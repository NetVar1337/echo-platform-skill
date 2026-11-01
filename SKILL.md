---
name: echo-platform
description: Build scripts and tools for the Echo Platform "Angel" Universal API — AngelScript (.as) scripts that run inside the Discord overlay over any process, with direct (physical) memory access, a full overlay renderer, a menu UI system, HTTP/WebSocket/process-socket networking, Zydis + Unicorn bindings, SIMD intrinsics, BVH tracing, and Source/Unreal/Unity/IW engine helpers. Use this skill whenever the user mentions Echo Platform, Angel, the Universal API, .as scripts, the Discord overlay, process memory tooling, signature scanning, or asks to write a script/tool for a game or process — even vaguely ("that overlay thing"). When in doubt about an Echo API question, load this skill.
---

# Echo Platform (Angel Universal API) — Playbook

The Universal API lets you build tools for any process — full overlay renderer + direct memory access — as plain `.as` AngelScript files. No project compilation: drop the file in, reload the menu, run it.

Official docs: https://echo-23.gitbook.io/angel (raw dump bundled at `references/api-full.txt`).

## 1. Environment setup checklist

- Discord installed and **running**, overlay **enabled**, **legacy overlay disabled**.
- The target must be in **borderless or windowed** mode (fullscreen exclusive is not supported).
- Scripts live in `%APPDATA%\scripts\*.as` (e.g. `C:\Users\<you>\AppData\Roaming\scripts`). After adding a file, **close and reopen the menu** to refresh the script list.
- Memory dumps go to a `dmp` folder inside the scripts folder.
- Language is [AngelScript](https://www.angelcode.com/angelscript/) — C-style, statically typed, reference-counted handles (`@`), arrays `array<T>`, string concat with `+`.

## 2. Script skeleton & lifecycle

```cpp
void on_unload() {
    // optional teardown: save data, release resources, undo changes to the target
}

bool main() {
    // entry point — script won't compile without it
    // return false to signal failure (unloads the script); true = success
    return true;
}
```

Globals, handles and arrays are cleaned up automatically on unload — only undo *external* effects in `on_unload()`.

## 3. Golden rules (the gotchas that actually bite)

1. **Attach first.** Every memory/sig/socket call needs `process::attach(...)`. Check `.pid == 0` for failure. Call `process::detach()` in `on_unload()`.
2. **Direct physical memory.** Page protections and copy-on-write are bypassed; the target is never injected into. Bad addresses won't just return 0 — validate addresses (e.g. via `process::virtual_query`) before writing.
3. **Prefer `read_buffer(address, size)`** for reading several fields from one region (one read, then `read_uint32(offset)` etc.), and the `bool read_x(addr, out&)` overloads when you need the success flag. The value-returning overloads silently return defaults on failure.
4. **`read_string` is null-terminated, 256 chars max** — use `read_bytes` for wide strings, then `util::wide_to_string`.
5. **Resolve dynamic addresses properly:** sig-scan the module (`get_module_base` → `find_signature`), then `get_relative_address(insn_addr, offset, inst_size)` for RIP-relative operands. Never hardcode addresses — ASLR.
6. **Threading:** ticking threads (`create_thread(tps, cb)`, tps 1–128) can **only be created from `main()`**; one-shot threads (tps 0) from anywhere. `on_render(cb)` runs on the render path.
7. **Never share unsynchronized globals between threads** — a data race can take down the whole process. Use `mutex` (lock/try_lock/lock_shared), atomics (`atomic_int32` etc., use `compare_exchange` for read-modify-write), or the snapshot-publish pattern (see `references/threading-data.md`).
8. **Render calls only from the render context**, unless marked "callable from any context" (`render::get_fps`, `is_foreground`, `get_viewport_size`, `load_font`, `load_image`, `set_anti_aliasing`). Same for `render::update_font` / `draw_*`.
9. **`http::get/post` with a callback are only callable from `main()`**; the fire-and-forget overloads (no callback) work from any context incl. `on_unload()`. UA `prism/1.0`, 10s timeout, redirects followed.
10. **`ws::` callbacks fire on the connection thread** — guard shared state. `process::open_socket` allows **one** connection at a time and requires an attached process.
11. **File I/O is sandboxed** to the data directory: max 512 MB, depth 2, executable extensions blocked, dirs one level deep.
12. **UI build order matters:** `ui::add_tab` → `ui::add_subtab` → `ui::add_category` → elements. Elements attach to the most recent category; categories to the most recent tab/sub-tab. Call `ui::config_path("unique_name")` once to get a save/load config tab.

## 4. Recipe index (task → reference)

| Task | Go to |
|---|---|
| Attach, read/write memory, modules, sig scan, dumps, virtual_query | `references/process-memory.md` |
| Process WebSocket (JSON routes, error codes) | `references/process-memory.md` |
| Overlay drawing (text, boxes, lines, images, 3D→2D bbox) | `references/render-ui.md` |
| Menu UI (tabs, categories, checkboxes, sliders, callbacks, config) | `references/render-ui.md` |
| Alerts / console logging | `references/render-ui.md` |
| Threads, mutexes, atomics, snapshot sharing, hash set/map | `references/threading-data.md` |
| HTTP, WebSocket client, JSON, filesystem, util/format, hashing | `references/network-data.md` |
| Zydis disassembly, Unicorn emulation, SIMD/bit ops | `references/disasm-emulation.md` |
| Source/Source2/Unreal/Unity/IW helpers, math, vectors, matrices, BVH, clipping | `references/engines-math.md` |
| Anything not covered above — full verbatim API | `references/api-full.txt` |

## 5. Canonical script shapes

**Memory reader tool:**
```cpp
bool main() {
    attach_data a = process::attach("target.exe");
    if (a.pid == 0) { alert::show("attach failed", 255, 80, 80); return false; }
    uint64 mod = process::get_module_base("target.exe");
    uint64 addr = process::find_signature(mod, "48 83 EC 28 FF 15 ? ? ? ?", mod, process::get_module_size(mod));
    uint64 field = process::get_relative_address(addr, 3, 7);
    create_thread(30, function() {
        float v; if (process::read_float(field + 0x10, v)) print(v);
    });
    return true;
}
```

**Overlay loop:** `create_thread` for logic, `on_render(@cb)` for drawing — read state in the tick thread, publish via snapshot, draw in the render thread (never draw from a tick thread).

**Menu-driven tool:** build UI in `main()` (tabs → categories → elements), read element values each tick with `ui::get(...)`, react via callbacks.

## 6. Working agreements

- Target only processes you own or are authorized to analyze; don't use the platform to gain an unfair advantage in multiplayer games.
- When writing scripts for a user, prefer sig scans + relative resolution over hardcoded offsets, and validate every read with the `bool` overloads.
- Keep SKILL.md small: if you need the full signature list, read the matching reference file rather than the whole raw dump.

## Companion resources (bundled — see BUNDLE.md)

- Binary RE / decompilation / AoB scanning of the attached target → run [mcp-reverse](https://github.com/1llu50ry/mcp-reverse) as an MCP server (MIT, 180+ tools).
- Anti-cheat / kernel / engine research background → install skills from [awesome-game-security](https://github.com/gmh5225/awesome-game-security) via `npx skills add`.
- Windows driver attack-surface research → [BYOVD-AI-Guide](https://github.com/gmh5225/BYOVD-AI-Guide) (link only; authorized testing only).

These are linked, not vendored (multi-GB repos). Use everything here only on software you own or are authorized to analyze.
