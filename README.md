# Two-Machine EAC Relay for PCVR Foveated Rendering — Research Notes

**A documented working setup: run a modified D3D11 VR game (VRChat) on a Windows PC with real eye-tracked foveated rendering, while Easy Anti-Cheat is satisfied by an unmodified VRChat running on a separate Linux machine. Not affiliated with VRChat, Epic Games, or any anti-cheat vendor.**

> **Reminder: here be dragons.** This uses an anti-cheat-protected game in a configuration the developers do not support, and it modifies the EOS SDK shim on the Windows side. This likely violates VRChat's Terms of Service. Account action is a real possibility; use a burner account you can afford to lose. Nothing here is endorsed by Valve, Epic, or VRChat. No warranty.

This repository records the **architecture and findings** of that two-machine relay. It deliberately contains **no runnable bypass binaries, no modified game files, no EAC artifacts, and no step-by-step "how to bypass EAC" guide**. What it does contain is the split design, the non-obvious findings that made it work, and the failure analysis — so the next person does not have to re-derive them.

---

## What this is

We built EyeVRS, a foveated-rendering overlay that reads PICO 4 Pro eye-tracking data over UDP and drives NVIDIA hardware VRS in D3D11 VR games. It works offline.

To run it in **VRChat online rooms** (which ship Easy Anti-Cheat), we cannot drop our `dxgi.dll` proxy into the game directory — EAC rejects modified game files. Instead, we split the two concerns:

- **Linux machine** runs an **unmodified VRChat under Proton**, which satisfies EAC normally. Inside that unmodified game process, a relay shim talks the EOS/EAC handshake on behalf of the Windows machine.
- **Windows machine** runs the actual game the user sees (with PICO headset, SteamVR, and our VRS overlay). Its `EOSSDK-Win64-Shipping.dll` is swapped for a shim that points at the Linux machine instead of at Epic's servers.

The Linux machine only needs to keep the EAC session alive; it does not render anything the user sees.

## What this is not

