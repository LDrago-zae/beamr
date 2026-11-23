# RemoteDesk: Full Implementation Plan

Working name. A Flutter app (iOS + Android) that gives full remote control of Windows and macOS machines from a phone. Written so it can be saved as `docs/PLAN.md` and used directly in Cursor.

This file is the source of truth for the beamr repository. The product working name is RemoteDesk. The repo folder is `beamr`.

## 0. How to use this in Cursor

- Save this as `docs/PLAN.md`. Copy section 14 into `.cursor/rules`.
- Work one phase at a time. Prompt pattern: Read docs/PLAN.md. Implement Phase N, task N.x only. Follow the protocol in section 5 exactly. Add tests. Stop when the acceptance criteria are met.
- Section 5 (protocol) is the single source of truth. Change the spec first, then the code on all three sides (mobile, agent, server).

## 1. Product scope

**Goal:** a user installs an agent on their Mac or Windows PC, pairs it with their phone once, and can then see and control that machine from anywhere.

**Full-control feature set:**

| # | Feature | Notes |
| --- | --- | --- |
| 1 | Live screen view | Multi-monitor switching, quality presets |
| 2 | Touch mouse | Tap, long-press right click, drag, two-finger scroll, pinch zoom |
| 3 | Keyboard | System keyboard plus a special-keys bar (Esc, Tab, Ctrl, Alt, Cmd, Win, F-keys, arrows) |
| 4 | Clipboard sync | Text first, images later |
| 5 | File transfer | Browse remote folders, upload from phone, download to phone |
| 6 | Quick actions | Lock, sleep, restart, shutdown, volume, brightness, media keys |
| 7 | Session privacy | Blank the remote screen, block local input while controlled |
| 8 | Unattended access | Connect without anyone at the PC, per-device policy |
| 9 | Audio streaming | Later phase, OS-dependent |
| 10 | Remote terminal and process manager | Later phase |
| 11 | Accounts, device list, revoke, audit log | Required for security |
| 12 | Subscription | RevenueCat, free tier of 1 device |

**Out of scope for v1:** controlling the Windows login screen on locked machines without a service (Phase 6 adds this), macOS FileVault pre-boot screen (not possible), Linux agent, web client.

## 2. Architecture

```
Phone (Flutter) <==== WebRTC: DTLS-SRTP video + DataChannels ====> Desktop Agent
      |                                                               |
      +---------- WSS ---> Signaling/API server <--- WSS -------------+
      +---------- STUN/TURN (coturn) <------------------------------+
```

**Connection flow**

1. Agent starts, authenticates to the server with its device credential, opens a WSS connection, and stays registered as online.
2. Phone logs in, fetches its device list with online status, and taps a device.
3. Phone calls `POST /v1/sessions` and receives a short-lived session token plus time-limited TURN credentials.
4. Phone and agent exchange SDP offer/answer and ICE candidates through the signaling WebSocket.
5. Both sides verify the other's signed DTLS fingerprint against the public key pinned at pairing.
6. P2P is attempted first. TURN relay is the fallback. Media stays end-to-end encrypted even over TURN.
7. Video flows agent to phone. Input and control flow phone to agent over DataChannels.
8. Either side ends the session. The server writes an audit record.

## 3. Technology decisions

| Area | Choice | Reason |
| --- | --- | --- |
| Mobile | Flutter, Riverpod, go\_router, `flutter_webrtc` | Existing skill, one codebase |
| Desktop agent (MVP) | Electron + TypeScript | Chromium gives hardware-accelerated capture, H.264 encode and a mature WebRTC stack for free |
| Native layer | Rust addon via `napi-rs` | Input injection, power actions, window and display info, later the Windows service |
| Backend | Node.js, Fastify, `ws`, TypeScript | Fast to build, same language as the agent |
| Auth | Firebase Auth | Email, Google, Apple sign-in; Apple sign-in is required on iOS if you offer other social logins |
| Database | Postgres (Supabase or Neon) | Relational device and pairing data |
| TURN/STUN | coturn on a VPS with `use-auth-secret` | Cheap, standard, time-limited credentials |
| Hosting | Fly.io or Railway for the server, a small VPS for coturn | Needs a public IP and UDP/TCP 3478 and 443 |
| Billing | RevenueCat | Handles both stores |
| Crash and analytics | Sentry, PostHog | Debugging remote sessions without seeing the user's screen |

