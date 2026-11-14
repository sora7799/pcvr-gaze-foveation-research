# Gaze-Driven Foveated Rendering in an EAC-Protected VR Game — Research Notes

**Research documentation only.** This repo records the architecture and findings of a two-machine setup that ran an eye-tracked foveated-rendering overlay against an EAC-protected VR game (VRChat) online. It contains no runnable bypass tooling, no modified game files, and no step-by-step evasion recipe. Not affiliated with VRChat, Epic Games, Easy Anti-Cheat, Valve, PICO, or NVIDIA.

> **Here be dragons.** The setup described here uses an anti-cheat-protected game in an unsupported configuration and modifies the EOS SDK shim on the client side. It likely violates the game's Terms of Service; account action is a real possibility. If you reproduce anything, use an account you can afford to lose. No warranty, no endorsement.

---

## What this is

EyeVRS is a foveated-rendering overlay that reads PICO 4 Pro eye-tracking data over UDP and drives NVIDIA hardware Variable Rate Shading (VRS) in a D3D11 VR game. It works offline against the game's offline build.

Running it against **online rooms** (which ship Easy Anti-Cheat) cannot be done by dropping a proxy DLL into the game directory — EAC rejects modified game files. The split design used for the online experiment:

- **Linux relay machine** runs an unmodified game under Proton on native hardware, which satisfies EAC normally. A relay shim inside that process carries the EOS/EAC handshake on behalf of the Windows machine. The Linux box renders nothing the user sees; it only keeps the anti-cheat session alive.
- **Windows gaming machine** runs the game the user sees, with the PICO headset, SteamVR, and the VRS overlay. Its EOS SDK DLL is swapped for a shim that points at the Linux machine instead of Epic's servers.

## What this is not

