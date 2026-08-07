# GT-VBAO Implementation Spec

**状态**: Draft v1 · 关联 wayfinder 地图 `.scratch/gt-vbao-spec/map.md` · 术语表见根目录 `CONTEXT.md`

---

## 1. Goals & success criteria

目标：交付一个 **UE 5.7 独立 Plugin**（`Plugins/GTVBAO/`），用 **Visibility Bitmask AO (VBAO)** 作为 drop-in 替代内置 SSAO/GTAO。

- 功能完成度 **全功能对等** 参考仓 HBAOPlus（双 Mode、关内置 AO、主 pass→双边模糊→时域、HalfRes、Quality 0–3、Debug、CVar 表、README、Win64、Substrate）
- 算法仅来自开源移植：**VBAO bitmask（cdrinmatane/SSRT3, MIT）+ XeGTAO（Intel, MIT）**，禁抄任何闭源实现
- 成功标准 = 第 7 章 Acceptance checklist 全部勾选

## 2. Non-goals / out of scope

- **禁止**复制/逆向 HBAOPlus（含其 `.usf` shader）、matiasgql 商业 GT-VBAO、ARK.KRA VBAO、Unity HDRP 受限代码、闭源 NVIDIA HBAO+
- 不修改引擎 Renderer 源码（不 fork）
- 无 Editor Project Settings UI（与参考仓一致，仅 CVar）
- 非 Win64 平台本期不支持
- 不做 Fab / Marketplace 上架打包
- DebugMode 不扩展到 bitmask/horizon 可视化（保持 AO-only）
- 不做 SSGI/间接光照深集成（Mode1 写入 `ScreenSpaceAO` 后由引擎光照正常消费即达目标）

## 3. Integration

### 3.1 模块与注册

- `Plugins/GTVBAO/GTVBAO.uplugin`：Runtime、PostConfigInit、EngineVersion 5.7.x、Win64-only（`"Platforms": {"Win64": true}`）
- 模块 `GTVBAO`：私有依赖 `Renderer`，含 Renderer private/internal include 路径（需要 `FViewInfo`、`FSceneTextures`）
- `FSceneViewExtensionBase` 子类，`PostEngineInit` 时 `FAutoRegister` 构造注册
- Shader 映射目录 `/Plugin/GTVBAO`；`FilterPlugin.ini` 声明
- `README.md`：启用步骤、CVar 表、调参说明

### 3.2 双 Mode

**Mode 1 — 替换 ScreenSpaceAO（默认 drop-in 路径）**

- 覆写 `PostRenderBasePassDeferred_RenderThread(FRDGBuilder&, FSceneView&, const FRenderTargetBindingSlots&, TRDGUniformBufferRef<FSceneTextureUniformParameters>)`
- 时机：deferred base pass 之后、`CompositionLighting::ProcessAfterBasePass` 内置 AO 写入之前
- 输出目标：`FViewInfo::GetSceneTexturesChecked()->ScreenSpaceAO`；若未分配则创建**全 extent**（`Depth.Target->Desc.Extent`）纹理，格式对齐引擎 desc（倾向 `PF_G8`，`FClearValueBinding::White`）
- 计算范围：`ViewInfo.ViewRect`；半分辨率时以 `AddDrawTexturePass` 从 AO 缓冲 stretch-blit 进全 extent 纹理的 ViewRect 区域（**blit 目标是全 extent 纹理，不是 ViewRect 尺寸纹理**）
- 引擎光照后续在 `IndirectLightRendering` 消费该缓冲

**Mode 0 — Post 合成 SceneColor**

- 覆写 `SubscribeToPostProcessingPass(EPostProcessingPass, const FSceneView&, FAfterPassCallbackDelegateArray&, bool)`（**5.5+ 签名，必须带 `const FSceneView&`**；旧无 View 重载已 deprecated）
- 订阅 `EPostProcessingPass::BeforeDOF`；AO 合成到 SceneColor
- 不替换光照用 SSAO；非 deferred / 无 GBuffer 环境的兜底路径

### 3.3 关内置 AO 契约（Enabled 联动）

| CVar | Enabled>0 时 | 停用时 | 角色 |
|------|--------------|--------|------|
| `r.AmbientOcclusionLevels` | 设为 `0` | 恢复原值（常为 `-1`） | **主开关**：`Levels>0` 才跑内置 AO |
| `r.AmbientOcclusion.Method` | Mode1 时 save → `0` | restore | 停 GTAO 路径（`0=SSAO, 1=GTAO`，**单独不够**） |
| `r.AmbientOcclusionMipLevelFactor` | Mode1 时 save → `0` | restore | 额外保险 |

- 一律 `ECVF_SetByCode`；在 `BeginRenderViewFamily` / Enabled 回调中做 save/restore
- 不强制 `Levels=0` 时内置 AO 会覆盖 Mode1 输出 — 必须保证

### 3.4 护栏

- 跳过 `bIsSceneCapture` / `bIsReflectionCapture`
- **无 Substrate global UB（`SubstrateViewData.SubstrateGlobalUniformParameters` 为空）→ 本帧 Skip Mode1**（已锁决策）
- 非 deferred 路径：Mode0 可用，Mode1 直接 skip

