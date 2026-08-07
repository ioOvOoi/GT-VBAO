# Domain glossary

本文件只收术语，不含实现细节。

| Term | Meaning |
|------|---------|
| **GT-VBAO** | 本仓产品：UE 5.7 独立 Plugin，用 Visibility Bitmask AO 替换内置 SSAO/GTAO |
| **HBAOPlus (reference)** | 完成度参考仓；功能/壳对齐对象。闭源 All Rights Reserved，**禁止抄代码** |
| **Mode 1** | Deferred：在 `PostRenderBasePassDeferred` 写入引擎 `ScreenSpaceAO`，供光照消费（推荐 drop-in） |
| **Mode 0** | Post：在 `BeforeDOF` 把 AO 合成到 SceneColor |
| **ScreenSpaceAO** | 引擎场景纹理中的屏幕空间 AO 缓冲；Mode 1 的写入目标 |
| **Parity** | 与 HBAOPlus **功能与工程完成度**对等，不是源码对等 |
| **Provenance** | 移植块必须注释来源 URL/文件/License，并记入 `Docs/THIRD_PARTY.md` |
