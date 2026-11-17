# Nintendo service compatibility matrix

## Scope and evidence

This matrix describes guest-facing HLE and the PS4 host endpoints documented
in the checked-in porting audits. The shared runtime is represented by
`patches/botw-ps4.patch`; the complete runtime sources and generated BOTW
modules are not checked in. “Implemented” therefore means the named HLE or
platform path is evidenced, not that every Nintendo service method is
implemented.

The port uses direct guest import HLE for several APIs. For these imports,
“service” does not necessarily mean a Nintendo IPC server. Conversely,
`nx::ipc::Service::handle` logs an unregistered command and returns zero; that
fallback is not evidence that the command’s semantics are implemented.

## Matrix

| Service | Status | Implemented features | Stubbed or reduced features | Missing features / evidence gaps | Dependencies and call chain |
|---|---|---|---|---|---|
| `nn::fs` | **Partial HLE implemented** | Imported file/directory operations, mount resolution, file handles, reads/writes, commits, save/cache paths, and folder or `romfs.pak` + `romfs.idx` content. Pack entries are indexed for random access; `.bwav` read-ahead is opt-in by default and budgeted. | Unsupported/unregistered IPC commands use the generic IPC fallback. The pack loader skips malformed index lines; exact malformed-input behavior is not a Nintendo-compatible contract. `CommitSaveData` is aliased to the host `Commit` implementation for SDK 7. | Full Nintendo `fsp-srv` protocol, complete error-code fidelity, save-slot schema, account binding, crash-safe/atomic commit guarantees, and guest archive-to-object ownership are not established. | Guest imported `nn::fs` symbol → `runtime/src/hle_fs.cpp` HLE → `plat::content_root()` / host files. `rom:` resolves to extracted RomFS or a process-lifetime pack; `save:`/`cache:` resolve under host user data. See [services.md](./services.md#filesystem-and-asset-loading) and [the filesystem patch](../../patches/botw-ps4.patch#L803). |
| `nn::hid` | **Partial HLE implemented** | Npad state and full-key history for up to four players; buttons/sticks mapped from `PadState`; style-set events and connection polling; controller-support result. On PS4 the platform maps signed-in users to DualShock 4 handles. | Vibration calls are accepted and dropped. Six-axis and BOTW SDK 7 seven-axis paths return no sensor state; some setters are no-ops. Sensor shared memory is not served. | Touch input, motion sensors, vibration hardware, and complete Npad style/assignment semantics are not implemented or not evidenced. | PS4 `main` → `sample_input()` / `scePadReadState()` → `read_pad()` or `read_extra_pad()` → `hle_hid.cpp` Npad HLE → guest reads. Player hotplug signals the guest style event. See [services.md](./services.md#threading-architecture) and [the HID patch](../../patches/botw-ps4.patch#L1182). |
| `nn::audio` | **Partial HLE implemented** | Guest AudioRenderer manager/open path; SDK DSP command execution on a host CPU thread every 5 ms; guest event signaling; stereo signed-16 PCM output queue. BOTW SDK 7 names are registered, and SDK fields/functions are located by symbols/instruction decoding where available. | Device-name listing is reduced; active channel count returns 2; device-output volume is a zero-return stub. Device-switch notification is represented by a HLE path rather than a real device-change service. Host output is fixed to 48 kHz stereo S16. | Hardware DSP, device selection/routing, true output-volume control, device hotplug notifications, bounded queue/backpressure, and normal audio teardown are not established. BOTW BGM/SE/voice ownership remains **UNVERIFIED**. | Guest `nn::audio` → `hle_audio.cpp` renderer and SDK `ExecuteAudioRendererRendering` → `plat::audio_submit()` → PS4 `sceAudioOutOutput()` thread. `.bwav` reads use the filesystem path, not a proven game mixer implementation. See [services.md](./services.md#audio-architecture) and [the audio patch](../../patches/botw-ps4.patch#L694). |
| `nn::account` | **Service request observed; dedicated HLE not established** | Startup logs show `acc:u0`; PS4 platform code initializes `sceUserService` and obtains signed-in users for controller assignment. | Unknown IPC commands return zero through the generic IPC fallback. This may satisfy callers that ignore the result, but is not account emulation. | Account profile queries, account identity returned to guest code, account-to-save binding, and a guest account HLE implementation are not evidenced. | PS4 `main()` → `sceUserServiceInitialize()` / user enumeration; guest service-manager request → IPC service dispatch. Do not equate PS4 controller-user assignment with BOTW save ownership. See [services.md](./services.md#save-data-and-user-services). |
| `nn::oe` | **Narrow HLE implemented** | BOTW SDK calls to `TryPopLaunchParameter` and `TryPopNotificationMessage` are registered. | Both return `false` (“nothing popped”); no applet storage or notification payload is created. | Launch-parameter delivery, applet lifecycle, focus/suspend/resume, and notification handling are not implemented by these registrations. | Guest imported call → `runtime/src/hle_oe.cpp` → false result. The patch comments cite BOTW’s boot abort when this path was not handled. See [the `nn::oe` patch](../../patches/botw-ps4.patch#L1456) and [startup evidence](./architecture.md#verified-startup-chain). |
| `nn::socket` | **Not established as a guest service** | PS4 host DNS wrappers and an optional read-only SMB/NFS `/net` VFS exist. | The optional network filesystem is a host libc wrapper, not `nn::socket`; no direct guest `nn::socket` import path is evidenced. | Guest socket API imports, network service IPC, protocol behavior, and network lifecycle are **UNVERIFIED**. | Host calls may use `port/ps4_net.cpp` resolver wrappers or `port/ps4_netfs.cpp` libc wrappers. These are not callers/callees of a proven guest `nn::socket` HLE. See [services.md](./services.md#nintendo-service-dependency-inventory). |
| `nn::vi` | **Partial display HLE implemented** | Guest layer/native-window setup and a vsync event are represented. The PS4 surface bridge returns a headless Vulkan surface; its WSI maps Vulkan swapchain images to VideoOut buffers. | The PS4 output is fixed at 1920×1080; the platform has no ordinary resizable desktop window. Vsync is a host-generated event, not the display service’s full timing/mode API. Catch-up behavior is optional and disabled by default because it was associated with a guest race. | Full display modes, display hotplug, resize, focus, and all VI method semantics are not established. | Guest `nn::vi` layer setup → VI HLE/native-window handle → NVN/Vulkan surface → `ps4_vk_wsi.cpp` → `sceVideoOutRegisterBuffers` and flip submission. See [graphics.md](./graphics.md#ps4-wsi) and [the VI patch](../../patches/botw-ps4.patch#L1637). |

## Cross-service boundaries

- **Filesystem is not an asset manager.** The VFS proves guest path access and
  archive bytes; it does not prove a general BFRES/BNTX/AAMP/BYML loader,
  resource cache, or actor/material lifetime.
- **Audio is two layers.** `nn::audio` runs the guest SDK renderer/DSP path;
  `sceAudioOut` is only the PS4 PCM sink. Replacing the sink does not replace
  the guest renderer contract.
- **Input is host-polled.** `nn::hid` consumes a platform snapshot; the guest
  does not call ScePad directly.
- **Account and saves are decoupled in evidence.** The same runtime has PS4
  user initialization and guest save mounts, but no established identity edge
  joins them.
- **IPC fallback is not a service implementation.** Unhandled commands return
  zero and are logged at sampled intervals. Treat a command as supported only
  where an explicit handler and behavior are evidenced.

## Evidence references

- [Architecture audit](./architecture.md): startup, module entry, guest/runtime boundary.
- [Graphics audit](./graphics.md): NVN, Vulkan, and presentation details.
- [Services audit](./services.md): filesystem, audio, threading, user/save, and service inventory.
- [PS4 platform implementation patch](../../patches/botw-ps4.patch#L10416).
- [PS4 CRT and libc/pthread shims](../../port/ps4_runtime.cpp).
