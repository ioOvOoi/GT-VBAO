# UE5.7 ScreenSpaceAO hook

Type: research  
Status: resolved

## Question

在 **UE 5.7.0** 上，对齐 HBAOPlus 的双 Mode 挂钩是否仍然成立？Spec 应写成哪些 API / CVar / 护栏？

必须核实：
1. Mode1：`PostRenderBasePassDeferred_RenderThread` 写入/创建全分辨率 `FSceneTextures::ScreenSpaceAO` 再 blit ViewRect — 5.7 符号与时机是否仍有效
2. Mode0：`SubscribeToPostProcessingPass(BeforeDOF)` 合成路径是否仍合适
3. 关内置：`r.AmbientOcclusionLevels` / `r.AmbientOcclusion.Method` / `r.AmbientOcclusionMipLevelFactor` 的 save/restore 是否仍是正确手段
4. Substrate 置换与 scene/reflection capture 跳过是否仍为硬要求
5. 相对 HBAOPlus 参考实现，5.7 有无必须写进 Spec 的 API 漂移

证据优先：本地 UE 5.7 引擎头/源（若可得）+ HBAOPlus 公开结构（路径级，勿粘贴闭源算法）。验收：Spec「Integration」章节可直接粘贴的要点列表。

## Answer

- **Verdict: dual Mode OK**（本地验证 **UE 5.7.4** `F:\EPIC\UE_5.7`，Release-5.7 分支；非精确 5.7.0 CL，同系列）

### Spec Integration 要点

- 模块：`FSceneViewExtensionBase`；私有依赖 `Renderer` + Renderer private/internal includes（`FViewInfo`、`FSceneTextures`）
- **Mode1**：覆写 `PostRenderBasePassDeferred_RenderThread`（`SceneViewExtension.h`）；时机在 deferred base pass 之后、内置 AO 写入之前（`BasePassRendering.cpp` → `CompositionLighting::ProcessAfterBasePass`）
- **Mode1 写入**：经 `GetSceneTexturesChecked()` 操作全 extent 的 `FSceneTextures::ScreenSpaceAO`；在 ViewRect 算 AO，再 blit 进全尺寸缓冲；格式目标倾向 `PF_G8` / white clear（与引擎 SSAO desc 对齐）
- **Mode0**：必须用带 `const FSceneView&` 的 `SubscribeToPostProcessingPass`（5.5+；旧无 View 重载已 deprecated）；订阅 `EPostProcessingPass::BeforeDOF`；合成 SceneColor，**不**替换光照用 SSAO
- **护栏**：跳过 scene/reflection capture；Substrate：Mode1 需 Substrate view data/UB，缺失时 Spec 需定义 skip 或降级行为
- **漂移**：内置 AO 在 `CompositionLighting/`；`r.AmbientOcclusion.Method` 的 `0=SSAO, 1=GTAO`，**单独不是关 AO**

### CVar disable/restore

| CVar | Disable | Restore | Role |
|------|---------|---------|------|
| `r.AmbientOcclusionLevels` | `0` | prior（常 `-1`） | **主开关**：`Levels>0` 才跑内置 AO |
| `r.AmbientOcclusion.Method` | Mode1: save→`0` | restore | 停 GTAO 路径； alone 不够 |
| `r.AmbientOcclusionMipLevelFactor` | Mode1: save→`0` | restore | 额外保险 |

`ECVF_SetByCode`；关插件 / 离 Mode1 时 restore。

### Risks

依赖私有 Renderer API；若不强制 `Levels=0`，内置 AO 会盖掉 Mode1；Mode0 ≠ lighting drop-in；Substrate-null 行为需 Spec 写死。
