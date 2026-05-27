---
name: pci-sm-to-hash-migration
description: Use when执行PCI密钥轮换迁移 - 即数据库中已有的 `xxx_sm`（SxfAksUtils 加密密文）字段需要新增对应的 `xxx_hash` (HMAC-SHA256) 列以支持等值查询和黑名单/规则命中比较，触发场景包括"PCI 轮换""SM 改 hash""黑名单命中比对失败""加密字段等值查不到"等
author: Mr.HayesLin
---

# PCI 密钥轮换：xxx_sm → xxx_hash 迁移

## 背景

SxfAksUtils 加密库升级（PCI 密钥轮换）后，同一明文加密两次会得到**不同密文**（但都能解密）。
因此所有以 `_sm` 密文做等值匹配的代码（Java `.equals()` / SQL `WHERE xxx_sm = ?`）都会失效。

**解决方案**：为每个 `xxx_sm` 增加一列 `xxx_hash`（HMAC-SHA256），将"密文等值"改为"hash 等值"。

## 命名与依赖约定

| 项 | 值 |
|----|----|
| 加密列后缀 | `_sm`（SM4 密文） |
| 掩码列后缀 | `_mask`（脱敏展示） |
| **新增 hash 列后缀** | `_hash`（HMAC-SHA256，等值查询） |
| Hash 工具类 | `com.cogo.digest.QueryDigestUtil` |
| 明文 → hash | `QueryDigestUtil.digestFromPlain(plain)` |
| 密文 → hash | `QueryDigestUtil.digestFromCipher(cipher)` |
| 升级依赖 | `com.cogolinks:cogo-metric-core:2.4.5-test-SNAPSHOT` |

## 工作流总览（6 阶段）

```dot
digraph workflow {
    "P1.1 输出影响点清单 [Opus+xhigh]" [shape=box];
    "HARD-GATE-1: 用户确认影响点" [shape=diamond];
    "P1.2 输出改动计划 plan [Opus+xhigh]" [shape=box];
    "HARD-GATE-2: 用户确认 plan" [shape=diamond];
    "P2 切 V1 分支 [Sonnet+subagent]" [shape=box];
    "P3 升级依赖 [Sonnet+subagent]" [shape=box];
    "P4 V1: 新增 hash 列 (兼容写入) [Sonnet+subagent 并行]" [shape=box];
    "P5 切 V2 分支 (基于 V1) [Sonnet+subagent]" [shape=box];
    "P6 V2: 改 sm 等值为 hash 等值 [Sonnet+subagent 并行]" [shape=box];

    "P1.1 输出影响点清单 [Opus+xhigh]" -> "HARD-GATE-1: 用户确认影响点" -> "P1.2 输出改动计划 plan [Opus+xhigh]" -> "HARD-GATE-2: 用户确认 plan" -> "P2 切 V1 分支 [Sonnet+subagent]" -> "P3 升级依赖 [Sonnet+subagent]" -> "P4 V1: 新增 hash 列 (兼容写入) [Sonnet+subagent 并行]" -> "P5 切 V2 分支 (基于 V1) [Sonnet+subagent]" -> "P6 V2: 改 sm 等值为 hash 等值 [Sonnet+subagent 并行]";
}
```

**铁律**：
- **Phase 1 强制 Opus + xhigh thinking mode**（深度调研推理），P1.1 与 P1.2 各一次 HARD-GATE，缺一不可（对齐 R8）。
- **Phase 2 起切 Sonnet 模型**，按 P1.2 产出的 plan 派发 **subagent 并行执行**；执行阶段零自由度（R10），偏离 plan 立即暂停回到规划。
- **subagent 并行原则**：P3/P4/P6 中按"字段 / 表 / 模块"维度拆分，相互不影响的任务必须并行派发（同一条消息内多 Agent 调用，对齐 R4 + R9）。
- V1 做"加列 + 写入兼容 + XML 预留 hash 的 `<if>`"，但**不动任何 Java 比较逻辑、不动 Service 入参**——XML 中新加的 `xxx_hash` `<if>` 因为没人传参所以不会拼出，行为零变更。
- V2 才把 Java `_sm` 等值比较切到 `_hash`，并把 Service 入参从 `setXxxSm` 切到 `setXxxHash`。XML 在 V1 已完工，V2 不再动 XML。

