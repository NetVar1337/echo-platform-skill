# Zydis disassembly, Unicorn emulation, SIMD/bit ops reference

## Zydis (x86-64 disassembler)

Decoded instructions are ref-counted `disasm_insn@` handles; operands are value-type `disasm_operand` structs.

```cpp
// operand type: op_type::unused|reg|mem|ptr|imm
// visibility:   op_visibility::invalid|explicit|implicit|hidden
// action flags: op_action::read|write|readwrite|condread|condwrite|read_condwrite|condread_write (bitmask)

struct disasm_operand {
    int type; int visibility; int action;
    int reg;                 // when type == reg; pass to zydis::reg_name()
    bool imm_is_signed; bool imm_is_relative; uint64 imm_value;   // when type == imm
    int mem_segment; int mem_base; int mem_index; uint8 mem_scale; int64 mem_disp; bool mem_has_disp; // when type == mem
}

struct disasm_insn {
    uint64 runtime_address; // base addr; rip-relative operands resolve against this
    uint8 length; int mnemonic; string text;      // Intel syntax
    uint8 operand_count; uint8 operand_count_visible;
    disasm_operand get_op(uint8 idx) const;       // default operand if out of range
}

string zydis::reg_name(int reg);
bool zydis::disasm(const array<uint8>@ &in bytes, array<disasm_insn@> &out insns);
bool zydis::disasm(const array<uint8>@ &in bytes, uint64 rip, array<disasm_insn@> &out insns);
bool zydis::disasm_func(const array<uint8>@ &in bytes, array<disasm_insn@> &out insns); // until first ret/iret
bool zydis::disasm_func(const array<uint8>@ &in bytes, uint64 rip, array<disasm_insn@> &out insns);
// disasm_func returns false if the 4096-instruction limit is hit without a return
```

Typical flow: attach → `get_module_base("ntdll.dll")` → `get_module_export(base, "NtCreateFile")` → `read_bytes(target, 256)` → `zydis::disasm_func(bytes, target, insns)` → iterate `insn.text` / `get_op(i)`.

## Unicorn (x86-64 CPU emulator)

Handles are opaque `uint64` (0 = creation failed). `uc::close(h)` releases mappings+hooks; leftover handles are auto-cleaned on script unload. Always x86_64.

```cpp
uc::prot::none|read|write|exec|all                     // OR-combinable
uc::mem_type::read|write|fetch|read_unmapped|write_unmapped|fetch_unmapped|read_prot|write_prot|fetch_prot
uc::error::ok|nomem|arch|handle|mode|version|read_unmapped|write_unmapped|fetch_unmapped|fetch_prot|read_prot|write_prot|map|hook|insn_invalid

struct uc_memory_region { uint64 address; uint64 end; uint32 perms; };

uint64 uc::create();
uint64 uc::create_process(); // backed by the ATTACHED process: pages faulted in on demand from the
                             // process virtual memory layout at region granularity with correct Win32
                             // protections; write faults to unmapped regions are NOT auto-mapped and stop
                             // emulation; the process is never modified; 0 if no process attached
void uc::close(uint64 h);

bool uc::setup_stack(uint64 h, uint64 base, uint64 size, uint64 stop_address); // maps stack, rsp=top, writes stop addr (ret halts)
int  uc::start(uint64 h, uint64 begin, uint64 end, uint64 timeout = 0, uint64 count = 0); // uc::error; ok = clean exit; timeout µs, count = instrs (0 = unlimited)
void uc::stop(uint64 h);                 // from within a hook
bool uc::flush_code(uint64 h);           // after self-modifying code
uint32 uc::last_error(uint64 h); uint64 uc::fault_address(uint64 h);

// funcdefs — return false to stop emulation
bool code_hook_fn(uint64 h, uint64 address, uint32 size) {}
bool mem_hook_fn(uint64 h, uc::mem_type type, uint64 address, uint32 size, uint64 value) {}
bool uc::hook_code(uint64 h, code_hook_fn@ cb, uint64 begin = 1, uint64 end = 0); // begin>end = all addresses
bool uc::hook_memory_unmapped(uint64 h, mem_hook_fn@ cb, uint64 begin = 1, uint64 end = 0);

// registers
x86_reg::rax..r15, rsp, rbp, rip, eflags, cs..ss, fs_base, gs_base, mxcsr, xmm0..15, ymm0..15
bool uc::reg_read64(uint64 h, x86_reg reg, uint64 &out val);
bool uc::reg_write64(uint64 h, x86_reg reg, uint64 val);
bool uc::reg_read128(uint64 h, x86_reg reg, array<uint8> &out data);   // array exactly 16 bytes
bool uc::reg_write128(uint64 h, x86_reg reg, const array<uint8> &in data);
bool uc::reg_read256(uint64 h, x86_reg reg, array<uint8> &out data);   // exactly 32 bytes (ymm; low 128 alias xmm)
bool uc::reg_write256(uint64 h, x86_reg reg, const array<uint8> &in data);

// memory
bool uc::mem_map(uint64 h, uint64 address, uint64 size, uint perms);
bool uc::mem_unmap(uint64 h, uint64 address, uint64 size);
bool uc::mem_protect(uint64 h, uint64 address, uint64 size, uint perms);
bool uc::mem_write(uint64 h, uint64 address, const array<uint8> &in data);
bool uc::mem_read(uint64 h, uint64 address, uint32 size, array<uint8> &out data);
int  uc::mem_query(uint64 h, uint64 address);  // uc::prot bitmask, -1 if unmapped; for create_process handles falls back to host process protections
bool uc::mem_regions(uint64 h, array<uc_memory_region> &out regions);
```

