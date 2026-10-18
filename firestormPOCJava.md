# firestormPOCJava

> Proof of concept: viewer Second Life / OpenSim in Java.

## Concetti
[[Virtual Worlds]] [[OpenSim]]

## Panoramica
Viewer didattico in Java ispirato a Firestorm, con struttura che rispecchia il codice C++ ufficiale.
- Login a grid SL (Agni/Aditi) e OpenSim via XML-RPC
- Handshake UDP completo (`UseCircuitCode`, `CompleteAgentMovement`, `AgentThrottle`, `RegionHandshake`)
- Streaming `ObjectUpdate`/`ObjectUpdateCompressed`
- Movimento WASD, rendering 3D LWJGL/OpenGL 3.3 (terreno, oggetti, LOD, frustum culling), UI Swing con mappa 2D
- Keep-alive con ping e `AgentUpdate`

Pacchetti: `agent`, `core` (LLVector3, LLQuaternion, LLUUID, LLColor4), `login`, `network` (LLCircuit, LLByteReader/Writer).

## Tech Stack
Java 17, Maven, LWJGL. Repo: https://github.com/pakkio/firestormPOCJava

## Collegamenti
[[phoenix-firestorm]], [[opensim-cpp]], [[libopenmetaverse]], [[craftstorm]]
