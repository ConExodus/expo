# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

This is the main Expo monorepo (a fork of `expo/expo`). It contains the Expo SDK (native mobile libraries), the Expo CLI, Expo Router, documentation, and the Expo Go app. It is a Yarn workspace with dozens of packages under `packages/` and `packages/@expo/`, plus several apps under `apps/`.

Development does **not** support Windows natively — WSL is required on Windows.

## Before Doing Anything

- **Dependencies are not installed in a fresh clone.** Run `yarn install` from the root first. The `postinstall` step also builds `@expo/cli`, applies `patch-package` patches, and validates workspace dependencies, so it takes several minutes.
- **Native builds need a Mac (iOS) or Android Studio (Android).** In a Linux-only or cloud environment, `yarn ios`, `yarn android`, and the `test:ios` / `test:android` scripts cannot run. Only JS/TypeScript build, lint, and unit tests work there.
- **`react-native-lab/react-native` is a git submodule and is usually empty.** It is only needed to build Expo Go. Do not treat the empty folder as a problem.
- **Always set `EXPO_NONINTERACTIVE=1` (or `CI=1`) when running `yarn build`, `yarn test`, or `yarn typecheck`.** Without it, `expo-module-scripts` adds `--watch` whenever stdout is a terminal and the command never exits. See `packages/expo-module-scripts/utils/commandUtils.js`.

## Key Areas

- `packages/expo` — The main `expo` npm package that every Expo app installs. Re-exports and wires up the rest of the SDK.
- `packages/expo-*` — Individual SDK modules (e.g. `expo-camera`, `expo-constants`). Each is its own npm package with its own native code.
- `packages/expo-router` — File-based navigation. End-to-end tests for it live in `apps/router-e2e`.
- `packages/@expo/` — Internal tooling and supporting packages (`@expo/cli`, `@expo/config`, `@expo/config-plugins`, `@expo/metro-config`, etc.).
- `packages/expo-modules-core` — The foundation every native module is built on. Defines the Swift/Kotlin module system.
- `packages/expo-module-scripts` — Shared build, lint, and test tooling used by every SDK package via the `expo-module` CLI.
- `apps/bare-expo` — The primary app for testing SDK changes locally on a device/simulator. Links all packages from `packages/` and runs `test-suite`.
- `apps/test-suite` — Automated E2E tests (Jasmine-based) that run inside `bare-expo`.
- `apps/native-component-list` — Manual smoke-test app for UI/interactive features.
- `apps/expo-go` — The Expo Go app (difficult to set up; requires API tokens; use `bare-expo` for most work).
- `tools/` — Internal build and release tooling, exposed as the `et` / `expotools` command.
- `docs/` — Source for docs.expo.dev (Next.js app).
- `templates/` — Project templates used by `npx create-expo-app`.
- `guides/` — Internal style guides and process docs. `guides/Expo JavaScript Style Guide.md` and `guides/contributing/Updating Changelogs.md` are the ones that matter most for day-to-day changes.

## Commands

All commands below should be run from the **package directory** unless noted otherwise.

### Working on an SDK Package

```sh
# From a package directory (e.g. packages/expo-constants)
EXPO_NONINTERACTIVE=1 yarn build          # Compile TypeScript into build/ (one-shot)
EXPO_NONINTERACTIVE=1 yarn test           # Run unit tests once
EXPO_NONINTERACTIVE=1 yarn test src/__tests__/Constants-test.ts   # Run a single test file
yarn lint                                  # ESLint + Prettier
yarn lint --fix                            # Auto-fix lint issues
yarn clean                                 # Delete the build/ directory
```

Type-checking: most SDK packages do **not** define a `typecheck` script. Use the underlying tool directly, which works in any package that depends on `expo-module-scripts`:

```sh
EXPO_NONINTERACTIVE=1 yarn expo-module typecheck
```

Some packages have extra build targets (e.g. a config plugin in `plugin/` or a CLI in `cli/`). Build/test them with `yarn build plugin`, `yarn test plugin`, `yarn build cli`, etc. The `expo-module` program handles the sub-directory automatically.

Run `yarn expo-module --help` in any package to see all available sub-commands.

### Testing in bare-expo (macOS / Android Studio required)

```sh
# From apps/bare-expo
yarn ios            # Build and run on iOS simulator
yarn android        # Build and run on Android emulator

yarn test:ios       # Run automated test suite on iOS
yarn test:android   # Run automated test suite on Android

yarn edit:ios       # Open Xcode workspace
yarn edit:android   # Open Android Studio
```

After making **native** changes (Swift, Kotlin, Objective-C), you must rebuild the app. JS-only changes can be hot-reloaded.

### Internal tooling (`et`)

`et` is the repo's internal command-line helper (changelog entries, docs generation, release tasks). It lives in `bin/et` and is added to `PATH` by `direnv` via `.envrc`. If `direnv` is not active, run it as `./bin/et` from the repository root.

```sh
./bin/et --help
./bin/et add-changelog                              # Interactive helper to add a changelog entry
./bin/et generate-docs-api-data -p <package-name>   # Regenerate API docs for a package
```