---

## Phase 1 · 调研：影响点 + 改动计划（Opus + xhigh thinking）

> **模型与思考模式**：本阶段**必须使用 Opus 模型 + xhigh thinking mode**。Phase 1 是整次迁移成败的根，覆盖率漏一处 = V2 上线某条路径直接失效，**禁止用 Sonnet 或低思考预算执行 Phase 1**。
>
> **执行约束**：本阶段不得动任何代码，不得切分支，不得装依赖。只产出两份文档，每份产出后 STOP 等用户确认。

Phase 1 拆为两个子阶段，**串行执行，各自一次 HARD-GATE**：

| 子阶段 | 产物 | 闸门 |
|--------|------|------|
| **P1.1 输出影响点清单** | `docs/pcihash/<YEAR>-PCI密钥轮换影响.md` | HARD-GATE-1 |
| **P1.2 输出改动计划 plan** | `docs/pcihash/plan/<YEAR>-PCI改动计划.md` | HARD-GATE-2 |

---

### Phase 1.1 · 输出影响点清单

**目标**：盘清楚本仓库受影响的所有 `_sm` 字段、所在表、Java/XML 中的等值比较点、控制器→服务→Mapper 调用链。**只盘点，不出改造方案**。

**输出文件**：`docs/pcihash/<YEAR>-PCI密钥轮换影响.md`（不创建任何其他文档文件，遵循 R6）。

**模板**：

```markdown
# <YEAR>-PCI 密钥轮换影响清单

## 一、_sm 字段清单
| 表名 | 字段 |
|------|------|
| `t_xxx_xxx` | `a_sm`, `b_sm` |
| `t_yyy_yyy` | `c_sm` |

## 二、等值/包含比较点（V2 需改）
| 文件 | 行号 | 形式 | 待改 |
|------|------|------|------|
| `XxxServiceImpl.java` | 42 | `s.getASm().equals(req.getASm())` | → `getAHash().equals(digestFromPlain(...))` |
| `XxxMapper.xml` | 88 | `AND a_sm = #{aSm}` | → `AND a_hash = #{aHash}` + 保留 `_sm` if |

## 三、调用链
| 入口功能（一句话） | 调用链 |
|--------------------|--------|
| 黑名单列表分页查询 | Controller `/api/xxx/search` → `XxxServiceImpl#search` → `XxxMapper#search`（使用 `a_sm` 等值） |
| 新增黑名单时按账号验重 | Controller `/api/xxx/add` → `XxxServiceImpl#add` → `XxxMapper#accurateQuery`（使用 `a_sm` 验重） |
| 支付路由命中黑名单内存过滤 | Handler `CheckXxxHandler#handle` → 内存中 `.equals(getASm())` 比较 |

## 四、风险与上线动作
- **V1 期间 hash 与 sm 双写**：所有写入路径（add / batchInsert / updateById / batchUpdate / Job 同步）同时回填 `_hash` 与 `_sm`，新数据 hash 不会为空。
- **V1 上线后必须补刷历史数据**：执行一次性回填脚本，把存量 `xxx_sm` 解密后计算 hash 写入 `xxx_hash`。补刷完成是 V2 上线的**前置条件**。
  ```sql
  -- 示意：实际由应用侧跑批（需调用 SxfAksUtils.decrypt + QueryDigestUtil.digestFromPlain）
  UPDATE t_xxx_xxx
     SET a_hash = <HMAC(decrypt(a_sm))>
   WHERE a_sm IS NOT NULL AND a_hash IS NULL;
  ```
- **回滚预案**：V2 上线后若 hash 命中异常，可回退到 V1——XML 中 `_sm` 的 `<if>` 仍保留，老调用方传 SM 仍可工作。
```

**搜索命令模板**：
```bash
# 1. 找所有 _sm 字段（XML + Java）
grep -rEn '\b\w+_sm\b' --include='*.xml' --include='*.java' -- <repo>

