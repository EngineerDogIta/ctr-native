# macOS (Apple Silicon) Port — Status & Roadmap

This document tracks the work to run CTR-Native on macOS / Apple Silicon
(arm64), the milestones reached so far, and the concrete path to a playable
build. It is the macOS-specific companion to `docs/MEMORY_MODEL.md`, which
explains the underlying memory model and the two "Known Gap" sections referenced
below.

## TL;DR

- The Windows and Linux builds are **forced 32-bit** (`-m32`) so that pointers
  are 4 bytes and match the PS1-shaped, on-disc data layouts the decompiled game
  code assumes.
- macOS has no 32-bit support, so the macOS build is **64-bit arm64**. Every
  place the retail code treats a pointer as a 32-bit value has to be made
  pointer-width-correct.
- **Current state:** the arm64 binary builds, links, launches, and boots/idles
  stably against real NTSC-U assets — the model pack, level, and driver-model
  relocation gaps (Milestone M3) and the per-frame render-path field
  widening (Milestone M4) are fixed. Driving an actual race (Milestone M5)
  is blocked on a `PROC_BirthWithObject` crash; not yet investigated.

## Why this is hard

The game code is decompiled PS1 retail source. On the PS1 a pointer is 32 bits,
and the on-disc file formats (BIGFILE entries, MPK model packs, level files,
HOWL audio) are **binary overlays** that embed 32-bit pointers directly. The
32-bit desktop builds work because `sizeof(void *) == 4` there, so the C structs
match the files byte-for-byte and the retail pointer arithmetic is lossless.

On a 64-bit host two distinct problems appear:

1. **Pointer-width truncation (mechanical).** Retail code that stashes a pointer
   in an `int`/`u32`, walks a buffer with `(int)ptr + n`, or hardcodes a struct
   size, silently drops the high 32 bits. Fix: widen the *arithmetic* to
   `uintptr_t` (which is 4 bytes on the 32-bit builds, so they are unaffected)
   or use `sizeof(...)` instead of a retail literal. These are the fixes that
   got us from "instant crash" to "16 s of boot".

2. **On-disc 4-byte pointers (architectural).** Pointer *tables* and pointer
   *fields inside* loaded assets are 4 bytes on disc. A 64-bit host cannot store
   its addresses in 4 bytes, and the obvious escape — mapping all game memory
   below 4 GiB so addresses fit — is **impossible on macOS**: the kernel
   `SIGKILL`s any arm64 binary with a non-default `__PAGEZERO`, and with the
   default 4 GiB pagezero the entire low 4 GiB is reserved. This is Milestone M3.

## Build & run

```sh
./build_macos.sh           # configures CMake into build/, builds build/ctr_native
```

Requires Xcode Command Line Tools and CMake (`brew install cmake`).

### Assets

The game needs NTSC-U retail assets under `assets/` (next to `build/`, or in the
repo root — the loader checks the executable dir then its parent). Extract them
from a CTR **NTSC-U** disc image (`BUILD=926` is UsaRetail; a PAL disc will not
match). A PSX `MODE2/2352` `.bin` can be unpacked with a small ISO9660 reader:
Form1 files (`BIGFILE.BIG`, `SOUNDS/KART.HWL`, `TEST.STR`, `XA/ENG.XNF`) as 2048
user-bytes/sector, and `.XA` files as raw 2352-byte sectors (the audio layer
accepts 2352 or 2336; the CD layer reads 2048-byte sectors). The disc's
directory layout matches the loader's expected paths exactly.

### Debugging recipe

The fastest loop is crash-driven, because each pointer bug surfaces as a clean
`EXC_BAD_ACCESS` at a truncated address:

```sh
cmake --build build -j
./build/ctr_native            # let it crash
```

macOS writes a fully symbolicated report to
`~/Library/Logs/DiagnosticReports/ctr_native-*.ips`. The triggered thread's
backtrace names the exact `file:line`, and the faulting address is almost always
a recognisably truncated pointer (low 32 bits only, or two 4-byte entries
concatenated). `lldb -b -o run -o bt ./build/ctr_native` also works but the
`.ips` report is more reliable for late crashes.