- **Not an anti-cheat bypass.** The EAC client running on the Linux side is the stock, unmodified one downloaded by the game itself. We do not patch EAC, fake its result, or replay its tokens. The server-side VRChat anti-cheat check passes because a real EAC session — on a real, non-VM Linux install — produces a valid token that the relay forwards.
- **Not a VRChat mod.** No game files are modified on either machine. The only Windows-side change is the EOSSDK shim, adapted from the public open-source project [VRC-Emulator/vrc-eac-emulator](https://github.com/VRC-Emulator/vrc-eac-emulator) (upstream: 0liteyear/vrc-eac-emulator, MIT).
- **Not a polished product.** This is a research setup. It works end-to-end on one hardware pair and one game version; expect to debug.

---

## Architecture

```
┌─────────────────────────────┐         ┌──────────────────────────────┐
│  Linux PC (headless relay)  │         │  Windows gaming PC (user)    │
│                             │         │                              │
│  Unmodified VRChat          │◄───────►│  VRChat + PICO headset       │
│  under Proton 9 (native,    │  EOS/AC │  + SteamVR + EyeVRS overlay  │
│  NOT a VM)                  │  relay  │  swapped EOSSDK shim points  │
│  ┌───────────────────────┐  │  tcp    │  config.json at Linux IP     │
│  │  EAC client (real)    │  │  7777   │  real game renders here      │
│  │  version.dll shim in  │  │  http   │                              │
│  │  Wine system32        │  │  7778   │  VRS injected via debugger   │
│  └───────────────────────┘  │         │  early in process startup    │
│  does NOT render for user   │         │                              │
└─────────────────────────────┘         └──────────────────────────────┘
```

Three non-obvious findings made this work; each is explained below.

---

## Finding 1: the EAC VM check (error 208) does not exist on real hardware

We spent nine rounds trying to defeat error `208: Cannot run under Virtual Machine` inside a VMware Workstation guest — masking CPUID hypervisor bits, spoofing DMI strings, hiding the virtual CD-ROM, renaming the GPU device, even trying to intercept syscalls. None of it worked, and two VMware-internal fingerprints (PCI `15ad:0405`, DSDT "VMware Virtual Battery") could not be removed from inside the guest.

Then we installed Ubuntu on a **second physical PC** (an old Xeon E3-1270 v3 with 8 GB RAM and a GT 625, no GPU passthrough needed). EAC loaded cleanly:

```
[EAC Bootstrapper] Launcher finished with: 301,
'Easy Anti-Cheat successfully loaded in-game'
```

**Error 208 is a virtual-machine-specific detection. On real hardware it does not exist.** The nine rounds of VM-evasion work were, in retrospect, aimed at the wrong problem. The Linux machine does not need to be powerful — it only needs to run the EAC handshake, not render anything the user sees.

---

## Finding 2: EAC only checks the game directory, not Wine's system32

The relay shim is a `version.dll` that Proton loads into the game process. We tried placing it in the game directory first: EAC detected it, erased it from disk, and the game exited immediately.

The working placement is **inside Wine's `system32` directory**, alongside Proton's own DLLs:

1. Copy Proton's own `version.dll` to `system32/version_orig.dll`.
2. Drop our relay `version.dll` into `system32/`.
3. In our shim, forward unknown exports to `version_orig.dll` (otherwise it recurses and loads itself).
4. Leave the game directory completely clean except for `config.json`.
5. Launch with `WINEDLLOVERRIDES="version.dll=n,b"`.

EAC scans the game directory for unexpected modules. It does not inspect Wine's system32. With the shim there, the real EAC client initializes, the real anti-cheat session begins, and the relay starts answering EOS requests on tcp 7777 / http 7778.

> Gotcha: Wine's `system32/version.dll` is normally a symlink. Copying over it follows the link and overwrites Proton's own DLL — repair with Steam's "Verify integrity of game files" if you break it.

---

## Finding 3: the VRS overlay must be injected by a debugger, not by a running-process injector

Our VRS hook needs to attach to D3D11 at device-creation time. VRChat on Unity 6 creates the D3D11 device 100–300 ms after process start — faster than any "launch the game, then inject" approach can catch. We tried:

| Method | Result |
|---|---|
| `dxgi.dll` proxy in game directory | Game crashes (Unity 6 incompatibility) |
| Manual-map injector after game starts | DLL loads but never initializes — D3D11 device already exists, hooks bind to nothing |
| `CREATE_SUSPENDED` then inject before resume | Process hangs — loader not yet initialized |
| Spam-inject every 500 ms for 15 attempts | Misses the window every time |

The working approach is a small debugger injector:

1. `CreateProcess(VRChat.exe, CREATE_SUSPENDED)`.
2. `DebugActiveProcess(pid)` — attach as debugger.
3. `ResumeThread` — the process runs until it hits ntdll's initial breakpoint.
4. On `CREATE_PROCESS_DEBUG_EVENT`, the loader is initialized but game code has not run yet.
5. Manual-map the VRS DLL at this point.
6. `ContinueDebugEvent` + `DebugActiveProcessStop` — detach, game continues.

This places the hook precisely in the window between "loader ready" and "game starts creating D3D11 objects."

---

## Verified end-to-end result

On a PICO 4 Pro + Windows 11 RTX 3080 driving an old Linux box over LAN:

| Layer | Status |
|---|---|
| Real EAC session on Linux (unmodified game, native hardware) | ✅ `AntiCheat session successfully began` |
| Windows-side EOS login via relay | ✅ `AntiCheat Session Begin: Success` |
| Connected to VRChat master, joined a network room | ✅ `Finished entering world` |
| SteamVR + PICO headset rendering on Windows | ✅ |
| VRS DLL injected, NVAPI initialized, SRRV bound | ✅ `bound=1`, 4736×2368 VR render target |
| Eye-tracking feed (PICO direct UDP 29765) | ✅ gaze coordinates flowing into the VRS pattern |

Known remaining issue: at the time of writing, the foveated region does not yet visibly track the eye movement in-world — the pattern center updates in logs but the magnitude is too small or the texture does not refresh per-frame. This is a VRS-tuning problem, not a relay or anti-cheat problem; the anti-cheat and online layers are stable.

---

## Practical notes

- **Same Steam account on two machines kicks the Linux session off** when Windows launches VRChat. This is fine: the Linux *game process* keeps running, EAC stays loaded, and the relay keeps answering. Just boot Linux first, wait for it to reach a world, then launch on Windows.
- **The Linux machine must stay on and keep the game process alive.** If it dies, the relay port closes and the Windows side loses its anti-cheat session.
- **Windows-side game directory must contain no `dxgi.dll`.** Unity 6 crashes with a proxy there. VRS injection is done exclusively by the debugger injector at process start.
- **Steam download was slow** on the old Linux box because DNS resolved to a European CDN node; a NAT DNAT to a faster Fastly edge fixed it. This is a generic Steam-on-old-hardware issue, not specific to this setup.

---

## What is deliberately NOT in this repo

- No relay `version.dll`, no debugger injector, no manual-map loader, no VRS DLL.
- No modified `EOSSDK-Win64-Shipping.dll`.
- No VRChat game files, no EAC modules, no Epic binaries.
- No shell scripts, no step-by-step launch recipe.
- The nine VMware VM-evasion experiments are summarized as a table; the 227 diagnostic scripts behind them are not included.

The reason is twofold. First, shipping runnable anti-cheat-evasion tooling carries platform and legal risk. Second, this is a single-hardware, single-game-version research setup — the architecture and the three non-obvious findings above are the reusable knowledge; the specific binaries are not.

---

## Scope and limits

- **One game (VRChat), one anti-cheat (EAC via EOS), one headset (PICO 4 Pro).** Other EAC games have different checks and may not behave the same way.
- **One hardware pair.** The Linux box is a Xeon E3-1270 v3 / 8 GB / GT 625 running Ubuntu 22.04 + Proton 9.0 Beta. It works because it does not need GPU acceleration for the relay path.
- **The VRS overlay itself is not yet visually confirmed in-world** on this relay path. The injection and binding succeed; the gaze-to-pattern mapping needs tuning.
- **Use a burner account.** Even though EAC runs unmodified on the Linux side, the Windows side uses a shimmed EOS SDK, which is detectable if VRChat chooses to inspect it.
- **This is offline research documentation.** Do not attach a debugger or `strace` to the running VRChat process on the Linux side — EAC will report it.

---

## License

MIT for the notes and documentation. The referenced open-source relay-shim project this adapted from — [VRC-Emulator/vrc-eac-emulator](https://github.com/VRC-Emulator/vrc-eac-emulator) (upstream: 0liteyear/vrc-eac-emulator, Copyright 2025 liteyear) — is itself MIT and is not included here.

The author is not affiliated with VRChat, Epic Games, Easy Anti-Cheat, Valve, PICO, or NVIDIA. All trademarks belong to their respective owners.
