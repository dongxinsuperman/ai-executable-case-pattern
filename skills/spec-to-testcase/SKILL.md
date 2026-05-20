---
name: spec-to-testcase
description: >-
  Parse feature-list spec documents (with EARS-format rules, Mermaid flows,
  ASCII wireframes, and acceptance criteria) and generate structured functional
  test cases in a tab-indented 8-level hierarchy. Use when the user asks to
  generate test cases, create test plans, or produce QA checklists from a
  feature spec / PRD / feature-list markdown file.
---

# Spec → 功能测试用例生成

## 公开版说明

本 Skill 是“AI 可消费 case 模式”的公开 baseline，用于展示如何从 feature list / PRD 生成更适合 AI 执行器消费的测试用例。

它不是任何公司的完整落地规范。使用前应先根据团队自己的产品形态补齐：

- feature list / PRD 结构
- 测试账号、环境、入口路径
- 数据构造工具与异常注入方式
- UI 证据来源（截图 / 设计稿 / 录屏 / 用户答复）
- AI PHONE 或其他执行器的自动化边界

## 执行前必读

**最高权威·准出铁律**（不可妥协，三门铁律咬合输出）：

- [ai-executable-rules.md](ai-executable-rules.md) — 写法层：用例写法对照集（5 条铁律 + 22 对正反例 + 完整 case 综合对照）
- [execution-context-questionnaire.md](execution-context-questionnaire.md) — 信息层：信息缺口反问规范（7 类反问信号）
- [manual-case-rules.md](manual-case-rules.md) — 路由层：人工 case 判定规范（step-first 原则 + VLM 边界 6 类 + 非人工判定理由清单）

**辅助参考：**

- [spec-parsing-rules.md](spec-parsing-rules.md) — Spec 解析细则（EARS / Mermaid / ASCII / 验收项）
- [test-design-methods.md](test-design-methods.md) — 测试设计方法（等价类 / 边界值 / 状态转换 / 判定表 / 场景 / 错误推测）
- [output-examples.md](output-examples.md) — 输出格式产物示例
- [requirement-supplement.md](requirement-supplement.md) — 需求特定补充（可空）

## 概述

本 Skill 将遵循固定格式的产品需求 Spec（feature-list.md）解析为结构化的功能测试用例。Spec 采用 EARS 格式描述业务规则，包含 Mermaid 交互流程图、ASCII 示意图和验收项。

## 工作流程

```
读取 Spec 文件
  → Phase 1:   解析模块与功能点结构
  → Phase 2:   拆解测试功能点
  → Phase 3:   为每个测试功能点生成测试用例
  → Phase 4:   自检 lint（覆盖度 + 写法回扫 + 信息回扫）
  → Phase 4.5: 人工 case 总检（VLM 边界判定打【人工】标签）
  → 输出结果（同时生成 .txt 和 .md 两个文件）
```

> **反问能力贯穿全流程。** 任一 Phase、任一字段命中 7 类反问信号即触发反问，规则以 [execution-context-questionnaire.md](execution-context-questionnaire.md) 为准。允许批量攒问、用户答复后再次回扫整批 case；**轮数不限，必须问到 7 类信号全部归零才能放行**。不存在"过了某个 Phase 就豁免反问"的语义。

---

## Phase 1：解析 Spec 结构

从 Spec 中识别以下层级：

| Spec 元素 | 提取内容 |
|-----------|---------|
| `## 模块 N：xxx` | 模块名称、优先级（P0/P1/P2） |
| `#### 功能点 X：xxx` | 产品功能点名称 |
| `**业务规则**` 下的 EARS 条目 | 前置条件 + 触发动作 + 预期行为 |
| `**异常处理**` 下的 EARS 条目 | 异常条件 + 降级行为 |
| `**交互说明**` / `**交互逻辑**` | 交互细节与约束 |
| `**验收项**` 的 checkbox 列表 | 已有的验收骨架 + 测试数据提示 |
| Mermaid `flowchart` | 关键路径与分支路径 |
| ASCII 示意图 + 状态流转 | 状态集合与转换条件 |
| `**埋点需求**` 表格 | 不生成用例，仅作辅助参考 |

### EARS 解析规则

EARS 条目格式为 `EARS-X（类型）：**当**...**时**，系统应...`

| EARS 类型 | 含义 | 测试关注点 |
|-----------|------|-----------|
| EARS-U（普遍规则） | 任何情况下系统都应满足 | 作为基线断言，每条都需验证 |
| EARS-E（事件驱动） | 当某事件发生时触发 | 触发事件 → 预期响应 |
| EARS-S（状态驱动） | 当系统处于某状态时 | 不同状态下的差异行为 |
| EARS-W（当...时） | 异常/边界条件 | 异常场景 + 降级/兜底行为 |