**Planned migration:** the Electron agent ships v1. If the install size (about 100 MB) or memory use becomes a problem, rewrite capture and encoding natively in Rust later. The protocol does not change, so mobile and server are unaffected.

## 4. Repository layout (monorepo)

```
remotedesk/
  docs/PLAN.md
  .cursor/rules
  apps/
    mobile/            Flutter app
    agent/             Electron + TypeScript
    server/            Fastify signaling + REST API
  native/
    input-rs/          Rust napi addon (input, power, displays)
    win-service/       Rust Windows service (Phase 6)
  packages/
    protocol/          Message schemas (TypeScript, zod) + generated Dart models
  infra/
    coturn/            turnserver.conf, docker-compose
    server/            Dockerfile, fly.toml
  scripts/
```

The `protocol` package defines every message once. Generate Dart classes from the same JSON Schema so the mobile app can never drift from the agent.

The current checkout lives in `beamr/`. Use that folder as the monorepo root. `apps/mobile` is the Flutter app. Do not leave the Flutter project at the repository root once Phase 0 starts.

## 5. Protocol specification

All messages are JSON with the envelope `{ v, type, id, ts, payload }`. `v` is the protocol version (start at 1). Unknown types are ignored and logged, never crash.

### 5.1 Signaling messages (WebSocket)

| type | Direction | Payload |
| --- | --- | --- |
| `hello` | client to server | session token, role (`agent` or `mobile`), device id |
| `offer` | mobile to agent | SDP, signed DTLS fingerprint |
| `answer` | agent to mobile | SDP, signed DTLS fingerprint |
| `ice` | both | ICE candidate |
| `bye` | both | reason code |
| `presence` | server to mobile | device id, online or offline, agent version |
| `error` | server to client | code, message |

### 5.2 DataChannels

| Channel | Reliability | Use |
| --- | --- | --- |
| `input-fast` | unordered, `maxRetransmits: 0` | Mouse move, scroll deltas |
| `control` | reliable, ordered | Clicks, key events, commands, display switch, quality, clipboard |
| `files` | reliable, ordered | File listing and chunked transfer |
| `telemetry` | unordered | Agent to phone: RTT, fps, bitrate, dropped frames |

### 5.3 Input events

Coordinates are normalized floats from 0 to 1 relative to the selected display. The agent converts to pixels using that display's bounds and DPI scale. Never send raw pixel positions from the phone.

| type | Payload |
| --- | --- |
| `mouse.move` | display id, x, y |
| `mouse.button` | button (left, right, middle), state (down, up) |
| `mouse.scroll` | dx, dy in normalized wheel units |
| `key.event` | key code (USB HID usage), state, modifiers bitmask |
| `key.text` | UTF-8 string, injected as unicode text for the soft keyboard |
| `display.select` | display id |
| `quality.set` | preset (auto, data-saver, balanced, sharp), max fps, max bitrate |

Use physical key codes (HID usage IDs) for `key.event`, and map them per OS inside the agent. Use `key.text` for typed characters so layouts and IME input work.

### 5.4 Control commands

`power.lock`, `power.sleep`, `power.restart`, `power.shutdown`, `audio.volume`, `display.brightness`, `media.key`, `privacy.blank`, `privacy.blockLocalInput`, `clipboard.set`, `clipboard.changed`, `session.end`. Destructive power commands require a confirmation round trip: the phone sends the command, the agent replies with a nonce, and the phone must send the nonce back within 10 seconds.

### 5.5 File transfer

Messages: `fs.list` (path), `fs.stat`, `fs.get` (path, offset), `fs.put.begin` (path, size, sha256), `fs.chunk` (id, index, bytes, base64 or binary frame), `fs.put.end`, `fs.error`. Chunk size 16 KB, backpressure via `bufferedAmountLowThreshold`. Verify sha256 at the end. The agent enforces an allowlist of roots (Home, Desktop, Documents, Downloads by default).

