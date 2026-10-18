# libremetaverse

> ```

## Concetti
[[C Sharp]]

## Panoramica
```
▗▖   ▄ ▗▖    ▄▄▄ ▗▞▀▚▖▗▖  ▗▖▗▞▀▚▖   ■  ▗▞▀▜▌▄   ▄ ▗▞▀▚▖ ▄▄▄ ▄▄▄ ▗▞▀▚▖
▐▌   ▄ ▐▌   █    ▐▛▀▀▘▐▛▚▞▜▌▐▛▀▀▘▗▄▟▙▄▖▝▚▄▟▌█   █ ▐▛▀▀▘█   ▀▄▄  ▐▛▀▀▘
▐▌   █ ▐▛▀▚▖█    ▝▚▄▄▖▐▌  ▐▌▝▚▄▄▖  ▐▌        ▀▄▀  ▝▚▄▄▖█   ▄▄▄▀ ▝▚▄▄▖
▐▙▄▄▖█ ▐▙▄▞▘          ▐▌  ▐▌       ▐▌                                
                                   ▐▌                                
```
# LibreMetaverse

LibreMetaverse is a fork of libOpenMetaverse which in turn was a fork of
libSecondLife, a library for developing Second Life-compatible virtual world
clients. LibreMetaverse returns the focus to up-to-date Second Life and OpenSim
compatibility with an eye to performance, multi-threading, and memory management.

The canonical source for LibreMetaverse can be found at:
https://github.com/cinderblocks/libremetaverse

## Quick Start

### New to LibreMetaverse?

Check out our example applications to learn the library:

- **[SimpleBot](Programs/examples/SimpleBot/)** - Build an interactive bot that responds to IMs
- **[PrimInspector](Programs/examples/PrimInspector/)** - Inspect 3D objects and their properties
- **[InventoryExplorer](Programs/examples/InventoryExplorer/)** - Browse and export inventory data

See all examples in [`Programs/examples/`](Programs/examples/) | [Examples README](Programs/examples/README.md)

### Need Tools?

- **[OSDInspector](Programs/tools/OSDInspector/)** - Inspect, validate, and convert OSD data files

See all tools in [`Programs/tools/`](Programs/tools/) | [Tools README](Programs/tools/README.md)

## Requirements

- .NET SDK 8.0 or 9.0 installed (recommended). Older .NET SDKs may build some projects but are not tested.
- `dotnet` CLI available on PATH. Verify with:

```
dotnet --info
```

- On Windows you can use Visual Studio (2022/2023) with .NET workloads installed. On Linux/macOS use the official .NET SDK installers.

## Recommended quick build (CLI)

From repository root:

1. Restore packages:

```
# Restore packages for the solution
# Example: dotnet restore LibreMetaverse.sln
# Or restore and build directly from the root directory:

dotnet restore
```

2. Build the solution (Release):

```
# Build the entire repo (builds discovered projects)
# Recommended: use 'dotnet build' which runs source generators automatically

dotnet build -c Release
```

This builds projects for their configured target frameworks (net8.0/net9.0/netstandard2.0, etc.). 
If you need to force a single framework for a specific project use `-f` on the `dotnet build` command for that project.

Notes:
- Some sample programs and tests target Windows-only frameworks (e.g. .NET Framework) 
  and will produce warnings or errors on non-Windows hosts. You can ignore those when building cross-platform.
- If you prefer MSBuild directly, be aware: some projects include custom source generators under the 
  `SourceGenerators` folder. When using MSBuild (or `dotnet msbuild`) you may need to build/run those 
  generator projects manually before building the consumer projects. Using `dotnet build` is recommended 
  because it will execute source generators as part of the normal build flow when configured correctly.

If you still want to run MSBuild directly:

```
# Build generator projects first (example):
# dotnet build SourceGenerators/PacketSourceGenerator/PacketSourceGenerator.csproj -c Release
# dotnet build SourceGenerators/VisualParamGenerator/VisualParamGenerator.csproj -c Release

