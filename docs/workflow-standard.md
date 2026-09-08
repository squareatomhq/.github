# Square Atom GitHub Actions Standard

Moonpage is the behavioral reference for CI depth, Android releases, artifact hygiene, and repository activity reporting. Shared mechanics live in this repository; application repositories retain small caller workflows and stack-specific inputs.

## Workflow families

- `reusable-expo-ci.yml`: Expo/React Native validation
- `reusable-flutter-ci.yml`: Flutter analysis, tests, code generation, and debug build
- `reusable-android-ci.yml`: Native Gradle tests, lint, and debug build
- `reusable-android-release.yml`: Common Android artifact, GitHub Release, and optional Play publishing
- `reusable-discord-notification.yml`: Safe metadata-only Discord reporting
- `reusable-artifact-cleanup.yml`: Manual, dry-run-first artifact cleanup

## Adoption map

| Stack | Repositories |
| --- | --- |
| Expo | moonpage, habitly |
| Flutter | momentum, finbit, dearbaby |
| Native Android | adbase, comet, days-since, skills |
| Monorepo | jibon |

## Safety

Callers define event triggers. A `pull_request_target` caller must only invoke metadata-only notification logic and must never check out or execute pull-request code. Production signing secrets remain repository- or environment-scoped. The shared Discord webhook may be an organization secret.

## Versioning

Test callers against the feature branch, merge this repository first, then create a protected `v1` tag. Production callers should reference `@v1`.
