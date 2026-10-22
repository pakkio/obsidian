# opensim-cpp

> A from-scratch C++20 reimplementation of OpenSimulator, built in dependency

## Concetti
[[C++]] [[OpenSim]] [[Python]] [[TypeScript]] [[Virtual Worlds]]

## Panoramica
# opensim-cpp

A from-scratch C++20 reimplementation of OpenSimulator, built in dependency
order so there is always something that runs. **Phase 0** is a wire-accurate
LLUDP codec, the XML-RPC login service, and the region-entry handshake for a
single empty region — i.e. the whole path from "viewer clicks Log In" to
"avatar placed in the region".

The protocol details are taken directly from the viewer's own message template
(`message_template.msg`) and from OpenSim's `OpenSim/Region/ClientStack/Linden/UDP`.

## Build & test

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
ctest --test-dir build --output-on-failure   # or: ./build/opensim-cpp selftest
./build/opensim-cpp serve 9000               # login (HTTP) + region (UDP)
python3 scripts/integration_test.py 9000     # against a running server
```

No third-party dependencies — just a C++20 compiler, POSIX sockets, and
(optionally) Python 3 for the integration test.

## What Phase 0 does

A viewer logging in reaches an avatar-in-region state through these stages, all
implemented here:

**Login (HTTP / XML-RPC), `src/login`**

| Endpoint | Purpose |
|----------|---------|
| `POST /` `login_to_simulator` | returns agent/session IDs, circuit code, sim IP/port, seed cap, inventory skeleton |
| `POST /CAPS/<id>/` | seed-capability stub (empty LLSD map) |

The issued circuit code is stored in a `SessionRegistry`; when `UseCircuitCode`
arrives over UDP it is validated against it (logs `[VALIDATED]`).

**Region (UDP / LLUDP), `src/sim`** — the login handshake state machine:

| Direction | Message | Status |
|-----------|---------|--------|
| viewer → region | `UseCircuitCode` | parsed, circuit established |
| region → viewer | `RegionHandshake` | sent (reliable) with region info |
| viewer → region | `RegionHandshakeReply` | accepted |
| viewer → region | `CompleteAgentMovement` | accepted |
| region → viewer | `AgentMovementComplete` | sent, avatar placed |
| viewer ↔ region | `StartPingCheck` / `CompletePingCheck` | answered |
| both | `PacketAck` (reliable acking) | sent for inbound reliable packets |

The codec handles the full packet header (big-endian sequence, the
High/Medium/Low/Fixed message-number encodings), **zero-coding** (RLE on `0x00`),
and **appended acks**. Message bodies are little-endian, UUIDs are 16 raw bytes,
`Variable 1`/`Variable 2` length prefixes are implemented. All of this is
exercised by `selftest`.

## Layout

```
include/oscpp/core/    Uuid, Vector3, ByteBuffer (LE reader/writer)
include/oscpp/lludp/   Message: flags, frequencies, message numbers, decode/encode
include/oscpp/net/     UdpSocket: POSIX UDP wrapper + Endpoint
include/oscpp/http/    HttpServer: minimal HTTP/1.1 over TCP
include/oscpp/login/   LoginService: XML-RPC login + caps stub
include/oscpp/sim/     RegionServer (handshake) + SessionRegistry
src/                   implementations + main (serve | selftest)
scripts/               integration_test.py (login + UDP, end to end)
```

## Pointing a viewer at it

Run `./build/opensim-cpp serve 9000`, add a grid in the viewer's grid manager
with login URI `http://<host>:9000/`, and log in. The viewer will complete login
and open the UDP circuit (watch the server log for `[VALIDATED]`). It will *not*
fully render the world yet — that needs capabilities and scene data (below).

## Not yet implemented (the road ahead)

Phase 0 proves the full **login → circuit → in-region handshake** path. Still
missing before a viewer draws the world:

1. **Capabilities (HTTP/LLSD)** — real caps from the seed endpoint
   (`EventQueueGet`, `GetTexture`, `GetMesh`, inventory/appearance caps), which
   means an LLSD XML/binary (de)serializer.
2. **Reliable transport** — retransmission of unacked outbound reliable packets,
   throttles/token buckets, `PacketPool`-style buffer reuse.
3. **Scene bring-up** — `LayerData` (terrain), `ObjectUpdate`, `CoarseLocationUpdate`,
   `AvatarAppearance`, so the world and the avatar are actually drawn.

## Tech Stack
- Linguaggi: C++ (15), Python (1), TypeScript (1)
- File totali (scan): 46
- Estensioni principali: .hpp=10, .cmake=8, .txt=5, .cpp=5, .make=5, (noext)=3, .out=1, .md=1

## Struttura (top-level)
```
CMakeLists.txt
README.md
a.out
build
include
scripts
src
```

## Documentazione presente nel repo
- `README.md`

## Progetti correlati
- [[MegaFuse]]
- [[cck]]
- [[craftsim]]
- [[craftstorm]]
- [[libopenmetaverse]]
- [[libremetaverse]]
- [[llmwiki]]
- [[llmwiki1]]
- [[local_llm]]
- [[mcp-unity]]
- [[nexus]]
- [[nispa-ocr]]

## Link
- Path locale: `/home/pakkio/w/opensim-cpp`
