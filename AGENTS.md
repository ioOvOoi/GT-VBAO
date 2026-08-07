# PROJECT KNOWLEDGE BASE

**Generated:** 2026-08-07

## OVERVIEW

Project: **GT-VBAO** (Ground Truth Visibility Bitmask Ambient Occlusion)

Goal: Ship an Unreal Engine plugin that **replaces the built-in SSAO/GTAO** with Visibility Bitmask AO (VBAO). Final product is a drop-in ambient occlusion path for deferred rendering, not a research demo.

Stack:
* Unreal Engine **5.7**
* C++ Plugin (`ISceneViewExtension` / `FSceneViewExtensionBase`)
* HLSL compute / pixel shaders (post-GBuffer AO pass)

Authoring mode:
* This repository is written primarily by AI agents.
* Core AO math must be **ported from open-source / published references**, not invented from scratch.
* Every ported or adapted block must cite source in comments (repo/URL, file, license).

## STRUCTURE

Empty repo today. Target layout:

```
Plugins/GTVBAO/
  GTVBAO.uplugin
  Source/GTVBAO/          # C++ module: ViewExtension, CVars, module startup
  Shaders/                # HLSL: depth reconstruct, horizon/bitmask, filters
  Docs/THIRD_PARTY.md     # License + provenance ledger for all ported code
AGENTS.md                 # This file (robot readme)
```

* `Source/GTVBAO/`: Plugin lifecycle, Scene View Extension registration, CVar wiring, AO texture handoff into engine lighting.
* `Shaders/`: Screen-space VBAO (visibility bitmask) + spatial/temporal filter shaders.
* `Docs/THIRD_PARTY.md`: Mandatory provenance list; keep in sync when porting.

## COMMANDS

| Action | Command |
|--------|---------|
| Install | Enable plugin in host `.uproject` → `Plugins/GTVBAO` |
| Build | Build host project with UE 5.7 (Editor target includes plugin) |
| Run | Launch Editor; disable built-in AO (`r.AmbientOcclusion.Method=0` or equivalent) and enable `r.GTVBAO.*` |
| Test | Visual A/B vs built-in GTAO on contact shadows / thin geometry; no automated suite yet |

## CODING STANDARDS

* **Language**: UE5 C++ + HLSL. Follow Unreal naming (`U`/`A`/`F`/`E`/`I`/`T`), `UPROPERTY`/`UFUNCTION`, no STL containers for UObject pointers.
* **Style**: Surgical diffs; reuse engine patterns for post-process / view extensions; Chinese comments for **why** (business rules, port reasons, safety); identifiers in English.
* **Rules**:
  * YAGNI / ponytail: shortest working diff; mark deliberate shortcuts with `// ponytail: …`.
  * **Provenance mandatory**: ported code comments must include `来源: <URL or repo> | <file> | <License>`.
  * Do **not** copy closed-source commercial plugins (see policy below).
  * Prefer existing engine hooks over forking Renderer module.

## WHERE TO LOOK

* **Source (planned)**: `Plugins/GTVBAO/Source/GTVBAO/`
* **Shaders (planned)**: `Plugins/GTVBAO/Shaders/`
* **Docs**: `Plugins/GTVBAO/Docs/THIRD_PARTY.md`, this `AGENTS.md`
* **Engine reference (local UE install)**: `Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessAmbientOcclusion.cpp` and related GTAO shaders

## OPEN SOURCE POLICY

Priority order for algorithm / filter code:

| Priority | Source | License | Use for |
|----------|--------|---------|---------|
| 1 | [GameTechDev/XeGTAO](https://github.com/GameTechDev/XeGTAO) | MIT | GTAO baseline (horizon integration, quality tiers) |
| 2 | [cdrinmatane VBAO / SSAOVB blog code](https://cdrinmatane.github.io/posts/ssaovb-code/) | See original post | Visibility bitmask core |
| 3 | UE `PostProcessAmbientOcclusion.cpp` + GTAO shaders | Epic | ViewExtension / pipeline integration patterns |
| Backup | [GameTechDev/ASSAO](https://github.com/GameTechDev/ASSAO), [GPUOpen-Effects/FidelityFX-CACAO](https://github.com/GPUOpen-Effects/FidelityFX-CACAO) | MIT | Filter / quality-tier reference |
| Forbidden | matiasgql **GT-VBAO** (Fab), ARK.KRA VBAO | Closed-source | Do not reverse or copy |

Papers / context (read, do not invent):
* Jimenez et al. 2016 — *Practical Realtime Strategies for Accurate Indirect Occlusion* (GTAO): https://www.iryoku.com/downloads/Practical-Realtime-Strategies-for-Accurate-Indirect-Occlusion.pdf
* Therrien, Levesque, Gilet — Visibility Bitmask AO: https://arxiv.org/abs/2301.11376

If a needed algorithm piece has no clear OSS source: **stop and research** before coding.

## IMPLEMENTATION PATH (guidance, not done yet)

1. Scaffold `Plugins/GTVBAO` for UE 5.7.
2. Register `FSceneViewExtensionBase`; inject AO after GBuffer in deferred path.
3. Disable built-in AO (`r.AmbientOcclusion.Method=0` or project setting) while plugin is active.
4. Port VBAO: depth reconstruct → directional horizon / bitmask → integrate → spatial/temporal filter (cite sources).
5. Write result into the texture/path the lighting / SSGI stack already consumes for screen-space AO.
6. Expose CVars: `r.GTVBAO.Mode`, `Radius`, `Intensity`, `DebugMode`, quality tiers, etc.
7. Keep `Docs/THIRD_PARTY.md` updated on every port.

## NOTES

* Since UE 4.24+, built-in “SSAO” is often **GTAO** (`r.AmbientOcclusion.Method=1`, `r.GTAO.*`). This project targets **VBAO-class** quality (thin surfaces / multi-horizon bitmask), not a rebrand of Crytek SSAO.
* Commercial UE “GT-VBAO” listings are closed-source; this repo is an independent OSS-first reimplementation for learning/replacement use — cite OSS only.
* Other agent context files: none yet (no `CLAUDE.md` / `.cursorrules` in tree). Prefer updating this `AGENTS.md` when stack or layout changes.
