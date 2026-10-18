# mcp-unity

> [![](https://badge.mcpx.dev?status=on 'MCP Enabled')](https://modelcontextprotocol.io/introduction)

## Concetti
[[C Sharp]] [[JavaScript]] [[MCP]] [[TypeScript]] [[Unity]]

## Panoramica
# MCP Unity Editor (Game Engine)

[![](https://badge.mcpx.dev?status=on 'MCP Enabled')](https://modelcontextprotocol.io/introduction)
[![](https://img.shields.io/badge/Unity-000000?style=flat&logo=unity&logoColor=white 'Unity')](https://unity.com/releases/editor/archive)
[![](https://img.shields.io/badge/Node.js-339933?style=flat&logo=nodedotjs&logoColor=white 'Node.js')](https://nodejs.org/en/download/)
[![](https://img.shields.io/github/stars/pakkio/mcp-unity 'Stars')](https://github.com/pakkio/mcp-unity/stargazers)
[![](https://img.shields.io/github/last-commit/pakkio/mcp-unity 'Last Commit')](https://github.com/pakkio/mcp-unity/commits/main)
[![](https://img.shields.io/badge/License-MIT-red.svg 'MIT License')](https://opensource.org/licenses/MIT)

| [🇺🇸English](README.md) | [🇨🇳简体中文](README_zh-CN.md) | [🇯🇵日本語](README-ja.md) |
|----------------------|---------------------------------|----------------------|

```        
                              ,/(/.   *(/,                                  
                          */(((((/.   *((((((*.                             
                     .*((((((((((/.   *((((((((((/.                         
                 ./((((((((((((((/    *((((((((((((((/,                     

             ,/(((((((((((((/*.           */(((((((((((((/*.                
            ,%%#((/((((((*                    ,/(((((/(#&@@(                
            ,%%##%%##((((((/*.             ,/((((/(#&@@@@@@(                
            ,%%######%%##((/(((/*.    .*/(((//(%@@@@@@@@@@@(                
            ,%%####%#(%%#%%##((/((((((((//#&@@@@@@&@@@@@@@@(                
            ,%%####%(    /#%#%%%##(//(#@@@@@@@%,   #@@@@@@@(                
            ,%%####%(        *#%###%@@@@@@(        #@@@@@@@(                
            ,%%####%(           #%#%@@@@,          #@@@@@@@(                
            ,%%##%%%(           #%#%@@@@,          #@@@@@@@(                
            ,%%%#*              #%#%@@@@,             *%@@@(                
            .,      ,/##*.      #%#%@@@@,     ./&@#*      *`                
                ,/#%#####%%#/,  #%#%@@@@, ,/&@@@@@@@@@&\.                    
                 `*#########%%%%###%@@@@@@@@@@@@@@@@@@&*´                   
                    `*%%###########%@@@@@@@@@@@@@@&*´                        
                        `*%%%######%@@@@@@@@@@&*´                            
                            `*#%%##%@@@@@&*´                                 
                               `*%#%@&*´                                     
                                                       
     ███╗   ███╗ ██████╗██████╗         ██╗   ██╗███╗   ██╗██╗████████╗██╗   ██╗
     ████╗ ████║██╔════╝██╔══██╗        ██║   ██║████╗  ██║██║╚══██╔══╝╚██╗ ██╔╝
     ██╔████╔██║██║     ██████╔╝        ██║   ██║██╔██╗ ██║██║   ██║    ╚████╔╝ 
     ██║╚██╔╝██║██║     ██╔═══╝         ██║   ██║██║╚██╗██║██║   ██║     ╚██╔╝  
     ██║ ╚═╝ ██║╚██████╗██║             ╚██████╔╝██║ ╚████║██║   ██║      ██║   
     ╚═╝     ╚═╝ ╚═════╝╚═╝              ╚═════╝ ╚═╝  ╚═══╝╚═╝   ╚═╝      ╚═╝   
```       

MCP Unity is an implementation of the Model Context Protocol for Unity Editor, allowing AI assistants to interact with your Unity projects. This package provides a bridge between Unity and a Node.js server that implements the MCP protocol, enabling AI agents like Cursor, Windsurf, Claude Code, Codex CLI, GitHub Copilot, Google Antigravity, and OpenCode to execute operations within the Unity Editor.

## Features

### IDE Integration - Package Cache Access

MCP Unity provides automatic integration with VSCode-like IDEs (Visual Studio Code, Cursor, Windsurf, Google Antigravity) by adding the Unity `Library/PackedCache` folder to your workspace. This feature:

- Improves code intelligence for Unity packages
- Enables better autocompletion and type information for Unity packages
- Helps AI coding assistants understand your project's dependencies

### MCP Server Tools

The followi

## Tech Stack
- Linguaggi: TypeScript (175), C# (90), JavaScript (74), LaTeX (3)
- File totali (scan): 511
- Estensioni principali: .ts=175, .meta=129, .cs=90, .js=74, .md=10, .html=8, .json=6, (noext)=4
- Remote: `git@github.com:pakkio/mcp-unity.git`

## Struttura (top-level)
```
AGENTS.md
AGENTS.md.meta
CLAUDE.md
CLAUDE.md.meta
Editor
Editor.meta
LICENSE.md
LICENSE.md.meta
README-ja.md
README-ja.md.meta
README.md
README.md.meta
README_zh-CN.md
README_zh-CN.md.meta
Server~
docs
docs.meta
glama.json
glama.json.meta
llms.txt
llms.txt.meta
opencode.json
opencode.json.meta
package.json
package.json.meta
```

## Documentazione presente nel repo
- `AGENTS.md`
- `CLAUDE.md`
- `LICENSE.md`
- `README-ja.md`
- `README.md`
- `README_zh-CN.md`
- `docs/articolo-mcp-unity-pakkio.md`
- `docs/arxiv-mcp-unity-pakkio.md`
- `docs/chilloutvr-cck-integration.md`

## Manifest
### package.json
```
{
  "name": "com.pakkio.mcp-cck",
  "displayName": "MCP Unity Server",
  "author": "pakkio",
  "version": "1.7.1",
  "mcpname": "io.github.pakkio/mcp-cck",
  "unity": "2022.3",
  "license": "MIT",
  "description": "The purpose of this package is to provide a MCP Unity Server for executing Unity operations and request Editor information from AI MCP enabled hosts",
  "dependencies": {
    "com.unity.nuget.newtonsoft-json": "3.2.1",
    "com.unity.editorcoroutines": "1.0.0",
    "com.unity.test-framework": "1.3.3"
  },
  "type": "library",
  "hideInEditor": false
}
```

## Progetti correlati
- [[Gen]]
- [[cck]]
- [[craftsim]]
- [[libopenmetaverse]]
- [[libremetaverse]]
- [[llmwiki]]
- [[llmwiki1]]
- [[mcp-blender]]
- [[nispa-ocr]]
- [[opensim-baseline]]
- [[opensim-cpp]]

## Link
- Repository: git@github.com:pakkio/mcp-unity.git
- Path locale: `/home/pakkio/w/mcp-unity`