### 5.6 REST API

| Endpoint | Purpose |
| --- | --- |
| `POST /v1/device-code` | Agent requests a pairing code (returns code, QR URL, polling token) |
| `POST /v1/device-code/confirm` | Logged-in phone confirms the code |
| `GET /v1/devices` | List devices with online status |
| `PATCH /v1/devices/:id` | Rename, change access policy |
| `DELETE /v1/devices/:id` | Revoke device |
| `POST /v1/sessions` | Start session, returns token and TURN credentials |
| `GET /v1/audit` | Session history |

### 5.7 Database tables

`users`, `devices` (id, user id, name, os, public key, policy, last seen), `phones` (id, user id, public key, push token), `sessions` (id, device id, phone id, started, ended, connection type p2p or relay, bytes), `audit_logs`. Index sessions by device and by start time.

## 6. Backend tasks

1. Fastify project, zod validation on every route, structured logging (pino), env config.
2. Firebase ID token verification middleware.
3. Device-code pairing flow (RFC 8628 style): agent displays a code and QR, phone confirms while signed in, server issues a long-lived device credential that is stored in the OS keychain by the agent.
4. WebSocket hub: authenticate on `hello`, route messages only between the two peers of one session, drop anything else.
5. TURN credential generator: HMAC-SHA1 of `expiry:userId` with the coturn shared secret, 1 hour TTL.
6. Presence: agent heartbeat every 15 s, mark offline after 45 s.
7. Rate limits: per IP and per account on pairing, session start and failed auth. Temporary lockout after repeated failures.
8. Audit log writer, plus push notification to the phone when a session starts from another phone.
9. Metrics and health endpoint.

**Acceptance:** two scripted clients can complete a full offer/answer/ICE exchange through the server; unauthorized clients cannot join someone else's session; load test of 500 concurrent sockets passes.

## 7. Desktop agent tasks

### 7.1 Shared (Electron main process)

- Tray app, auto-start at login, single instance, auto-update (electron-updater).
- Device-code sign-in screen with QR.
- Signaling client with reconnect and exponential backoff.
- WebRTC runs in a hidden renderer window. Capture with `desktopCapturer` and `getUserMedia` using the chromeMediaSource constraints. Prefer H.264, fall back to VP8. Target 30 fps, allow up to 60 on good links.
- Adaptive quality: read `getStats()` every second, lower resolution first, then fps, then bitrate. Request a keyframe on packet loss spikes.
- Input dispatcher: validates every incoming message against the schema, rate-limits mouse events, and forwards to the Rust addon.
- Always-visible session indicator (tray icon state plus a small overlay) while someone is connected. This is non-negotiable for security and for store trust.
- Access policy engine: unattended allowed, ask on screen each time, or deny. Optional local password.

### 7.2 Rust addon (`native/input-rs`)

- Functions: `mouseMove`, `mouseButton`, `mouseScroll`, `keyEvent`, `typeText`, `listDisplays`, `lock`, `sleep`, `restart`, `shutdown`, `setVolume`, `setBrightness`, `mediaKey`, `setClipboard`, `getClipboard`.
- Windows: `SendInput` (use `KEYEVENTF_UNICODE` for text), `LockWorkStation`, `SetSuspendState`, `ExitWindowsEx`, Core Audio for volume.
- macOS: `CGEventCreateMouseEvent`, `CGEventCreateKeyboardEvent`, `CGEventKeyboardSetUnicodeString`, `pmset sleepnow`, `CGSession` lock via the screensaver command, `osascript` or CoreAudio for volume.
- Per-OS tables mapping HID usage IDs to native key codes, covered by unit tests.

### 7.3 Windows specifics