## Milestones

### M0 — Build system (done)

`build_macos.sh`, the `APPLE` branches in `CMakeLists.txt`, and a 64-bit arm64
configuration. No `-m32`, no MIPS toolchain.

### M1 — Compiles & links (done)

The blocker was ~546 `offsetof`/`sizeof` layout asserts that bake in retail
4-byte-pointer offsets and (correctly) fail once a struct embeds an 8-byte host
pointer. All such layout asserts now go through `CTR_STATIC_ASSERT_LAYOUT`
(`include/platform/native_static_assert.h`, force-included via `CMakeLists.txt`):
a real `_Static_assert` on the 32-bit builds (which stay the byte-parity
guardians) and a no-op on `__LP64__`. Value asserts (enum/flag constants — no
`offsetof`/`sizeof`) stay as plain `_Static_assert` everywhere. See the second
"Known Gap" in `docs/MEMORY_MODEL.md`.

### M2 — Boots into asset loading (done)

A chain of pointer-width-truncation fixes, each verified by re-running to the
next crash. All are `uintptr_t`/real-pointer widenings or `sizeof`-instead-of-
literal changes, identical in behaviour on the 32-bit builds:

- `MEMPACK.c` — arena arithmetic and `MEMPACK_NewPack`'s `(u32)start`.
- `LOAD_LangFile` — pointer-typed signature; the on-disc 4-byte string-offset
  table is resolved into a native `char *[]` on 64-bit instead of being rewritten
  in place.
- `HOWL_Load` header parse walks; `HOWL_Bank` SPU source pointer.
- `HOWL_Voiceline` — `LIST_Init` pool strides via `sizeof`, not retail `0x20`/`0x10`.
- `MainInit` — JitPool free-list walk follows real `Item` links and seeds the
  payload self-pointer at the true (header-relative) offset.
- `MainFrame` — `otSwapchainDB` local kept at pointer width.
- `ptrMPK` retyped from `int` to a real pointer (`regionsEXE.h` + callback).
- A sweep of `(T *)((int|u32)ptr …)` truncating casts across `game/`.

### M3 — Asset pointer relocation (done — Option A adopted)

**Approach chosen: Option A (load-time transform).** A native-only module
(`platform/native_reloc.c`, `#ifdef CTR_RELOC64`) rebuilds each binary-overlay
asset into native structs with real 8-byte pointers right after load, so every
`game/` dereference site stays byte-identical to retail. The 32-bit Win/Linux
builds keep `LOAD_RunPtrMap` and compile none of it.

**Phase 1 (model pack / MPK) — done and verified.** The boot used to fault in
`LibraryOfModels_Store` walking the MPK model-pointer table; it now boots past it
and runs stably (renderer up). `Reloc64_ModelPack` rebuilds the `PLYROBJECTLIST`
table and the full `Model → ModelHeader → {TextureLayout[], ModelAnim[], AnimTex}`
graph plus the `mpkIcons → LevTexLookup → IconGroup` icon graph; leaf blobs
(command lists, vertex/frame data, CLUTs) stay resident in the original buffer
and are pointed at directly. Length-less pointer arrays (e.g. `ptrTexLayout`) are
bounded by the embedded DRAM pointer-map; arrays with a stored count
(`numHeaders`, `numAnimations`, `numIconGroup`) use it. Hooked in
`LOAD_DramFileCallback` (dispatched by `callback == LOAD_Callback_DriverModels`);
the few raw-offset reads (`ptrMPK+4`, `*ptrMPK`, `mpkIcons+4`) now go through
`Reloc64_Mpk*` accessors / struct fields. `ModelHeader.ptrCommandList` and
`gGT->mpkIcons` were widened to `uintptr_t`.

