# Runtime and platform memory model

## Scope

This model separates guest virtual address space, PS4 physical/direct memory,
host allocator memory, Vulkan resources, filesystem buffers, and audio queue
ownership. Values below are PS4 implementation details in the port patch or
`port/` sources; they are not universal desktop defaults.

## Memory domains

| Domain | Allocation / mapping | Owner and lifetime | Constraints / evidence |
|---|---|---|---|
| Guest address space | `platform_ps4.cpp::reserve` reserves fixed ranges; `commit` marks 64 KiB chunks committed. | Runtime memory manager owns reservations; guest allocators own logical ranges. Decommit returns mapped chunks for reuse. | Guest module/heap/alias/stack/kernel ranges are in the runtime’s guest address layout. PS4 reservation is address-space-only until first touch. |
| Guest backing memory | Lazy fault handler maps direct-memory chunks at committed addresses on first access; chunks are zeroed before guest exposure. Direct memory is acquired in 32 MiB blocks, halved when fragmented allocation fails. | `lazy::Slice` tracks state and physical chunk; decommit unmaps and returns chunks to the free list. | 64 KiB granularity; PS4 direct-memory pool is documented as about 4.5 GiB shared by guest and GPU. A map failure becomes a fault/crash and is counted by the memory guard. |
| Host heap / `new` / `malloc` | `port/ps4_runtime.cpp::createHeap` uses dlmalloc mspaces, preferring system flexible memory, then regular flexible memory, then a direct-memory fallback. | Runtime and platform C++ allocations; mspace ownership is global/process lifetime. | Candidate flexible heaps: 256/192/128 MiB system pool, then regular available memory after a reserve, then 256/128 MiB direct memory. Heap allocation is separate from guest lazy address mapping. |
| Guest thread stacks | `platform_ps4.cpp::start_thread` assigns 64 MiB slots in the reserved lazy range, with a 64 KiB guard and the top 1 MiB faulted in up front. | Stack slot is marked reusable after its thread exits and joined before reuse. If no slot is available, a detached thread receives an 8 MiB system stack. | The mapped portion of a normal guest stack is backed through the lazy/direct-memory allocator. `port/ps4_runtime.cpp::__wrap_pthread_create` enforces a 4 MiB minimum for ordinary PS4-created pthreads. |
| NVN guest memory pools | NVN memory-pool objects describe guest memory or mappings; GPU addresses are allocated in the HLE namespace. `pool_host` resolves an address to guest/host backing. | Pool registration maps GPU ranges; finalization waits for users, removes mappings, invalidates caches, then destroys/defer-destroys the object. | Pool lookup caches are invalidated on pool removal/replacement. Virtual pools can resolve through another pool’s mapping. |
| Vulkan images and pipelines | NVN textures are represented by HLE objects and translated Vulkan images; image aliases track guest memory and content serials. Pipelines/descriptors are created by the Vulkan replay path. | Images can be reused or retired after a completed submit serial; objects/snapshots may be deferred while replays refer to them. | Guest texture copies/adoptions matter because BOTW copies `NVNtexture` structures. Exact full runtime ownership implementation is absent, so details beyond the patch deltas remain bounded. |
| Upload/staging memory | PS4 NVN upload ring is 64 MiB. Each submit has a ring interval; overflows can allocate per-submit spill buffers; large worker allocations take ring chunks. | Ring bytes are retained until the owning submit slot retires; spill/staging buffers are queued for deferred cleanup at a later serial. | A full ring may wait for the oldest submit; allocation failure is logged. Pipelining is memory-bounded by slot/fence retirement, not an unbounded copy queue. |
| VideoOut swapchain buffers | PS4 WSI creates linear RGBA8 images in host-visible memory, aligns registered buffer addresses to 2 MiB, and registers them with VideoOut. | `Swapchain` owns Vulkan images, memory, and VideoOut buffer registration until destruction; destruction waits for device idle and unregisters buffers. | WSI advertises 2–3 images, internally allows up to four, uses fixed 1920×1080 extent, and presents via flips. |
| RomFS pack and file handles | `load_pack()` opens the shared `romfs.pak` once and builds in-memory path/child indexes from `romfs.idx`. Packed file handles store base offset/size and share the pack `FILE*`. | Pack/index are process-lifetime globals in the patch. Per-handle buffers are freed when the read-ahead window ends or the handle closes. | A packed file is a window into the shared pack, not a separately opened file. Positioned reads avoid changing a shared file cursor. |
| Audio read-ahead | Each eligible streamed `.bwav` handle may hold a read-ahead vector; windows are at most 512 KiB with a 16 MiB shared budget. | Per-handle storage is returned at stream end or file close; over-budget reads fall back to direct reads. | Read-ahead begins only after sequential reads. It is synchronous read caching, not a dedicated I/O worker. |
| Audio output queue | Pending PCM entries hold a source pointer, byte count, cursor, and completion context; output thread copies into alternating 256-frame buffers. | The submitting side must keep PCM data alive until completion callback. | Queue is protected by `g_audio`; no queue-capacity limit is evidenced. Output is S16 stereo at 48 kHz; underrun/pause yields silence. |