Classic pattern for emulating a small routine (e.g. pointer decryption `ror(ptr ^ key, 0x11)`): `create()` → `mem_map` code+stop regions → `setup_stack` → `mem_write` code → set input regs → `start(code_base, stop_base)` → check `uc::error::ok` → `reg_read64` results → `close(h)`.

## SIMD & bit operations

All SSE functions require `array<uint8>` of exactly 16 bytes.

```cpp
// rotations / byte swap / bit counts
uint8 rol8(uint8, int); uint8 ror8(uint8, int); // + 16/32/64 variants
uint16 bswap16(uint16); uint32 bswap32(uint32); uint64 bswap64(uint64);
int popcnt32(uint32); int popcnt64(uint64); int lzcnt32(uint32); int lzcnt64(uint64); int tzcnt32(uint32); int tzcnt64(uint64);

// SSE logical: out = a OP b
void mm_xor_si128(a, b, out); mm_or_si128(a, b, out); mm_and_si128(a, b, out); mm_andnot_si128(a, b, out); // (~a)&b

// shifts (imm8)
mm_slli_epi16/32/64, mm_srli_epi16/32/64(a, imm8, out); mm_slli_si128, mm_srli_si128(a, imm8, out); // whole-register byte shift

// shuffles
mm_shuffle_epi8(a, b, out);   // pshufb — mask bit 7 of each byte zeros it
mm_shuffle_epi32(a, imm8, out); mm_shufflehi_epi16(a, imm8, out); mm_shufflelo_epi16(a, imm8, out);

// unpack (interleave high/low lanes)
mm_unpackhi_epi8/16/32/64(a, b, out); mm_unpacklo_epi8/16/32/64(a, b, out);

// arithmetic
mm_add_epi8/16/32/64, mm_sub_epi8/16/32/64(a, b, out); mm_mullo_epi16, mm_mullo_epi32(a, b, out);

// set / broadcast
void mm_setzero_si128(out);
mm_set_epi64x(e1, e0, out); mm_set_epi32(e3, e2, e1, e0, out);
mm_set1_epi64x(a, out); mm_set1_epi32(a, out); mm_set1_epi16(a, out); mm_set1_epi8(a, out);
broadcast_qword(uint64 a, out); broadcast_dword(uint32 a, out);

// extract (zero-extended)
int64 mm_extract_epi64(a, index 0..1); int mm_extract_epi32(a, 0..3); int mm_extract_epi16(a, 0..7); int mm_extract_epi8(a, 0..15);

// compare — out lanes: 0xFF match / 0x00 no match
void mm_cmpeq_epi8/16/32(a, b, out);
```