**Phase 2 (level + instances) — done, committed.** `Reloc64_Level`
(`platform/native_reloc.c`) rebuilds `struct Level` and its full graph (PVS, BSP
tree + hitbox array, QuadBlock array, mesh info, skybox, water verts, spawn
types, nav header/table, vis mem, InstDef array/ptr array, AnimTex chains).
Wired into `LOAD_Callback_PatchMem`. Boots and idles stably; not yet exercised
by an actual race load.

**Phase 3 (individually-loaded driver models) — done, committed.** The `-2`
sentinel callback (`LOAD_DriverMPK_SetPointer`) queues per-character driver
model files through the same MPK format as the main model pack, but
dispatched to `LOAD_DramFileCallback` differently — that branch only matched
`LOAD_Callback_DriverModels`, so these files fell through to the truncating
`LOAD_RunPtrMap`. Fixed by also matching the `-2` sentinel and routing through
`Reloc64_ModelPack`/`Reloc64_MpkModels`; `driverModelExtras` widened to
`uintptr_t[3]` (`VehBirth_GetModelByName` reinterprets its address as
`struct Model**`, needing a real 8-byte stride). Also fixed a related bug:
`LOAD_ReadFile_ex` always returns the raw pre-relocation buffer it allocated,
discarding the rebuilt native pointer `Reloc64_ModelPack` stores into
`data.currSlot.ptrDestination` — the `-2` path's result capture in
`LOAD_DramFile` now re-reads that field under `CTR_RELOC64`.

**Phase 4 (the actual post-splash freeze) — done, committed.** Phases 1-3
above were verified by "boots and idles," which never drove the loader far
enough to hit any of this. Driving a real load past the splash screen
surfaced three more bugs, all found by attaching `lldb` to the "frozen"
process rather than guessing — it wasn't a hang, the main thread was
permanently parked in `MEMPACK_AllocMem`'s retail OOM halt
(`CTR_ErrorScreen(0xFF,0,0); for(;;){}`, see `game/MEMPACK.c`), which looks
identical to a frozen window from the outside (no more frames presented) but
shows up immediately in a backtrace:
- `ModelAnim.numFrames`' top bit is an interpolation flag (retail masks it
  with `& 0x7fff` at every read site — `INSTANCE.c:407`, `CS_Instance.c:155`,
  `VehFrame.c:44`, `RenderBucket_QueueExecute.c:1876`). `Reloc64_ModelAnim`
  read it raw, so any compressed animation sized its frame-data allocation in
  the megabytes instead of low hundreds of bytes.
- Normal race tracks load their LEV through `LOAD_Callback_LEV`, not
  `LOAD_Callback_PatchMem` (that path is Adventure-Hub-only). It was never
  wired into the `CTR_RELOC64` dispatch, so it fell through to the
  truncating `LOAD_RunPtrMap` — `sdata->ptrLevelFile` ended up misread at
  native struct offsets, crashing in `LibraryOfModels_Store` with garbage
  `numModels`/NULL `ptrModelsPtrArray`.
- The native MEMPACK arena was pinned to retail's exact ~1.3 MiB MPK window
  (`CTR_NATIVE_MEMPACK_RETAIL_PRESSURE`). That budget assumed retail's packed
  4-byte pointers; Option A's rebuilt native structs (8-byte pointers, plus
  the original file buffer staying resident alongside the rebuilt spine) need
  more headroom for a whole level's instance/model/animation graph — not a
  decode bug, a real budget increase. Defaulted off on `__LP64__`/`_WIN64`.

Also added a sanity cap in `Reloc64_Alloc`: since `MEMPACK_AllocMem` can't
report OOM (it halts forever instead), a future bug of this shape would
otherwise burn the whole pool with no diagnostic.

