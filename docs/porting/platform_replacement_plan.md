# Host platform replacement plan

## Goal and scope

This is a proposed host-port mapping for Windows, Linux, and Steam Deck using
SDL3, Vulkan 1.3, and standard filesystem APIs. These targets are not claimed
to be implemented by the checked-in PS4 port. The plan replaces PS4 endpoints
while retaining guest ABI and NVN compatibility behavior unless a separate
change is justified.

## Current host dependencies

| Dependency | Current PS4 use | Replaceable boundary | Porting note |
|---|---|---|---|
| OpenOrbis CRT / PS4 SDK | `_start`, `__libc_start_main`, system module loading, user/pad/audio/display init | Process entry, startup services, logging, package paths | Use native CRT and application startup. Remove OpenOrbis CRT, eboot packaging, and system-module IDs. |
| libkernel memory APIs | Fixed virtual reservations, direct/flexible-memory mapping, memory accounting, sleep, thread affinity/priority | Platform memory and thread abstractions | Windows needs `VirtualAlloc`; Linux/SteamOS can use `mmap`/`mprotect`. Preserve guest VA assumptions and guard/decommit semantics where required. |
| `scePthread*` / pthread wrappers | Thread creation, affinity, priority, names, minimum stacks, PS4 ABI workarounds | Runtime `plat::start_thread`, naming/placement | Use `std::thread` or native host threads; remove PS4 ABI wrappers, core masks, and syscall names. Keep explicit stack sizing only if the host/runtime needs it. |
| SceUserService | Initial-user selection and login list; maps signed-in users to controller players | Local app profile/controller assignment | Do not assume a console account exists on desktop or that saves are automatically bound to the active controller. |
| ScePad | Poll DualShock 4 state and map buttons/sticks to guest `PadState` | SDL3 Gamepad API | Preserve Switch-style guest button mapping, dead zones, trigger thresholds, player IDs, hotplug events, and optional button-layout setting. |
| `sceAudioOut` | Fixed S16 stereo 48 kHz sink with queued grains and a dedicated output thread | SDL3 audio device stream/callback | Keep `nn::audio` CPU DSP unchanged initially. Adapt sample format/rate, buffer ownership, pause, underrun, and completion callbacks at the sink. |
| VideoOut / GNM | Registered linear framebuffer buffers; GPU-side flip or queue-idle fallback; flip events | SDL3 window + Vulkan WSI | Delete PS4 swapchain shim and use native Vulkan surface/swapchain acquire/present/fences. Do not strip semaphores. |
| Linked RADV ICD dispatch | Vulkan entrypoints forwarded through RADV internal proc resolvers | Vulkan 1.3 loader and device dispatch | Use the platform Vulkan loader/driver; preserve runtime’s NVN-to-Vulkan translation and feature checks. |
| PS4 host filesystem | `/data/<app>` data/config/logs; host stdio and PS4 `fopen_utf8` abstractions | `std::filesystem` plus standard streams / positioned-read abstraction | Separate install/content root, writable profile data, cache, save, and logs. Preserve `romfs.pak`/`romfs.idx` compatibility if desired. |
| PS4 network wrappers | SceNet DNS workaround and optional SMB/NFS `/net` libc wrappers | Native host resolver/sockets; optional explicit VFS | Optional network filesystem is not a required guest service path. Keep it out of the core port unless a real use case is verified. |
| PS4 crash/profiler APIs | Signal context, sampler, fault capture, direct-memory stats, boot traces | Native diagnostics / profiler | Use platform crash reporting and profiling; do not carry PS4 signal/register layouts or kernel module queries. |

## Replacement matrix

