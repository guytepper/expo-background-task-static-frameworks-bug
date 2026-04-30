# Repro: `expo-background-task@55.0.17` breaks `useFrameworks: "static"` builds

## Summary

[expo/expo PR #44646](https://github.com/expo/expo/pull/44646) (shipped in `expo-background-task@55.0.17`) added a single line to `ExpoBackgroundTask.podspec`:

```ruby
s.dependency 'ExpoTaskManager'
```

Under `expo-build-properties.ios.useFrameworks: "static"` (commonly required by `@react-native-firebase`), this new transitive pod-level dependency `ExpoBackgroundTask → ExpoTaskManager` causes Clang to compile ExpoTaskManager's headers in a stricter cross-module context.

The existing `#import <ExpoModulesCore/EX*Interface.h>` lines in `EXTask.h` / `EXTaskExecutionRequest.h` / `EXTaskService.h` then trip `-Werror=non-modular-include-in-framework-module`, because `ExpoModulesCore` is force-disabled from `use_frameworks` by Expo autolinking.

Result: 5 errors, `xcodebuild` exits with code 65.

## Reproduce

```bash
bun install
bun run prebuild   # expo prebuild --platform ios --clean
bun run ios        # 5 non-modular include errors, code 65
```

The `package.json` pins `expo-background-task` to `55.0.17` (broken). To verify the workaround, change it to `"55.0.8"`, delete `node_modules` and `bun.lock`, then reinstall and rebuild.

## Version timeline

| `expo-background-task` | Podspec change                                   | Build under `useFrameworks: "static"` |
| ---------------------- | ------------------------------------------------ | ------------------------------------- |
| `55.0.8`               | original (had `vendored_frameworks` conditional) | ✅                                    |
| `55.0.9 – 55.0.14`     | conditional dropped, plain `source_files`        | ✅                                    |
| `55.0.15 – 55.0.16`    | added `s.test_spec 'Tests'` block                | ✅                                    |
| **`55.0.17`**          | **added `s.dependency 'ExpoTaskManager'`**       | ❌                                    |

Single-line diff between 55.0.16 and 55.0.17 in `ExpoBackgroundTask.podspec`:

```diff
+ s.dependency 'ExpoTaskManager'
```

`expo-task-manager`'s version is irrelevant — the bug reproduces with any version when `expo-background-task` is `55.0.17`. The `s.test_spec` block added in 55.0.15 is _not_ the trigger by itself.

## Workaround

Pin only `expo-background-task`:

```json
"expo-background-task": "55.0.8"
```

(Adding `"ExpoTaskManager"` to `expo-build-properties.ios.forceStaticLinking` also works but is more invasive.)

## Environment

- expo: 55.0.18
- expo-background-task: `55.0.8`–`55.0.16` work, **`55.0.17` broken**
- expo-modules-core: 55.0.23
- react-native: 0.83.6
- @react-native-firebase/app: 23.8.x
- Xcode: 26.x
- CocoaPods: 1.16.2

## Related

- [expo/expo PR #44646](https://github.com/expo/expo/pull/44646) — introduced the regression (originally fixed a runtime "Could not find TaskService module" lookup).
- [expo/expo issue #39607](https://github.com/expo/expo/issues/39607) — same family of root cause around `useFrameworks: "static"` + non-modular includes, on `@react-native-firebase`.