**Remaining:** none known in the asset-relocation (Option A) work itself. Two
issues found during Phase 3's investigation are still tracked as separate
follow-up work, not part of M3:
- `game/INSTANCE.c` (`INSTANCE_LevInitAll`) copies `InstDef` fields into
  `Instance` via a raw `int*` blit at hardcoded retail byte offsets/strides —
  corrupts every placed level instance on 64-bit once a real track loads
  instances (pointer fields grow 4→8 bytes, shifting every field after them).
  Needs a field-by-field assignment instead of the blit.
- `struct InstDrawPerPlayer`'s `ptrCommandList`/`ptrColorLayout`/
  `ptrDeltaArray` fields are truncating (see M4 below).

With Phase 4's fixes, a real track now loads all the way to driver/vehicle
spawn and beyond. The `VehBirth_Player` crash and five more along the same
crash-driven chain are fixed (one commit each):

- `PROC_BirthWithObject` truncated `th->object` (`(u32)stackObj` instead of
  `(uintptr_t)stackObj`, inconsistent with three other casts in the same
  function) — the actual `VehBirth_Player` `memset` crash.
- `JitPool_Add` returned `(int)item` instead of a real pointer — corrupted
  every instance/thread spawned from a JitPool (next hit:
  `INSTANCE_Birth3D`).
- `Reloc64_InstDefPtrArray` wasn't NULL-terminated; consumers walk it as a
  NULL-terminated list, so they read one slot past the array into heap
  garbage (`LevInstDef_UnPack`). Mirror of the already-correct
  `ptrModelsPtrArray` pattern.