# 2. 找 Java 等值/包含/查找类比较（穷举常见方法名，避免漏网）
#    覆盖：equals / equalsIgnoreCase / equalsAny / equalsAnyIgnoreCase
#         contains / containsIgnoreCase / containsAny
#         indexOf / lastIndexOf / startsWith / endsWith
#         Objects.equals / StringUtils.equals*
grep -rEn '\.get\w+Sm\(\)\s*\.\s*(equals|equalsIgnoreCase|equalsAny|equalsAnyIgnoreCase|contains|containsIgnoreCase|containsAny|indexOf|lastIndexOf|startsWith|endsWith)\s*\(' --include='*.java' -- <repo>
# 反向：把 SM 当参数传给上述方法
grep -rEn '\.(equals|equalsIgnoreCase|equalsAny|equalsAnyIgnoreCase|contains|containsIgnoreCase|containsAny|indexOf|lastIndexOf|startsWith|endsWith)\s*\([^)]*\.get\w+Sm\(\)' --include='*.java' -- <repo>
# 工具类静态比较：Objects.equals(xxxSm, ...) / StringUtils.equals*(xxxSm, ...)
grep -rEn '(Objects|StringUtils)\.\w*[Ee]quals\w*\s*\([^)]*\.get\w+Sm\(\)' --include='*.java' -- <repo>
# Collection.contains(xxxSm) —— sm 作为单参传入
grep -rEn '\.contains\s*\(\s*\w+\.get\w+Sm\(\)\s*\)' --include='*.java' -- <repo>

# 3. 找 SQL 中以 _sm 字段做条件的语句（等值 / IN / LIKE）
grep -rEn '(?i)(and|where|on)\s+\w*\.?\w+_sm\s*(=|in|like)' --include='*.xml' -- <repo>

# 4. 找通过 BeanUtils.copyProperties 写入 _sm 字段的路径
#    BeanUtils.copyProperties(src, dest) 会把 src 中所有 xxxSm getter 的值批量 copy 到 dest，
#    不会出现显式的 setXxxSm，因此需要单独搜索
#    策略：找到调用 BeanUtils.copyProperties 且 dest 的类型包含 _sm 字段的地方
grep -rEn 'BeanUtils\.copyProperties' --include='*.java' -- <repo>
#    结果需人工逐一确认：dest 对象的类型是否包含 xxx_sm 字段（即 xxxSm getter/setter）
#    若包含则该调用点视为 _sm 字段的写入路径，必须纳入 P4.4 的补 hash 改造范围
```

**人工兜底**：上述 grep 只能覆盖常见模式，**必须**再人工通读 P1 清单里每个 `_sm` 字段所在 PO 的 `getXxxSm()` 调用链（IDE Find Usages），把任何"两个 `_sm` 值进行比较 / 判断"的位置全部记入 P1 文档第二节，**漏一处 = V2 上线某条路径直接失效**。

**⚠️ 写入路径缺失时的处理规则**：  
对 P1 清单中每个 `xxx_sm` 字段，若以下两种写入来源**均未找到**：
1. Java 层显式调用 `setXxxSm(...)`
2. `BeanUtils.copyProperties` 将含 `xxxSm` 的对象 copy 到 PO/DO

则**不得自行假设"无写入路径"或"只读字段跳过"**，须在 P1 影响文档中标记：

```markdown
| `t_xxx_xxx` | `a_sm` | ⚠️ 未找到写入路径（setXxxSm + BeanUtils 均未命中），需人工确认 |
```

并在 Phase 1 完成后汇报时，单独列出所有标记项，明确请求用户确认后再继续。

**Phase 1.1 完成后 STOP（HARD-GATE-1）**：把影响清单文档路径报告给用户 + 字段汇总表 + ⚠️ 标记项汇总，明确请求"是否确认影响点清单、进入 P1.2 输出改动计划"。**禁止跳过 HARD-GATE-1 直接产出 plan**（对齐 R8 HARD-GATE / R10 执行阶段零自由度）。

---

### Phase 1.2 · 输出改动计划 plan

**前置**：HARD-GATE-1 已通过（用户确认影响点清单完整）。

**目标**：基于 P1.1 影响点清单，产出**精确到文件路径 + 函数名 + 行号 + 具体改造内容**的 plan，作为 Phase 2 起 subagent 执行的**唯一真相源**（Spec is Truth，对齐 R8）。

**输出文件**：`docs/pcihash/plan/<YEAR>-PCI改动计划.md`

**plan 必须满足**：
- 精确到 `文件:行号:具体改造`（对齐 R10 规划阶段"精确到函数签名"）
- 按"独立可并行任务"维度拆分，明确标注哪些任务可并行（喂给 subagent）
- 每个任务自包含：上下文 / 文件路径 / 改前代码片段 / 改后代码片段 / 验证方式
- 覆盖 P3 / P4（4.1–4.4）/ P5 / P6（6.1–6.2）所有改造点

**plan 模板**：

```markdown
# <YEAR>-PCI 密钥轮换改动计划