---

## Phase 2：拆解测试功能点

产品功能点颗粒度粗，需拆解为更细的测试功能点。从以下 6 个维度审视每个产品功能点：

| # | 维度 | 说明 | 判断依据 |
|---|------|------|---------|
| 1 | **正向流程** | 用户完成主干操作的完整路径 | 交互说明中的主流程 + Mermaid 主路径 |
| 2 | **反向/取消** | 用户中途放弃、回退、取消 | 交互说明中的取消/关闭描述 |
| 3 | **边界条件** | 极限值、临界点、空数据、满数据 | EARS-W 条目 + 业务规则中的数值约束 |
| 4 | **异常与容错** | 网络失败、数据异常、接口超时 | `**异常处理**` 整节 |
| 5 | **状态驱动** | 不同初始状态下的行为差异 | EARS-S 条目 + 状态流转图 |
| 6 | **多端/适配** | 不同设备、屏幕、分辨率下的表现 | 响应式适配章节 + 交互说明中的设备相关描述 |

**拆解原则：**

- 每个测试功能点应聚焦一个独立的可测试行为
- 如果一个维度在该功能点中无适用场景（如纯后端逻辑无多端差异），则跳过该维度
- 维度 6（多端/适配）：仅在 Spec 中明确提及设备差异时才拆出，不强制每个功能点都有

---

## Phase 3：生成测试用例

### 输出格式（严格遵守）

每次生成时，**同时输出两个文件**：

| 文件 | 格式 | 用途 |
|------|------|------|
| `xxx测试用例.txt` | Tab 缩进 8 级层次 | 粘贴进飞书思维笔记（编辑器会把 Tab 转空格，所以必须用 `.txt`） |
| `xxx测试用例.md` | Markdown 标题 + 表格 | 在 IDE / GitHub / 飞书文档中阅读和协作 |

两个文件包含**完全相同的用例内容**，仅排版格式不同。

---

#### 8 级层级与缩进映射

| Level | 内容 | 来源 | `.txt` 缩进 | `.md` 缩进 |
|-------|------|------|-------------|------------|
| 1 | Spec 版本名 + "测试用例" | Spec 文档元信息 | 0 Tab | 0 空格 |
| 2 | 模块名（优先级） | Spec 模块总览表 | 1 Tab | 2 空格 |
| 3 | 产品功能点名 | Spec `#### 功能点 X：xxx` | 2 Tab | 4 空格 |
| 4 | 测试功能点名 | Phase 2 拆解产出 | 3 Tab | 6 空格 |
| 5 | 测试标题：【场景标签】xxx | Phase 3 生成 | 4 Tab | 8 空格 |
| 6 | 前置条件：xxx | 从 EARS / 验收项提取 | 5 Tab | 10 空格 |
| 7 | 操作步骤：xxx | 用户视角操作动作 | 6 Tab | 12 空格 |
| 8 | 预期结果：xxx | 可感知的系统响应 | 7 Tab | 14 空格 |

#### 格式硬性约束

**通用（两种格式均适用）：**

1. 标题 / 前置条件 / 操作步骤 / 预期结果 **必须完整写在一行**，不可换行拆分
2. 多个前置条件用中文分号 `；` 分隔
3. 操作步骤多步用中文顿号 `、` 分隔
4. **预期结果只允许写 1 条断言**（"一条"是计数单位）——这条断言可以是单一视觉特征，也可以**聚合围绕同一观察对象的多个同类视觉特征**（如形态 + 配色）；但跨对象 / 跨业务面的并列断言必须拆成多条 case。其余事实降级为"也观察"不写入预期。详见 [ai-executable-rules.md](ai-executable-rules.md) 第 21 节
5. **case 是单线瀑布**——前置 / 步骤 / 预期任一字段出现「如果...则...」「若不存在则...」「找不到改用...」「按相同规则...」等决策结构 → 立即触发 [execution-context-questionnaire.md](execution-context-questionnaire.md) 反问，把决策外移成"确定值"，不可输出
6. 两个文件的层级结构、内容文字、场景标签必须完全一致（包括【】标签）
7. **emoji 与颜色写死禁出现**——case 任何字段不可含 emoji / 表情符号（spec 或截图含 emoji 时只引用其文字部分）；任何颜色描述必须紧跟 `（颜色仅参考）` 容差注释。详见 [ai-executable-rules.md](ai-executable-rules.md) 第 22、23 节

**`.txt` 专项：** 每级缩进 1 个 Tab（Level 1 = 0 Tab，Level 8 = 7 Tab）

**`.md` 专项：** 每级缩进 2 个空格，以 `- ` 开头（Level 1 = 0 空格，Level 8 = 14 空格）

完整产物示例见 [output-examples.md](output-examples.md)。

