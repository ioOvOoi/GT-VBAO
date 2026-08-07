# AO math baseline

Type: research  
Status: resolved

## Question

Spec 的 AO 数学基线应锁成哪一条，且每条路径的开源出处与许可证是否可商用进 MIT 仓？

候选（需证据，勿发明）：
1. **VBAO bitmask 为核心**（cdrinmatane SSAOVB / arXiv:2301.11376）+ XeGTAO 作 horizon/质量档/滤波参考
2. **先 XeGTAO GTAO 打通**，再演进 bitmask（分期 Spec）
3. 其它明确可引用的 OSS 组合

验收：给出推荐选项 + 关键文件/URL + License + 「禁止引用」清单（含 HBAOPlus.usf）；写清与 HBAOPlus 功能壳（Quality/HalfRes/Blur/Temporal）如何对接而不抄其 shader。

## Answer

- **推荐选项 1**：Spec 锁定 **VBAO bitmask 为 AO 积分核心**；**XeGTAO (MIT)** 作 slice/horizon 脚手架、质量档、depth MIP、空间去噪参考。对齐产品名与 AGENTS 优先级；功能壳对齐 HBAOPlus，不分期成「先只交 GTAO」。可复制 bitmask 代码优先 **cdrinmatane/SSRT3 (MIT)**；博客 + arXiv 作算法说明（博客无 SPDX，勿当 MIT 授权）。选项 2 推迟差异化；选项 3 单独不够。

### Source table

| URL | File / area | License | Used for |
|-----|-------------|---------|----------|
| https://arxiv.org/abs/2301.11376 | Paper | arXiv nonexclusive-distrib（论文分发，非代码许可） | 算法规格 |
| https://cdrinmatane.github.io/posts/ssaovb-code/ | UpdateSectors 等 | © CDRIN 2023；页上无 SPDX | 仅澄清算法 |
| https://github.com/cdrinmatane/SSRT3 | AO/bitmask | **MIT** | 主 bitmask / thickness / slice–step |
| https://github.com/GameTechDev/XeGTAO | XeGTAO.h / .hlsli | **MIT** | GTAO 基质、Quality 0–3、denoise |
| ASSAO / FidelityFX-CACAO | — | **MIT** | 滤波/半分辨率备援 |
| UE PostProcessAmbientOcclusion | Engine | Epic EULA | 仅挂钩模式，不 vendor shader |

### Forbidden

HBAOPlus（含 `.usf`）、Fab matiasgql GT-VBAO、ARK.KRA VBAO、Unity HDRP 衍生体、闭源 NVIDIA HBAO+。

### Shell mapping（概念）

- Quality 0–3 → XeGTAO QualityLevel；映射到 VBAO Rotation/Step（及固定 SECTOR_COUNT，如 32）— 数值表另票
- HalfRes → XeGTAO/CACAO 半分辨率思路，自研 upsample
- Blur → XeGTAO Denoise / ASSAO·CACAO edge-aware；排除 Unity denoiser
- Temporal → XeGTAO NoiseIndex + 引擎 TAA，和/或自研 history（可参考 SSRT3 作者路径，非 HBAOPlus）

### Risks

博客无明确代码许可；Quality 数值映射未锁；勿把 UE GTAO `.usf` 塞进 MIT 仓。
