# Tribes: Vengeance — RTX Remix

Path-tracing compatibility mod for **Tribes: Vengeance** (2004) using [Remix Plus](https://github.com/RemixProjGroup/dxvk-remix), Kim2091's community fork of NVIDIA [RTX Remix](https://github.com/NVIDIAGameWorks/rtx-remix).

*Tribes: Vengeance* renders most of its world with vertex and pixel shaders, which RTX Remix cannot path-trace directly. This repository provides a `d3d9.dll` proxy—built on the [remix-comp-proxy](https://github.com/xoxor4d/remix-comp-base) framework from the [Vibe Reverse Engineering](https://github.com/Ekozmaster/Vibe-Reverse-Engineering) toolkit—that sits between the game and Remix Plus. It converts those draw calls to Direct3D 9's fixed-function pipeline and patches `Engine.dll` in memory so Remix receives the complete scene with stable meshes.

> **Note:** This repository only contains the mod files. No game assets are included—you must supply your own copy of *Tribes: Vengeance*.

---

## Features

- **Accurate world placement:** Level geometry, static meshes, and terrain pass into Remix with their real world transforms.
- **Full level lighting:** Sends every light in the loaded level to Remix, rather than only the handful the engine normally keeps active near the player. Jetpack flames are excluded so they don't leave ghost lights behind.
- **Reduced culling:** Disables frustum, antiportal, and cull-distance checks so off-screen geometry continues to cast shadows and bounce light. Zone/portal visibility stays active, and small meshes beyond `8000` units still use engine culling to protect frame rates.
- **Stable meshes for the Remix Toolkit:** Renders walls, floors, and ceilings as whole BSP sections in a fixed order, draws static meshes one actor at a time, and centers static mesh pivots on their geometry. Asset replacements, lights, and material tags stay attached reliably.
- **Fixed-function terrain:** Blends ground layers via texture stages instead of a pixel shader so Remix's terrain baker can process them.
- **Conflicting effects disabled:** Turns off the legacy glow post-process and projected character shadows (which Remix otherwise renders as opaque planes under characters).

### Known Limitations

- **Early build:** Tested so far on a limited number of maps.
- **Static water:** Water surfaces are flat and still because their original wave animation relies on a vertex shader.
- **No replacement assets:** No RTX Remix Toolkit mod or upgraded textures/models are included.

---

## Requirements

- **Tribes: Vengeance (installed):** Throughout this guide, **"the game folder"** refers to `<InstallDir>\Program\Bin` (the folder containing `TV_CD_DVD.exe`).
- **Remix Plus v2.0.0:** Download [`Remix_Plus_v2.0.0.zip`](https://github.com/RemixProjGroup/dxvk-remix/releases/tag/remix-plus-2.0.0). This port was specifically tested against v2.0.0.
- **Compatible hardware:** A GPU and driver supported by Remix Plus.

### Tested Game Build

The proxy hooks and patches `Engine.dll` and `D3DDrv.dll` in memory. It has been tested against these exact binaries:

| File | Size | SHA-256 |
|---|---|---|
| `Engine.dll` | 4,800,512 bytes | `E85CA1A1F5795CAF3506B150E00D7969DEAA894FDAB54CD4221F9E9762267590` |
| `D3DDrv.dll` | 540,672 bytes | `B042C78B130BD7E164A07D1AD453DC8407AF75911ACCE245CAF5E38D5FAFE26F` |

To verify your files, run this command in PowerShell from the game folder:

```powershell
Get-FileHash Engine.dll, D3DDrv.dll