> 本 plan 是 Phase 2+ 的唯一执行依据。任何偏离 plan 的修改 = 暂停回到规划（R10）。

## 任务依赖图
- P2（切分支）→ P3（升依赖）→ P4 任务组 → P5（切 V2 分支）→ P6 任务组
- P4 任务组内部按字段拆分，**相互独立，并行执行**
- P6 任务组内部按模块/调用点拆分，**相互独立，并行执行**

## P3 · 升级依赖
| 文件 | 当前版本 | 目标版本 |
|------|----------|----------|
| `xxx-data/build.gradle` | `cogo-metric-core:2.4.0` | `cogo-metric-core:2.4.5-test-SNAPSHOT` |

## P4 任务组（按字段拆分，可并行）

### P4-T1: t_xxx_xxx.a_sm → a_hash
- **可并行**: 与 P4-T2 / P4-T3 完全独立
- **改造点 1**: ALTER SQL — `docs/pcihash/<project>.sql` 追加
  ```sql
  ALTER TABLE `<db>`.`t_xxx_xxx` ADD COLUMN `a_hash` ...
  ```
- **改造点 2**: PO 字段 — `XxxPO.java:35` 新增 `private String aHash;`
- **改造点 3**: XML — `XxxMapper.xml`
  - `<resultMap>` 第 18 行下方新增 `<result column="a_hash" property="aHash"/>`
  - 公共 `<sql id="baseColumns">` 第 25 行追加 `a_hash,`
  - `<insert id="add">` 第 45 行 columns + 第 50 行 values 各加一处
  - `<update id="updateById">` 第 70 行 `<set>` 内新增 `<if test="aHash != null">a_hash = #{aHash},</if>`
  - `<insert id="batchInsert">` 第 90 行 columns + values + `ON DUPLICATE KEY UPDATE` 三处
  - `<select id="search">` 第 110 行 WHERE 新增 `<if test="aHash != null and aHash != ''">AND a_hash = #{aHash}</if>`（与原 `a_sm` 的 `<if>` 并列保留）
- **改造点 4**: 写入路径补 hash
  - `XxxServiceImpl.java:88` 的 `add()` — 在 `setASm(...)` 后追加 `setAHash(QueryDigestUtil.digestFromPlain(req.getA()))`
  - `XxxJob.java:42` 的 `syncBatch()` — 在 `setASm(...)` 后追加 `setAHash(QueryDigestUtil.digestFromCipher(po.getASm()))`
  - `XxxConverter.java:25` 的 `BeanUtils.copyProperties(src, dest)` 之后追加补 hash（参见 SKILL 4.4）
- **改造点 5**: Request DTO — `XxxAddReq.java:30` 新增 `private String aHash;`（确保 MyBatis OGNL 不抛 no getter）
- **验证**: `./gradlew :xxx-data:compileJava` + 跑 `XxxMapperTest`

