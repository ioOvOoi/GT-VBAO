# Wayfinder map: GT-VBAO Spec

Label: `wayfinder:map`

**Map status: complete**（2026-08-07，SPEC 已落盘，见 Decisions 票 05；Quality 表 TBD 由票 04 补入，不重新开图）

## Destination

一份可交接的实现 Spec：`docs/SPEC.md`。读完后任意实现会话能按清单交付 **UE 5.7 纯 Plugin**，功能完成度对齐 [HBAOPlus](http://39.170.56.176:3680/publicgroup/HBAOPlus)（双 Mode、关内置 SSAO、滤波/时域/CVar/Debug/README 等），算法仅用开源移植，不抄闭源。

## Notes

- Domain: UE5 deferred Screen-Space AO plugin；术语见仓库根 `CONTEXT.md`
- Skills: grilling, domain-modeling, research；实现阶段再开
- Tracker: 本地 markdown（`.scratch/gt-vbao-spec/`）
- Standing preferences（grilling 已锁）:
  - Spec 落盘：`docs/SPEC.md` 单文件；九章大纲见票 03
  - 仓形态：纯 Plugin，外挂任意 UE 5.7 工程
  - 完成度：HBAOPlus **全功能清单**进 Spec 验收
  - 挂钩：双 Mode；无 Substrate UB → **Skip Mode1**
  - CVar：`r.GTVBAO.*` 镜像 HBAOPlus 字段名（默认以参考仓代码为准）
  - 平台：Win64-only；Substrate：必须支持
  - 本仓 License：MIT
  - **禁止**抄 HBAOPlus / matiasgql GT-VBAO 闭源
- Remote: `https://github.com/ioOvOoi/GT-VBAO.git`（origin 已配置）
- Wayfinder 本努力默认 **只做决策 / Spec**，不写产品代码

## Decisions so far

- [AO math baseline](issues/01-ao-math-baseline.md) — Spec 锁 VBAO bitmask（SSRT3 MIT）+ XeGTAO 辅助；禁抄 HBAOPlus/商业 GT-VBAO
- [UE5.7 ScreenSpaceAO hook](issues/02-ue57-screen-space-ao-hook.md) — 5.7.4 双 Mode 仍成立；主杀内置靠 `AmbientOcclusionLevels=0`；Mode1 写全 extent `ScreenSpaceAO`
- [Spec outline and acceptance](issues/03-spec-outline-grilling.md) — 九章大纲 + 可勾选验收表；Quality 表 TBD；Substrate 缺失 Skip Mode1
- [Quality sample table](issues/04-quality-sample-table.md) — Quality 0–3 = (1,2)/(2,2)/(3,3)/(9,3) × 32 扇区（镜像 XeGTAO/bevy）；半分辨率不降采样；已补入 SPEC §5
- [Write docs/SPEC.md](issues/05-write-spec.md) — `docs/SPEC.md` Draft v1 已落盘；符合票 03 完成定义 → **地图完成**

## Not yet specified

- Temporal jitter / history 失效策略细则（超出对齐 HBAOPlus 行为之外）— Spec 可先写「行为对齐参考仓」，细则雾区
- DebugMode 是否扩展到 bitmask/horizon 可视化（HBAOPlus 仅 AO-only）— 默认不扩展，进 Spec Non-goals 除非另开票
- 半分辨率与时域在极端运动下的验收阈值（数值 vs 主观 A/B）— 验收以功能勾选为主

（Quality 采样表已由票 04 解决，从雾区移除。）

## Out of scope

- 复制或逆向 HBAOPlus / 商业 GT-VBAO 闭源实现
- 修改引擎 Renderer 源码（fork）
- Fab / Marketplace 上架打包
- Editor Project Settings UI（HBAOPlus 亦无，仅 CVar）
- 多平台（非 Win64）本期 Spec 不要求
