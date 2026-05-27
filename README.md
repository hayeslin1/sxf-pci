# sxf-pci

SXF PCI 合规 Claude Code 插件，提供 PCI 密钥轮换迁移的完整作业技能。

## 技能列表

### `pci-sm-to-hash-migration`

**触发场景**：PCI 密钥轮换、SM 改 hash、黑名单命中比对失败、加密字段等值查不到。

**背景**：SxfAksUtils 加密库升级（PCI 密钥轮换）后，同一明文加密两次会得到不同密文，导致所有以 `_sm` 密文做等值匹配的代码失效。解决方案是为每个 `xxx_sm` 新增 `xxx_hash`（HMAC-SHA256）列，将"密文等值"改为"hash 等值"。

**作业分 6 阶段**：

| 阶段 | 内容 |
|------|------|
| P1 · 调研 | 盘清受影响的 `_sm` 字段、等值比较点、调用链，输出影响文档后 **HARD-GATE** |
| P2 · 切 V1 分支 | `BR_2026_pci_ask_hash_v1` |
| P3 · 升级依赖 | `com.cogolinks:cogo-metric-core:2.4.5-test-SNAPSHOT` |
| P4 · V1 改造 | 新增 hash 列、补写入逻辑、XML 预留 hash `<if>`（只加不删） |
| P5 · 切 V2 分支 | 基于 V1：`BR_2026_pci_ask_hash_v2` |
| P6 · V2 改造 | 切等值查询到 hash，移除原 `setXxxSm` 传参 |

## 安装

```bash
claude plugins  marketplace add hayeslin1/sxf-pci
claude plugins install  sxf-pci
```

## 更新 
```bash
claude plugins  update sxf-pci@sxf-pci
```



## 依赖约定

| 项 | 值 |
|----|----|
| Hash 工具类 | `com.cogo.digest.QueryDigestUtil` |
| 明文 → hash | `QueryDigestUtil.digestFromPlain(plain)` |
| 密文 → hash | `QueryDigestUtil.digestFromCipher(cipher)` |
| 依赖版本 | `com.cogolinks:cogo-metric-core:2.4.5-test-SNAPSHOT` |

## 作者

Mr.HayesLin