### P4-T2: t_xxx_xxx.b_sm → b_hash
（同上模板，独立任务）

### P4-T3: t_yyy_yyy.c_sm → c_hash
（同上模板，独立任务）

## P6 任务组（按比较点/查询入口拆分，可并行）

### P6-T1: XxxServiceImpl 内存比较改造
- **可并行**: 与 P6-T2 / P6-T3 独立
- **改造点**: `XxxServiceImpl.java:42`
  - 改前: `s.getASm().equals(req.getASm())`
  - 改后: `s.getAHash().equals(QueryDigestUtil.digestFromPlain(req.getA()))`
- **验证**: 单测 + curl 受影响接口

### P6-T2: CheckXxxHandler 命中比较改造
（同上模板，独立任务）

### P6-T3: Service 查询入口切 hash
- **改造点**: `XxxQueryServiceImpl.java:55`
  - 改前: `req.setASm(SxfAksUtils.encrypt(req.getA()))`
  - 改后: `req.setAHash(QueryDigestUtil.digestFromPlain(req.getA()))`（**移除** setASm，参见 SKILL 6.2 注意）

## 风险提示
- P4 写入路径任一遗漏 → 历史补刷脚本无法覆盖该路径新写入的数据
- P6 移除 setXxxSm 时若 XML 中 `_sm` 的 `<if>` 仍接受参数，需确认入参确实不再传入

## 上线节奏（与 SKILL 风险段一致）
- V1 上线 → 跑历史补刷脚本 → 验证补刷完成 → V2 上线
```

**plan 写完即停**：plan 写好之后，**禁止自动进入 P2**。

**Phase 1.2 完成后 STOP（HARD-GATE-2）**：把 plan 文档路径报告给用户 + 并行任务拆分摘要（`P4 有 X 个独立任务可并行 / P6 有 Y 个独立任务可并行`），明确请求"是否确认 plan、切到 Sonnet 模型从 P2 开始执行"。**禁止跳过 HARD-GATE-2**。


---

## Phase 2 · 创建 V1 分支

> **模型切换**：从此阶段开始**切换到 Sonnet 模型**。
> **执行方式**：严格按 P1.2 产出的 plan 执行；P2 是单点动作，由主 Agent 直接执行即可，不必派 subagent。

- 创建新分支，命名：`BR_<YYYYMMDD>_pci_ask_hash_v1`（日期取用户给定或当前日期）。

```bash
git checkout -b BR_<DATE>_pci_ask_hash_v1
```

---

## Phase 3 · 升级依赖

> **执行方式**：Sonnet + 按 plan 执行。若 plan 中涉及多个 `build.gradle` 文件且相互独立，**派多个 subagent 并行修改**（对齐 R4 + R9）。

定位每个 `build.gradle`（或 `pom.xml`）中 `com.cogolinks:cogo-metric-core` 的引用，统一改为：

```gradle
implementation 'com.cogolinks:cogo-metric-core:2.4.5-test-SNAPSHOT'
```

若项目中**没有**该依赖，则在主 data/core 模块的 `build.gradle` 添加。
依赖锁定使用确定版本号，禁止 `latest` / `+`（对齐 R5）。

---

## Phase 4 · V1：新增 hash 列、写入兼容、XML 预留 hash `<if>`

> **执行方式**：Sonnet + **subagent 并行**。P1.2 plan 中已按"字段 / 表"维度拆分独立任务（P4-T1 / P4-T2 / ...），**必须在一条消息内并行派发多个 subagent**（对齐 R4 + R9）；每个 subagent 负责一个 P4-Tx，严格按其改造点列表执行（R10 执行阶段零自由度）。
> **subagent 选型**：使用 `general-purpose`（需写文件）；任务粒度遵循"一个 subagent 一件事"——单个字段单个表为一个 subagent。
> **禁止偏离 plan**：subagent 发现 plan 缺漏 → 暂停 → 报告主 Agent → 主 Agent 回到 P1.2 修 plan → 重新派发（Reverse Sync，R8）。

针对 P1 清单中的每一个 `xxx_sm`，按以下 4 个动作改造：

### 4.1 ALTER SQL

Alter SQl 文件：`docs/pcihash/<project_name>.sql`

```sql
ALTER TABLE `<db>`.`<table1>`
  ADD COLUMN `xxx_hash` varchar(512) NULL COMMENT 'xxx-hash' AFTER `xxx_sm`,
  ADD COLUMN `xxx_hash` varchar(512) NULL COMMENT 'xxx-hash' AFTER `xxx_sm`;