- Phase 1 to 5 run as a normal user process. It cannot see or click UAC prompts or the lock screen. This is expected and documented in the UI.
- **Phase 6 service:** a Rust Windows service running as SYSTEM. It watches session changes, launches a capture helper into the active session with `WTSQueryUserToken` and `CreateProcessAsUser`, and switches the helper to the input desktop (`OpenInputDesktop`, `SetThreadDesktop`) so UAC and the lock screen can be captured and controlled. Capture with DXGI Desktop Duplication in that mode.
- Ship an MSI or NSIS installer and sign it with an EV or standard code-signing certificate to avoid SmartScreen warnings.

### 7.4 macOS specifics

- Needs two TCC permissions: **Screen Recording** and **Accessibility**. Build a guided onboarding that detects missing permissions, deep-links to the right System Settings pane, and re-checks automatically.
- Distribute outside the Mac App Store as a Developer ID signed and notarized app with the hardened runtime. Sandboxed Mac App Store builds cannot do this.
- Start at login via a LaunchAgent or `SMAppService`.
- The agent cannot control the login window or the FileVault pre-boot screen. Say so in the UI.
- Apple Silicon and Intel universal build.

**Acceptance (per OS):** from a phone on LAN, view the screen at 30 fps, move the mouse, click, drag, type text and shortcuts, scroll, switch displays, and trigger lock and sleep, all with end-to-end input latency under 150 ms on LAN.

## 8. Flutter mobile app

### 8.1 Packages

`flutter_webrtc`, `flutter_riverpod`, `go_router`, `dio`, `web_socket_channel`, `firebase_auth`, `google_sign_in`, `sign_in_with_apple`, `mobile_scanner`, `local_auth`, `flutter_secure_storage`, `freezed` and `json_serializable`, `file_picker`, `purchases_flutter`, `sentry_flutter`.

### 8.2 Structure

Feature folders: `auth`, `devices`, `pairing`, `session` (rtc, input, ui), `files`, `settings`, `paywall`. Keep the WebRTC session logic in a plain Dart class behind a Riverpod provider so it can be tested without UI.

### 8.3 Screens

Onboarding, sign in, device list (online dot, OS icon, last seen), add device (scan QR or enter code), remote session, file browser, device settings (name, access policy, revoke), account and subscription.

### 8.4 Remote session UI and gestures

| Gesture | Action |
| --- | --- |
| One-finger drag | Move the cursor (trackpad mode) |
| Tap | Left click |
| Two-finger tap | Right click |
| Long-press then drag | Click and drag |
| Two-finger drag | Scroll |
| Pinch | Zoom the viewport locally, not the remote display |
| Three-finger swipe | Switch display or app, configurable |

Two input modes: **Trackpad** (relative movement, default) and **Touch** (tap where you point, absolute). Floating toolbar with keyboard toggle, special-keys bar, quick actions sheet, quality picker, display switcher, privacy toggle, and disconnect. Landscape lock option. Show a connection badge (P2P or Relay), RTT and fps in a collapsible overlay.

### 8.5 Mobile specifics

- Keep the screen awake during a session. Handle app backgrounding: pause video, keep the socket briefly, then end the session after 60 s.
- iOS: camera permission for QR, Face ID usage string, local network usage string for LAN discovery.
- Use biometric re-auth (`local_auth`) before starting a session, configurable.

**Acceptance:** a user can sign in, pair a PC via QR, connect, and operate the PC with all gestures, without reading any docs.

## 9. Security requirements

This app grants full control of someone's computer. Treat each item below as a release blocker.