## 4. Render pipeline

```
深度/GBuffer（引擎） → 主 pass → bilateral blur X → bilateral blur Y → temporal（TAA/TSR 时） → 输出
```

- **主 pass**：深度重建 → 每方向 slice 的 horizon/bitmask 采样 → 位域积分 → AO 值
- **Bilateral blur**：深度+法线感知（`BlurSharpness` / `NormalSharpness` 控制锐度）；X/Y 两趟
- **Temporal**：仅在 TAA 或 TSR 开启时生效（`TemporalAccumulation` + `TemporalBlend` + jitter）；行为对齐参考仓，历史缓冲自建
- **HalfRes**：半分辨率计算 + 上采样（自研/OSS 思路，禁抄参考仓 shader），再按 Mode 输出
- 工作缓冲格式实现可依质量档选择（如 R16F）；最终 Mode1 输出对齐引擎 `ScreenSpaceAO` 期望格式
- DistanceFade：`DistanceFadeStart` → `DistanceFadeEnd` 线性淡出；`MaxPixelRadius` 防过采样
- RDG pass 命名 + `DECLARE_GPU_STAT_NAMED` 可辨识（如 "GT-VBAO Main/Blur/Temporal"）

## 5. Algorithm

**基线（票 01 已锁）**：VBAO bitmask 为 AO 积分核心，XeGTAO 作 slice/horizon 脚手架、深度 MIP、空间去噪参考。

- 论文：Therrien, Levesque, Gilet — *Screen Space Indirect Lighting with Visibility Bitmask*（arXiv:2301.11376 / Vis Comput 2022）——算法规格
- 可移植代码来源（按优先级）：
  1. `cdrinmatane/SSRT3`（**MIT**）— bitmask 采样、thickness 前后沿、slice/step
  2. `GameTechDev/XeGTAO`（**MIT**）— `XeGTAO.h` / `XeGTAO.hlsli`：深度重建、horizon、depth MIP、空间去噪、QualityLevel 语义
  3. 博客 ssaovb-code（© CDRIN，页面无 SPDX）— 仅作算法澄清，**不作复制来源**
- 关键常数：`SECTOR_COUNT = 32`（默认，来自 bitmask 方案）
- 每个移植/改写块必须注释 `来源: <URL> | <file> | <License>`，并同步 `Docs/THIRD_PARTY.md`
- **禁止**从 HBAOPlus `.usf` 反推采样数或复制任何参考仓 shader

### Quality 采样表（票 04 已锁）

`r.GTVBAO.Quality` 0–3 → Rotation / Step / SECTOR_COUNT；数值镜像 XeGTAO `vaGTAO.hlsl` 四档（`CSGTAOLow/Medium/High/Ultra`）与 bevy VBAO 实现，默认 Quality=2（对齐 XeGTAO `GTAOSettings` 与参考仓默认）。

| Quality | Rotation (dirCount) | Step (stepCount) | SECTOR_COUNT | Samples/px (rot×step×2) | Notes |
|---------|--------------------|------------------|--------------|--------------------------|-------|
| 0 (Low) | 1 | 2 | 32（固定） | 4 | 镜像 XeGTAO `CSGTAOLow` / bevy `Low` |
| 1 (Medium) | 2 | 2 | 32 | 8 | 镜像 XeGTAO `CSGTAOMedium` |
| 2 (High) **← 默认** | 3 | 3 | 32 | 18 | 镜像 XeGTAO `CSGTAOHigh`；XeGTAO/bevy 默认即 High |
| 3 (Ultra) | 9 | 3 | 32 | 54 | 镜像 XeGTAO `CSGTAOUltra` |

- `SECTOR_COUNT = 32` 依据：论文 §3.3/§4.1 —— 32 扇区恰好一个 `uint`，且为伪影阈值；SSRT3 `#define MAX_RAY 32`。
- 可选 `Thickness`（常数、世界空间）：SSRT3 默认 1.0（0.01–10）、bevy 默认 0.25、论文用 0.2；关 ⇒ 无限厚度（退化为 GTAO 高度场行为）。
- **半分辨率不降低采样**（论文 §4.3：同采样数半分辨率 ~4× 提速，代价为轻微闪烁/模糊；交给 denoise/TAA）。thickness 为世界空间量，无需缩放。
- 来源：XeGTAO `Source/Rendering/Shaders/vaGTAO.hlsl` + `XeGTAO.h`（MIT）；SSRT3 `SSRTCS.compute` + `SSRT_HDRP.cs`（MIT）；bevy `crates/bevy_pbr/src/ssao/mod.rs`（MIT/Apache-2.0）；论文 arXiv:2301.11376。

## 6. CVars

前缀 `r.GTVBAO.*`，字段名与语义镜像参考仓公开 CVar 清单；**默认值取参考仓公开清单，实现代码自写**。