ALTER TABLE `<db>`.`<table2>`
  ADD COLUMN `xxx_hash` varchar(512) NULL COMMENT 'xxx-hash' AFTER `xxx_sm`;  
```

多字段在同一文件按顺序列出，每个 `AFTER` 紧跟对应 `_sm`。

### 4.2 Domain (PO) 新增字段

在 PO 中追加（保持 `@Data`、Lombok 风格不变）：

```java
@Schema(description = "xxx-hash")
private String xxxHash;
```

### 4.3 Mapper XML：写入路径全覆盖

**必须**同时修改以下所有出现位置，缺一会导致 hash 列空：

| 位置 | 改造 |
|------|------|
| `<resultMap>` | 增加 `<result column="xxx_hash" property="xxxHash"/>` |
| 公共字段列 `<sql id="...">` | 增加 `xxx_hash,` |
| `insert` / `add` | 列名 + values 两处 |
| `updateById` 的 `<set>` | 增加 `<if test="xxxHash != null">xxx_hash = #{xxxHash},</if>` |
| `batchInsert` | columns + values + `ON DUPLICATE KEY UPDATE` 三处 |
| `batchUpdate` | `<set>` + （如有）`<foreach>` |
| `select` / `search` / `dynamicWhere` 等查询语句 WHERE | **新增** `<if test="xxxHash != null and xxxHash != ''">AND xxx_hash = #{xxxHash}</if>`，与原有 `xxx_sm` 的 `<if>` **并列保留** |

**注意**：
- P4 阶段对 WHERE 的策略是**只加不删**：把 `xxx_hash` 的 `<if>` 块加进去，但原 `xxx_sm` 的 `<if>` 块**绝对保留**——V1 期间没人传 `xxxHash` 参数，新加的 `<if>` 不会拼出，行为完全等价于改造前；这样 V2 阶段无需再回头改 XML。
- Request DTO 在 P4 也要同步加 `xxxHash` 字段，确保 MyBatis OGNL `xxxHash != null` 不会抛 "no getter" 异常。
- select 列默认通过公共 `<sql>` 覆盖，不必每条 select 单独加列。

### 4.4 写入逻辑：补 hash

在每个写入 PO 的地方，**保留原 `setXxxSm()` 不变**，紧随其后补一行：

```java
// 兼容写入：补 xxx_hash（用于后续等值查询）
po.setXxxHash(QueryDigestUtil.digestFromCipher(po.getXxxSm()));
```

如果当前上下文已经有明文（典型如 `add` 入口），优先用更便宜的 `digestFromPlain`：

```java
// 入参带明文：先算 hash 再加密，避免重复解密
req.setXxxHash(QueryDigestUtil.digestFromPlain(req.getXxx()));
req.setXxxSm(SxfAksUtils.encrypt(req.getXxx()));
```

**自检**：所有写入路径（add / batchInsert / updateById / batchUpdate / Job 同步）都要补；没有写入路径的只读表跳过。

> **特别注意 `BeanUtils.copyProperties`**：若 P1 调研中发现某个写入路径是通过 `BeanUtils.copyProperties(src, dest)` 批量赋值的，dest 对象的 `xxxSm` 会被自动填入，但不会有显式的 `setXxxSm` 可以追加 hash。  
> 此时改造策略：在 `BeanUtils.copyProperties(src, dest)` 调用之后，**紧跟**补写 hash：
> ```java
> BeanUtils.copyProperties(src, dest);
> // 兼容写入：BeanUtils copy 后补 xxx_hash（copy 不会设置 hash，需手动补）
> if (dest.getXxxSm() != null) {
>     dest.setXxxHash(QueryDigestUtil.digestFromCipher(dest.getXxxSm()));
> }
> ```