- **Not an anti-cheat bypass.** The EAC client on the Linux side is the stock, unmodified one downloaded by the game itself. No EAC patch, no forged result, no replayed token. The server-side check passes because a real EAC session on real non-VM hardware produces a valid token that the relay forwards.
- **Not a game mod.** No game files are modified on either machine. The only client-side change is the EOS SDK shim, adapted from the public open-source project [VRC-Emulator/vrc-eac-emulator](https://github.com/VRC-Emulator/vrc-eac-emulator) (upstream: 0liteyear/vrc-eac-emulator, MIT).
- **Not a polished product.** Research setup: one hardware pair, one game version. Expect to debug.

---

## Architecture

```
┌──────────────────────────────┐         ┌──────────────────────────────┐
│  Linux PC (relay, headless)  │         │  Windows gaming PC (user)     │
│                              │         │                               │
│  Unmodified game             │◄───────►│  Game + PICO headset          │
│  under Proton, native,       │  EOS/AC │  + SteamVR + VRS overlay      │
│  NOT a VM                    │  relay  │  shimmed EOS SDK points at    │
│  ┌────────────────────────┐ │  tcp    │  the Linux machine            │
│  │  EAC client (real)      │ │  7777   │                               │
│  │  relay shim in Wine     │ │  http   │  VRS overlay attached early   │
│  │  system32               │ │  7778   │  in process startup           │
│  └────────────────────────┘ │         │                               │
│  does NOT render for user    │         │                               │
└──────────────────────────────┘         └──────────────────────────────┘
```

Three non-obvious findings made the experiment work; each is explained below.

---

## Finding 1: the EAC VM check (error 208) does not exist on real hardware

Nine rounds were spent trying to defeat error `208: Cannot run under Virtual Machine` inside a VMware Workstation guest — hypervisor-bit masking, DMI spoofing, virtual CD-ROM hiding, GPU device renaming, syscall interception. None worked; two VMware-internal fingerprints could not be removed from inside the guest.

Installing Ubuntu on a **second physical PC** (an old low-power box, no GPU passthrough needed) made EAC load cleanly:

```
[EAC Bootstrapper] Launcher finished with: 301,
'Easy Anti-Cheat successfully loaded in-game'
```

**Error 208 is a virtual-machine-specific detection. On real hardware it does not exist.** The VM-evasion work was aimed at the wrong problem. The relay machine does not need to be powerful — it only carries the anti-cheat handshake, not rendering.

---

## Finding 2: EAC scans the game directory, not Wine's system32

The relay shim is a `version.dll` that Proton loads into the game process. Placing it in the game directory was detected: EAC erased it from disk and the game exited immediately.

The working placement was **inside Wine's `system32` directory**, alongside Proton's own DLLs, with unknown exports forwarded to Proton's original `version.dll` (otherwise the shim recurses into itself), and the game directory kept clean apart from a config file. EAC inspects the game directory for unexpected modules; it does not inspect Wine's system32. With the shim there, the real EAC client initializes, the real anti-cheat session begins, and the relay starts answering EOS requests on tcp 7777 / http 7778.

> Gotcha: Wine's `system32/version.dll` is normally a symlink. Copying over it follows the link and overwrites Proton's own DLL — repair with Steam's "Verify integrity of game files" if you break it.

---

## Finding 3: the VRS overlay has to attach at process-startup time, not after launch

The VRS hook needs to attach to D3D11 at device-creation time. The Unity 6 game creates the D3D11 device 100–300 ms after process start — faster than any "launch the game, then inject" approach can catch.

| Method | Result |
|---|---|
| Proxy DLL in game directory | Game crashes (Unity 6 incompatibility) |
| Manual-map injector after game starts | DLL loads but never initializes — D3D11 device already exists, hooks bind to nothing |
| Suspended process, inject before resume | Process hangs — loader not yet initialized |
| Repeated injection attempts over 15 tries | Misses the window every time |

The working approach attached as a debugger to a suspended process, mapped the VRS DLL in the window between loader initialization and the first D3D11 object creation, then detached. The timing window, not the injection mechanism, was the hard part.

---

## Verified end-to-end result

On a PICO 4 Pro + Windows 11 (RTX 3080) driving an old Linux box over LAN:

| Layer | Status |
|---|---|
| Real EAC session on Linux (unmodified game, native hardware) | ✅ `AntiCheat session successfully began` |
| Windows-side EOS login via relay | ✅ `AntiCheat Session Begin: Success` |
| Connected to master, joined a network room | ✅ `Finished entering world` |
| SteamVR + PICO headset rendering on Windows | ✅ |
| VRS DLL injected, NVAPI initialized, SRRV bound | ✅ `bound=1`, 4736×2368 VR render target |
| Eye-tracking feed (PICO direct UDP 29765) | ✅ gaze coordinates flowing into the VRS pattern |

### Known remaining issues (VRS tuning, not relay/anti-cheat)

- **Horizontal tracking works** in a vertical-strip pattern that spans the full viewport height.
- **Vertical mapping is truncated.** The driver samples only part of the shading-rate texture, so a circular foveated region lands offset vertically; the strip mode that spans top-to-bottom is the only usable pattern.
- **The hook can drop after entering a world**, and the in-game hot toggle to restore it has a known issue.
- The relay and online layers themselves are stable; the open problems are all on the gaze-to-pattern mapping side.

---

## Practical notes

- **The same Steam account on two machines kicks the Linux session off** when Windows launches the game. This is fine: the Linux game process keeps running, EAC stays loaded, the relay keeps answering. Boot Linux first, wait until it reaches a world, then launch on Windows.
- **The Linux machine must stay on and keep the game process alive.** If it dies, the relay port closes and the Windows side loses its anti-cheat session.
- **No proxy DLL in the Windows game directory.** Unity 6 crashes with one there; the VRS overlay attaches exclusively at process startup.
- Steam download on the old Linux box was slow because DNS resolved to a European CDN node; routing to a faster edge fixed it. This is a generic Steam-on-old-hardware issue, not specific to this setup.

---

## What is deliberately NOT in this repo

- No relay `version.dll`, no debugger injector, no manual-map loader, no VRS DLL.
- No modified EOS SDK DLL.
- No game files, no EAC modules, no Epic binaries.
- No shell scripts, no launch recipe.
- The VMware VM-evasion experiments are summarized as a failure table; the diagnostic scripts behind them are not included.

The reason is twofold. First, shipping runnable anti-cheat-evasion tooling carries platform and legal risk. Second, this is a single-hardware, single-game-version research setup — the architecture and the three non-obvious findings above are the reusable knowledge; the specific binaries are not.

Separately, a public binary bundle for two anti-cheat-free PCVR games (Hyper Dash, DCS World) is published only as a GitHub Release attachment; this source tree itself contains no runnable components.

---

## Scope and limits

- **One game (VRChat), one anti-cheat (EAC via EOS), one headset (PICO 4 Pro).** Other EAC games have different checks and may not behave the same way.
- **One hardware pair.** The relay box is an old Ubuntu 22.04 + Proton 9.0 Beta machine with a weak iGPU; it works because the relay path needs no GPU acceleration.
- **The VRS overlay is not yet a clean in-world foveation experience.** Injection and binding succeed; the gaze-to-pattern mapping still has the vertical-mapping and hook-drop issues listed above.
- **Use a burner account.** Even though EAC runs unmodified on the Linux side, the Windows side uses a shimmed EOS SDK, which is detectable if the game chooses to inspect it.
- **This is offline research documentation.** Do not attach a debugger or `strace` to the running game process on the Linux side — EAC will report it.

---

## License

MIT for the notes and documentation. The referenced open-source relay-shim project this adapted from — [VRC-Emulator/vrc-eac-emulator](https://github.com/VRC-Emulator/vrc-eac-emulator) (upstream: 0liteyear/vrc-eac-emulator, Copyright 2025 liteyear) — is itself MIT and is not included here.

The author is not affiliated with VRChat, Epic Games, Easy Anti-Cheat, Valve, PICO, or NVIDIA. All trademarks belong to their respective owners.
