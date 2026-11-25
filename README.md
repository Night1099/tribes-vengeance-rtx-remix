# Tribes: Vengeance — RTX Remix

Path-tracing compatibility mod for **Tribes: Vengeance** (2004) using NVIDIA [RTX Remix](https://github.com/NVIDIAGameWorks/rtx-remix).

*Tribes: Vengeance* renders most of its world with vertex and pixel shaders, which RTX Remix cannot path-trace directly. This repository provides a `d3d9.dll` proxy—built on the [remix-comp-proxy](https://github.com/xoxor4d/remix-comp-base) framework from the [Vibe Reverse Engineering](https://github.com/Ekozmaster/Vibe-Reverse-Engineering) toolkit—that sits between the game and Remix. It converts those draw calls to Direct3D 9's fixed-function pipeline and patches `Engine.dll` in memory so Remix receives the complete scene with stable meshes.

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
- **RTX Remix runtime:** Tested with Remix Plus v2.0.0 ([`Remix_Plus_v2.0.0.zip`](https://github.com/RemixProjGroup/dxvk-remix/releases/tag/remix-plus-2.0.0)), which the installation steps below use.
- **Compatible hardware:** A GPU and driver supported by RTX Remix.

### Tested Game Build

The proxy hooks and patches `Engine.dll` and `D3DDrv.dll` in memory. It has been tested against these exact binaries:

| File | Size | SHA-256 |
|---|---|---|
| `Engine.dll` | 4,800,512 bytes | `E85CA1A1F5795CAF3506B150E00D7969DEAA894FDAB54CD4221F9E9762267590` |
| `D3DDrv.dll` | 540,672 bytes | `B042C78B130BD7E164A07D1AD453DC8407AF75911ACCE245CAF5E38D5FAFE26F` |

To verify your files, run this command in PowerShell from the game folder:

```powershell
Get-FileHash Engine.dll, D3DDrv.dll
```

Other builds are untested. The proxy checks the bytes or the build stamp before most of its patches and skips any that don't match (see [Troubleshooting](#troubleshooting)).

---

## Installation

1. **Back up the game folder.**
2. **Install the Remix runtime:** Open `Remix_Plus_v2.0.0.zip` and copy everything inside its `game folder` into the game folder:
   - `d3d9.dll` — the Remix bridge
   - `.trex\` — the Remix runtime. If the game folder already has a `.trex` folder, delete it first instead of merging into it.
3. **Rename the bridge:** In the game folder, rename `d3d9.dll` (the one you just copied, about 1 MB) to **`d3d9_remix.dll`**. Leave `.trex\d3d9.dll` as it is.
4. **Install this mod:** Copy the four files from this repo's [`Bin`](Bin) folder into the game folder (use **Code → Download ZIP**, or open each file and use **Download raw file**):

   | File | Purpose |
   |---|---|
   | `d3d9.dll` | The proxy |
   | `remix-comp-proxy.ini` | Proxy settings |
   | `rtx.conf` | Remix settings for this game: world orientation and scale, texture tags (sky, UI, decals, particles, lightmaps, terrain) |
   | `user.conf` | *Optional:* the author's graphics settings. Leave it out to start from the runtime's own defaults |

5. **Disable static mesh batching:** Open `TV_CD_DVD.ini` in the game folder (the INI named after `TV_CD_DVD.exe`; the game writes it on first launch). Under `[Engine.GameEngine]`, add or change this line:

   ```ini
   UseStaticMeshBatching=False
   ```

   Without it, the engine merges static meshes into large batches that Remix cannot place or replace.
6. **Recommended — smoother mouse:** In the same `TV_CD_DVD.ini`, under `[D3DDrv.D3DRenderDevice]`, set `ReduceMouseLag=False`. With `True`, the renderer stalls every frame to sync with the GPU, which costs several milliseconds per frame through Remix. In `User.ini` in the game folder, add:

   ```ini
   [Engine.PlayerInput]
   MouseSmoothingMode=0
   MouseAccelThreshold=0.0
   ```

Besides the game's own files, the game folder now contains:

```
Program\Bin\
  d3d9.dll               this proxy
  d3d9_remix.dll         the Remix bridge (renamed from its d3d9.dll)
  .trex\                 the Remix runtime
  remix-comp-proxy.ini
  rtx.conf
  user.conf              (optional)
```

---

## Playing

- Launch `TV_CD_DVD.exe` as usual. The first launch compiles Remix's shaders and can take several minutes; this happens once per Remix version.
- A console window titled **RTX-Comp Debug Console** opens alongside the game and shows the proxy's log. Leave it open: closing it can end the game.
- **Alt+X** opens the Remix menu. **F4** opens the proxy's debug overlay.

---

## Configuration

`remix-comp-proxy.ini` is read when the game starts; every key is described in the file. The main ones:

| Section | Key | Shipped | Meaning |
|---|---|---|---|
| `[Engine]` | `DisableGlow` | 1 | Glow post-process off. Keep it at 1 while any `[Culling]` switch is 1 |
| `[Engine]` | `DisableDynamicShadows` | 1 | Projected character shadows off |
| `[Engine]` | `WholeBspSections` | 1 | Level geometry drawn whole, in a fixed order |
| `[Engine]` | `FixedFunctionTerrain` | 1 | Terrain layers blended on texture stages, for Remix's terrain baker |
| `[Meshes]` | `CentrePivots` | 1 | Each static mesh centred on its geometry |
| `[Culling]` | `FullScene` | 0 | 1 = the whole level every frame. Use it for Remix Toolkit captures; costs a lot of frame rate on large maps |
| `[Culling]` | `DisableFrustum`, `DisableDistance` | 1, 1 | Engine frustum, antiportal and distance culling off |
| `[Culling]` | `KeepRadius`, `KeepAngularSize` | 8000, 0.05 | Mesh actors drawn regardless of visibility: those within this many units, and larger ones farther away |
| `[Lights]` | `Enabled`, `Intensity`, `IgnoreClasses` | 1, 2.0, Jetpack | Level lights sent to Remix, their brightness, and the actor classes left out |

> **For Toolkit mod authors:** `WholeBspSections`, `CentrePivots` and `UseStaticMeshBatching` each change the mesh hashes Remix sees. Keep them as shipped, both when capturing and when playing, so replacements made by others apply.

---

## Troubleshooting

Logs, in the game folder:

- `rtx_comp\console.log` — the proxy (same text as the console window)
- `rtx-remix\logs\remix-dxvk.log` — Remix

| Symptom | Check |
|---|---|
| The game runs, but without Remix; `console.log` says `Remix enabled but could not load 'd3d9_remix.dll'` | `d3d9_remix.dll` must be in the game folder (installation step 3). Without it the proxy falls back to Windows' own Direct3D 9. |
| `console.log` says `Engine.dll is not retail 1.0.1; ... left on` or `... BSP sections not hooked` | Your `Engine.dll` is a different build from the one tested (see [Tested Game Build](#tested-game-build)). The named patch is skipped and the game runs without it. |
| `console.log` says `D3DDrv.dll has timestamp ..., expected ...` | Your `D3DDrv.dll` is a different build from the one tested. The proxy does not hook it; the port takes the engine's transforms from that hook and has not been tested without it. |
| The proxy disappeared after reinstalling or updating Remix | Remix's `d3d9.dll` replaced the proxy. Rename it to `d3d9_remix.dll` again and copy this repo's `d3d9.dll` back. The bridge and the `.trex` folder must always come from the same Remix release. |
| The game crashes at start | Set `[Remix] Enabled=0` in `remix-comp-proxy.ini` to start the game with the proxy but without Remix. That shows whether Remix or the proxy is the cause. |

For a diagnostic dump of the proxy's draw calls, press **F4** and use the **Diagnostics** tab; it writes `rtx_comp\diagnostics.log`.

---

## Uninstall

Restore the backup from installation step 1. Otherwise, delete these from the game folder: `d3d9.dll`, `d3d9_remix.dll`, `.trex`, `remix-comp-proxy.ini`, `rtx.conf`, `user.conf`, `rtx_comp`, and `rtx-remix` (which also holds any captures or mods you made). Then remove the `UseStaticMeshBatching=False` line from `TV_CD_DVD.ini`. The mouse settings from step 6 can stay.

---

## Credits

| Who | What |
|---|---|
| [xoxor4d](https://github.com/xoxor4d) | [remix-comp-base](https://github.com/xoxor4d/remix-comp-base): D3D9 proxy architecture, ImGui integration, module system |
| [kim2091](https://github.com/kim2091) | remix-comp-proxy: fixed-function conversion, diagnostics, tracer, INI config. [Remix Plus](https://github.com/RemixProjGroup/dxvk-remix), the RTX Remix runtime this port was tested with |
| [momo5502](https://github.com/momo5502) | The codebase remix-comp-base was built on |
| [Ekozmaster](https://github.com/Ekozmaster/Vibe-Reverse-Engineering) | The Vibe Reverse Engineering toolkit |
| NVIDIA | [RTX Remix](https://github.com/NVIDIAGameWorks/rtx-remix) |

The Tribes: Vengeance port (engine patches, BSP section drawing, terrain, level lights, mesh centring): this repo.

---

## License

The proxy is MIT licensed: see [LICENSE](LICENSE). `d3d9.dll` includes MinHook, Dear ImGui and the RTX Remix bridge API headers; their licenses are in [THIRD_PARTY_LICENSES.md](THIRD_PARTY_LICENSES.md).

The Remix runtime is not included here; it has its own licenses. *Tribes: Vengeance* is a trademark of its owners. This project is not affiliated with or endorsed by them, or by NVIDIA.