| CVar | 默认 | 语义 |
|------|------|------|
| `r.GTVBAO.Enabled` | 1 | 0 关 / 1 开（联动 §3.3 关内置契约） |
| `r.GTVBAO.Mode` | 0 | 0=Post 合成 SceneColor；1=Deferred 替换 ScreenSpaceAO |
| `r.GTVBAO.Radius` | 100.0 | AO 半径（世界单位） |
| `r.GTVBAO.Bias` | 0.1 | 深度偏差 |
| `r.GTVBAO.Intensity` | 1.5 | 强度 |
| `r.GTVBAO.Quality` | 2 | 质量档 0–3（采样表见 §5，TBD） |
| `r.GTVBAO.BlurSharpness` | 16.0 | 模糊深度锐度 |
| `r.GTVBAO.NormalSharpness` | 8.0 | 模糊法线锐度 |
| `r.GTVBAO.MaxPixelRadius` | 100.0 | 最大像素半径（防过采样） |
| `r.GTVBAO.HalfRes` | 1 | 0 全分辨率 / 1 半分辨率 |
| `r.GTVBAO.DistanceFadeStart` | 15000.0 | 距离淡出起点 |
| `r.GTVBAO.DistanceFadeEnd` | 30000.0 | 距离淡出终点 |
| `r.GTVBAO.DebugMode` | 0 | 0 Off / 1 AO only |
| `r.GTVBAO.TemporalAccumulation` | 1 | TAA/TSR 开启时启用时域累积+jitter |
| `r.GTVBAO.TemporalBlend` | 0.95 | 时域混合权重（高=更多历史） |

## 7. Acceptance checklist

### 工程壳

- [ ] `Plugins/GTVBAO/GTVBAO.uplugin`：Runtime、PostConfigInit、EngineVersion 5.7.x、Win64-only
- [ ] `FilterPlugin.ini`、`README.md`（启用步骤 + CVar 表 + 调参）
- [ ] Shader 映射 `/Plugin/GTVBAO`；模块依赖 Renderer private/internal（模式对齐，不抄算法）
- [ ] MIT `LICENSE`；`Docs/THIRD_PARTY.md` 存在且与移植同步

### 挂钩

- [ ] `FSceneViewExtensionBase`，PostEngineInit 注册
- [ ] Mode1：`PostRenderBasePassDeferred_RenderThread` → 全 extent `ScreenSpaceAO` + ViewRect blit
- [ ] Mode0：`SubscribeToPostProcessingPass(BeforeDOF)` 合成 SceneColor（5.5+ 签名）
- [ ] Enabled 时 `r.AmbientOcclusionLevels=0`；Mode1 另 save/restore Method + MipLevelFactor
- [ ] 跳过 scene/reflection capture；无 Substrate global UB → Skip Mode1

### 管线

- [ ] Main pass → bilateral blur X/Y → temporal（TAA/TSR 时生效）
- [ ] HalfRes；DistanceFadeStart/End；MaxPixelRadius
- [ ] DebugMode：0 Off / 1 AO-only
- [ ] RDG/GPU stats 有名可辨

### CVars（§6 全表 15 项）

- [ ] Enabled, Mode, Radius, Bias, Intensity, Quality(0–3)
- [ ] BlurSharpness, NormalSharpness, MaxPixelRadius, HalfRes
- [ ] DistanceFadeStart, DistanceFadeEnd, DebugMode
- [ ] TemporalAccumulation, TemporalBlend

### 合规

- [ ] 移植注释含 `来源:`；无 HBAOPlus / 商业 GT-VBAO / Unity HDRP 受限源码

## 8. Provenance / THIRD_PARTY

| 来源 | 文件/区域 | License | 用途 |
|------|-----------|---------|------|
| [arXiv:2301.11376](https://arxiv.org/abs/2301.11376) | 论文 | arXiv 非独占分发（论文许可，非代码） | 算法规格 |
| [cdrinmatane/SSRT3](https://github.com/cdrinmatane/SSRT3) | bitmask / thickness / slice–step | MIT | 主移植来源 |
| [ssaovb-code 博客](https://cdrinmatane.github.io/posts/ssaovb-code/) | UpdateSectors 等示意 | © CDRIN 2023（无 SPDX） | 仅算法澄清 |
| [GameTechDev/XeGTAO](https://github.com/GameTechDev/XeGTAO) | XeGTAO.h / .hlsli | MIT | GTAO 基质/去噪/质量语义 |
| [GameTechDev/ASSAO](https://github.com/GameTechDev/ASSAO) | — | MIT | 滤波备援 |
| [GPUOpen-Effects/FidelityFX-CACAO](https://github.com/GPUOpen-Effects/FidelityFX-CACAO) | — | MIT | 半分辨率/滤波备援 |
| UE 引擎 `CompositionLighting/` 等 | — | Epic EULA | 仅挂钩模式，不 vendor shader |

义务：每次移植/改写同步 `Docs/THIRD_PARTY.md`；本仓对外 MIT。

## 9. Open questions

- ~~**Quality 采样表**~~ — 已由票 04 解决，见 §5 数值表
- Temporal jitter / history 失效策略细则 — 默认「行为对齐参考仓」，细则留实现期
- DebugMode 扩展（bitmask/horizon 可视化）— 默认不扩展
- 半分辨率+时域在极端运动下的验收阈值 — 默认主观 A/B（与内置 GTAO 对照）