1. **End-to-end encryption:** WebRTC DTLS-SRTP. TURN and the server never see media or input.
2. **Identity pinning:** each agent and phone has an Ed25519 key pair generated on device and stored in the Keychain, Keystore or OS credential store. Public keys are exchanged at pairing. Each side signs its DTLS fingerprint in the SDP, and the peer verifies the signature. This stops a malicious signaling server from doing a man-in-the-middle.
3. **Pairing confirmation:** show a 6-digit short authentication string derived from both public keys on phone and PC, and require the user to confirm they match.
4. **Short-lived tokens:** session tokens valid for minutes, single use, bound to the device and phone pair.
5. **Access policies:** per device, choose unattended, ask each time, or deny. Optional local password on the agent.
6. **Visibility:** persistent indicator on the PC during a session, push notification to all of the owner's phones when a session starts.
7. **Revocation:** removing a device invalidates its credential immediately and drops live sessions.
8. **Destructive actions:** nonce confirmation for shutdown, restart, and file delete.
9. **File access:** root allowlist, path normalization to block traversal, size limits.
10. **Abuse controls:** rate limits, lockouts, and audit logs. Add terms of service that forbid unauthorized access and respond to abuse reports quickly. Scammers use remote-access tools, and stores take this seriously.
11. **Hygiene:** dependency scanning, secrets in a vault, signed releases, a published security contact, and a third-party penetration test before public launch.
12. **Privacy:** no screen recording, no frame storage on the server, minimal metadata retention, a clear privacy policy.

## 10. Phased delivery plan

Week counts assume one full-time developer using Cursor heavily. Add 30 to 50 percent for part-time work.

### Phase 0: Foundations (week 1)

- 0.1 Monorepo, CI (GitHub Actions), lint, formatting, commit hooks.
- 0.2 `packages/protocol` with zod schemas and Dart model generation.
- 0.3 Firebase project, Postgres, Fly.io app, coturn on a VPS with TLS.
- 0.4 Cursor rules file and this plan committed.

**Done when:** CI is green and a hello-world message round-trips through all three apps.

### Phase 1: Windows P2P proof of concept (weeks 2 to 3)

- 1.1 Agent captures the primary display and publishes a WebRTC video track.
- 1.2 Minimal Flutter viewer connects to the agent via a hard-coded local signaling server on LAN.
- 1.3 Rust addon: mouse move, click, scroll, key, text.
- 1.4 Wire `input-fast` and `control` DataChannels.

**Done when:** you can control a Windows PC from a phone on the same Wi-Fi.

### Phase 2: Cloud connectivity and accounts (weeks 4 to 5)

- 2.1 Full backend from section 6.
- 2.2 Device-code pairing, QR flow, key exchange, SAS confirmation.
- 2.3 TURN integration and relay fallback testing on mobile data, symmetric NAT and a corporate network.
- 2.4 Device list and presence in the app.

**Done when:** control works from LTE to a home PC with no port forwarding, and relay mode works when P2P is blocked.

### Phase 3: Mobile UX (weeks 5 to 6)

- 3.1 Gesture engine with trackpad and touch modes.
- 3.2 Soft keyboard plus special-keys bar.
- 3.3 Quality presets and adaptive bitrate.
- 3.4 Reconnect handling after network switches (Wi-Fi to LTE) using ICE restart.

**Done when:** a 30-minute session survives a Wi-Fi to LTE switch and feels usable.

### Phase 4: macOS agent (weeks 6 to 8)

- 4.1 Port the Rust addon to macOS, add key code tables.
- 4.2 Permission onboarding for Screen Recording and Accessibility.
- 4.3 Signing, notarization, universal build, LaunchAgent.
- 4.4 Fix retina scaling and multi-display coordinate mapping.

**Done when:** the Windows acceptance checklist passes on Intel and Apple Silicon Macs.

### Phase 5: Full-control features (weeks 8 to 10)

- 5.1 Clipboard sync (text), then images.
- 5.2 File browser and transfer with resume.
- 5.3 Quick actions with nonce confirmation.
- 5.4 Multi-monitor selection, privacy blank screen, local input lock.
- 5.5 Access policy UI and on-PC approval prompt.

### Phase 6: Windows service and secure desktop (weeks 10 to 12)

- 6.1 SYSTEM service plus session helper architecture.
- 6.2 Capture and control UAC prompts and the lock screen.
- 6.3 Installer, service lifecycle, crash recovery.

This is the hardest phase. Time-box it, and ship v1 without it if needed.

### Phase 7: Hardening and launch (weeks 12 to 14)

