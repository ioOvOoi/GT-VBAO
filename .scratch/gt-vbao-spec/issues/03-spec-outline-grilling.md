# Spec outline and acceptance

Type: grilling  
Status: resolved  
Blocked by: 01, 02

## Question

`docs/SPEC.md` 应包含哪些章节、每章写到什么粒度，以及 **HBAOPlus 全功能对等** 的验收清单如何措辞，才算「可交接」？

已锁约束（勿再议，除非推翻 Destination）：
- 单文件 `docs/SPEC.md`
- 全功能对等 HBAOPlus 清单（双 Mode、关内置、主 pass→双边模糊→时域、HalfRes、Quality、Debug、CVar 表、README、Win64、Substrate）
- CVar：`r.GTVBAO.*` 镜像字段名
- 算法基线与挂钩细节分别来自票 01 / 02 的 Answer
- MIT；禁止抄闭源

本票要产出：章节目录草案 + 验收表（可勾选）+ 「写完 Spec 即关闭本地图」的完成定义。

## Answer

### 章节目录（九章）

1. **Goals & success criteria** — drop-in 替换内置 SSAO/GTAO；HBAOPlus 全功能对等；OSS 算法
2. **Non-goals / out of scope** — 不抄闭源；不改引擎；无 Editor Settings UI；非 Win64 本期不做
3. **Integration** — 双 Mode；关内置 CVar 契约；capture/Substrate 护栏（**无 Substrate UB → Skip Mode1**）
4. **Render pipeline** — Main → bilateral blur X/Y → temporal；HalfRes；缓冲格式（倾向 R16F 工作缓冲 → blit SSAO）
5. **Algorithm** — VBAO bitmask（SSRT3）+ XeGTAO 辅助；Quality 数值表 **TBD**（票 04）
6. **CVars** — 完整 `r.GTVBAO.*` 表（镜像 HBAOPlus 字段名/语义/默认，以参考仓 **代码** 为准非 README）
7. **Acceptance checklist** — 下方可勾选表
8. **Provenance / THIRD_PARTY** — 来源表 + `Docs/THIRD_PARTY.md` 义务
9. **Open questions** — 含 Quality TBD、temporal 细则、Debug 是否扩展等

粒度：Integration/Pipeline/CVars/Acceptance 写到实现可照抄；Algorithm 写映射与出处，不贴大段 shader。

### 验收清单（可勾选，HBAOPlus 功能对等）

**工程壳**
- [ ] `Plugins/GTVBAO/GTVBAO.uplugin`：Runtime、PostConfigInit、EngineVersion 5.7.x、Win64-only
- [ ] `FilterPlugin.ini`、`README.md`（启用步骤 + CVar 表 + 调参）
- [ ] Shader 映射 `/Plugin/GTVBAO`；模块依赖 Renderer private/internal（模式对齐，不抄算法）

**挂钩**
- [ ] `FSceneViewExtensionBase`，PostEngineInit 注册
- [ ] Mode1：`PostRenderBasePassDeferred_RenderThread` → 全 extent `ScreenSpaceAO` + ViewRect blit
- [ ] Mode0：`SubscribeToPostProcessingPass(BeforeDOF)` 合成 SceneColor
- [ ] Enabled 时 `r.AmbientOcclusionLevels=0`；Mode1 另 save/restore Method + MipLevelFactor
- [ ] 跳过 scene/reflection capture；无 Substrate global UB 时 **Skip Mode1**

**管线**
- [ ] Main pass → bilateral blur X/Y → temporal（TAA/TSR 时生效）
- [ ] HalfRes；DistanceFadeStart/End；MaxPixelRadius
- [ ] DebugMode：0 Off / 1 AO-only
- [ ] RDG/GPU stats 有名可辨

**CVars（前缀 `r.GTVBAO.`，语义对齐 HBAOPlus 代码默认）**
- [ ] Enabled, Mode, Radius, Bias, Intensity, Quality(0–3)
- [ ] BlurSharpness, NormalSharpness, MaxPixelRadius, HalfRes
- [ ] DistanceFadeStart, DistanceFadeEnd, DebugMode
- [ ] TemporalAccumulation, TemporalBlend

**合规**
- [ ] MIT LICENSE；`Docs/THIRD_PARTY.md`；移植注释含 `来源:`
- [ ] 无 HBAOPlus / 商业 GT-VBAO 源码

### 地图完成定义

当 `docs/SPEC.md` 按上列九章写完、Acceptance 表可直接给实现会话勾选、Open questions 仅剩已登记雾区（含 Quality TBD 指向票 04）时，关闭本 wayfinder 地图。**写 Spec 本身是票 05，不是本票。**
