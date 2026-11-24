# T3 Code Mobile

> [!NOTE]
> To install the released app from the App Store or Google Play, see [Mobile app](../../docs/user/install.md#mobile-app). This guide covers building it from source.

## Quickstart

> [!NOTE]
> Uses native modules so using Expo Go is not supported. You need to use the Expo Dev Client.

This app has three variants:

- `development`: Expo dev client, installable side-by-side as `T3 Code Dev`
- `preview`: persistent internal preview build, installable side-by-side as `T3 Code Preview`
- `production`: store/release build as `T3 Code`

Run commands from `apps/mobile`.

T3 Connect is optional and disabled in a fresh clone. Public configuration belongs in the
repository-root `.env` or `.env.local`, not an `apps/mobile/.env` file. See
[`../../.env.example`](../../.env.example).

## Development

For simulator, emulator, or Android phone development, boot or connect a device, then install the
native client that matches this checkout before starting Metro:

```bash
vp run mobile:install ios [simulator-udid]
# Or: vp run mobile:install android [emulator-or-phone-serial]
vp run dev:client
```

The device argument can be omitted when exactly one device is booted or connected. The helper
compares this checkout's Expo fingerprint and the installed binary with its last install record,
and reuses a matching client. Otherwise it installs the build for that fingerprint from
`~/.cache/t3code/native-clients/artifacts`, then the shared task cache, and builds only when
neither has it. A build runs in the `Mobile Dev Client` workflow (Blacksmith runners with warm
Gradle and ccache state, and a macOS runner for iOS) when `gh` may dispatch workflows on
`pingdotgg/t3code` and HEAD is pushed; the helper waits for the client in the shared cache.
Otherwise, or when HEAD lacks your native edits or `T3_NATIVE_CLIENT_BUILD=local` is set, it builds
on this machine. Main's client is built there on every native change, so most checkouts hit. One iOS simulator build serves every iPhone and iPad simulator, and one APK per
ABI (x86_64 or arm64-v8a) serves every emulator and phone of that ABI. Build output goes to the
log file it prints (`T3_NATIVE_CLIENT_VERBOSE=1` streams it instead). Builds are shared back when
your task-cache login (`bash infra/task-cache/install.sh`) can write. `T3_NATIVE_CLIENT_CACHE=read`
skips the upload and `off` skips the shared cache. Physical iPhones still use `vp run ios:dev`.

`--session <name>` installs into an agent-device session opened elsewhere, such as a device pool
or a cloud simulator, instead of a local device: `vp run mobile:install android --session pool-1
--metro http://<this-machine>:8081`. It uses only the cached build (x86_64 by default; `--abi`
for ARM) and fails with the next step when no machine has built this fingerprint yet.

For a read-only decision, run `node ../../scripts/mobile-native-client.ts check <ios|android>
[device]`: exit 0 means compatible, 2 means an install is needed, and 1 means an operational error.
An externally installed client is unverified until the helper installs it once.

Start Metro for an already verified dev client:

```bash
vp run dev:client
```

Metro keeps its transform cache between ordinary starts. If the cache itself is causing stale or
invalid output, clear it for one development-client start:

```bash
vp run dev:client:reset
```

Run that reset once after installing or changing the Uniwind dependency patch. Cached transforms
can otherwise reference its previous pnpm package path. Ordinary Metro starts still keep the cache.

Component edits use Fast Refresh. See [mobile development lifecycle](../../docs/internals/13_mobile-development.md)
before changing runtime ownership or refresh behavior.

Build and run the local iOS dev client:

```bash
vp run ios:dev
```

After changing a native dependency patch, rerun CocoaPods before rebuilding an existing iOS
project. pnpm gives each patch hash a new package path; Pods can otherwise keep compiling the
previous directory.

If your Xcode account only has a Personal Team, use a bundle identifier you control and opt into the
reduced-capability local build. Personal Team builds omit the widget and share extensions, push
entitlement, and native Sign in with Apple entitlement; builds without this opt-in are unchanged.

```bash
T3CODE_IOS_PERSONAL_TEAM=1 \
T3CODE_IOS_PERSONAL_TEAM_BUNDLE_ID=com.example.t3code.dev \
vp run ios:dev
```

Build and install a self-contained Release app that does not need Metro:

```bash
vp run ios:release
```

The Personal Team equivalent also needs a unique bundle identifier:

```bash
T3CODE_IOS_PERSONAL_TEAM=1 \
T3CODE_IOS_PERSONAL_TEAM_BUNDLE_ID=com.example.t3code \
vp run ios:release
```

Build and run the local iOS preview app:

```bash
vp run ios:preview
```

Inspect the resolved Expo config for a variant:

```bash
vp run config:dev
vp run config:preview
```

Run static checks for mobile native code:

```bash
node ../../scripts/mobile-native-static-check.ts
```

The native lint task runs SwiftLint for Swift plus ktlint and detekt for Kotlin. Missing native tools are reported as warnings and skipped locally. CI installs the default toolset from `apps/mobile/Brewfile` before running the native checks.

## EAS Builds

Preview and production variants use Expo fingerprinting so OTA updates only reach binaries with matching native dependencies, config plugins, and patches. CI uses the `preview:dev` profile to reuse a compatible native build when possible.

The development variant uses `appVersion` to avoid recalculating the native fingerprint for each Metro launch manifest. `MOBILE_VERSION_POLICY` can override either default. If you distribute a custom Release build with the development identity and publish OTA updates to it, set `MOBILE_VERSION_POLICY=fingerprint` for both its build and updates. Changing the runtime policy requires a native rebuild for OTA matching; an existing dev client can still load local Metro bundles.

For preview or production EAS environments, set `T3CODE_CLERK_PUBLISHABLE_KEY`,
`T3CODE_CLERK_JWT_TEMPLATE`, and `T3CODE_RELAY_URL`
as EAS environment variables. Expo config maps the canonical values into the mobile build.

Create a PR preview dev-client build manually:

```bash
vp run eas:ios:preview:dev
```

Create a cloud dev-client build:

```bash
vp run eas:ios:dev
```

Create a persistent preview build:

```bash
vp run eas:ios:preview
```

Android equivalents:

```bash
vp run eas:android:dev
vp run eas:android:preview:dev
vp run eas:android:preview
```

## iOS 17 fork notes

This fork (`hikki-fan/t3code`, branch `ios17-compat`) lowers the iOS minimum from 18.0 to 17.0 so
the app can be built and installed on iOS 17 devices. 本分支把 iOS 最低版本从 18.0 降到 17.0，
以便在 iOS 17 设备上自行编译安装。

What changed / 改动:

- `app.config.ts`: `expo-build-properties` `ios.deploymentTarget` `"18.0"` → `"17.0"`.
- Local native module podspecs (`t3-native-controls`, `t3-web-socket`, `t3-preview-tunnel`,
  `t3-voice-transcription`) now declare iOS 17.0. Their Swift code already gates newer APIs
  (iOS 26 / 27.1) with `#available`; nothing in them needs iOS 18.
- `packages/contracts/src/relay.ts`: relay device schemas accept `iosMajorVersion >= 17`.
- Widget and share extensions keep the Expo plugin default (16.4).

Build on a Mac with a recent Xcode (Xcode 26 or newer; the native code uses the iOS 26 SDK) and a
free Apple ID / Personal Team. 在装有较新 Xcode（26 及以上）的 Mac 上，用个人免费 Apple ID 编译：

```bash
cd apps/mobile
T3CODE_IOS_PERSONAL_TEAM=1 \
T3CODE_IOS_PERSONAL_TEAM_BUNDLE_ID=com.hikkifan.t3code \
vp run ios:release
```

Caveats / 注意事项:

- Do not use the official bundle id `com.t3tools.t3code`; use your own reverse-DNS id.
  不要使用官方 bundle id `com.t3tools.t3code`。
- Free Apple ID signing certificates expire after 7 days; rebuild/reinstall weekly.
  免费 Apple ID 签名 7 天后过期，需要每周重新安装。
- Personal Team builds omit the widget and share extensions, push notifications, and native Sign in
  with Apple. 个人团队构建不包含小组件、分享扩展、推送和 Sign in with Apple。
- The official T3 Connect relay is not changed by this fork and may still reject iOS 17 device
  registration (push / Live Activities). Pairing over LAN or Tailscale works without the relay.
  官方 T3 Connect 中继服务器可能仍拒绝 iOS 17 设备注册；局域网或 Tailscale 配对不受影响。