### 场景标签

每条测试用例标题前加标签，标识场景类型：

| 标签 | 含义 | 来源 |
|------|------|------|
| `【正向】` | 正常流程验证 | Phase 2 维度 1 |
| `【反向】` | 用户取消/回退/拒绝 | Phase 2 维度 2 |
| `【边界】` | 极限值/空值/临界值 | Phase 2 维度 3 |
| `【异常】` | 网络错误/数据异常/接口失败 | Phase 2 维度 4 |
| `【状态】` | 不同前置状态下的行为 | Phase 2 维度 5 |
| `【兼容】` | 多端/多屏适配 | Phase 2 维度 6 |

### 执行方式标签

写作阶段**不预判**——所有 case 按 VLM 可执行写作。`【人工】` 路由标签由 Phase 4.5 总检统一判定，规则详见 [manual-case-rules.md](manual-case-rules.md)。

### 优先级规则

优先级标注在模块（Level 2）层级，直接继承 Spec 中模块总览表的优先级。所属该模块下的所有用例默认继承该优先级。格式：`模块名（P0）`。

### 测试设计方法

生成用例时，按测试功能点的特征选用合适的设计方法。详见 [test-design-methods.md](test-design-methods.md)。

### 写作质量标准

具体写法以 [ai-executable-rules.md](ai-executable-rules.md) 为最终约束。

---

## Phase 4：自检 lint（三层 lint 全过才能输出）

生成完毕后，对每条 case 执行三层回扫。任一层不过 → 定位违规字段 → 改写或触发反问 → 重过 lint。**不重生成全量，只修违规条目。**

### A. 覆盖度回扫

- [ ] Spec 中每个 `#### 功能点` 都有对应的测试功能点
- [ ] Spec 中每条 EARS 业务规则都被至少一条用例覆盖
- [ ] Spec 中每条异常处理都有对应的【异常】用例
- [ ] Spec 中每条验收项都能在用例中找到对应
- [ ] Mermaid 流程图中的每个分支路径都被覆盖
- [ ] 状态流转图中的每个状态转换都被覆盖
- [ ] 6 个维度中适用的维度均有用例覆盖
- [ ] `.txt` 和 `.md` 两个文件的用例内容完全一致（数量、标题、前置条件、操作步骤、预期结果）

### B. 写法回扫

逐条 case 按 [ai-executable-rules.md](ai-executable-rules.md) 的「准出铁律 + 22 对正反例 + 完整 case 综合对照」回扫；命中任一反例特征即定位违规字段，改写后再过 lint。

### C. 信息回扫

逐条 case 按 [execution-context-questionnaire.md](execution-context-questionnaire.md) 的 7 类反问信号回扫；命中即触发反问，不存在"已经过 Phase 3 就不用再问"的豁免。反问得到答复后写入 case，重过 lint。

---

## Phase 4.5：人工 case 总检（入口判定 + 出口复检 双把关）

Phase 4 三层 lint 全过后，**回扫所有 case**，按 [manual-case-rules.md](manual-case-rules.md) 双层执行：

### 入口判定（step-first → 6 类边界）

1. 对每条 case 先做 **step-first 反问**：「这个状态能否由步骤路径触发到位？」能 → 改写步骤完整构造、不打【人工】
2. step-first 通不过的，再判 6 类 VLM 边界：命中任一类先反问「能否改自动」，改不了在标题最前面打 `【人工】` 路由标签

### 出口复检（与入口同源约束）

3. 对所有已打【人工】的 case 逐条核对 [manual-case-rules.md](manual-case-rules.md) 的「非人工判定理由」清单：
   - [ ] 不是以"预置成本高"为由打的【人工】
   - [ ] 不是以"步骤稍长（< 60 步）"为由打的【人工】
   - [ ] 不是以"UI 推测"为由打的【人工】（应走信息层反问）
   - [ ] 不是以"状态构造繁琐 / 需先达到 X 态"为由打的【人工】
   - 命中任一项 → 立即降为 VLM 重写步骤、再过 Phase 4 三层 lint

> 【人工】是路由标签，不放宽任何写法 / 信息层铁律——前两门已在 Phase 3 / 4 闭合，本步骤只决定执行方式。
> 入口 + 出口同源约束，避免 AI 以"省力捷径"误打【人工】。

---

## 配套数据文件处理

当 Spec 目录下存在 JSON 格式的标签映射 / 筛选逻辑文件时：

1. 读取 JSON 文件，理解数据结构
2. 对存在等价类关系的数据（多套结构同形的业务模板 / 多套同质标签集），每套取 1 个代表生成用例
3. 在用例的前置条件或操作步骤中引用具体的标签值作为测试数据
4. 不对所有排列组合穷举，避免用例爆炸
