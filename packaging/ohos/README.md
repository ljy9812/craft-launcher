# Craft Launcher for HarmonyOS (OpenHarmony) — ArkWeb shell

Runs the official `artcraft-launcher-web` build — the same static site published as
`artcraft-launcher-web-<version>.zip` on the GitHub release — inside a HarmonyOS app. An
ArkTS shell wraps an ArkWeb `Web` component and serves the site from
the HAP's own `rawfile`: no network origin, no server-side code.

This branch (`ohos-webshell`) is the integrated form: the ArkWeb shell
packaging under `packaging/ohos/` plus the wasm-side CJK font change
merged from the `cjk-web-face` branch (which remains the clean
single-purpose form for an upstream PR). Build either variant from this
one checkout.

## How the shell works

| Concern | Mechanism |
|---|---|
| Serving | `onInterceptRequest` maps the virtual origin `https://local.craft-launcher/` to `entry/src/main/resources/rawfile/web/` |
| Opening files | `onShowFileSelector` → system `DocumentViewPicker`; the picked URI is handed back to the page's `<input type=file>` |
| Saving | `a[download]` clicks — attached (document listener) and programmatic/detached (`HTMLAnchorElement.prototype.click` override) — are intercepted by a hook installed in `onPageEnd`; the blob is fetched, base64-chunked over a `MessageChannel` port to ArkTS, and written through `DocumentSavePicker` (with a filename field) |
| GitHub data | `api.github.com` and `raw.githubusercontent.com` requests are relayed through ArkTS: the virtual origin only serves the local site, and the wasm's fetch of GitHub JSON would otherwise have no route |
| Platform detection | `navigator.userAgent` is pinned to a Windows desktop string at serve time: the launcher derives its download offering from the UA, and an OpenHarmony UA matches none of its known platforms — every card would show "Not available" |
| External links | dual path: `onLoadIntercept` for main-frame http(s) navigations + `multiWindowAccess(true)`/`onWindowNew` for `window.open` popups — both route to the system browser |

## Build (Windows host, DevEco Studio toolchain)

Prerequisites: DevEco Studio (for `hvigorw`, `ohpm`, and the HarmonyOS SDK),
`hdc` on PATH, and `gh` (GitHub CLI) for the release download.

```powershell
cd packaging\ohos
.\scripts\fetch-web.ps1     # deploy the web build into rawfile (latest release; zip name artcraft-launcher-web-*)
.\scripts\build-hap.ps1     # build the Release HAP and install to the connected device
```

### Signing

The committed tree is deliberately signing-free (`"signingConfigs": []` in
`build-profile.json5`). To build a signed HAP, put your own signing fragment
at `packaging\ohos\local\signing.json` (gitignored) — one `signingConfigs`
entry as DevEco Studio writes it. `build-hap.ps1` injects it for the build
and restores the pristine file afterwards. Adjust `bundleName` in
`AppScope/app.json5` (currently the `com.example.craft-launcher.web` placeholder)
to match your signing material.

## Verified status and known limitations

Verified on a HarmonyOS laptop (HAD-W24, API 23): install & launch, catalog
round-trips through the GitHub relay, and both external-link routes on
device — the ⋯ menu's "GitHub repository" and the primary "Download for
Windows" button open the system browser (the release asset URL matching
the launcher's catalog entry byte-for-byte).

Known limitations:

- Anonymous GitHub API quota is 60 requests/hour (fixed-hour reset, not
  rolling); the launcher re-queries 14 repos every 5 minutes, so a
  long-running session periodically shows Offline until the window resets.
  A token or shell-side cache would fix it; neither is in scope here.
- The UA pin makes the launcher offer its Windows installers — there are
  no OpenHarmony assets upstream, so "Not available" would otherwise be
  the correct-but-useless state for a real OHOS visitor. Treat the pinned
  platform as a test vehicle.
- The zip glob quirk: this app's release assets are named
  `artcraft-launcher-web-<version>.zip` (not `craft-launcher-web-*`);
  `fetch-web.ps1` already accounts for it.