# Then run msbuild on the solution (adjust solution filename as needed):
# dotnet msbuild LibreMetaverse.sln -t:Build -p:Configuration=Release
```

## Building specific projects

To build an individual project (example `TestClient`):

```
dotnet build Programs/examples/TestClient/TestClient.csproj -c Release
```

Outputs for each project are under that project's `bin/<Configuration>/<TargetFramework>/` directory 
(for example `LibreMetaverse/bin/Release/net8.0/`).

## Running examples

Change into the example project's output folder and run with `dotnet` (or execute the produced 
exec

## Tech Stack
- Linguaggi: C# (674)
- File totali (scan): 1147
- Estensioni principali: .cs=674, .animatn=118, .gesture=76, .tga=62, .clothing=49, .bodypart=48, .llm=29, .csproj=25
- Remote: `https://github.com/cinderblocks/libremetaverse.git`

## Struttura (top-level)
```
CONTRIBUTING.md
Directory.Build.props
Directory.Packages.props
LICENSE.txt
LibreMetaverse
LibreMetaverse.LslTools
LibreMetaverse.RLV
LibreMetaverse.Rendering.MeshFoundry
LibreMetaverse.Rendering.Meshmerizer
LibreMetaverse.Rendering.Simple
LibreMetaverse.Rendering.Tests
LibreMetaverse.StructuredData
LibreMetaverse.Tests
LibreMetaverse.Types
LibreMetaverse.Utilities
LibreMetaverse.Voice.Vivox
LibreMetaverse.Voice.WebRTC
LibreMetaverse.sln
LibreMetaverse.sln.DotSettings
PrimMesher
Programs
QUICK_REFERENCE.md
README.md
SourceGenerators
bin
data
global.json
util
```

## Documentazione presente nel repo
- `CONTRIBUTING.md`
- `Programs/examples/README.md`
- `Programs/tools/README.md`
- `QUICK_REFERENCE.md`
- `README.md`

## Manifest
### LibreMetaverse.sln
```
﻿
Microsoft Visual Studio Solution File, Format Version 12.00
# Visual Studio Version 18
VisualStudioVersion = 18.0.11222.15
MinimumVisualStudioVersion = 10.0.40219.1
Project("{9A19103F-16F7-4668-BE54-9A1E7A4F7556}") = "IRCGateway", "Programs\examples\IRCGateway\IRCGateway.csproj", "{89049BBC-0000-0000-0000-000000000000}"
EndProject
Project("{9A19103F-16F7-4668-BE54-9A1E7A4F7556}") = "LibreMetaverse", "LibreMetaverse\LibreMetaverse.csproj", "{27C70F3A-0000-0000-0000-000000000000}"
	ProjectSection(ProjectDependencies) = postProject
		{3577716B-6891-41CA-BF57-7DDE4E83745C} = {3577716B-6891-41CA-BF57-7DDE4E83745C}
	EndProjectSection
EndProject
Project("{9A19103F-16F7-4668-BE54-9A1E7A4F7556}") = "LibreMetaverse.Rendering.Meshmerizer", "LibreMetaverse.Rendering.Meshmerizer\LibreMetaverse.Rendering.Meshmerizer.csproj", "{95479B1D-0000-0000-0000-000000000000}"
	ProjectSection(ProjectDependencies) = postProject
		{2E2B643F-F18B-4791-BA4B-6E82D0E794B6} = {2E2B643F-F18B-4791-BA4B-6E82D0E794B6}
	EndProjectSection
EndProject
Project("{9A19103F-16F7-4668-BE54-9A1E7A4F7556}") = "LibreMetaverse.Rendering.Simple", "LibreMetaverse.Rendering.Simple\LibreMetaverse.Rendering.Simple.csproj", "{29E206AC-0000-0000-0000-000000000000}"
EndProject
Project("{9A19103F-16F7-4668-BE54-9A1E7A4F7556}") = "LibreMetaverse.StructuredData", "LibreMetaverse.StructuredData\LibreMetaverse.StructuredData.csproj", "{89D7A3E5-0000-0000-0000-000000000000}"
EndProject
Project("{9A19103F-16F7-4668-BE54-9A1E7A4F7556}") =
```

## Progetti correlati
- [[Gen]]
- [[cck]]
- [[craftsim]]
- [[craftstorm]]
- [[libopenmetaverse]]
- [[mcp-unity]]
- [[nexus]]
- [[opensim-baseline]]
- [[opensim-cpp]]

## Link
- Repository: https://github.com/cinderblocks/libremetaverse.git
- Path locale: `/home/pakkio/w/libremetaverse`