| Capability | Windows | Linux | Steam Deck / SteamOS |
|---|---|---|---|
| Window and events | SDL3 video/window/event loop; SDL-created Vulkan surface | SDL3 video/window/event loop; SDL-created Vulkan surface | Same as Linux; validate Wayland and gamescope session behavior through SDL3. |
| Input | SDL3 Gamepad; map buttons/sticks to guest `PadState` | SDL3 Gamepad; same guest mapping | SDL3 Gamepad with Deck controls; do not assume Steam Input bindings unless explicitly enabled. |
| Audio | SDL3 audio device stream at native or converted S16 stereo 48 kHz | SDL3 audio device stream | SDL3 audio device stream; verify suspend/resume and device changes in SteamOS. |
| Rendering API | Vulkan 1.3 loader/driver; standard Win32 Vulkan WSI | Vulkan 1.3 loader/driver; X11/Wayland WSI provided through SDL3/native loader | Linux Vulkan driver (commonly Mesa RADV); standard platform WSI, no GNM/VideoOut. |
| NVN translation | Keep NVN HLE → Vulkan replay and guest object/memory model | Same translation layer | Same translation layer; tune memory and shader pipelines on Deck GPU. |
| Filesystem | `std::filesystem` paths; standard streams or a positioned-read adapter; Windows path/UTF-8 tests | `std::filesystem` paths and standard streams/positioned reads | Same as Linux; store writable data under per-user application data, not Steam installation paths. |
| Threading | `std::thread` / Windows runtime synchronization; no PS4 affinity assumptions | `std::thread` / POSIX runtime; optional measured CPU affinity | Same as Linux; avoid Jaguar/PS4 fixed core placement. |
| Guest VA backing | `VirtualAlloc` reserve/commit/decommit with guard pages as needed | `mmap` reservation, `mprotect`, `munmap` | Same as Linux; check address-space and overcommit behavior. |
| User/save model | Local profile directory; explicit save isolation | Local profile directory; explicit save isolation | Local profile directory; optionally integrate Steam Cloud outside the guest filesystem contract. |
| Diagnostics | Windows crash dump/profiler and SDL/Vulkan diagnostics | `perf`, core dumps, Vulkan validation | Linux diagnostics plus Deck-specific performance capture; no PS4 sampler. |

The table is a target design, not proof that each source integration exists.
The checked-in repository’s PS4 implementation is in the patch and `port/`
tree; platform build support must be verified separately before claiming a
working target.

## Migration sequence and acceptance gates

1. **Extract platform interfaces.** Keep guest `nn::fs`, `nn::hid`,
   `nn::audio`, `nn::vi`, kernel, and NVN HLE contracts separate from PS4
   implementations. Keep public behavior and platform resource lifetime in
   narrow `plat::*` boundaries.
2. **Replace process and data roots.** Use native CRT; resolve content,
   writable saves, cache, logs, and shader cache separately. Test both
   extracted RomFS and packed `romfs.pak`/index access, UTF-8 paths, directory
   enumeration, offsets, and error mapping.
3. **Replace input and audio endpoints.** Use SDL3 Gamepad and audio APIs;
   preserve player mapping, hotplug events, guest return semantics, and audio
   callback lifetime. Test disconnect/reconnect, pause, underrun, and queue
   drain.
4. **Replace PS4 WSI.** Use an SDL3-created Vulkan surface and standard
   Vulkan 1.3 swapchain. Restore normal acquire/present synchronization and
   semaphore/fence semantics; preserve NVN frame pacing separately from
   display vblank.
5. **Replace memory and thread adapters.** Implement guest VA reservation,
   commit/decommit, fault/guard handling, and thread-start contracts per OS.
   Remove PS4 direct/flexible-memory accounting, fixed core masks, and
   ABI-wrapping pthread code.
6. **Validate compatibility before optimization.** Compare service return
   codes, guest memory visibility, command replay ordering, rendered output,
   save bytes, and audio cadence. Only then add desktop-specific pipeline,
   upload, or worker tuning.

## Non-negotiable compatibility invariants

- Do not replace NVN with native engine rendering; Vulkan is the host backend
  of the guest NVN API in this repository.
- Preserve guest address and memory-pool semantics where recompiled code
  depends on them, even though the desktop physical allocation model differs.
- Preserve queue ordering, fence-visible guest writebacks, descriptor/object
  lifetime, and swapchain image lifetime.
- Keep save/cache paths user-local and do not infer save/account binding.
- Keep PS4-only WSI, GNM, VideoOut, and Sony SDK calls behind platform
  implementations; none should leak into shared guest HLE logic.

## Risks to resolve before claiming a target

- The complete runtime source tree is not checked in; platform parity must be
  validated against the matching shared-runtime revision, not this patch alone.
- Windows/Linux/Steam Deck support cannot be inferred merely from SDL3 and
  Vulkan availability. Guest ABI, address reservations, filesystem behavior,
  memory pressure, and shader coverage still need target runs.
- The PS4 WSI strips semaphores and relies on flip ordering; copying that
  workaround to desktop would violate ordinary WSI expectations.
- The currently documented save/profile relationship, guest I/O scheduling,
  and normal shutdown are incomplete; replacement code must not invent those
  contracts.

## Evidence

- [Services audit](./services.md): concrete service and platform boundaries.
- [Graphics audit](./graphics.md): NVN/Vulkan/RADV/VideoOut path.
- [PS4 platform and WSI patch](../../patches/botw-ps4.patch#L10416).
- [PS4 pthread wrappers](../../port/ps4_runtime.cpp).
- [PS4 GNM adapter](../../port/mesa/ac_ps4_drm.c).
- [Build and staging scripts](../../scripts/stage-data.sh).
