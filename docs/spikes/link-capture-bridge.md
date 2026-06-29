# Spike — Link-capture bridge & mobile capture architecture (Plan 11)

**Question:** How should the Chrome extension hand captures to the desktop app — and does
the desktop app have to be running? Can the same architecture serve iOS/Android
share-sheet capture later, all with **zero Reflect-hosted infrastructure**?
(Desk research for the [Plan 11](../plans/11-link-capture.md) bridge spike; the code
spike below still has to validate packaging.)

**Verdict: native messaging, with the host as a *spooler*, not a relay.** The host binary
writes the capture into a **capture inbox** (a spool directory of JSON + asset files);
the desktop app drains the inbox — immediately via its file watcher when running, or on
next launch when not. The inbox *is* the IPC. The same envelope + drain action serves
iOS (app-group inbox written by a share extension) and Android (ACTION_SEND) later.
**No hosted API is needed for any scenario**, and the desktop app does **not** need to be
open to capture.

## Key findings

### 1. Native messaging works with the app closed — that decides it

- Chrome itself spawns the host binary per `sendNativeMessage` call and communicates over
  stdio; no daemon, launchd job, or running app is required
  ([Chrome native messaging](https://developer.chrome.com/docs/extensions/develop/concepts/native-messaging)).
  A capture succeeds with Reflect closed **iff the host can do something useful alone** —
  which a spool-to-inbox host can. (Counterexample: Bitwarden's host is a live proxy to
  the app and hangs when the app is closed —
  [bitwarden/clients#20929](https://github.com/bitwarden/clients/issues/20929).)
- Size limits don't bite: **extension→host is 64 MiB** in current Chrome (4 GB in
  Firefox); only **host→extension is 1 MB**, so replies must stay small JSON acks. A
  base64 screenshot of a few MB fits comfortably. This removes the plan's stated reason
  for the loopback-HTTP fallback ("screenshot payload size").
- `runtime.sendNativeMessage` spawns one host process per message and treats the first
  reply as the response — the right shape for infrequent captures, with no MV3
  service-worker keepalive concerns (a `connectNative` port would also keep the SW alive
  if ever needed —
  [SW lifecycle](https://developer.chrome.com/docs/extensions/develop/concepts/service-workers/lifecycle)).
- Protocol discipline: 4-byte native-order length prefix + UTF-8 JSON; **stdout must stay
  protocol-pure** (log to stderr); binary data travels base64 inside JSON
  (`tabs.captureVisibleTab` already returns a base64 data URL).

### 2. The spool inbox beats a socket relay

KeePassXC/Bitwarden/1Password all relay host→socket→running-app because they need the
app's live unlocked state. Captures have no such dependency, so the relay (and its
must-be-running flaw) is pure cost. Instead:

- Host validates the payload, decodes the screenshot, and **atomically writes**
  (`tmp/` + `rename(2)`, Maildir discipline, one uniquely-named file per capture) into
  the inbox.
- App running → its watcher sees the new file within ~1 s and runs the drain pipeline:
  resolve target → privacy gate → copy assets to `assets/` → write raw `[[Links]]`
  entry + provenance → remove spool file → schedule async enrichment (meta scrape +
  BYOK AI description) → reindex. App closed → the capture waits; the app drains the
  inbox on next launch. Raw link is never lost.
- The host finds the inbox via a small pointer file the app maintains in a fixed app-data
  location (active graph path). No graph configured → the host replies with a typed error
  the extension can surface ("Open Reflect and pick a graph first").
- Security: filesystem permissions (user-only) replace ports and tokens entirely, and the
  browser enforces the `allowed_origins` extension-ID allowlist on who may launch the
  host — strictly stronger than any loopback token scheme.
- Nothing surveyed ships exactly this (password managers can't), but every mobile capture
  app does (§5) — it's the same inbox pattern, which is the point.

### 3. Packaging & registration (the real risk, now mapped)

- **Sidecar:** a tiny Rust crate in the workspace, bundled via `externalBin`; lands in
  `Reflect.app/Contents/MacOS/` and is **signed + notarized with the app** by the Tauri
  bundler ([sidecar docs](https://v2.tauri.app/develop/sidecar/)). Tauri doesn't auto-build
  workspace bins — build + rename to `<name>-<triple>` in `beforeBuildCommand` (the
  official Node-sidecar guide's pattern). Watch open bug
  [tauri#11992](https://github.com/tauri-apps/tauri/issues/11992) ("nested code is
  modified" notarization failure with externalBin) — first thing the code spike checks.
- **Do not** make the main binary dual-mode (argv flag): the browser SIGKILLs the host on
  disconnect, `tauri-plugin-single-instance` would forward-and-exit (severing Chrome's
  stdio pipes), and WebView/log chatter on stdout corrupts framing. A ~300 KB dedicated
  host crate is the ecosystem norm (KeePassXC `keepassxc-proxy`, 1Password
  `BrowserSupport`, Claude Desktop `chrome-native-host`).
- **Registration:** the app **rewrites user-level manifests on every launch** (no admin
  rights anywhere; self-heals macOS app-translocation and app moves — what Claude Desktop
  and KeePassXC's "update manifests at startup" do). Only write for **browsers actually
  detected** (Claude Desktop drew criticism for spraying manifests for uninstalled
  browsers). Locations:
  - macOS/Linux: per-browser `NativeMessagingHosts/` dirs — Chrome (+Beta/Dev/Canary),
    Chromium, Edge, Brave, Vivaldi, Opera, and **Arc has its own**
    (`~/Library/Application Support/Arc/User Data/NativeMessagingHosts/`).
  - Windows: one `HKCU\Software\Google\Chrome\NativeMessagingHosts\<name>` key covers
    Chrome/Brave/Vivaldi/Opera, and Edge falls back to it by documented search order; add
    an explicit Edge key for completeness.
  - The manifest `path` may point inside the .app bundle (standard practice).
- **Stable extension IDs:** the Chrome Web Store mints the listing ID, while unpacked
  dev builds keep a separate key-derived ID via the manifest `key` trick. List both in
  `allowed_origins`; add the Edge Add-ons ID there too if/when an Edge listing ships
  (wildcards are not allowed). Firefox later: different manifest (`allowed_extensions` +
  fixed gecko ID); Safari: a wholly different pipeline (extension inside a containing
  app, `SFSafariWebExtensionHandler`) — out of first wave, but see §6.

### 4. Loopback HTTP: demoted to packaging-failure fallback only

Joplin's design (127.0.0.1:41184+, `/ping` discovery scan, accept/reject pairing dialog
minting a token) is the best-of-breed loopback pattern if we ever need it. But it shares
one structural flaw — **nothing works when the app is closed** (Joplin's #1 support
issue) — plus port-squatting/impersonation by local processes, DNS-rebinding history
(Zotero), Chrome's Local-Network-Access churn (extensions exempt with host permissions,
but real breakage shipped in Chrome 142–143, fixed ≥144), and the standing cautionary
tale of unauthenticated loopback servers
([CVE-2025-49596](https://www.oligo.security/blog/critical-rce-vulnerability-in-anthropic-mcp-inspector-cve-2025-49596)).
Native messaging avoids the entire class. Deep links stay a last resort: ~2 KB practical
URL budget; Obsidian's clipper abandoned URI transfer for the clipboard because of it.

### 5. Mobile: the same inbox, a different transport

The capture envelope + inbox drain is **exactly** the shipping consensus on mobile, which
is what makes "down the road" cheap:

- **iOS share extension** (the share-button flow Reflect V1 users know): a separate
  Xcode target; receives the page URL via `NSItemProvider`, and title/selection via the
  `NSExtensionJavaScriptPreprocessingFile` JS hook (runs in the Safari page, returns
  `{title, selection, …}` to the extension). It **cannot** write to the app's Documents
  or launch the app (App-Review-risky responder-chain hacks aside) — it writes a capture
  JSON into the **App Group shared container** and exits. The main app drains on next
  launch/foreground into its local graph clone and enriches with BYOK keys (sharable via
  keychain access group if ever needed). Joplin, Logseq (Capacitor send-intent), Drafts
  ("Quick Capture… without further interaction"), and Bear all ship this shape; use one
  uniquely-named file per capture (Joplin's single fixed filename can drop a second
  capture — a visible flaw to avoid). ~120 MB extension memory cap is irrelevant for
  URL+selection captures.
- **Tauri 2 fits:** extra Xcode targets are added in the XcodeGen `project.yml` Tauri
  already templates (`src-tauri/ios.project.yml`), entitlements (App Groups) in the
  generated-and-committed entitlements files — proven workable
  ([tauri#10074](https://github.com/tauri-apps/tauri/issues/10074),
  [tauri#14332](https://github.com/tauri-apps/tauri/issues/14332)), with
  [IT-ess/tauri-plugin-mobile-sharetarget](https://github.com/IT-ess/tauri-plugin-mobile-sharetarget)
  as a working iOS+Android example (Rust-side FIFO queue for cold-start webview timing).
- **Android is easier:** an ACTION_SEND intent filter in the committed
  `gen/android` manifest; Chrome shares URL (`EXTRA_TEXT`) + title (`EXTRA_SUBJECT`); the
  share launches the **full app process**, so it can ingest and even sync immediately.
- **Sync closes the loop:** the mobile app drains into its *own* Git clone (Plan 12);
  GitHub propagates to desktop. No relay, no hosted API, no cross-device IPC.

### 6. Scenario matrix

| Scenario | Path | Outcome |
|---|---|---|
| Desktop app running | extension → host → inbox → watcher | Saved + enriched in seconds; extension shows success |
| App installed, not running | extension → host → inbox | Saved (host acks "queued"); enriched on next app launch |
| App not installed | `connectNative` fails ("host not found") | Extension explains + links download; queues capture in `chrome.storage`, retries later |
| No graph configured | host finds no inbox pointer | Typed error; extension says "open Reflect first" |
| Private target | normal path; privacy gate in app | Raw link saved, zero outbound AI (Plan 11 step 3 unchanged) |
| iOS (later) | share extension → app-group inbox | Saved instantly; ingested + enriched on next app open; syncs via GitHub |
| Android (later) | ACTION_SEND → full app | Saved + enriched immediately |
| iOS Safari extension (much later) | same WXT codebase → `sendNativeMessage` → Swift handler → app-group inbox | Possible; plumbing-heavy (`safari-web-extension-converter`); an optimization, not the first mobile path |
| Any scenario needing Reflect-hosted infra | — | **None exist.** Zotero's hosted fallback only buys "app not installed", which is inherently unservable without infra and acceptably degrades to install-prompt + retry queue |

## Implications for Plan 11

1. **Reframe step 2:** the host doesn't "forward to the running desktop app" — it spools
   to the capture inbox; the app's watcher/launch-drain is the receiver. Drop any socket.
2. **Shared contract:** define the capture envelope (zod) in `@reflect/core`
   (`actions/capture`) as platform-agnostic — `{url, title, selection?, screenshotRef?,
   note?, capturedAt, source}` — written identically by the NM host (desktop), share
   extension (iOS), and intent handler (Android). `drainCaptureInbox` is one core action.
3. **Registration detail:** manifests rewritten **every launch** (not "on install"), only
   for detected browsers; both store IDs in `allowed_origins`; `key`-pinned dev ID.
4. **Watcher change:** the Rust watcher currently only reports `.md` under `daily/` and
   `notes/` — it needs to also watch the inbox dir (or a second watcher).
5. **Loopback HTTP fallback narrows** to "native-messaging packaging proves unshippable"
   (size is a non-issue); if ever built, copy Joplin's pairing flow + Host-header checks.
6. **Acceptance criterion to add:** capturing with the desktop app **closed** queues the
   capture and it lands (raw, then enriched) on next launch — the spool design gives this
   for free, and it's the headline UX win.
7. **Mobile door-openers now (cheap):** keep the envelope platform-agnostic (done via #2);
   nothing else in the desktop design needs to change. Reserve the App Group ID when iOS
   work actually starts.

## What the code spike must still verify

- `externalBin` sidecar **signs + notarizes** cleanly with the current Tauri version
  ([tauri#11992](https://github.com/tauri-apps/tauri/issues/11992) is open).
- End-to-end WXT MV3 → `sendNativeMessage` → Rust host → inbox file → app watcher drain,
  with a real ~2–4 MB `captureVisibleTab` screenshot (latency + memory).
- Manifest registration empirically on Arc and one non-Chrome Chromium (Brave or Edge);
  Windows HKCU path.
- Host behavior with no inbox pointer / unwritable inbox (typed errors reach the
  extension UI).

## Caveats

- Enrichment is deferred when the app is closed — the daily-note entry appears on next
  launch, not "live". That's inherent to no-hosted-API and matches the product principle;
  the extension should say "queued" honestly rather than "saved to today's note".
- AppImage (Linux) executable paths are unstable across mounts — manifest registration on
  Linux must point at a stable installed path (deb/rpm) or re-register per run; deal with
  it when Linux packaging lands (Plan 15).
- Firefox/Safari support are real but separate workstreams (different manifest dialect /
  containing-app pipeline); the envelope + inbox are unchanged for both.