- 7.1 Security review from section 9, then an external penetration test.
- 7.2 RevenueCat paywall, free tier limits.
- 7.3 Store assets, privacy policy, terms, support site, abuse reporting process.
- 7.4 TestFlight and Google Play closed testing with 20 to 50 real users on varied networks.
- 7.5 Public release, then iterate on audio streaming, terminal, and process manager.

## 11. Testing strategy

- **Unit:** protocol schemas, key mapping tables, coordinate conversion, token logic.
- **Integration:** two headless WebRTC peers through the real signaling server in CI.
- **Network matrix:** same LAN, home to LTE, symmetric NAT forced to TURN, 3 percent packet loss and 150 ms latency using `tc netem` or Network Link Conditioner.
- **Device matrix:** Windows 10 and 11, macOS 13 to 15, at least two iOS and three Android devices, high-DPI and multi-monitor setups.
- **Latency budget:** measure glass-to-glass (input to visible change) and track it per release. Target under 150 ms on LAN and under 300 ms over the internet.
- **Security tests:** token replay, cross-account session join, malformed message fuzzing, path traversal on file APIs.

## 12. Release, distribution and business

- **iOS:** the app must be a pure remote client for the user's own machines. Review App Review Guideline 4.2.7 on remote desktop clients before building the UI, and avoid anything that looks like it executes code on the phone.
- **Android:** Google Play has a policy on remote-access and deceptive behavior. State clearly in the listing and in-app that the user controls their own devices.
- **Desktop:** distribute from your own site. Windows needs code signing. macOS needs Developer ID and notarization.
- **Pricing idea:** free for 1 device and short sessions, Pro for unlimited devices, file transfer, unattended access and priority relay. TURN bandwidth is your main variable cost, so meter relay usage.
- **Positioning:** Chrome Remote Desktop, AnyDesk, TeamViewer, Splashtop and Jump Desktop already exist. Pick an angle such as phone-first UX (macros, one-tap shortcuts, widgets), a clean developer experience, or AI commands, and put it in the first version rather than adding it later.

## 13. Main risks

| Risk | Mitigation |
| --- | --- |
| Latency feels poor on mobile networks | Adaptive bitrate, H.264 hardware decode, regional TURN servers |
| Windows UAC and lock screen | Phase 6 service, clearly documented limitation before that |
| macOS permission drop-off at onboarding | Guided flow with live permission detection |
| Abuse by scammers | Visible indicators, owner-only pairing, rate limits, fast takedown |
| TURN bandwidth cost | Prefer P2P, cap relay on free tier, bitrate limits |
| Electron footprint | Accept for v1, plan native rewrite only if metrics demand it |
| Store rejection | Read the guidelines early, keep iOS a pure client |

## 14. Cursor rules (paste into `.cursor/rules`)

```
Project: RemoteDesk. Read docs/PLAN.md before any task.
- Never change protocol messages without first updating packages/protocol and docs/PLAN.md section 5.
- Validate every incoming message with the zod schema before use. Never trust the peer.
- Mobile: Flutter, Riverpod, go_router. No business logic inside widgets. Keep WebRTC logic in plain Dart classes.
- Agent: TypeScript strict mode. All OS calls go through native/input-rs, never inline shell commands with user data.
- Server: Fastify with zod on every route. No route without auth unless listed in the plan.
- Never log tokens, keys, clipboard contents, file paths or screen data.
- Secrets only from environment variables or the OS keychain. Never commit secrets.
- Every feature ships with tests. Every PR must pass lint, tests and type checks.
- Implement only the requested phase and task. Do not refactor unrelated code.
- Security items in section 9 are release blockers. Flag any change that weakens them.
```

## 15. First three Cursor prompts

1. Set up the monorepo from section 4 with CI, lint and formatting, and create `packages/protocol` with zod schemas for every message in section 5, plus a script generating Dart models.
2. Build `apps/server` per section 6 tasks 1 to 4, with an integration test that runs two scripted clients through offer, answer and ICE.
3. Build the Phase 1 Windows agent capture and WebRTC publisher in `apps/agent`, with the Rust addon in `native/input-rs` exposing `mouseMove`, `mouseButton`, `mouseScroll`, `keyEvent` and `typeText`.
