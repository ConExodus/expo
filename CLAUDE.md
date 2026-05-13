# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

This is the main Expo monorepo. It contains the Expo SDK (native mobile libraries), the Expo CLI, Expo Router, documentation, and the Expo Go app. It is a Yarn workspace with dozens of packages under `packages/` and `packages/@expo/`, plus several apps under `apps/`.

Development does **not** support Windows natively — WSL is required on Windows.

## Key Areas

- `packages/` — All Expo SDK modules (e.g. `expo-camera`, `expo-constants`). Each is its own npm package with its own native code.
- `packages/@expo/` — Internal tooling and supporting packages (e.g. `@expo/cli`, `@expo/config`, `@expo/config-plugins`).
- `packages/expo-modules-core` — The foundation every native module is built on. Defines the Swift/Kotlin module system.
- `packages/expo-module-scripts` — Shared build, lint, and test tooling used by every SDK package via `expo-module` CLI.
- `apps/bare-expo` — The primary app for testing SDK changes locally. Links all packages from `packages/` and runs `test-suite`.
- `apps/test-suite` — Automated E2E tests (Jasmine-based) that run inside `bare-expo` on device/simulator.
- `apps/native-component-list` — Manual smoke-test app for UI/interactive features.
- `apps/expo-go` — The Expo Go app (difficult to set up; requires API tokens; use `bare-expo` for most work).
- `react-native-lab/react-native` — A fork of React Native used specifically to build Expo Go.
- `tools/` — Internal build and release tooling (`et` / `expotools`).
- `docs/` — Source for docs.expo.dev (Next.js app).
- `templates/` — Project templates used by `npx create-expo-app`.

## Commands

All commands below should be run from the **package directory** unless noted otherwise.

### Working on an SDK Package

```sh
# From the root — install everything
yarn install

# From a package directory (e.g. packages/expo-constants)
yarn build          # Compile TypeScript (starts file watcher)
yarn lint           # Lint with ESLint + Prettier
yarn lint --fix     # Auto-fix lint issues
yarn test           # Run unit tests (Jest)
yarn typecheck      # Type-check without emitting
yarn clean          # Delete the build/ directory
```

The `expo-module` binary (from `expo-module-scripts`) powers all of those scripts. Run `yarn expo-module --help` in any package for details.

### Testing in bare-expo (iOS / Android)

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

### Docs

```sh
# From docs/
yarn            # Install dependencies (requires Node per volta in docs/package.json)
yarn dev        # Start dev server on http://localhost:3002
```

### Root-level

```sh
yarn lint       # Lint the entire workspace
```

## How Native Modules Work

Every SDK package follows the **Expo Modules API** pattern:

**TypeScript side** — `src/` contains the JS/TS entry point. It imports native functions exposed by the module.

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

**`build/` is committed to Git.** Each package's compiled output lives in `build/` and must be committed so that other contributors don't need to rebuild everything on every checkout.

## Unit Testing

Tests live in `src/__tests__/` with filenames like `MyModule-test.ts`. Platform-specific tests use extensions like `.test.ios.ts`, `.test.android.ts`, `.test.web.ts`.

When adding new bridged native functions, add mocks for them in `packages/jest-expo/src/preset/moduleMocks/expoModules.js`. See `guides/Generating Jest Mocks.md` for the automated generation process.

## E2E Testing

E2E tests live in `apps/test-suite/tests/`. They use a minimal Jasmine-like API (no snapshot testing). When adding a new test file, register it in `apps/test-suite/TestUtils.js`. If it should run in automated CI from `bare-expo`, also add it to `apps/bare-expo/e2e/TestSuite-test.native.js`.

## Code Style

- ESLint config extends `universe/native`, `universe/node`, `universe/web`. Run `yarn lint --fix` before submitting.
- Prettier settings (from `.prettierrc`): single quotes, 100-char print width, 2-space indent, trailing commas (ES5).
- Commit message format: `[platform][api] Description` — e.g. `[ios][video] Fixed black screen on older devices`.

## Changelogs

Every package has its own `CHANGELOG.md`. Add an entry under `## Unpublished` in the correct category whenever you change a package:

- **🛠 Breaking changes** — implies major version bump
- **🎉 New features** — implies minor bump
- **🐛 Bug fixes**
- **⚠️ Notices** — deprecations or behavior edge cases
- **💡 Others** — internal changes

Entries must be plain text with a PR link and author link. No nested markdown.

## Docs API Reference

Package API docs are partially generated from TypeScript sources. After changing a public API, regenerate docs with:
```sh
et generate-docs-api-data -p <package-name>
```
Any change to the current versioned docs should also be copied into `docs/pages/versions/unversioned/`.