Running `et` the first time builds `tools/` from source. It also expects Ruby 3.3+ and `bundle install` to have been run at the root for some commands.

### Docs

```sh
# From docs/
yarn            # Install dependencies (pinned to Node 22.13.1 via volta in docs/package.json)
yarn dev        # Start dev server on http://localhost:3002
```

### Root-level

```sh
yarn lint       # Lint the entire workspace (slow — prefer running inside a single package)
```

Running `yarn tsc` from the root is intentionally blocked; run it from an individual package instead.

## How Native Modules Work

Every SDK package follows the **Expo Modules API** pattern:

**TypeScript side** — `src/` contains the JS/TS entry point. It imports native functions exposed by the module. Platform-specific implementations use file suffixes: `Foo.ts` (shared), `Foo.native.ts`, `Foo.web.ts`, `Foo.ios.ts`, `Foo.android.ts`.

**iOS (Swift)** — A class extending `Module` (from `ExpoModulesCore`) implements a `definition()` method using a DSL:
```swift
public class MyModule: Module {
  public func definition() -> ModuleDefinition {
    Name("MyModule")
    AsyncFunction("doThingAsync") { (arg: String) -> String in ... }
  }
}
```

**Android (Kotlin)** — Same pattern with `Module` and `ModuleDefinition`:
```kotlin
class MyModule : Module() {
  override fun definition() = ModuleDefinition {
    Name("MyModule")
    AsyncFunction<String>("doThingAsync") { arg: String -> ... }
  }
}
```

**`expo-module.config.json`** — Declares which platforms the module supports. The autolinking system reads this to wire the module into app builds.

**`build/` is committed to Git.** Each package's compiled output lives in `build/` and must be committed so that other contributors don't need to rebuild everything on every checkout. After changing anything under `src/`, run `EXPO_NONINTERACTIVE=1 yarn build` and commit the resulting `build/` changes too.

## Unit Testing

Tests live in `src/__tests__/` and are named `<Name>-test.ts` (hyphen, not dot). Platform-specific tests add a platform suffix before the extension:

- `Constants-test.ts` — runs on all platforms
- `Localization-test.native.ts` — iOS + Android only
- `Localization-test.web.ts` — web only
- `Contacts-test.ios.ts` — iOS only

The Jest preset runs each test under iOS, Android, and web projects. If a feature doesn't support a platform, use the suffix to exclude it rather than skipping inside the test.

When adding new bridged native functions, add mocks for them in `packages/jest-expo/src/preset/moduleMocks/expoModules.js`. See `guides/Generating Jest Mocks.md` for the automated generation process.

## E2E Testing

E2E tests live in `apps/test-suite/tests/`. They use a minimal Jasmine-like API (no snapshot testing). When adding a new test file, register it in `apps/test-suite/TestUtils.js`. If it should run in automated CI from `bare-expo`, also add it to `apps/bare-expo/e2e/TestSuite-test.native.js`.

## Code Style

- ESLint config extends `universe/native`, `universe/node`, `universe/web`. Run `yarn lint --fix` before submitting.
- Prettier settings (from `.prettierrc`): single quotes, 100-char print width, 2-space indent, trailing commas (ES5).
- Async functions end in `Async` — e.g. `getWebViewUserAgentAsync`, `runAsync`. This applies to JS/TS and to native function names exposed to JS.
- Commit message format: `[platform][api] Description` — e.g. `[ios][video] Fixed black screen on older devices`. Platform tags seen in history: `[ios]`, `[android]`, `[web]`, `[cli]`, `[docs]`, `[autolinking]`, `[router]`.

## Changelogs

Every package has its own `CHANGELOG.md`. Add an entry under `## Unpublished` in the correct category whenever you change a package:

- **🛠 Breaking changes** — implies major version bump
- **🎉 New features** — implies minor bump
- **🐛 Bug fixes**
- **⚠️ Notices** — deprecations or behavior edge cases
- **💡 Others** — internal changes

Entries must be plain text with a PR link and author link. No nested markdown. If the PR number isn't known yet, write the message only — the repo's GitHub bot suggests the PR/author links as review comments.

Changes that don't belong to any single package go in the root `CHANGELOG.md`.

## Pull Requests

The PR template (`.github/PULL_REQUEST_TEMPLATE`) has four sections: **Why**, **How**, **Test Plan**, **Checklist**. Fill all four. The checklist asks whether a changelog entry was added and `build/` was rebuilt, whether the change works with `npx expo prebuild` / EAS Build, and whether docs changes follow `guides/Expo Documentation Writing Style Guide.md`.

CI runs different workflow files depending on which directories changed (see `.github/workflows/`). Keeping docs changes in a separate PR from code changes makes CI finish faster.

## Docs API Reference

Package API docs are partially generated from TypeScript sources. After changing a public API, regenerate docs with:
```sh
./bin/et generate-docs-api-data -p <package-name>
```
Any change to the current versioned docs under `docs/pages/versions/vXX.0.0/` should also be copied into `docs/pages/versions/unversioned/`.
