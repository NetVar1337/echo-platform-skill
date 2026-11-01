# Process & Memory reference

Direct physical memory access to an attached process. Page protections and copy-on-write have no effect; the target process is never injected into.

## Attach / detach

```cpp
struct attach_data { uint64 pid; uint64 peb_address; uint64 base_address; }
struct module_entry { string name; uint64 base; uint64 size; }
struct memory_region { uint64 base; uint64 size; bool present; bool writable; bool executable; }

attach_data process::attach(string name);   // check .pid == 0 on failure
attach_data process::attach(uint32 pid);
void process::detach();
bool process::is_alive();
```

## Reading

```cpp
// Bulk read into a buffer — best for multi-field struct access (one read total)
memory_buffer@ process::read_buffer(uint64 address, uint64 size); // null on failure
// memory_buffer::read_uint8/16/32/64, read_int8/16/32/64, read_float, read_double,
//   read_vec2/3/4_float, read_vec2/3/4_double(uint64 offset), size()

// Value-returning reads — return default (0, 0.0, false) on failure, no error signal
array<uint8>@ process::read_bytes(uint64 address, uint32 size);
int8/uint8/int16/uint16/int32/uint32/int64/uint64/float/double process::read_*(uint64 address);
string process::read_string(uint64 address); // null-terminated, 256 chars max; use read_bytes for wide strings

// Bool reads — return true on success (prefer these when a bad address matters)
bool process::read_*(uint64 address, T &out); // same type list + read_string(addr, string &out)
```

## Writing

```cpp
bool process::write_bytes(array<uint8> bytes, uint64 address);
bool process::write_int8/uint8/int16/uint16/int32/uint32/int64/uint64/float/double(uint64 address, T value);
```

## Region / module utilities

```cpp
// Contiguous region with uniform flags containing address. If present==false, base/size
// describe the unmapped hole — advance with base+size to enumerate all regions incl. gaps.
memory_region process::virtual_query(uint64 address);

uint64 process::get_module_base(string name);                 // 0 if not found
uint32 process::get_module_size(uint64 base);                 // 0 if not found
uint64 process::get_module_export(uint64 base, string name);  // 0 if not found
array<module_entry>@ process::get_module_list();
```

## Pattern scanning

Two sig formats: `sig` = hex bytes, `mask` uses `x` (match) and `?` (wildcard) — e.g. `find_signature(base, "4883ec28ff15", "xxxxxx")`. IDA style takes one string: `"48 83 EC 28 FF 15 ? ? ? ? 85 C0"`. Both return 0 if not found; `find_signatures` returns all matches. Optional `range_offset`/`range_length` narrow the scan window.

```cpp
uint64 process::find_signature(uint64 base, string sig, string mask);
uint64 process::find_signature(uint64 base, string sig, string mask, uint64 range_offset, uint64 range_length);
array<uint64>@ process::find_signatures(uint64 base, string sig, string mask[, uint64 range_offset, uint64 range_length]);
// IDA-style overloads: find_signature(uint64 base, string sig[, range_offset, range_length]) etc.

// Resolve a relative (RIP-relative) address from an instruction
uint64 process::get_relative_address(uint64 address, uint32 offset, uint32 inst_size);
```

## Dumping

```cpp
bool process::dump(string filename);                  // main executable
bool process::dump(string module_name, string filename); // specific module
// saved to the "dmp" folder inside the scripts folder
```

## Process socket (JSON over WebSocket)

```cpp
// Requires an attached process. Only ONE connection at a time; false if busy/not attached.
bool process::open_socket(const string url);
// Every request: {"type": ..., "id": optional int (default 0)} — id echoed in all responses.
```

Routes:

| type | request fields | response |
|---|---|---|
| `ping` | — | `ping_result`: attached, pid, base (hex string; pid/base omitted when not attached) |
| `read` | address (hex str), size | `read_result`: success, address, size, data (lowercase hex; on failure size=0, data="") |
| `read_batch` | reads: [{address, size}...] (per-entry ≤ 0x10000) | `read_batch_result`: results[] in order, outer success always true if processed |
| `modules` | — | `modules_result`: modules[] {name, base, size} |
| `sig_scan` | sig, mask, module (default main exe), find_all | `sig_scan_result`: results[] hex addrs; success = ≥1 found |
| `sig_scan_ida` | pattern ("48 83 EC .. ? ?"), module, find_all | `sig_scan_ida_result` |
| `string_scan` | text, module, find_all | `string_scan_result`: results[] |
| `resolve_relative` | address, offset=0, inst_size=0 | `resolve_relative_result`: address |
| `rtti_resolve` | address | `rtti_resolve_result`: name e.g. "class PlayerController" |
| `rtti_resolve_batch` | addresses[] | `rtti_resolve_batch_result`: results[] same order |
| `enumerate_functions` | module | `enumerate_functions_result`: count, results[] {address, size, method} |
| `disassemble` | address, size=0 (0 = auto extent) | `disassemble_result`: complete, count, results[] {address, length, bytes, text} |

Error frame (replaces normal result on failure): `{"type":"error","id":..,"code":..,"message":..,"detail":..}`
Codes: 1000 unknown type · 1001 not attached · 1002 invalid/missing args · 1003 module not found · 1004 size out of range (1..1048576) · 1005 read failed · 1006 function enumeration failed · 1007 disassembly failed.

## Typical flow

1. `attach` → check pid
2. `get_module_base("game.exe")` → base
3. `find_signature(base, sig, mask)` → instruction address (0 = not found, bail out)
4. `get_relative_address(insn, offset, inst_size)` → data pointer
5. `read_buffer(ptr, struct_size)` → typed field reads by offset
