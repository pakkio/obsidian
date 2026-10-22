---
tags:
  - craftsim
  - voice
  - webrtc
  - janus
  - wolfvoice
  - synthvoice
  - operations
aliases:
  - WebRTC voice
  - os-webrtc-janus
  - wolfvoice
---

# Voice (WebRTC, Janus & SynthVoice)

← Back to [[CraftSim]] | Related: [[Build, Configuration & Self-Hosting]], [[MCP Server & Live Diagnostics]], [[Known and Unresolved Problems]], [[Performance & Benchmarks]]

> [!WARNING] This note was rebuilt on 2026-10-17
> The earlier version of this page was lost (the file was found empty, and the empty file had already been committed). This is a **reconstruction** from `CLAUDE.md` (section "WebRTC voice"), `README.md` (`[WebRtcVoice]` / `[WebRTCVoice]` tables), `JANUS-SETUP.md`, `TEST-PLAN-VOICE.md` and the sources named below. It follows those documents; it was **not** checked against a running voice server or a viewer. Compare with the old version if you can restore it (Obsidian *File recovery*, MEGA version history).

## 1. What is there

Voice in CraftSim is WebRTC voice for modern Second Life-derived viewers (Firestorm). Three pieces exist in the tree:

| Piece | Where | Role |
| :--- | :--- | :--- |
| **`os-webrtc-janus` addon** | `OpenSim/Addons/os-webrtc-janus/` (4 projects: `WebRtcVoice`, `WebRtcVoiceRegionModule`, `WebRtcVoiceServiceModule`, `Janus`) | the region side: capabilities `ProvisionVoiceAccountRequest` and `VoiceSignalingRequest`, the voice service connector, session bookkeeping. **Active.** Upstream's addon byte for byte plus local patches (168 changed lines in `WebRtcVoiceRegionModule.cs`, 5 in `JanusMessages.cs`) |
| **Internal module** | `OpenSim/Region/OptionalModules/Avatar/Voice/WebRTCVoice/WebRTCVoiceModule.cs` (1245 lines) | an older in-tree implementation with a selectable backend (`janus` or `wolfvoice`), group / IM voice and session lifecycle. **Kept, disabled** (`[WebRTCVoice] enabled = false`) |
| **Voice server** | outside this repo: [wolfvoice](https://github.com/pakkio/mcp-wolfvoice) (`pakkio/mcp-wolfvoice`, a copy of Wolf Software Systems' project plus two logging commits) | mixes the audio. Older documents call the same program **synthvoice** (names `SYNTHVOICE_*`, binary `synthvoice`) |

**Only one of the two region-side implementations may run.** They register the same capabilities and the same `VoiceServerType` simulator feature; if both are on, the last to register wins, "which is a race and not a choice". The shipped `bin/OpenSim.ini` turns the internal module off and the addon on.

## 2. How a call is wired

1. The viewer asks the region for a voice account (`ProvisionVoiceAccountRequest`) and then exchanges signalling (`VoiceSignalingRequest`: SDP and ICE candidates).
2. The region does not mix audio. `WebRtcVoiceServiceConnector` forwards both requests to the voice server as **JSON-RPC 2.0 over HTTPS**, methods `provision_voice_account_request` and `voice_signaling_request`, parameters `request` / `userID` / `scene`. That contract matches wolfvoice's `docs/PROTOCOL.md`.
3. wolfvoice needs `channel_type` of `local` or `multiagent` with a non-empty `channel`, which is why the region fills in the empty channel of a one-to-one (p2p) call before forwarding.
4. The connector gives the backend **10 seconds** to answer a provision (`WebUtil.PostToService(uri, request, 10000, true)`); after that the request fails on the region side even if the server answers later.
5. The region announces `VoiceServerType=webrtc` and, from `StunServers`, a `stun-servers` simulator feature.

Why the backend is wolfvoice and not Janus (from `TEST-PLAN-VOICE.md`; a statement about Firestorm's source, not re-verified here): Firestorm treats voice as unusable until the SCTP data channel `SLData` opens. In `llvoicewebrtc.cpp` the state `VOICE_STATE_WAIT_FOR_DATA_CHANNEL` advances only when the data interface exists, with no timeout and no exception for OpenSim. Janus AudioBridge does not carry that channel and the addon never negotiates it. Position, per-user gain, mute and the roster travel over it. So a hybrid ("spatial voice on wolfvoice, groups on Janus") is not possible: the non-spatial branch of the viewer sits behind the same data-channel check. wolfvoice also mixes **per listener**, which spatial audio needs.

Janus is still supported by the code. `JANUS-SETUP.md` describes what a stock Janus needs: the `janus.plugin.audiobridge` plugin enabled, the HTTP transport with the voice base path (the addon speaks REST, not WebSocket), and the admin API with its secret. Its four settings (`JanusGatewayURI`, `APIToken`, `JanusGatewayAdminURI`, `AdminAPIToken`) must all be present or the addon disables itself at start-up (`Janus/WebRtcJanusService.cs`).

## 3. Configuration

`[WebRtcVoice]` in `bin/OpenSim.ini` configures the active addon (not `bin/config/wolfvoice.ini`, which is only wolfvoice's own suggestion):

| Key | Shipped value | Meaning |
| :--- | :--- | :--- |
| `Enabled` | `true` | the addon is on |
| `SpatialVoiceService`, `NonSpatialVoiceService` | `WebRtcVoice.dll:WebRtcVoiceServiceConnector` | both kinds of voice go to the JSON-RPC connector |
| `WebRtcVoiceServerURI` | `<voice-server>` | the wolfvoice server. **HTTPS is mandatory**: the SDP carries the DTLS fingerprint, so anyone who can rewrite plain signalling could become the media endpoint |
| `StunServers` | `stun:stun.l.google.com:19302` | advertised as the `stun-servers` feature. Needed with **any** backend: Firestorm hard-wires Second Life's STUN servers otherwise |
| `P2PRing` | `sessionstart` | when the callee of a one-to-one call is rung (section 4) |
| `MessageDetails` | `false` | log every message exchanged with the viewer; very verbose |

`[WebRTCVoice]` configures the internal module (`enabled = false` in the shipped file): `Backend` (`janus` default or `wolfvoice`), `WolfVoiceServerURI`, `JanusUrl`, `JanusApiSecret` (`${Secrets|JanusApiSecret}`, see [[Build, Configuration & Self-Hosting]]), `StunServers`. The host names in the two sections must agree.

The voice server host, its TLS certificate (the host name must match) and TCP 9443 reachable from the region hosts are prerequisites. Voice permissions must be on in **both** the estate and About Land.

## 4. One-to-one calls and ringing

`P2PVoiceCalls` (`WebRtcVoiceRegionModule/P2PVoiceCalls.cs`) is pure bookkeeping, with no scene or event queue: who is on a one-to-one voice session and whether the callee has already been rung.

- **The problem.** Under WebRTC voice the viewer sends `ChatSessionRequest` `start p2p voice` for **every** P2P conversation, text IMs included (`llimview.cpp`: the IM session is flagged `mP2PAsAdhocCall` because `LLWebRTCVoiceClient` has no outgoing-call interface), and the request cannot tell a call from an IM. A voice invitation opens the callee's incoming-call dialog, so ringing at session start can ring for a plain text IM.
- **The switch**, `[WebRtcVoice] P2PRing`:
  - `sessionstart` (default): ring when the session starts; the behaviour verified to ring on a call.
  - `provision`: ring when the caller provisions the one-to-one channel, so never for a text IM; it only works if the caller's viewer provisions before the callee accepts, which has to be confirmed in-world.
  - `none`: never send the invitation.
- **Status: unresolved.** Not tested with a viewer. The in-world procedure is in the `bin/OpenSim.ini` comment. See [[Known and Unresolved Problems]].
- **Tests.** `OpenSim/Tests/P2PVoiceCallsTests.cs` has 9 cases, among them: the callee is rung when the caller joins and not again; the callee answering does not ring the caller; a text IM that never starts voice never rings; a later call on the same session rings again after everyone left; re-calling while the other is still in does not ring; a caller who hangs up before the answer can ring again; leaving an unknown viewer session is harmless; removing an agent drops them from their calls; no peer or a self peer never rings.
- `OnRemovePresence` also clears `m_p2pVoicePeers`, which used to keep both directions of every call forever.

## 5. Fixes carried in `ChatSessionRequest`

Taken from Wolf's `c22b966` on `intelligentwolf/os-webrtc-janus`, which upstream does not have:

- `start p2p voice` without `params` is **refused** instead of being given `UUID.Random()`: a random id re-keys the viewer's floater away from the XOR id that inbound IMs arrive under, so replies land in another session.
- `start conference` fails fast with `no_ability_error` instead of a 30 s viewer timeout.
- `call`, `invite`, `mute update` and `session update` answer 200 instead of 400; `fetch history` returns an explicit empty array.
- Raff's `accept invitation` handling, which announces both parties, is kept instead of Wolf's plain 200.

## 6. Bringing up one region (from `TEST-PLAN-VOICE.md`)

The plan targets Debian 13 and one production region, in four phases: **A** the base without voice; **B** the OpenSim half, still without a backend; **C** the voice server on a LAN (recommended, optional); **D** production. With wolfvoice the guided install is the `setup.sh` of its repo (certbot, a `wolfvoice` user, a systemd unit). Name mapping between the plan and wolfvoice: binary `synthvoice` → `/usr/local/bin/wolfvoice`; `SYNTHVOICE_PUBLIC_IP` → `WOLFVOICE_PUBLIC_IP`; `/etc/synthvoice/tls/` → `/etc/wolfvoice/tls/` (`fullchain.pem`, `privkey.pem`).

Decisions the plan records: the setup is a single standalone process (`StandaloneHypergrid.ini`), so Robust is not involved and of the addon's four projects three are needed (`WebRtcVoiceServiceModule` is the grid-side piece); the database is MySQL.

## 7. When something is wrong

| Symptom | Where to look |
| :--- | :--- |
| no `[REGION WEBRTC VOICE]` log line | the module did not load: missing DLLs or `Enabled = false` |
| `VoiceServerType=webrtc` but no provision request | voice permissions, in the estate **and** About Land |
| provision request but a connector error | `WebRtcVoiceServerURI`, firewall, a certificate the region host does not trust |
| the viewer stays in `WAIT_FOR_DATA_CHANNEL` | the backend did not accept the data channel |
| `answer SDP carries NO ICE candidates` | fatal configuration error: `WOLFVOICE_PUBLIC_IP` (or `SYNTHVOICE_PUBLIC_IP`) wrong, or the media port range cannot be allocated |
| audio but no white dot on speakers | the roster travels on the data channel, not the media |
| voice fine close up, silent beyond some distance | expected: silent at 60 m, attenuated from 10 m |

## 8. What is not proven

- Nothing here was exercised with a viewer or a voice server in the documentation passes; the `P2PVoiceCallsTests` cover the bookkeeping only.
- The claims about Firestorm's source (`llvoicewebrtc.cpp`, `llimview.cpp`) come from the repo's own documents, not from reading Firestorm again.
- The wolfvoice repository and its protocol document are outside this repo; the contract above is as `CLAUDE.md` states it.
- Whether the addon and the internal module can be removed or merged is open: removing the internal module is left to the colleague who wrote it.
