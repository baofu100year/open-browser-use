# Open Browser Use Installation

Read this reference when the user asks to install, verify, repair, or explain Open Browser Use setup.

## Components

- Chrome extension: the browser-side controller. Installing or enabling it may require the user to approve Chrome prompts.
- Native host and CLI: the local `open-browser-use` binary, also exposed as `obu` when installed from supported packages.
- SDKs: JavaScript, Python, and Go clients that connect to the active native host socket.

## Install The CLI

Use one of the supported package routes:

```sh
npm install -g open-browser-use
```

```sh
brew install iFurySt/open-browser-use/open-browser-use
```

Verify:

```sh
open-browser-use version
obu version
```

If the short alias is unavailable, use `open-browser-use`.
Running `open-browser-use` with no subcommand prints the CLI version, browser
extension detection status, extension version when available, and the next setup
or upgrade command.

## Set Up A Browser

After installing the CLI, register the native messaging host and open the Chrome Web Store page for the matching extension:

```sh
open-browser-use setup
```

Ask the user to install or enable Open Browser Use from the opened store page. Chrome may ask the user to confirm, enable the extension, or restart. Do not bypass this user step.

For Chrome Beta, register that browser explicitly:

```sh
open-browser-use setup --browser chrome-beta
```

For Microsoft Edge (Stable), register that browser explicitly:

```sh
open-browser-use setup --browser edge
```

Edge no longer reliably allows installing extensions from the Chrome Web
Store (Microsoft removed the "allow extensions from other stores" toggle in
stable Edge in late 2025), so `setup --browser edge` is best-effort: it still
registers the native messaging host and attempts the External Extensions
registration, but the extension install itself may not take effect. The
command's output calls this out explicitly. Prefer the beta path below for
Edge unless the store-based install is confirmed to work in the user's
environment:

```sh
open-browser-use setup beta --browser edge
```

For BitBrowser, install or load the extension in the target BitBrowser instance,
then register the native host manifest into that instance's user-data directory:

```sh
open-browser-use install-manifest --browser <bitbrowser-instance-id>
```

Use `open-browser-use profiles --connected --json` to see BitBrowser instance
ids in the `browserInstance` field once the instance is detectable.

While the Chrome Web Store item is unavailable or pending review, use the release ZIP path:

```sh
open-browser-use setup beta
```

This downloads the latest keyed `open-browser-use-chrome-extension-*.zip` from GitHub Releases and registers the native host for that stable extension id. It opens `chrome://extensions/` and reveals the ZIP in Finder or the system file manager only when the browser extension is missing or older than the CLI-expected version. Ask the user to enable Developer mode and drag that ZIP into the Chrome extensions page when setup prints that next step.

For Chrome Beta, use `open-browser-use setup beta --browser chrome-beta`. For
Edge, use `open-browser-use setup beta --browser edge`; unlike
`setup --browser edge`, this path does not depend on any extension store and
works the same way as it does for Chrome.

Repair only the native host manifest:

```sh
open-browser-use install-manifest
```

Use `--browser chrome-beta`, `--browser edge`, or
`--browser <bitbrowser-instance-id>` to repair a non-default browser.

Print the manifest without installing:

```sh
open-browser-use manifest
```

## Platform Notes

- macOS and Windows can require the user to approve or enable the extension after Chrome sees it.
- Linux external extension registration can require elevated permissions depending on Chrome installation paths.
- Chrome native messaging host name is `com.ifuryst.open_browser_use.extension`. Edge uses the same host name and the same `chrome-extension://` origin scheme, since Edge is Chromium-based.
- Edge is supported on Stable only (no Beta/Dev/Canary channel) on macOS, Windows, and Linux for profile discovery and native messaging manifest installation. External Extensions auto-install on Edge is best-effort only; see "Set Up A Browser" above.
- On Linux, only Chrome and Edge have full `--browser` selector support (profile roots, native messaging manifest, and External Extensions paths). Chrome Beta and BitBrowser are not yet supported on Linux.
- The default socket registry is under `/tmp/open-browser-use/` on Unix-like systems.

## Verification

Run:

```sh
open-browser-use ping --session-id "$OBU_SESSION_ID"
open-browser-use info --session-id "$OBU_SESSION_ID"
open-browser-use user-tabs --session-id "$OBU_SESSION_ID"
```

For one-off installation checks, a temporary session id is enough. Agent browser
tasks should still create and reuse a task-unique session id before opening or
claiming tabs.

If `ping` cannot communicate with Chrome, ask the user whether Chrome is installed and running, whether the extension is enabled, and whether they approved any Chrome prompt. Then use [troubleshooting.md](troubleshooting.md).