- `CutsceneObj.frameOverrideRoot` read only the low 32 bits of an 8-byte
  pointer (retail aliases it onto a struct's leading 4-byte field) — hit
  during a kart's intro cutscene.
- `UI_INSTANCE_BirthWithThread` carried a function pointer, a name string,
  and a `PushBuffer*` through `int` parameters, truncating all three at
  every one of 13 call sites — HUD/pickup-display init.

All found via the same technique: attach `lldb` at the crash (or set a
breakpoint just before it once the failure mode is understood), inspect the
actual pointer value, recognize it as a 32-bit-truncated address, find the
narrow field/parameter upstream.

**M4 — render-path field widening: DONE.** Three crash-driven fixes, same
technique as M3's Phase 4 chain:

- `struct InstDrawPerPlayer.ptrCommandList`/`.ptrColorLayout`/
  `.ptrDeltaArray` (`u32`/`u32`/`int`) were truncating real pointers copied
  from `ModelHeader.ptrCommandList`/`ptrColors` and `ModelAnim.ptrDeltaArray`
  — the crash in `RenderBucket_CopyScratchColorCache` reading
  `ctx->idpp->ptrCommandList`. Widened all three to `uintptr_t`, plus
  `RenderBucket_GetFrame`'s `deltaArrayOut` out-param and `ModelHeader.unk3`
  (aliases `ptrDeltaArray` for static models, also needed
  `Reloc64_Resolve` in `platform/native_reloc.c` instead of a raw copy).
  Touches `include/namespace_Instance.h`, `RenderBucket_QueueExecute.c`,
  `AH_WarpPad.c`, `native_reloc.c`.
- `InstDrawPerPlayer.otRangeNormal`/`.otRangeSecondary` (`int`) — same
  truncation shape, different cluster: `RenderBucket_AllocateOTRange`
  allocates a real native OT-heap pointer and biases it through
  `RenderBucket_AddressSubOffset`, which truncated it via
  `(int)(u32)(uintptr_t)lhs` before storing. ~16 `*AtRange`/`*AtOTEntry`
  function signatures and their local round-trips across
  `RenderBucket_QueueExecute.c` (plus `MM_Title_SetTrophyDPP`'s `e4`/`e8`
  locals) all widened to `intptr_t` to carry the value through intact.
- `Reloc64_Level` discarded its own `ptrMapOffsets`/`numPtrs` parameters
  (`ctx.ptrSet = NULL; ctx.ptrSetCount = 0;`), even though both real callers
  (`LOAD_Callback_PatchMem`, `LOAD_Callback_LEV`) already pass a real
  DRAM-pointer-map. Level-embedded `Model`/`ModelHeader` records (decorative
  props baked directly into the LEV file, not the shared model pack) walk
  through the same `Reloc64_ModelHeaderInto` as the model pack, and
  `ModelHeader.ptrTexLayout`'s length (`Reloc64_PtrRunLen`) depends on that
  map — with it empty, every level-embedded model's texture array sized to
  0, and any `texIndex >= 1` read aliased whatever the bump allocator placed
  next (caught live: the literal bytes of a `ModelAnim` named `"anim0"`).
  Fixed by building `ctx.ptrSet` in `Reloc64_Level` the same way
  `Reloc64_ModelPack` already does.

Debugging note: the first diagnosis of the `ptrTexLayout` crash (an
embedded-null entry truncating an otherwise-correct pointer-run heuristic)
turned out wrong — live instrumentation showed `ctx->ptrSetCount == 0`
entirely, not a gap in a populated map. Worth remembering: when a
length-less array's bound looks wrong, check whether the *map it's bounded
by* is even populated before assuming the bounding heuristic itself is at
fault.

**mediumStack JitPool overflow — DONE.** The `PROC_BirthWithObject`/
`LIST_RemoveFront` crash above (and an alternate manifestation in
`RenderBucket_PrepareDrawContext` with `inst=0x100000000`, same underlying
corruption, surfacing at whichever consumer read it first in a given run)
was caused by `struct CutsceneObj` (asserted `0x60`=96 bytes in retail, but
`CTR_STATIC_ASSERT_LAYOUT` no-ops the size check on `__LP64__`) actually
being **144 bytes** on this build, exceeding the shared `mediumStack`
JitPool's hardcoded `0x88` (136-byte) item size from `MainInit.c`.
`game/233/CS_Thread.c` made it worse by passing the stale retail size
(`0x60`) instead of `sizeof(struct CutsceneObj)` to
`PROC_BirthWithObject`/`INSTANCE_BirthWithThread`, so the pool's own safety
check incorrectly passed and a full 144-byte object got written into a
136-byte slot, corrupting the next free-list slot's header. Root-caused via
a hardware watchpoint on `gGT->JitPools.mediumStack.free.first` and live
`sizeof()`/`offsetof()` checks (the crash's `LinkedList` pointer didn't
match the first guess, `JitPools.thread` — computing each pool's live
offset from `gGT` pinned it to `mediumStack` instead). Fixed `CS_Thread.c`'s
three call sites to pass `sizeof(struct CutsceneObj)` (matching every other
caller in the codebase) and widened the pool's item size to `0xa8` (168,
fits `struct WarpPad` at 160 too — `AH_WarpPad.c` already correctly passed
`sizeof(struct WarpPad)`, so its safety check was silently rejecting every
warp pad spawn against the old undersized pool, a separate non-crashing bug
fixed by the same resize).

**DrawTires_Solid.c / DrawTires_Reflection.c OT-slot truncation — DONE.**
The deferred risk flagged during the OT-range fix above was real: both
files' `DrawTiresSolidScratch`/`DrawTiresReflectionScratch` (hand-laid-out,
byte-offset-addressed scratch mirrors, same `CTR_STATIC_ASSERT_LAYOUT`
convention) store `otRangeNormal`/`otRangeSecondary` (copied from the
now-`intptr_t` `InstDrawPerPlayer` fields) and derive `otRangeStart`/
`otRangeEnd` from them — all four still `int`, crashing
`DrawTiresSolid_LinkPrimitive` on the truncated dereference. Since
`otRangeStart`/`otRangeEnd` are the last two fields in both 0x178-byte
structs (nothing follows them) but `otRangeNormal`/`otRangeSecondary` sit
earlier with ~40 unrelated fields and ~50 hardcoded-offset call sites
between them, widened by **appending four new `intptr_t` fields at the end
of each struct** instead of widening in place, leaving the original narrow
fields as unused dead weight so no other offset moves — then redirected
only the ~12 call sites per file that actually touch these four values.