### 4.5 Phase 4 验证（R12）

- 编译通过：`./gradlew :<module>:compileJava`
- 跑一遍受影响 Mapper 的单测（若有）
- 把 P1 清单中所有"修改文件"逐一 checklist 勾选

---

## Phase 5 · 创建 V2 分支（基于 V1）

> **执行方式**：Sonnet + 单点动作，主 Agent 直接执行。

```bash
git checkout BR_<DATE>_pci_ask_hash_v1
git checkout -b BR_<DATE>_pci_ask_hash_v2
```

V2 必须基于 V1（不是基于 master），否则会丢掉 hash 列的写入兼容。

---

## Phase 6 · V2：切等值查询到 hash

> **执行方式**：Sonnet + **subagent 并行**。P1.2 plan 中已按"比较点 / 查询入口"维度拆分独立任务（P6-T1 / P6-T2 / ...），**一条消息内并行派发**；每个 subagent 负责一个 P6-Tx，严格按改造点列表执行，**不得改 XML、不得改 DTO**（V1 已完工）。
> **禁止偏离 plan**：同 P4——发现缺漏暂停回主 Agent 修 plan。

**前提**：P4 已在 XML 中**并列加上** `xxx_hash` 的 `<if>`，并在 Request DTO 中加好 `xxxHash` 字段。所以 V2 阶段**不再动 XML、不再动 DTO**，只改两类代码：

1. **Java 内存比较**：所有 `xxxSm.equals(yyySm)` / `Collection.contains(xxxSm)` / `Objects.equals(xxxSm, ...)` 等改为 hash 比较。
2. **Service 查询入口**：把"加密后塞 `setXxxSm`"改为"hash 后塞 `setXxxHash`"——由于 XML 中 `_sm` 的 `<if>` 仍在但 `xxxSm` 不再被 set，条件不会拼出；新设的 `xxxHash` 走 hash 条件。

**hash 来源选择**：
- 入参带明文 → `QueryDigestUtil.digestFromPlain(plain)`
- 上下文只有密文 → `QueryDigestUtil.digestFromCipher(cipher)`

### 6.1 Java 比较改造模板

**改前**：
```java
boolean hit = StringUtils.isEmpty(s.getAccountNoSm())
        || s.getAccountNoSm().equals(businessOrder.getAccountNoSm());
```

**改后**：
```java
// 改用 hash 比较，避免直接对比 SM 密文（同明文不同密文时可能误判）
boolean hit = StringUtils.isEmpty(s.getAccountNoHash())
        || s.getAccountNoHash().equals(QueryDigestUtil.digestFromCipher(businessOrder.getAccountNoSm()));
```

### 6.2 Service 查询入口改造模板

**改前**：
```java
req.setAccountNoSm(SxfAksUtils.encrypt(req.getAccountNo()));
```

**改后**：
```java
// 改用 hash 等值查询，不再传入 SM 密文
req.setAccountNoHash(QueryDigestUtil.digestFromPlain(req.getAccountNo()));
```

> **注意**：不要保留原 `setAccountNoSm(...)`——保留会让 XML 同时拼 `account_no_sm = ?` 和 `account_no_hash = ?` 两个条件，老 SM 密文又匹配不上，直接查空。


### 6.3 Phase 6 验证（R12）

- 全量 `./gradlew compileJava`
- 受影响接口手工 / curl 验证：相同明文新老密文都能命中查询
- 黑名单 / 规则命中类逻辑：构造一个新加密的密文，确认仍能被命中

---

## 常见陷阱（必看）

