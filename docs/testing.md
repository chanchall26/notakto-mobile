# Testing and CI/CD

## Running tests

Tests use [Jest](https://jestjs.io/) with the `jest-expo` preset and
[React Native Testing Library](https://callstack.github.io/react-native-testing-library/).

```bash
npm install
npm test            # run every test once
npm test -- --watch # re-run on file changes
npm run test:ci     # what CI runs (no coverage, single worker)
```

Before opening a PR, run the same checks CI runs:

```bash
npx tsc --noEmit
npm run check
npm run test:ci
```

## Where test files live

- Put a test next to the file it covers, named `<file>.test.ts` or `<file>.test.tsx`
  (for example `src/stores/globalModalStore.test.ts`).
- A `__tests__/` folder is also picked up if a feature needs several test files.
- Import project code with the `@/` alias, like the rest of the app. Biome rejects relative imports.

## No network in tests

`jest.setup.ts` runs before every test file and mocks everything that touches the network or
a native module:

- `@/src/lib/firebase` is replaced with a fake that exports `auth`, `signInWithGoogle`,
  `signOutUser` and `onAuthStateChangedListener` as `jest.fn()` mocks.
- `firebase/app`, `firebase/auth` and `@react-native-google-signin/google-signin` are mocked
  too, in case code imports them directly.

To change what a mock returns inside one test, import the module and use the mock helpers:

```ts
import { signInWithGoogle } from "@/src/lib/firebase";

(signInWithGoogle as jest.Mock).mockRejectedValueOnce(new Error("cancelled"));
```

For modules that only one test needs (for example `expo-router` or `react-native-toast-message`),
call `jest.mock(...)` at the top of that test file.

Tests do not need any real Firebase keys or `.env` file.

## Minimum testing standard for new features

A PR that adds or changes behaviour should include tests for it:

1. Zustand stores: test the initial state and every action. Reset the store in `beforeEach`
   with `useStore.setState(...)`.
2. Hooks and utility functions: test each branch (success, error, loading, empty).
3. Components: at least one render test for the visible text and one for each user interaction
   (`fireEvent.press`, and so on).
4. Anything that talks to Firebase or another service: mock it, never call the real one.
5. A bug fix should come with a test that fails without the fix.

Tests must pass in CI before a PR can be merged.

## CI

`.github/workflows/ci.yaml` runs on every pull request and on pushes to `main`:

1. `npm ci`
2. `tsc --noEmit`
3. Biome check
4. Jest tests
5. Expo Doctor
6. `expo export` (verifies the app bundles)

Any failing step fails the check. To block merging on failures, enable branch protection on
`main` and mark the `ci` job as a required status check.

## EAS builds

`.github/workflows/eas-build.yml` builds the app on [EAS Build](https://docs.expo.dev/build/introduction/).
It runs only when started by hand (Actions tab, "EAS Build", "Run workflow") and asks for a
platform (`android`, `ios`, `all`) and a profile (`development`, `preview`, `production`).
It does not run on pull requests, so it does not use EAS quota by accident.

Build profiles are defined in `eas.json`:

| Profile       | Use                                             |
| ------------- | ----------------------------------------------- |
| `development` | Development client, internal distribution       |
| `preview`     | Internal test build (Android APK)               |
| `production`  | Store build, version number auto-incremented    |

### Required secrets (SECRET REQUIRED HERE)

Nothing is committed with real values. The maintainers need to add:

| Where                                   | Name                                   | Used for                                    |
| --------------------------------------- | -------------------------------------- | ------------------------------------------- |
| GitHub Actions secret                   | `EXPO_TOKEN`                           | Lets the workflow log in to Expo/EAS        |
| `eas.json` env or EAS env variables     | `EXPO_PUBLIC_FIREBASE_API_KEY`         | Firebase config read in `src/lib/firebase.ts` |
| same                                    | `EXPO_PUBLIC_FIREBASE_APP_ID`          | same                                        |
| same                                    | `EXPO_PUBLIC_FIREBASE_AUTH_DOMAIN`     | same                                        |
| same                                    | `EXPO_PUBLIC_FIREBASE_MEASUREMENT_ID`  | same                                        |
| same                                    | `EXPO_PUBLIC_FIREBASE_MESSAGING_SENDER_ID` | same                                    |
| same                                    | `EXPO_PUBLIC_FIREBASE_PROJECT_ID`      | same                                        |
| same                                    | `EXPO_PUBLIC_FIREBASE_STORAGE_BUCKET`  | same                                        |
| same                                    | `EXPO_PUBLIC_FIREBASE_WEB_CLIENT_ID`   | Google Sign-In                              |

The values in `eas.json` are the placeholder text `SECRET REQUIRED HERE`. Replace them with real
values, or better, create them with `eas env:create` so they stay out of the repository.

One-time setup for the maintainers:

1. Create an access token at https://expo.dev/settings/access-tokens and save it as the
   `EXPO_TOKEN` repository secret.
2. Run `eas init` once to link the project to your Expo account (this adds the project ID to
   `app.json`).
3. Set `ios.bundleIdentifier` and `android.package` in `app.json` (needed for store builds).
4. Fill in the Firebase values as described above.

### Release flow

1. Open a PR. CI validates it (types, lint, tests, bundle).
2. When a device build is needed, run "EAS Build" with profile `preview`.
3. After merging to `main`, run "EAS Build" with profile `production`.
