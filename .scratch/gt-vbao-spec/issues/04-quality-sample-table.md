# Quality sample table

Type: research  
Status: resolved

## Question

`r.GTVBAO.Quality` 0–3 应映射到哪些 VBAO 采样参数（Rotation/dirCount、StepCount、SECTOR_COUNT、可选 thickness），且每档如何对齐 XeGTAO QualityLevel 与 HBAOPlus「0–3 越高越干净」的用户预期？

约束：

- 算法基线见 [AO math baseline](01-ao-math-baseline.md)（SSRT3 + XeGTAO）
- Spec §5 已留 TBD；本票产出可粘贴进 `docs/SPEC.md` 的数值表
- 禁止从 HBAOPlus.usf 反推采样数

验收：一张表 Quality → dirs / steps / sectors / notes；说明半分辨率下是否改采样；标出建议默认 Quality（对齐 HBAOPlus 代码默认 2）。

## Answer

| Quality | Rotation | Step | SECTOR_COUNT | Samples/px | Notes |
|---|---|---|---|---|---|
| 0 (Low) | 1 | 2 | 32 | 4 | 镜像 XeGTAO `CSGTAOLow` / bevy `Low` |
| 1 (Medium) | 2 | 2 | 32 | 8 | 镜像 XeGTAO `CSGTAOMedium` |
| 2 (High) **← 默认** | 3 | 3 | 32 | 18 | 镜像 XeGTAO `CSGTAOHigh`；XeGTAO/bevy 默认 High |
| 3 (Ultra) | 9 | 3 | 32 | 54 | 镜像 XeGTAO `CSGTAOUltra` |

- `SECTOR_COUNT=32`：论文 §3.3/§4.1（恰好一个 uint + 伪影阈值）；SSRT3 `#define MAX_RAY 32`。
- Thickness（世界空间常数）：SSRT3 默认 1.0 / bevy 默认 0.25 / 论文 0.2；关⇒无限厚度（GTAO 行为）。
- 半分辨率**不降低采样**（论文 §4.3：同采样 ~4× 提速，轻微闪烁交给 denoise/TAA）。
- 来源（均为一手）：XeGTAO `vaGTAO.hlsl`/`XeGTAO.h`（MIT）；SSRT3 `SSRTCS.compute`/`SSRT_HDRP.cs`（MIT）；bevy `crates/bevy_pbr/src/ssao/mod.rs`（MIT/Apache-2.0）；论文 arXiv:2301.11376。已补入 `docs/SPEC.md` §5。
