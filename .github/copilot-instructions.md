## Copilot instructions for the WinSavage repo

This short guide gives an AI coding agent the minimal, precise context it needs to be useful in this codebase.

Core idea
- This is a native Win32 C++ project (Win32/Visual Studio). The main job is loading and patching a packed OS/2 LE "flat" executable (`PSAVAGE.EXE`) into memory and hooking its runtime.

Key files to read first
- `loader.cpp` — decrypts, finds the LE image inside the packed blob, maps pages, applies fixups, sets protections, and provides `LoadSQ()`.
- `exeflat.h` — binary-layout structs used by the loader (`os2_flat_header`, `object_record`, `le_map_entry`). Any change here can break parsing.
- `savage.cpp` — project entry, calls `LoadSQ`, implements `PatchSQ` and many runtime hooks and patches.
- `patchutil.{cpp,h}` — helper utilities used for runtime patching/hooking.

Non-obvious, high-value facts
- The code assumes 32-bit x86 pointers in many places (casts to `uint32_t` when writing fixups). Do not switch to x64 without planning/fixing all pointer-size assumptions.
- `DecryptLE()` allocates `buf` with `new uint8_t[...]` and returns size; caller must `delete[]`. `LoadLE()` returns a `pages` buffer (caller-owned).
- `LoadSQ(path, virtualSize, patchFunc)` high-level flow: DecryptLE -> FindLE -> LoadLE -> patchFunc(mappedBase) -> ProtectLE -> free temporary exe buffer.
- `ProtectLE()` maps object flags (OBJ_READABLE/WRITEABLE/EXECUTABLE) to Win32 `VirtualProtect` protections — changing interpretation is risky.

Build & debug (practical)
- Build in Visual Studio:
  1. Open `winsavage.sln` in the root directory
  2. Ensure Platform is set to Win32 (x86) - this is critical due to 32-bit pointer assumptions
  3. Choose Debug or Release configuration
  4. Build Solution (F7)
- Alternative: Build from PowerShell:
  ```powershell
  msbuild .\winsavage.sln /p:Configuration=Debug /p:Platform=Win32
  ```
- Debugging tips:
  1. Build Debug/Win32 with symbols
  2. Place `PSAVAGE.EXE` in the same directory as the built executable
  3. Key breakpoints to set:
     - `loader.cpp`: DecryptLE() start, LoadSQ() after FindLE
     - `savage.cpp`: PatchSQ() entry and specific patch locations
  4. Watch for `DebugBreak()` calls - these fire on unexpected fixup types
  5. MSVC debug visualizers help with:
     - `os2_flat_header` struct contents
     - `object_record` flags and memory protections
     - Raw memory at patch locations

Runtime issues & error patterns
- Memory errors to watch for:
  - Access violations in fixup code - usually means mismatched pointer sizes or bad address calculation
  - DEP violations - check object flags and `ProtectLE()` protections
  - Stack traces with offsets like `0xXXXXXX (0xYYYYYY)` show relative addresses within `PSAVAGE.EXE`
- Common failure points:
  1. DecryptLE: `PrologueSize` check fails (wrong file) or XOR decrypt produces invalid header
  2. FindLE: No 'LE' signature found after walking MZ/BW headers
  3. LoadLE: Fixup types 5/19 are ignored, others trigger `DebugBreak()`
  4. PatchSQ: Version check fails (`SQ v0.92 May 13 1999`) or specific patches can't find expected byte patterns

Repo conventions / gotchas
- Always add `#define NOMINMAX` before `<windows.h>` to avoid macros colliding with C++ stdlib.
- Many files use low-level `memcpy`, pointer arithmetic and reinterpret_casts that depend on binary layout offsets — annotate changes.
- Error style: functions commonly return 0/nullptr for failure. Follow this for new helpers.

Search targets for automation or edits
- Look for `LoadSQ(`, `patchFunc(`, `WatcomHook`, `WatcomCall`, and string literals like "SQ v0.92" to locate version checks and patch sites.

Quick PR checklist (small changes)
- Build (Win32 Debug/Release): PASS
- Run a small smoke: call `LoadSQ("PSAVAGE.EXE", vsz, PatchSQ)` using the shipped exe; ensure `PatchSQ` runs and mapped base looks valid.
- Annotate casts/offsets and add comment notes where you change binary-layout dependent code.
- If any pointer-width changes are required (x64), include a short port plan and test cases for fixups and ProtectLE.

If anything here is ambiguous, tell me which file or workflow you want me to expand and I'll update this doc.