**RenderBucketEntry allocation + terminator write — DONE.** The
`RenderBucket_PrepareDrawContext`/`inst=0x100000000` crash (and its
`PROC_BirthWithObject`/`LIST_RemoveFront` twin from earlier) turned out to
be one bug, not the mediumStack one originally suspected — that fix was
real and necessary but didn't address this. `struct RenderBucketEntry`
(`inst`/`instPlayerBase`, both real pointers, retail-asserted 8 bytes
total, genuinely 16 bytes here) has two compounding issues:
1. `MainInit.c`'s native path borrowed a fixed slice of static RDATA
   scratch (`rdata.s_STATIC_GNORMALZ + 148`) sized for retail's 8-byte
   entries — switched to a real `MEMPACK_AllocMem` call scaled to the real
   entry size, since doubling usage of the static slice risked colliding
   with whatever real data follows it.
2. The actual crash cause: `RenderBucket_QueueAllInstances`
   (`MainFrame_RenderFrame.c`) wrote the list terminator through `int
   *RBI; *RBI = 0;` — on 64-bit this only zeroes the low 32 bits of
   `entry->inst`, leaving high-bits garbage from the (non-zero-initialized)
   freshly allocated slot. `RenderBucket_Execute`'s `entry->inst != 0` loop
   then walks into that garbage. Every appearance of "inst=0x100000000" was
   this same bug — same narrow write, different garbage in the high bits
   depending on what was in memory. Retyped `RBI` to `struct
   RenderBucketEntry *` and write `RBI->inst = 0` (full pointer width).

Found via the same lldb technique as every fix this session: a conditional
breakpoint on `RenderBucket_QueueDraw`'s entry never fired before the
crash, ruling out bad data entering the pipeline and pointing at
corruption after queuing. Verified: binary now runs **~64 seconds / 2000
frames** before its next crash — by far the longest stable run this
session.

**Next crash, confirmed independent (not RenderBucketEntry-related —
reproduces with a valid `cs` pointer):** `CS_ScriptCmd_ReadOpcode_Main`
(`game/233/CS_ScriptCmd.c:60`) — `cs->currOpcode[0]` reads a wildly garbage
address. Not yet investigated. `struct CutsceneObj` itself was already
audited (its pointer fields are correctly widened); the cause is likely
elsewhere (script data loading/relocation, or another consumer of the
cutscene system).

**Session continuation — 4 more fixes (commits `03fc1603e`..`0da45760d`):**

1. **`CsOpcodeArg::ptr` layout (`include/ovr_233.h`, `game/233/CS_Thread.c`).**
   `union CsOpcodeArg` had `char *ptr`, making it 8 bytes on native and
   silently shifting every `CsOpcodeMeta` field. Changed `ptr` to `u32`
   (retail 4-byte opcode offset); added `_Static_assert(sizeof==4)`;
   cast `(char *)(uintptr_t)` at the 3 branch-target use sites in CS_Thread.c.
   This likely fixes the `CS_ScriptCmd_ReadOpcode_Main` garbage-read crash.

2. **`INSTANCE_LevInitAll` skip offset (`game/INSTANCE.c`).** Hardcoded `+ 8`
   to skip `next+prev` at Instance head was correct on retail (two 4-byte
   pointers) but wrong on native (two 8-byte pointers). Changed to
   `+ offsetof(struct Instance, name)` — evaluates to 8 or 16 automatically.

3. **`sdata->ptrLoadSaveObj` truncation (`include/regionsEXE.h`,
   `game/SelectProfile.c`).** Field typed `int` truncated the native heap
   pointer on write; changed to `uintptr_t` (4B on 32-bit = layout preserved).

