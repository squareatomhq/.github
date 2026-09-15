# Square Atom GitHub Actions Standard

Moonpage is the behavioral reference for CI depth, Android releases, artifact hygiene, and repository activity reporting. Shared mechanics live in this repository; application repositories retain small caller workflows and stack-specific inputs.

## Workflow families

- `reusable-expo-ci.yml`: Expo/React Native validation
- `reusable-flutter-ci.yml`: Flutter analysis, tests, code generation, and debug build
- `reusable-android-ci.yml`: Native Gradle tests, lint, and debug build
- `reusable-discord-notification.yml`: Safe metadata-only Discord reporting
- `reusable-artifact-cleanup.yml`: Manual, dry-run-first artifact cleanup

## Adoption map

| Stack | Repositories |
| --- | --- |
| Expo | moonpage, habitly |
| Flutter | momentum, finbit, dearbaby |
| Native Android | adbase, comet, days-since, skills |
| Monorepo | jibon |

## Runner selection: hosted first, self-hosted fallback

Every workflow in every Square Atom repository prefers a GitHub-hosted runner and falls back to the organization self-hosted Fedora runner (`[self-hosted, Linux, X64]`) when hosted Actions are unavailable. Moonpage is the reference implementation.

GitHub has no native runner failover. `runs-on` is resolved once when a job is queued, and a job with no matching runner waits in the queue (up to 24 hours) rather than trying anywhere else. A `needs:`-based fallback cannot rescue that case either, because a job that never starts produces no result for a dependent job to react to. The choice has to be made *before* the real job is scheduled, so every workflow opens with a probe job:

```yaml
jobs:
  pick-runner:
    name: 🧭 Pick a runner
    runs-on: ubuntu-latest
    continue-on-error: true          # a blocked hosted runner must not redden the run
    timeout-minutes: 5
    outputs:
      runner: ${{ steps.pick.outputs.runner }}
    steps:
      - id: pick
        run: echo 'runner=["ubuntu-latest"]' >> "$GITHUB_OUTPUT"
```

The probe is a few seconds of work whose only job is to prove hosted Actions are usable. Exhausted minutes *fail* a job ("The job was not started because recent account payments have failed or your spending limit needs to be increased") rather than queueing it, so the output is simply never set and the empty value falls through to the self-hosted labels.

A repository-local job consumes it directly:

```yaml
  real-job:
    needs: pick-runner
    if: ${{ !cancelled() }}
    runs-on: ${{ fromJSON(needs.pick-runner.outputs.runner || '["self-hosted","Linux","X64"]') }}
```

A caller of a shared workflow passes it as the `runner` input instead:

```yaml
  quality:
    needs: pick-runner
    if: ${{ !cancelled() }}
    uses: squareatomhq/.github/.github/workflows/reusable-flutter-ci.yml@main
    with:
      runner: ${{ needs.pick-runner.outputs.runner || '["self-hosted","Linux","X64"]' }}
```

Every `reusable-*.yml` in this repository accepts that `runner` input, typed `string` and defaulting to `'["ubuntu-latest"]'`, and resolves it with `runs-on: ${{ fromJSON(inputs.runner) }}`. The default keeps a caller that has not adopted the probe on hosted runners, unchanged.

`if: ${{ !cancelled() }}` rather than `always()`: these workflows use `cancel-in-progress`, and `always()` would run the real job even for a superseded run.

The probe **cannot** be extracted into a shared workflow of its own, because `continue-on-error` is not supported on jobs that call reusable workflows — a failing probe would then mark the whole run red. It is inlined in every caller.

### Self-hosted hygiene

Any job that can land on Fedora checks its toolchain and clears the workspace before checkout. Both steps no-op on hosted runners:

```yaml
      - name: Check required tools
        if: runner.os == 'Linux'
        run: |
          missing=()
          for bin in git curl tar unzip xz; do
            command -v "$bin" >/dev/null || missing+=("$bin")
          done
          if ((${#missing[@]})); then
            echo "On Fedora: sudo dnf install -y ${missing[*]}"
            exit 1
          fi

      - name: Clean leftover workspace
        if: runner.environment == 'self-hosted'
        run: |
          mkdir -p "${{ github.workspace }}"
          find "${{ github.workspace }}" -mindepth 1 -maxdepth 1 -exec rm -rf {} +
```

A workflow must never *refuse* a hosted runner. Guards of the form "error out unless `runner.environment == 'self-hosted'`" predate this standard and are incompatible with it — the probe's whole purpose is to land the job on hosted when hosted is available.

### What it costs

Each workflow run adds one short hosted job. GitHub bills hosted jobs rounded up to the whole minute, so budget roughly one minute per workflow run while minutes are available. The `*-github-logs.yml` workflows fire on nearly every GitHub event and are the largest share of that; if minutes get tight, they are the first to pin back to self-hosted.

| Hosted | Self-hosted | Result |
| --- | --- | --- |
| available | either | runs hosted, no queue |
| blocked (minutes/billing) | online | probe fails, real job runs on Fedora |
| blocked | offline | queues — same as before, no worse |

### Discord must not starve CI

There is one org Fedora runner. The `*-github-logs.yml` workflows fire on almost every GitHub event. With `cancel-in-progress: false` and a unique run id per concurrency group, those jobs pile up `Queued` and real CI waits behind them. Every repository's activity reporter uses a single concurrency group with `cancel-in-progress: true`.

## Standard workflow set

Every application repository carries the same four files. Stack-specific gates and release builds stay repository-local.

| File | Purpose |
| --- | --- |
| `squareatom-ci.yml` | Thin caller into the stack's `reusable-*-ci.yml` |
| `<repo>-github-logs.yml` | Repository activity → Discord, metadata only |
| `cleanup-actions-artifacts.yml` | Manual, dry-run-first artifact cleanup |
| `android-release.yml` | Repository-local; signing differs per app |

## Safety

Callers define event triggers. A `pull_request_target` caller must only invoke metadata-only notification logic and must never check out or execute pull-request code. Production signing secrets remain repository- or environment-scoped. Release build workflows stay repository-local because each app has different signing integration and build scripts; they follow the same naming, permissions, validation, artifact, release, and Play-publishing standard without routing keystore credentials through a generic command runner. The shared Discord webhook may be an organization secret.

## Versioning

Test callers against the feature branch, merge this repository first, then create a protected `v1` tag. Production callers should reference `@v1`.
