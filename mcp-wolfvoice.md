# mcp-wolfvoice

> Voce WebRTC spaziale per OpenSimulator, in Rust.

## Concetti
[[OpenSim]] [[Virtual Worlds]] [[MCP]]

## Panoramica
**wolfvoice** è un backend voce per l'addon `os-webrtc-janus` di OpenSimulator: Firestorm 7.1.10+ e viewer browser si raggiungono tramite le capability della regione e si sentono tra loro in modo posizionale. wolfvoice risponde all'SDP, termina DTLS/SRTP/SCTP e mixa uno stream **separato per ogni ascoltatore**. L'addon (requisito obbligatorio) inoltra le cap via JSON-RPC; non serve un gateway Janus.

```
Viewer -> OpenSim + os-webrtc-janus -> wolfvoice (mix per ascoltatore) -> UDP media
```

## Tech Stack
- Linguaggio: Rust (`Cargo.toml`, `src/`), con `client/`, `examples/`, `docs/`, `setup.sh`
- Richiede host Linux con IP pubblico, DNS e TLS (il DTLS fingerprint è nell'SDP)
- Repo: https://github.com/pakkio/mcp-wolfvoice

## Collegamenti
- Lato server in [[craftsim]]: [[Voice (WebRTC, Janus & SynthVoice)]]
- Server: skill `pkc` (wolf/wolfvoice su edu3d.it)