| 陷阱 | 后果 | 防御 |
|------|------|------|
| V1 阶段顺手把 Service 入参从 `setXxxSm` 切成 `setXxxHash` | 灰度期老数据 hash 还没补刷 → 命中率骤降 | V1 **只加 XML 的 hash `<if>` 和 DTO 字段**；切入参放 V2，且 V2 前必须完成历史补刷 |
| V1 阶段把 XML 中 `xxx_sm` 的 `<if>` 删掉 | 回滚预案失效，老调用方传 SM 立刻查空 | 只**加**不**删**——`_sm` 和 `_hash` 两个 `<if>` 并列长期共存 |
| `batchInsert` 的 columns / values / ON DUPLICATE 三处只改一处 | INSERT 列数不匹配或 hash 不写入 | 用 grep `batchInsert` 定位三个锚点逐一验证 |
| `digestFromPlain` 和 `digestFromCipher` 用混 | hash 值对不上 → 永远 miss | 入参有明文用 plain；只有密文用 cipher，并加注释 |
| XML 编辑时贪婪匹配带走相邻 `<if>` | 其他 where 条件失效 | Edit `old_string` 取**最小**唯一上下文，验证 diff |
| 漏改写入路径 (Job、批量同步) | 历史数据 hash 永远空 | 在 P1 调研阶段穷举所有 `setXxxSm` 调用点 |
| V2 切完入参后忘记**移除**原 `setXxxSm()` | XML 同时拼 sm 和 hash 两个等值条件 → 老密文不匹配查空 | 见 6.2 "注意" |
| P4 加 XML hash `<if>` 但忘记给 DTO 加 `xxxHash` 字段 | MyBatis OGNL `xxxHash != null` 抛 "no getter" | 4.3 表格末行强制清单 |
| `BeanUtils.copyProperties` 写入路径漏补 hash | dest 对象 `xxxHash` 为 null → hash 列空，补刷脚本也无法覆盖 | P1 调研 grep `BeanUtils.copyProperties`，P4.4 在 copy 后紧跟补 hash |
| `BeanUtils.copyProperties` + `setXxxSm` 均未找到，直接当只读表跳过 | 实际有写入路径未被发现，导致 hash 列永久缺失 | 两种来源均缺失时，P1 文档标记 ⚠️，人工确认后才能继续 |

## 红色信号 - 立刻停下

- "我先把 P1 调研和 P4 改造一起做了" → 违反 HARD-GATE
- "V1 我把 WHERE 也顺手改了，反正一起" → 破坏灰度
- "这个表只有 select，应该不用补 hash" → 仍需补，可能被其他模块 INSERT
- "找不到 `QueryDigestUtil`" → 依赖 P3 没升级到位
- **"Phase 1 我用 Sonnet 跑就行"** → 违反铁律，必须 Opus + xhigh thinking
- **"P1.1 影响点直接连着 P1.2 plan 一口气产出"** → 违反双 HARD-GATE，必须先确认影响点再产 plan
- **"P1.2 plan 写完直接进 P2"** → 跳过 HARD-GATE-2
- **"P4 / P6 我串行一个一个改更稳"** → 违反 R4 + R9，独立任务必须 subagent 并行
- **"subagent 执行时发现 plan 缺一处，我顺手补上"** → 违反 R10 零自由度，必须暂停回 P1.2 修 plan 再派

## 输出规范（每阶段）

- **P1.1 完成**：贴出 `docs/pcihash/<YEAR>-PCI密钥轮换影响.md` 路径 + 字段汇总表 + ⚠️ 标记项，**等用户放行（HARD-GATE-1）**
- **P1.2 完成**：贴出 `docs/pcihash/plan/<YEAR>-PCI改动计划.md` 路径 + 并行任务拆分摘要（P4 / P6 各几个独立任务），**等用户放行（HARD-GATE-2）**
- **P2/P3/P5 完成**：贴当前分支名 + 修改文件列表 + 编译输出
- **P4/P6 完成**：贴当前分支名 + subagent 派发摘要（各 subagent 负责的任务 ID + 修改文件 + 编译输出）+ 整体 diff 摘要 + 验证证据（编译 / curl / 日志）

所有阶段对话语言：中文（对齐 R6）。所有新增 Java 字段、方法、关键逻辑：中文注释（对齐 R7）。
