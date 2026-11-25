# Tribes: Vengeance — RTX Remix

A compatibility mod that brings NVIDIA [RTX Remix](https://github.com/NVIDIAGameWorks/rtx-remix) path tracing to **Tribes: Vengeance** (2004).

Out of the box, *Tribes: Vengeance* uses older vertex and pixel shaders that RTX Remix cannot ray-trace directly. This mod adds a `d3d9.dll` proxy—built on [remix-comp-proxy](https://github.com/xoxor4d/remix-comp-base) and the [Vibe Reverse Engineering](https://github.com/Ekozmaster/Vibe-Reverse-Engineering) toolkit—that sits between the game and Remix. It translates the game's graphics into Direct3D 9's fixed-function pipeline and patches the engine in memory so Remix gets a complete, stable scene.

> **Note:** This repository only contains the mod files. You need your own copy of *Tribes: Vengeance* to play.

---

## What It Does

* **Fixes world placement:** Passes maps, terrain, and 3D objects into Remix at their true world positions.
* **Loads full level lighting:** Sends every light in the map to Remix instead of only the few closest to the player. Jetpack flames are filtered out so they don't leave behind floating ghost lights.
* **Reduces pop-in and missing shadows:** Turns off standard camera and distance culling so off-screen walls and objects still cast shadows and bounce light. To keep frame rates healthy, zone visibility stays active and small objects past `8000` units are still culled.
* **Keeps assets stable for modders:** Draws level geometry in a consistent order, renders static objects individually, and centers object pivots. This ensures RTX Remix texture packs, model replacements, and custom lights stay attached properly.
* **Makes terrain Remix-ready:** Converts ground blending to standard texture stages so Remix's terrain baker can process it.
* **Disables conflicting legacy effects:** Turns off the original glow filter and projected character shadows, which otherwise show up as solid dark boxes under characters in Remix.

### Known Limitations

* **Work in progress:** Only tested on a small selection of maps so far.
* **Still water:** Water surfaces are flat and unmoving because their original wave animation relies on a vertex shader.
* **No HD assets included:** This is a compatibility layer only—upgraded textures and models are not bundled.

---

## Requirements

* **Tribes: Vengeance (installed):** Throughout this guide, **"the game folder"** means `<InstallDir>\Program\Bin` (the folder containing `TV_CD_DVD.exe`).
* **RTX Remix runtime:** Tested with Remix Plus v2.0.0 ([`Remix_Plus_v2.0.0.zip`](https://github.com/RemixProjGroup/dxvk-remix/releases/tag/remix-plus-2.0.0)).
* **Supported hardware:** A GPU and driver that support NVIDIA RTX Remix.

### Verified Game Version

Because this mod patches `Engine.dll` and `D3DDrv.dll` in memory, it has only been tested with these exact files:

| File | Size | SHA-256 |
|---|---|---|
| `Engine.dll` | 4,800,512 bytes | `E85CA1A1F5795CAF3506B150E00D7969DEAA894FDAB54CD4221F9E9762267590` |
| `D3DDrv.dll` | 540,672 bytes | `B042C78B130BD7E164A07D1AD453DC8407AF75911ACCE245CAF5E38D5FAFE26F` |

To check your files, open PowerShell inside the game folder and run:

```powershell
Get-FileHash Engine.dll, D3DDrv.dll
```

Other versions are untested. Before applying most patches, the mod checks your file version and safely skips anything that doesn't match (see [Troubleshooting](#troubleshooting)).

---

## Installation

1. **Back up your game folder** (`Program\Bin`) so you can easily revert later.
2. **Install Remix Plus:** Open `Remix_Plus_v2.0.0.zip` and copy everything inside its `game folder` into your game folder:
   * `d3d9.dll` (the Remix bridge)
   * `.trex\` (the Remix runtime—if you already have a `.trex` folder, delete the old one first instead of merging them)
3. **Rename Remix's DLL:** In the game folder, rename the `d3d9.dll` you just copied (about 1 MB) to **`d3d9_remix.dll`**. Leave `.trex\d3d9.dll` untouched.
4. **Install this mod:** Download the repository (**Code → Download ZIP**, or download each file directly) and copy the four files from its [`Bin`](Bin) folder into your game folder:

   | File | What it does |
   |---|---|
   | `d3d9.dll` | The compatibility proxy |
   | `remix-comp-proxy.ini` | Proxy settings |
   | `rtx.conf` | Game-specific Remix setup (scale, orientation, and texture tags for skies, UI, decals, particles, lightmaps, and terrain) |
   | `user.conf` | *Optional:* The author's graphics presets. Skip this file to use Remix's defaults |

5. **Turn off static mesh batching:** Open `TV_CD_DVD.ini` in the game folder (launch the game once if this file doesn't exist yet). Under `[Engine.GameEngine]`, add or edit this line:

   ```ini
   UseStaticMeshBatching=False
   ```

   *Why:* Without this, the game groups objects into large chunks that Remix cannot track or replace properly.

6. **Recommended — Fix mouse lag and smoothing:**
   * In `TV_CD_DVD.ini`, under `[D3DDrv.D3DRenderDevice]`, set `ReduceMouseLag=False`. Leaving it `True` forces the renderer to pause every frame to sync with the GPU, noticeably hurting performance in Remix.
   * In `User.ini` (also in the game folder), add:

     ```ini
     [Engine.PlayerInput]
     MouseSmoothingMode=0
     MouseAccelThreshold=0.0
     ```

When you're finished, these new files should be sitting alongside the game's original files:

```text
Program\Bin\
  d3d9.dll               (this mod's proxy)
  d3d9_remix.dll         (renamed from Remix's d3d9.dll)
  .trex\                 (Remix runtime)
  remix-comp-proxy.ini
  rtx.conf
  user.conf              (optional)
```

---

## Playing

* **Launch the game:** Run `TV_CD_DVD.exe` normally. The very first launch takes several minutes to compile Remix shaders (this only happens once per Remix version).
* **Keep the console open:** A window titled **RTX-Comp Debug Console** opens with the game to display status logs. Leave it running—closing it can crash or close the game.
* **Shortcuts:** Press **Alt+X** for the RTX Remix menu, or **F4** for this mod's debug overlay.

---

## Configuration

The mod loads its settings from `remix-comp-proxy.ini` at startup. Full explanations are included inside the file, and the key options are listed below:

| Section | Key | Default | What it controls |
|---|---|---|---|
| `[Engine]` | `DisableGlow` | `1` | Disables the old glow effect. Keep at `1` whenever any `[Culling]` option is active |
| `[Engine]` | `DisableDynamicShadows` | `1` | Turns off flat projected character shadows |
| `[Engine]` | `WholeBspSections` | `1` | Draws level geometry in whole sections in a consistent order |
| `[Engine]` | `FixedFunctionTerrain` | `1` | Blends terrain layers via texture stages for Remix's terrain baker |
| `[Meshes]` | `CentrePivots` | `1` | Centers each static mesh's pivot point on its geometry |
| `[Culling]` | `FullScene` | `0` | Set to `1` to draw the entire map every frame. Useful for RTX Remix Toolkit captures, but heavy on FPS on large maps |
| `[Culling]` | `DisableFrustum`, `DisableDistance` | `1`, `1` | Turns off camera-view, antiportal, and distance culling |
| `[Culling]` | `KeepRadius`, `KeepAngularSize` | `8000`, `0.05` | Keeps objects drawn regardless of visibility if they are within `8000` units or appear large enough from afar |
| `[Lights]` | `Enabled`, `Intensity`, `IgnoreClasses` | `1`, `2.0`, `Jetpack` | Controls whether map lights are sent to Remix, their brightness, and which object types to ignore |

> **Note for Remix Toolkit modders:** Changing `WholeBspSections`, `CentrePivots`, or `UseStaticMeshBatching` alters the mesh hashes Remix generates. Keep all three at their default settings when capturing scenes and playing so community asset mods stay compatible.

---

## Troubleshooting

If something goes wrong, check the log files in your game folder:
* `rtx_comp\console.log` — Mod proxy log (matches the console window)
* `rtx-remix\logs\remix-dxvk.log` — RTX Remix log

| Problem | Cause & Fix |
|---|---|
| **Game runs without RTX Remix** (`console.log` says `Remix enabled but could not load 'd3d9_remix.dll'`) | `d3d9_remix.dll` is missing from the game folder. Re-check step 3 of the installation—without it, the mod falls back to standard Windows Direct3D 9. |
| **`console.log` says `Engine.dll is not retail 1.0.1; ... left on` or `BSP sections not hooked`** | Your `Engine.dll` doesn't match the [verified version](#verified-game-version). That specific patch is skipped automatically, and the game runs without it. |
| **`console.log` says `D3DDrv.dll has timestamp ..., expected ...`** | Your `D3DDrv.dll` doesn't match the verified version. The mod skips hooking it; world transforms rely on this hook, so results are untested. |
| **The mod stopped loading after updating Remix** | Reinstalling Remix overwrote the mod's `d3d9.dll`. Rename Remix's new `d3d9.dll` to `d3d9_remix.dll` and copy this mod's `d3d9.dll` back in. Always make sure `d3d9_remix.dll` and `.trex\` come from the same Remix release. |
| **Game crashes on startup** | Open `remix-comp-proxy.ini` and set `[Remix] Enabled=0`. This runs the game with the proxy alone so you can tell whether the crash comes from Remix or the proxy. |

Need deeper debugging? Press **F4** in-game and open the **Diagnostics** tab to dump draw-call details to `rtx_comp\diagnostics.log`.

---

## Uninstalling

Either restore your backup from step 1, or manually remove these files and folders from `Program\Bin`:
* `d3d9.dll`, `d3d9_remix.dll`, `remix-comp-proxy.ini`, `rtx.conf`, `user.conf`
* `.trex\`, `rtx_comp\`, and `rtx-remix\` *(note: `rtx-remix\` contains any captures or mods you created)*

Finally, delete the `UseStaticMeshBatching=False` line from `TV_CD_DVD.ini`. You can safely keep the mouse tweaks from step 6.

---

## Credits

* **[xoxor4d](https://github.com/xoxor4d)** — [remix-comp-base](https://github.com/xoxor4d/remix-comp-base) (D3D9 proxy architecture, ImGui integration, module system)
* **[kim2091](https://github.com/kim2091)** — remix-comp-proxy (fixed-function conversion, diagnostics, tracer, INI config) and [Remix Plus](https://github.com/RemixProjGroup/dxvk-remix)
* **[momo5502](https://github.com/momo5502)** — Original codebase behind remix-comp-base
* **[Ekozmaster](https://github.com/Ekozmaster/Vibe-Reverse-Engineering)** — Vibe Reverse Engineering toolkit
* **NVIDIA** — [RTX Remix](https://github.com/NVIDIAGameWorks/rtx-remix)

*Tribes: Vengeance*-specific work (engine patches, BSP section rendering, terrain conversion, level lights, and mesh pivot centering) is maintained in this repository.

---

## License

This proxy is released under the MIT License (see [LICENSE](LICENSE)). `d3d9.dll` bundles MinHook, Dear ImGui, and the RTX Remix bridge API headers—see [THIRD_PARTY_LICENSES.md](THIRD_PARTY_LICENSES.md) for their licenses.

The RTX Remix runtime is not included and is governed by its own licenses. *Tribes: Vengeance* is a trademark of its respective owners. This project is not affiliated with or endorsed by them or by NVIDIA.
