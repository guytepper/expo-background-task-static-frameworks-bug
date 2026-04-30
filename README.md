# Repro: `expo-background-task@55.0.17` breaks `useFrameworks: "static"` builds

## Summary

[expo/expo PR #44646](https://github.com/expo/expo/pull/44646) (merged 2026-04-21, shipped in `expo-background-task@55.0.17`) added a single line to `ExpoBackgroundTask.podspec`:

```ruby
s.dependency 'ExpoTaskManager'
```

Under `expo-build-properties.ios.useFrameworks: "static"` (commonly required by `@react-native-firebase`), this transitive pod-level dependency `ExpoBackgroundTask → ExpoTaskManager` causes Clang to compile ExpoTaskManager's headers in a stricter cross-module context.

The existing `#import <ExpoModulesCore/EX*Interface.h>` lines in `EXTask.h` / `EXTaskExecutionRequest.h` / `EXTaskService.h` then trip `-Werror=non-modular-include-in-framework-module`, because `ExpoModulesCore` is force-disabled from `use_frameworks` by Expo autolinking (`[Expo] Disabling USE_FRAMEWORKS for modules ExpoModulesCore, ExpoModulesJSI, Expo, ReactAppDependencyProvider, expo-dev-menu, RNFBApp` during pod install).

Result: 5 errors, `xcodebuild` exits with code 65.

## Reproduce

```bash
bun install
bun run prebuild   # expo prebuild --platform ios --clean
bun run ios        # 5 non-modular include errors, code 65
```

Default `package.json` resolves `expo-background-task` to `55.0.17` (broken). To verify the fix, change `"expo-background-task"` to `"55.0.8"` (exact pin, before the regressing podspec change), reinstall, and rebuild.

## Version timeline (bisected)

| `expo-background-task` | Podspec change                                   | Build under `useFrameworks: "static"` |
| ---------------------- | ------------------------------------------------ | ------------------------------------- |
| `55.0.8`               | original (had `vendored_frameworks` conditional) | ✅                                    |
| `55.0.9 – 55.0.14`     | conditional dropped, plain `source_files`        | ✅                                    |
| `55.0.15 – 55.0.16`    | added `s.test_spec 'Tests'` block                | ✅                                    |
| **`55.0.17`**          | **added `s.dependency 'ExpoTaskManager'`**       | ❌                                    |

Single-line diff between 55.0.16 and 55.0.17 (`ExpoBackgroundTask.podspec`):

```diff
+ s.dependency 'ExpoTaskManager'
```

`expo-task-manager`'s version is irrelevant — the bug reproduces with `55.0.9` (no `test_spec`) or `55.0.15` (with `test_spec`) when `expo-background-task` is `55.0.17`.

## Mechanism

- Without `expo-background-task`, or with `55.0.8`–`55.0.16`: nothing else declares a CocoaPods dependency on `ExpoTaskManager`. ExpoTaskManager compiles standalone; non-modular `<ExpoModulesCore/...>` includes in its headers are not enforced cross-module → builds.
- With `55.0.17`: `ExpoBackgroundTask` declares `s.dependency 'ExpoTaskManager'`. CocoaPods generates the build with ExpoTaskManager's headers reachable through ExpoBackgroundTask's framework-module compilation context, which activates `-Wnon-modular-include-in-framework-module -Werror` on `<ExpoModulesCore/EXTaskManagerInterface.h>` etc.
- `ExpoModulesCore` is force-disabled from `use_frameworks` by Expo autolinking, so its interface headers can't be reached as a framework module → warning fires → fatal under `-Werror`.

The `s.test_spec` block added in 55.0.15 is _not_ the trigger by itself; `55.0.16` (with `test_spec`, without the dep declaration) builds fine.

## Workaround

Pin only `expo-background-task`:

```json
"expo-background-task": "55.0.8"
```

`expo-task-manager` does not need to be pinned. Adding `"ExpoTaskManager"` to `expo-build-properties.ios.forceStaticLinking` also works but is more invasive.

## Bisect path (how this was found)

1. **Theory 1: `expo-task-manager@55.0.10`'s `test_spec` is the trigger.** Wrong — pinning `expo-task-manager` to `55.0.9` happened to fix a real project, but a minimal repro with `expo-task-manager@55.0.15` + `useFrameworks: "static"` + `@react-native-firebase/app` builds fine. The `test_spec` promotes ExpoTaskManager's modulemap to `framework module`, but that alone doesn't fail.
2. **Theory 2: pulsar's framework-pod presence.** Wrong — adding `react-native-pulsar` to the minimal repro doesn't break it.
3. **Theory 3: bisect a real failing project's full dep set.** Removed deps in halves, isolating the trigger to `expo-background-task`.
4. **Theory 4 (final): `expo-background-task` 55.0.17 podspec.** Confirmed by:
   - Diff `55.0.16` vs `55.0.17`: single line `+ s.dependency 'ExpoTaskManager'`.
   - Repro folder builds fine with `expo-background-task@55.0.16`, fails with `55.0.17`.
   - Linked to merged PR #44646 (2026-04-21).

## Suggested fix

The PR author noted in #44646 that `import ExpoTaskManager` in Swift caused build conflicts with ExpoModulesCore types and used an Obj-C bridge to work around it. The same root cause now bites Obj-C `#import <ExpoTaskManager/EXTask.h>`. Options:

1. Revert the `s.dependency 'ExpoTaskManager'` declaration and find another way to expose `EXTaskService.shared` (e.g. via `unimodules-app-loader`-style indirection, or expose the helper without making ExpoTaskManager a CocoaPods dependency).
2. Or fix the upstream cause: expose ExpoModulesCore's interface headers (`EX*Interface.h`) modularly so cross-pod `#import <ExpoModulesCore/...>` always resolves modularly, even when ExpoModulesCore is force-disabled from `use_frameworks`.

## Environment

- expo: 55.0.18
- expo-background-task: `55.0.8`–`55.0.16` work, **`55.0.17` broken**
- expo-task-manager: any (irrelevant to this bug)
- expo-modules-core: 55.0.23
- react-native: 0.83.6
- @react-native-firebase/app: 23.8.x
- Xcode: 26.x
- CocoaPods: 1.16.2

## Related

- [expo/expo PR #44646](https://github.com/expo/expo/pull/44646) — added the `s.dependency 'ExpoTaskManager'` line that triggers this. Originally fixed runtime "Could not find TaskService module" lookup.
- [expo/expo issue #39607](https://github.com/expo/expo/issues/39607) — same family of root cause around `useFrameworks: "static"` + non-modular includes (this one was on `@react-native-firebase`).
- [expo/expo PR #44956](https://github.com/expo/expo/pull/44956) — added `test_spec` to `expo-task-manager` (55.0.10). Initially suspected as the cause; ruled out by bisect.
