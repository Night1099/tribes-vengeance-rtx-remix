# Tribes: Vengeance — RTX Remix

Path-traced **Tribes: Vengeance** (2004) through [Remix Plus](https://github.com/RemixProjGroup/dxvk-remix), Kim2091's community fork of NVIDIA [RTX Remix](https://github.com/NVIDIAGameWorks/rtx-remix).

The game draws most of its world with vertex and pixel shaders, which Remix cannot path trace. The `d3d9.dll` in this repo is a proxy that sits between the game and Remix Plus. It redraws those draws through Direct3D 9's fixed-function pipeline and patches the game's `Engine.dll` in memory, so that Remix receives the whole scene with stable meshes. It is built on the [remix-comp-proxy](https://github.com/xoxor4d/remix-comp-base) framework from the [Vibe Reverse Engineering](https://github.com/Ekozmaster/Vibe-Reverse-Engineering) toolkit.

This repo holds only the files a player needs. It contains no game files: you need your own copy of Tribes: Vengeance.

## What the port does

- **World in place:** level geometry, static meshes and terrain reach Remix with their real world transforms.
- **Level lights:** every light in the loaded level is sent to Remix, not only the few the engine keeps near the player. Jetpack flames are left out, so they do not leave lights behind.
- **Less culling:** the engine's frustum, antiportal and cull-distance tests are off, so geometry behind the camera still casts shadows and bounce light. Zone and portal visibility stays on, and small mesh actors more than 8000 units away are left to the engine's own visibility, to keep the frame rate up.
- **Stable meshes for the Remix Toolkit:** walls, floors and ceilings are drawn a whole BSP section at a time in a fixed order, static meshes are drawn one actor at a time, and static meshes are centred on their geometry. Replacements, lights and material tags attached in the Toolkit stay attached.
- **Terrain:** the ground layers are blended on the texture stages instead of in a pixel shader, so Remix's terrain baker can blend them.
- **Engine effects Remix has no use for are off:** the glow post-process and the projected character shadows, which Remix showed as an opaque plane under every character.

Known limits:

- This is an early build, tested by the author on a small number of maps.
- Water is flat and still, because its waves were made in a vertex shader.
- No Remix Toolkit mod (replacement assets) is included.

## Requirements

- **Tribes: Vengeance**, installed. The game's own files are in `Program\Bin` of the install folder, next to `TV_CD_DVD.exe`. Below, "the game folder" means that `Program\Bin` folder.
- **Remix Plus v2.0.0**: [download `Remix_Plus_v2.0.0.zip`](https://github.com/RemixProjGroup/dxvk-remix/releases/tag/remix-plus-2.0.0). This port was tested with exactly this version.
- A GPU and driver that Remix Plus supports.

The proxy hooks and patches the game's `Engine.dll` and `D3DDrv.dll` in memory. It was tested with these two files:

| File | Size | SHA-256 |
|---|---|---|
| `Engine.dll` | 4,800,512 bytes | `E85CA1A1F5795CAF3506B150E00D7969DEAA894FDAB54CD4221F9E9762267590` |
| `D3DDrv.dll` | 540,672 bytes | `B042C78B130BD7E164A07D1AD453DC8407AF75911ACCE245CAF5E38D5FAFE26F` |

Check yours in PowerShell, from the game folder: `Get-FileHash Engine.dll, D3DDrv.dll`. Other builds are untested. The proxy checks the bytes or the build stamp before most of its patches and skips any that do not match (see [Troubleshooting](#troubleshooting)).

## Install

1. **Back up the game folder** (`Program\Bin`).
2. **Install Remix Plus.** Open `Remix_Plus_v2.0.0.zip` and copy everything inside its `game folder` into the game folder:
   - `d3d9.dll`, the Remix Plus bridge
   - the `.trex` folder, the Remix Plus runtime. If the game folder already has a `.trex` folder, delete it first instead of merging into it.
3. **Rename the bridge.** In the game folder, rename `d3d9.dll` (the one you just copied, about 1 MB) to **`d3d9_remix.dll`**. Leave `.trex\d3d9.dll` as it is.
4. **Install this port.** Copy the four files from this repo's [`Bin`](Bin) folder into the game folder:

   | File | What it is |
   |---|---|
   | `d3d9.dll` | the proxy |
   | `remix-comp-proxy.ini` | the proxy's settings |
   | `rtx.conf` | Remix settings for this game: world orientation and scale, texture tags (sky, UI, decals, particles, lightmaps, terrain) |
   | `user.conf` | optional: the author's graphics settings (Remix menu). Leave it out to start from Remix Plus's own defaults, which pick a preset for your GPU |

   To download them, use **Code → Download ZIP** on this page, or open each file and use **Download raw file**.
5. **Turn off static mesh batching.** Open `TV_CD_DVD.ini` in the game folder. It is the INI named after `TV_CD_DVD.exe`, and the game writes it the first time it starts; other `.ini` files in the folder are not read for this. Under `[Engine.GameEngine]`, add this line, or change it if it is already there:

   ```ini
   UseStaticMeshBatching=False
   ```

   Without it the engine merges static meshes into large batches that Remix cannot place or replace.
6. **Recommended: smoother mouse.** In the same `TV_CD_DVD.ini`, under `[D3DDrv.D3DRenderDevice]`, set `ReduceMouseLag=False`. With `True`, the renderer stalls every frame to sync with the GPU, which costs several milliseconds a frame through Remix. In `User.ini`, in the game folder, add:

   ```ini
   [Engine.PlayerInput]
   MouseSmoothingMode=0
   MouseAccelThreshold=0.0
   ```

The game folder now holds, besides the game's own files:

```
Program\Bin\
  d3d9.dll               this proxy
  d3d9_remix.dll         the Remix Plus bridge (renamed from its d3d9.dll)
  .trex\                 the Remix Plus runtime
  remix-comp-proxy.ini
  rtx.conf
  user.conf              (optional)
```

## Playing

- Start `TV_CD_DVD.exe` as usual. The first start with Remix Plus compiles shaders and can take several minutes. That happens once per Remix Plus version.
- A console window titled **RTX-Comp Debug Console** opens with the game. It shows the proxy's log. Leave it open: closing it can end the game.
- **Alt+X** opens the Remix menu. **F4** opens the proxy's debug overlay.

## Configuration

`remix-comp-proxy.ini` is read when the game starts; each key is described in the file. The main ones:

| Section | Key | Shipped | Meaning |
|---|---|---|---|
| `[Engine]` | `DisableGlow` | 1 | glow post-process off. Keep it at 1 while any `[Culling]` switch is 1 |
| `[Engine]` | `DisableDynamicShadows` | 1 | projected character shadows off |
| `[Engine]` | `WholeBspSections` | 1 | level geometry drawn whole, in a fixed order |
| `[Engine]` | `FixedFunctionTerrain` | 1 | terrain layers blended on texture stages, for Remix's terrain baker |
| `[Meshes]` | `CentrePivots` | 1 | each static mesh centred on its geometry |
| `[Culling]` | `FullScene` | 0 | 1 = the whole level every frame. Use it for Remix Toolkit captures; it costs a lot of frame rate on large maps |
| `[Culling]` | `DisableFrustum`, `DisableDistance` | 1, 1 | engine frustum, antiportal and distance culling off |
| `[Culling]` | `KeepRadius`, `KeepAngularSize` | 8000, 0.05 | mesh actors drawn regardless of visibility: those within this many units, and larger ones farther away |
| `[Lights]` | `Enabled`, `Intensity`, `IgnoreClasses` | 1, 2.0, Jetpack | level lights sent to Remix, their brightness, and the actor classes left out |

**For Toolkit mod authors:** `WholeBspSections`, `CentrePivots` and `UseStaticMeshBatching` each change the mesh hashes Remix sees. Keep them as shipped, both when capturing and when playing, so that replacements made by others apply.

## Troubleshooting

Logs, in the game folder:

- `rtx_comp\console.log`: the proxy (the same text as the console window)
- `rtx-remix\logs\remix-dxvk.log`: Remix Plus

| Symptom | Check |
|---|---|
| The game runs, but without Remix; `console.log` says `Remix enabled but could not load 'd3d9_remix.dll'` | `d3d9_remix.dll` must be in the game folder (install step 3). Without it the proxy falls back to Windows' own Direct3D 9. |
| `console.log` says `Engine.dll is not retail 1.0.1; ... left on` or `... BSP sections not hooked` | Your `Engine.dll` is a different build from the one tested (see [Requirements](#requirements)). The named patch is skipped and the game runs without it. |
| `console.log` says `D3DDrv.dll has timestamp ..., expected ...` | Your `D3DDrv.dll` is a different build from the one tested. The proxy does not hook it. The port takes the engine's transforms from that hook and has not been tested without it. |
| Everything worked, then the proxy disappeared after reinstalling or updating Remix Plus | Remix Plus's `d3d9.dll` replaced the proxy. Rename it to `d3d9_remix.dll` again and copy this repo's `d3d9.dll` back. The bridge and the `.trex` folder must always come from the same Remix Plus release. |
| The game crashes at start | Set `[Remix] Enabled=0` in `remix-comp-proxy.ini` to start the game with the proxy but without Remix. That shows whether Remix Plus or the proxy is the cause. |

For a diagnostic dump of the proxy's draw calls, press F4 and use the **Diagnostics** tab. It writes `rtx_comp\diagnostics.log`.

## Uninstall

Restore the backup from install step 1. Otherwise, delete these from the game folder: `d3d9.dll`, `d3d9_remix.dll`, `.trex`, `remix-comp-proxy.ini`, `rtx.conf`, `user.conf`, `rtx_comp`, and `rtx-remix`, which also holds any captures or mods you made. Then remove the `UseStaticMeshBatching=False` line from `TV_CD_DVD.ini`. The mouse settings from install step 6 can stay.

## Credits

| Who | What |
|---|---|
| [xoxor4d](https://github.com/xoxor4d) | [remix-comp-base](https://github.com/xoxor4d/remix-comp-base): D3D9 proxy architecture, ImGui integration, module system |
| [kim2091](https://github.com/kim2091) | remix-comp-proxy: fixed-function conversion, diagnostics, tracer, INI config. [Remix Plus](https://github.com/RemixProjGroup/dxvk-remix) |
| [momo5502](https://github.com/momo5502) | the codebase remix-comp-base was built on |
| [Ekozmaster](https://github.com/Ekozmaster/Vibe-Reverse-Engineering) | the Vibe Reverse Engineering toolkit |
| NVIDIA | [RTX Remix](https://github.com/NVIDIAGameWorks/rtx-remix) |

The Tribes: Vengeance port (engine patches, BSP section drawing, terrain, level lights, mesh centring): this repo.

## License

The proxy is MIT licensed: see [LICENSE](LICENSE). `d3d9.dll` includes MinHook, Dear ImGui and the RTX Remix bridge API headers; their licenses are in [THIRD_PARTY_LICENSES.md](THIRD_PARTY_LICENSES.md).

Remix Plus is not included here; it has its own licenses. Tribes: Vengeance is a trademark of its owners. This project is not affiliated with or endorsed by them, or by NVIDIA.