## Lazy guest memory lifecycle

```mermaid
flowchart LR
  RESERVE["Reserve fixed guest VA ranges"] --> COMMIT["Commit marks chunks usable"]
  COMMIT --> ACCESS["Guest or host touches committed address"]
  ACCESS --> FAULT["SIGSEGV/SIGBUS handler"]
  FAULT --> ALLOC["Take 64 KiB direct-memory chunk"]
  ALLOC --> ZERO["Zero before exposure"]
  ZERO --> MAP["Map chunk at guest VA"]
  MAP --> USE["Guest/runtime/GPU use"]
  USE --> DECOMMIT["Decommit/unmap"]
  DECOMMIT --> FREE["Return physical chunk to free list"]
```

The fault handler only maps addresses within the lazy region and only when
their chunk is committed. A second fault after another thread has mapped the
chunk retries the access. The handler uses an atomic spin lock rather than a
normal mutex. `read_at()` prefaults guest destination buffers because kernel
`pread` cannot invoke the process fault handler on behalf of the write.

The lazy allocator tracks chunk state and direct-memory offsets in per-GiB
tables. The patch’s free-list capacity is 131,072 chunks (8 GiB of addressable
chunks); this is a bookkeeping limit, not a claim that the console has 8 GiB
of direct memory. The memory guard reports total/free direct memory, lazy
guest bytes, and failed chunk mappings.

## Verified comment conflict: thread-stack backing

Two patch comments disagree:

- `platform_ps4.cpp` reserves stack slots inside the lazy guest range; its
  `start_thread()` commits and prefaults the slot through `lazy::commit()` and
  `lazy::prefault()`. This is the implementation path for those slots.
- `memguard_ps4.cpp` says flexible memory is used for thread stacks.

The allocation path supports the first description for guest threads using a
lazy stack slot. The fallback stack when no slot exists is allocated by
`pthread_attr_setstacksize` and uses the platform pthread implementation; its
physical backing is not described in that call site. The broad memory-guard
comment should therefore not be treated as evidence that all thread stacks
come from flexible memory.

## Lifetime and ownership rules

1. **Guest-memory writes and reads:** guest allocations own guest VA; the
   platform owns backing mappings. HLE file reads prefault destinations and
   copy bytes into guest memory.
2. **NVN memory pools:** guest-visible pool lifetime controls address
   translation. Finalization waits for replay workers and relevant GPU
   writebacks before destroying the pool object.
3. **Textures and images:** texture HLE objects may be copied/adopted; texture
   pools and descriptor snapshots can retain references. GPU images must
   survive through their last submit and are retired only after completion.
4. **Upload allocations:** command recording borrows the submit’s upload-ring
   slice. A Vulkan fence determines when the ring tail can advance and
   per-submit staging resources can be retired.
5. **Audio submissions:** the PCM source pointer remains live until the output
   thread consumes the bytes and invokes the completion callback.
6. **Pack files:** each file handle is a bounded offset/size view over a
   shared, process-lifetime pack file. Its optional read-ahead allocation is
   per handle.

## Target-port implications

- A desktop host should replace PS4 fixed-range reservation/mapping with a
  per-OS virtual-memory adapter while preserving guest address layout where
  required. Linux/SteamOS can use reserve/`mmap`/`mprotect`; Windows needs
  `VirtualAlloc` reserve/commit semantics. This is distinct from ordinary
  `std::filesystem` use.
- Guest backing, Vulkan allocations, host heap, and thread stacks compete for
  different PS4 pools in the current port. Do not combine their measurements
  into one “free memory” number.
- A desktop Vulkan driver will choose different memory types. Preserve
  upload-ring and resource-retirement invariants, not PS4’s 64 MiB ring,
  2 MiB VideoOut alignment, or direct-memory assumptions.
- The runtime patch sets the PS4 upload buffer to 64 MiB; this is a target
  tuning decision. It does not define a desktop memory budget.

## Evidence

- [PS4 lazy memory and stack allocator](../../patches/botw-ps4.patch#L10416).
- [PS4 memory guard](../../patches/botw-ps4.patch#L2035).
- [PS4 host heap and pthread wrappers](../../port/ps4_runtime.cpp).
- [Filesystem pack/read-ahead implementation delta](../../patches/botw-ps4.patch#L803).
- [NVN frame slots and upload ring](../../patches/botw-ps4.patch#L4966).
- [PS4 VideoOut WSI](../../patches/botw-ps4.patch#L12598).
