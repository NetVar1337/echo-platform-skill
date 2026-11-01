# echo-platform skillpack

A skill/playbook for building scripts on the **Echo Platform "Angel" Universal API** — AngelScript (`.as`) scripts that run inside the Discord overlay over any process, with direct memory access, a full overlay renderer, menu UI, HTTP/WebSocket networking, Zydis + Unicorn bindings, and Source/Unreal/Unity/IW engine helpers.

Built from the official public docs: https://echo-23.gitbook.io/angel (raw dump included at `references/api-full.txt`).

This repo **is** the skill — `SKILL.md` is at the root, so GitHub renders it below.

## Install

Clone straight into your agent's skills directory:

```bash
# pi / claude-style agents
git clone https://github.com/NetVar1337/echo-platform-skill.git ~/.agents/skills/echo-platform
```

Or copy `SKILL.md` + `references/` into an existing `echo-platform` skill folder. The skill triggers on mentions of Echo Platform, Angel, the Universal API, `.as` scripts, the Discord overlay, process memory tooling, or signature scanning.

## Contents

```
SKILL.md                       ← playbook: setup checklist, script skeleton, golden rules, recipe index
references/
├── api-full.txt               ← verbatim full API docs (llms-full.txt)
├── process-memory.md          ← attach, read/write, sig scan, dumps, process socket JSON routes
├── render-ui.md               ← overlay rendering + menu UI + logging
├── threading-data.md          ← threads, mutexes, atomics, snapshot pattern, hash containers
├── network-data.md            ← http/ws, JSON, filesystem, util, input
├── disasm-emulation.md        ← Zydis, Unicorn, SIMD/bit ops
└── engines-math.md            ← engine w2s helpers, math, vectors, matrices, BVH, clipping
```

## Requirements for using the platform itself

- Discord running with the overlay enabled (legacy overlay off); target in borderless/windowed mode.
- Scripts in `%APPDATA%\scripts\*.as`; close/reopen the menu to refresh the list.

## Responsible use

Target only processes you own or are authorized to analyze. Do not use the platform to gain an unfair advantage in multiplayer games or to bypass anti-cheat in ways that harm other players.

## Bundled companion resources

See **[BUNDLE.md](BUNDLE.md)** — this skillpack pairs with three public repos (linked, not vendored, since each is multi-GB):

- **[1llu50ry/mcp-reverse](https://github.com/1llu50ry/mcp-reverse)** (MIT) — MCP server for game RE: 180+ tools, 56 debug/decompiler backends, 4,200+ skills. Run it alongside your agent to analyze the binary your script attaches to.
- **[gmh5225/awesome-game-security](https://github.com/gmh5225/awesome-game-security)** (MIT) — game-security resource list + installable agent skills (`npx skills add ... --skill reverse-engineering-tools` etc.).
- **[gmh5225/BYOVD-AI-Guide](https://github.com/gmh5225/BYOVD-AI-Guide)** — Windows vulnerable-driver research guide. Reference link only; authorized testing only.
