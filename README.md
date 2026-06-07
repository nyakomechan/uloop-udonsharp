# uloop-udonsharp

UdonSharp operation skill for [unity-cli-loop (uloop)](https://github.com/nyakomechan/unity-cli-loop).

## Overview

Provides 15 categories of UdonSharp operations that can be executed from uloop's `execute-dynamic-code` skill:

- Attach / Detach UdonSharpBehaviour
- Read / Write fields (including synced variables)
- Proxy ↔ Backing synchronization (`CopyProxyToUdon`, `CopyUdonToProxy`)
- Compile UdonSharp programs
- Create UdonSharpProgramAsset
- PlayMode operations
- Scene validation and more

## Requirements

- Unity with VRChat Worlds SDK (com.vrchat.worlds >= 3.7.1)
- UdonSharp integration
- uloop with `execute-dynamic-code` skill

## Installation

Place this directory under your project's `.agents/skills/uloop-udonsharp/`.

## License

MIT License - Copyright (c) 2026 nyakomechan