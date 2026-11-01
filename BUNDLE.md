# Bundled companion resources

This skillpack is designed to work alongside these three public repos. They are **linked, not vendored** — each working tree is multi-GB, so clone or install them separately.

| Repo | What it is | License | How to use with this skillpack |
|---|---|---|---|
| [1llu50ry/mcp-reverse](https://github.com/1llu50ry/mcp-reverse) | MCP server for game reverse engineering: 180+ MCP tools, 56 RE/debug backends (Ghidra, IDA, Binja, x64dbg, WinDbg, Frida, Cheat Engine, ...), 4,200+ curated skills, multi-agent mesh | MIT | Run it as an MCP server next to your agent (stdio: `node dist/index.js` in the client's MCP config). Use it to analyze/decompile/scan the same binary your script attaches to with `process::attach` |
| [gmh5225/awesome-game-security](https://github.com/gmh5225/awesome-game-security) | The awesome-game-security resource list, plus installable AI-agent skills (anti-cheat, reverse-engineering tools, Windows kernel, DMA, game engines, ...) | MIT | `npx skills add https://github.com/gmh5225/awesome-game-security --skill <skill-name>` — e.g. `reverse-engineering-tools`, `anti-cheat-systems`, `windows-kernel-security` |
| [gmh5225/BYOVD-AI-Guide](https://github.com/gmh5225/BYOVD-AI-Guide) | Guide to researching vulnerable Windows drivers (collection playbook, static-audit reports, blocklist checks, VM-verified write primitives) | none (all rights reserved) | Reference reading for kernel/driver research — **linked only, not copied** into this repo |

## Why linked, not vendored

- mcp-reverse: ~3.3 GiB working tree · awesome-game-security: ~3.3 GiB · BYOVD-AI-Guide: ~107 MB. Vendoring would make this repo unusable.
- BYOVD-AI-Guide has no license, so its content must not be re-hosted here regardless of size.

## Responsible use

All four resources (this skillpack included) are for analyzing software you own or are explicitly authorized to test. Do not use them to gain an unfair advantage in multiplayer games, grief other players, or bypass protections that harm others. Each upstream repo has its own ethics/responsible-disclosure docs (e.g. mcp-reverse's `SECURITY.md`) — read them before use.