4. **`Reloc64_SCVertArray` (`platform/native_reloc.c`).** `ptrSCVert` was
   resolved as a single "leaf" resolve, leaving `scVert->v` as a raw 4-byte
   disc offset read as 8 native bytes → garbage pointer + wrong `scVert++`
   stride (20 vs 16 bytes). Added `Reloc64_SCVertArray` mirroring
   `Reloc64_WaterVertArray`: allocates a native array, resolves `v` per entry,
   copies 3 plain-int fields. Game now runs 20+ seconds past the
   `AnimateQuadVertex` crash site with no new crash report.

Background on why this is the architectural task — the MPK (and level) data are
binary overlays whose **on-disc pointers are 4 bytes**:

- Pointer *tables* (e.g. `PLYROBJECTLIST = ptrMPK + 4`) are arrays of 4-byte
  entries; reading them as native `struct Model **` (8-byte) is a stride
  mismatch (a fault address of two 4-byte entries glued together is the tell).
- Pointer *fields inside* structs shift: `struct Model.headers` is a pointer at
  retail offset `0x14`; at 64-bit it is 8 bytes and the compiler realigns it to
  `0x18`, so the C struct no longer matches the file.
- `LOAD_RunPtrMap` relocates these 4-byte slots with a truncating
  `*(int *)&origin[off] += (int)origin`.

This cannot be solved by widening (4-byte slots cannot hold 64-bit addresses)
nor by address placement (the pagezero constraint above). It needs one of:

- **Option A — load-time transform (recommended).** Per asset format (model,
  level, instance, …) write a deserializer that, right after load/relocation,
  rebuilds the data into native structs with real 8-byte pointers. Access sites
  stay unchanged; the cost is one transform per format. Most maintainable.
- **Option B — 4-byte handles + accessors.** Keep on-disc fields 4 bytes, change
  every loaded-asset pointer field to a handle type, and resolve it against the
  asset base at each dereference (mirrors the GPU OT-link token bridge already
  used in `native_gpu_links.c`). Touches far more call sites.

Both are scoped work, not one-line fixes. `LOAD_RunPtrMap` should be addressed as
part of whichever option is chosen (Option A relocates into the rebuilt struct;
Option B keeps file-relative offsets and never adds `origin`). See the first
"Known Gap" in `docs/MEMORY_MODEL.md`.

### M4 — Render-path field widening: done (see above)

### M5 — Playable

Drive past the per-frame render path into menus and an actual race. Expect
further truncation/layout issues in the instance/spawn paths — the same bug
classes, found the same crash-driven way. Currently blocked on
`PROC_BirthWithObject`/`LIST_RemoveFront` (see above).

### M6 — Distribution

Ad-hoc `codesign` is already applied by the linker. For shipping: a proper
signed/notarised `.app` bundle, a Universal binary if x86_64 is also wanted, and
bundling `assets/` per the layout in `README.md`.

**Longevity risk — OpenGL → Metal.** The native renderer
(`platform/native_renderer.c`, `native_glad.c`) uses an OpenGL 3.3 Core context.
Apple deprecated OpenGL in macOS 10.14 and caps it at 4.1 (we run on Apple's
"4.1 Metal" GL-over-Metal shim — see the boot log). It works today and is not a
"does it run" blocker, but Apple could remove it. The long-term escape is to move
the backend onto Metal — either via SDL3's `SDL_GPU` abstraction or by translating
GL through ANGLE/MoltenVK. This is a renderer concern, fully orthogonal to the
64-bit memory work above; flagged here so it is on the distribution radar.

## Conventions for new 64-bit fixes

- Prefer `uintptr_t` for pointer↔integer arithmetic; it is 32-bit on the
  `-m32` builds, so they keep retail behaviour exactly.
- Never hardcode a struct/pool size that contains a pointer — use `sizeof`.
- Keep edits in `game/` minimal and shaped like the surrounding retail code;
  gate genuinely native-only divergence behind `__LP64__` (or `CTR_NATIVE`),
  leaving the 32-bit path untouched so it remains the parity reference.
- Re-run after each fix and read the `.ips` backtrace; a truncated faulting
  address pinpoints the next site.
