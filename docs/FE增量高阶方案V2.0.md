# FE 增量高阶方案 V2.0

> 适用：`ia-fe-generator` Skill 的 `incremental` 模式（`tp-incremental-build` workflow）
> 遵循：设计文档 Skill 构建规范 v1.2.0 / 引擎 v2.1.0
> 与 PRD 增量方案 V3.0 在架构上同构、术语对齐
> 取代：v1.0
>
> **本方案是一份独立可读的产品方案文档**：自包含全部表格、依赖图、数据结构、流程定义；不依赖任何 YAML 或 Skill 实现文件即可被讨论、评审、培训。

---

## 目录

- 第一章 为什么需要这个方案（Why）
- 第二章 整体流程（What）
- 第三章 两条铁律
- 第四章 关键概念定义
- 第五章 原子变化点完整清单（18 个）
- 第六章 原子变化点 → FE 要素影响映射
- 第七章 always_affected 要素
- 第八章 依赖图（cascade 安全网）
- 第九章 六步执行流程详解
- 第十章 人工确认与最终交付物
- 第十一章 完整执行案例
- 附录 A 编号与引用体系
- 附录 B 与 PRD 增量方案的衔接
- 附录 C 与 v1.2.0 规范的同步事项

---

## 第一章 为什么需要这个方案（Why）

### 1.1 问题背景

企业 IT 系统进入 1→N 阶段后，业务侧每次提出优化需求都要回答相同的问题：

- 这条需求究竟在业务上变了什么？
- 这个变化影响 FE 文档的哪些章节（流程、功能、交互、非功能）？
- 哪些章节必须改、哪些可以不改、哪些**绝对不能动**（动了会破坏现有业务流程的连续性）？
- 改完后下游 PRD 阶段要对应做哪些设计？

这些问题有规律可循，但人工逐次分析效率低、遗漏多、结论不一致。本方案的目标是用一套结构化方法把"一段业务描述"转化为"精确的影响点 + 完整追溯链"，让业务方案设计人员只做最终确认而不是从零分析。

### 1.2 方案解决的核心问题

**问题一：影响域不清**
业务说"增加一个审批环节"，实际可能影响业务流程（流程图、活动总览、活动明细）、业务功能（新增功能、权限矩阵）、用户交互（页面承载）三个章节，且这些章节存在上下游依赖关系。人工分析容易只看到直接影响、漏掉传导影响。

**问题二：变更范围不明确**
业务侧描述常常是"新增某规则影响审批"，但实际审批可能涉及多个活动节点，**有些活动受影响、有些不受影响**。如果不显式声明哪些受影响、哪些不受影响，下游开发会自行扩展或漏改。

**问题三：业务方案细节缺失**
业务侧给的常常是一句话或一段描述，缺少 FE 文档需要的细节（活动输入输出、业务规则编号、角色清单等）。需要通过对话挖掘补齐，而且挖掘过程必须有据可依、不可猜测。

**问题四：追溯链断裂**
1→N 阶段经常一次接收多条业务需求，最终交付时无法回答"这个流程节点是为了满足哪条原始需求才加的"。本方案建立 **RR → 原子变化点 → ImpactPoint** 完整追溯链，并把这条链显式承载到下游 PRD 阶段。

### 1.3 方案的价值

通过结构化分析，把"几句业务语言的需求"转化为"精确的业务影响点 + 边界约束 + 完整追溯链"，让人只做最终确认。同时本方案产出的 ImpactPoint 与 RR 列表会作为下游 PRD 增量阶段的输入，**两阶段共享术语和数据结构**。

---

## 第二章 整体流程（What）

### 2.1 方案的角色定位

```
【输入层】
（A）原始需求 RR（用户提供的业务描述，可一条或多条）
（B）基线 FE 文档（必备，已 status=completed）
              ↓
【ChangeRouter 路由层】
（C）原子变化点 AtomicChange（用户描述映射到 18 个业务语言变化点之一）
              ↓
【影响域分析层】
（D）受影响 FE 要素清单
   - 主触发要素：原子变化点直接影响
   - 依赖传导要素：通过 dependency-graph 间接影响
   - always_affected 要素：每次增量必执行（original-requirement / requirement-type / glossary）
              ↓
【影响点确认层】
（E）ImpactPoint 影响点清单（含变更落点 + 边界约束）
              ↓
【输出层】
（F）增量 FE 文档（DELTA 标注的增量内容）
```

**分层说明**：

- **（A）原始需求 RR**：输入层，回答"业务想解决什么"。一次可输入多条 RR-1、RR-2…
- **（B）基线 FE**：输入层，是增量的事实基线。每条结论必须能回到基线某章节做对照。
- **（C）原子变化点**：路由层，回答"业务上到底变了什么"。共 18 个，覆盖 4 类业务域（PR/FN/UI/NF）。
- **（D）受影响 FE 要素**：承接层，回答"哪些 FE 章节需要变化"。
- **（E）ImpactPoint**：设计层，回答"每个章节具体怎么改、有什么边界约束、哪些活动/功能/页面不受影响"。
- **（F）输出层**：人工确认后落盘的最终交付物。

> **与 PRD 阶段的关键差异**：
>
> 1. **不读 PRD**：FE 阶段对 PRD 切干净，只读基线 FE 与用户输入
> 2. **不拆 Story**：Story 是 PRD 阶段的事，FE 阶段不涉及
> 3. **使用业务语言**：18 个原子变化点全部用业务语言表达（流程节点、活动、功能、页面、规则），与 PRD 的工程语言（FR、PAGE、INT 编号体系）形成对比

### 2.2 人做什么、AI 做什么

| 环节 | 执行者 | 内容 |
|---|---|---|
| 提供原始需求 | 人 | 业务描述（一句话或一段）；多条需求并存时分别给出 |
| 提供基线 FE | 人 | 路径或在 ongoing.md 中已记录 |
| Step 1~6 分析 | AI | 原子变化点识别、要素推导、影响点分析、对话挖掘补齐细节 |
| 暂停澄清 | 人 + AI | AI 暂停提问、人回答 |
| 审查草案 | 人 | 检查分析准确性、影响点完整性、是否漏掉边界约束 |
| 确认/修改/批准 | 人 | 对结果负责的节点 |
| 增量 FE 文档 | AI | 按确认结论结构化输出 |

---

## 第三章 两条铁律

这两条规则优先级高于一切，所有 Step 都须严格遵守。

### 3.1 铁律一：有理有据，不猜测

每条结论必须有明确依据。FE 增量场景下依据**只有两种**：

**evidence_source 两档**（与 PRD 增量场景的三档相比，FE 缺一档"上游文档"——FE 本身就是上游）：

| 档位 | 描述 | 标注示例 |
|---|---|---|
| `baseline_fe` | 来自基线 FE 文档原文 | "基线 FE §4.2 活动明细表-A05 行" |
| `dialog` | 来自用户当次澄清回答 | "根据用户澄清（Q3）：审批超时阈值定为 24 小时" |

**当依据不足时唯一合法的处理是暂停询问，不得推断或假设。**

具体表现：

- 描述说"增加预审环节"但没说预审角色 → 不能自行推断角色，必须询问
- 描述说"调整审批阈值"但没说目标值 → 不能假设阈值，必须询问
- 描述说"新增某功能"但没说功能边界 → 不能自行划定范围，必须询问

### 3.2 铁律二：随时暂停，主动澄清

分析过程不是一次性线性执行，**允许在任何 Step 中途暂停询问**。下列情形必须暂停：

| 触发情形 | 暂停时机 |
|---|---|
| 一句描述同时命中多个原子变化点，无法区分 | Step 1.1 识别变化点时 |
| 变化点识别置信度为"低" | Step 1.1 之后 |
| `conditional` 影响项的 condition 无法判断 | Step 2 |
| 受影响要素在基线 FE 中无对应章节 | Step 4 / Step 5 |
| ImpactPoint 的 target_state 无法从证据中推出（活动输入/输出、规则、角色等关键字段） | Step 5 |
| 边界约束（boundary_constraints）依赖业务侧约束知识，FE 无记录 | Step 5 |
| 同一基线活动/功能涉及多个原子变化点，影响是否叠加无法判断 | 任意 Step |

**统一暂停格式**：

```
⏸ 分析暂停 — 需要澄清以下问题，才能继续 [Step X / 要素 / 变化点]：

Q1：[具体问题]
背景：[为什么需要这个信息，不澄清会导致什么无法判断]

Q2：[具体问题]
背景：[同上]

请回答以上问题后，我将继续分析。
```

要求：
- 同一次暂停的问题集中一次性提出，不逐问逐答
- 每个问题必须配"背景"，让用户理解为什么这个问题必须回答
- 问题必须具体，禁止"请补充更多信息"这类泛化问句
- 收到回答后从暂停 Step 继续，输出中标注"根据用户澄清：[关键信息]"

---

## 第四章 关键概念定义

### 4.1 原始需求（RR — Raw Requirement）

```yaml
RR:
  id:           "RR-{序号}"           # 如 RR-01、RR-02
  description:  "用户原文，一字不改"
  source:       "对话输入 | 文档片段"  # 来源
  status:       "已分析 | 待澄清"     # 分析状态
```

**用法**：用户一次给多条需求时，AI 先把每条编号化（RR-01、RR-02…），便于后续追溯。

### 4.2 原子变化点（AtomicChange）

```yaml
AtomicChange:
  id:                 "{CATEGORY-NN}"   # 如 PR-01、FN-04
  source_requirement: "RR-{xx}"          # 引用的原始需求
  evidence:           "证据原文片段"
  evidence_source:    "baseline_fe | dialog"   # 证据来源档位
  confidence:         "high | medium | low"
  open_question:      "置信度非 high 时的待确认问题"
```

### 4.3 ImpactPoint（影响点，统一结构）

> **重要**：FE 增量场景与 PRD 增量场景共用同一种 ImpactPoint 结构。**取消 v1.2.0 规范第 3.7.3 节中的 `kind` 字段**，所有影响点统一为一个完整结构。"边界约束"作为可选子字段嵌入。
>
> **FE 阶段的特殊性**：
>
> - **引用更宽松**：FE 阶段没有 PRD 那种严格的 FR/E/PAGE 编号体系，baseline_ref 可使用章节路径或活动/功能/页面的业务名称（带编号最佳，无编号也可接受）
> - **必须显式标注"哪些不受影响"**：FE 增量经常出现"业务规则影响 2 个活动但第 3 个不影响"这种情况，**out_of_scope 字段比 PRD 阶段更重要**

```yaml
ImpactPoint:
  id:                  "IP-{全局序号}"
  source_requirement:  "RR-{xx}"          # 来源原始需求（可多个）
  source_change:       "{change_id}"       # 来源原子变化点
  trigger_type:        "primary | cascade" # 主触发 / 依赖传导
  cascade_rule:        ""                  # cascade 时填依赖图边的 reason
  element:             "{element_id}"      # 受影响的 FE 要素
  
  # ─── 变更落点 ───────────────────────
  baseline_ref:        "FE §X.Y 章节路径 / 业务名称（如'A05 部门预审活动'，可附带编号但不强求）"
  baseline_state:      "基线现状（直接引用基线内容；基线无此内容时写'基线无对应章节/字段'）"
  action:              "新增 | 修改 | 删除 | 复用 | 不涉及"
  target_state:        "变更后目标状态"
  target_state_evidence: "baseline_fe | dialog"  # 目标状态的证据来源
  
  # ─── FE 阶段特别强调的字段 ────────────
  in_scope:            ["明确包含的对象（活动/功能/页面/规则名称）"]
  out_of_scope:        ["明确排除的对象 — 同类对象中不受影响的部分"]
  out_of_scope_reason: "为什么这些对象不受影响（必填，避免下游误读为'忘了写'）"
  
  # ─── 边界约束（可选子字段）─────────
  boundary_constraints:
    - target:       "禁止改动的对象（活动名/功能名/页面名/规则编号等）"
      reason:       "禁止原因（业务连续性 | 合规约束 | 已上线依赖）"
      consequence:  "若违反会发生什么"
      evidence:     "依据来源（基线 FE 章节 / 用户澄清说明）"
```

**关于 `out_of_scope` 与 `out_of_scope_reason`（FE 阶段独有的强调）**：

PRD 阶段的 out_of_scope 主要是给下游 Coding Agent 划范围；FE 阶段的 out_of_scope 主要是**给业务评审人看**——业务侧最容易问"这个改动是不是把所有审批活动都影响了"，必须显式标注"是哪几个、不是哪几个、为什么"。

例如：业务规则 BR-审批阈值-001 从 5 万改为 10 万：
- in_scope：A02 采购申请审批活动、A05 出差申请审批活动
- out_of_scope：A08 报销审批活动
- out_of_scope_reason：报销审批走的是另一套规则 BR-报销阈值-002，不在本次调整范围

**与旧术语映射**：

| 旧概念（v1.0 / v1.2.0 规范）| 新概念（V2.0）|
|---|---|
| ChangePoint | ImpactPoint 主体 |
| ForbiddenItem | ImpactPoint.boundary_constraints[] |
| 改动点清单 + 禁止改动项清单 | 影响点清单（一份完整列表） |
| modify + forbid 分组 | 不分组，统一展示 |

---

## 第五章 原子变化点完整清单（18 个）

按 4 类业务域组织。每个变化点有唯一 ID、名称、用户视角描述、识别关键词、典型例子。

### 5.1 PR 类——业务流程变化（8 个）

| id | name | description_zh | detection_keywords | 例子 |
|---|---|---|---|---|
| PR-01 | 流程节点新增 | 在业务流程中增加新步骤/环节/活动 | 增加步骤、新增环节、加一步、增加活动 | "审批前增加部门预审环节" |
| PR-02 | 流程节点删除/合并 | 去掉或合并某个流程步骤 | 去掉步骤、删除环节、简化流程、合并 | "去掉部门预审，直接走总监审批" |
| PR-03 | 流程顺序调整 | 调整流程节点的执行顺序 | 调整顺序、流程顺序、放到前面、放到后面 | "把付款放到发货之前" |
| PR-04 | 角色调整 | 流程中执行某节点的角色变化 | 换角色、改角色、执行人变化 | "审批人从部门主管改为项目经理" |
| PR-05 | 活动输入变化 | 某步骤的输入信息字段或来源变化 | 增加输入、改输入、新增字段、改数据来源 | "申请提交时增加预算编码字段" |
| PR-06 | 活动输出变化 | 某步骤的输出结果或状态变更逻辑变化 | 输出变化、状态变化、产出 | "审批通过后新增同步到 ERP" |
| PR-07 | 业务规则新增/修改 | 业务规则增加、调整阈值或废弃 | 新增规则、规则调整、阈值、改规则 | "审批阈值从 5 万改为 10 万" |
| PR-08 | 异常处理新增/调整 | 异常分支或异常处理逻辑变化 | 异常处理、驳回、撤回、超时、退回 | "新增审批超时 24 小时自动转交" |

### 5.2 FN 类——业务功能变化（4 个）

| id | name | description_zh | detection_keywords | 例子 |
|---|---|---|---|---|
| FN-01 | 功能新增 | 增加一个完整的业务功能 | 新增功能、加功能、增加 | "增加批量导出功能" |
| FN-02 | 功能描述调整 | 已有功能的操作步骤、范围调整 | 改功能、调整功能、操作步骤 | "导出功能增加按时间筛选" |
| FN-03 | 功能下线 | 去掉某个已有功能 | 去掉功能、下线、移除 | "下线手工补录功能" |
| FN-04 | 业务权限变化 | 角色对功能的访问权限调整 | 权限调整、角色权限、访问权限 | "增加部门主管查看全部数据的权限" |

### 5.3 UI 类——用户交互变化（4 个）

| id | name | description_zh | detection_keywords | 例子 |
|---|---|---|---|---|
| UI-01 | 页面新增 | 新增一个完整页面 | 新增页面、加页面、新页面 | "新增订单详情页" |
| UI-02 | 页面字段/控件调整 | 在已有页面增/删字段、按钮、控件 | 加字段、改字段、加按钮、控件调整 | "申请表单增加优先级字段" |
| UI-03 | 页面流转变化 | 调整页面之间的跳转关系 | 流转调整、跳转关系、页面跳转 | "提交后直接跳详情页" |
| UI-04 | 页面下线 | 移除某个已有页面 | 下线页面、删页面、不需要页面 | "下线手工补录页" |

### 5.4 NF 类——非功能变化（2 个）

| id | name | description_zh | detection_keywords | 例子 |
|---|---|---|---|---|
| NF-01 | 性能要求变化 | 调整响应时间、并发、吞吐等指标 | 性能、响应、并发、吞吐 | "响应时间从 3 秒收紧到 1 秒" |
| NF-02 | 安全/隐私要求变化 | 调整加密、脱敏、合规要求 | 安全、加密、脱敏、合规 | "增加 GDPR 合规要求" |

---

## 第六章 原子变化点 → FE 要素影响映射

`impact_level` 三档：

- **certain**：一定影响，自动加入 effective_sequence
- **likely**：通常影响，默认加入但可由用户跳过
- **conditional**：条件影响，附 `condition`，按条件判断或交由用户决定

> 表头说明：每行展示一个变化点对各 FE 要素的影响等级与原因。
> FE 要素简称：BP=业务流程 / BF=业务功能 / UI=用户交互 / NFR=非功能需求。

| 变化点 | 受影响要素与等级 / 原因 |
|---|---|
| **PR-01** 流程节点新增 | BP(certain：流程图、活动总览、活动明细新增节点) / BF(likely：新节点通常对应新功能) / UI(conditional：新节点是否需 UI 承载) |
| **PR-02** 流程节点删除/合并 | BP(certain) / BF(likely：节点对应功能可能下线) / UI(conditional：节点对应页面是否仍保留) |
| **PR-03** 流程顺序调整 | BP(certain：流程图重排) / BF(conditional：顺序变化是否影响功能描述操作步骤) / UI(conditional：顺序变化是否影响页面流转) |
| **PR-04** 角色调整 | BP(certain：角色清单调整) / BF(certain：业务权限矩阵必然调整) |
| **PR-05** 活动输入变化 | BP(certain：输入子要素) / BF(likely：功能描述涉及的输入信息变化) / UI(likely：输入字段变化通常对应页面字段变化) |
| **PR-06** 活动输出变化 | BP(certain：输出子要素) / BF(likely) / UI(conditional：输出是否在页面呈现) |
| **PR-07** 业务规则新增/修改 | BP(certain：业务规则子要素) / BF(likely：功能描述可能引用规则) |
| **PR-08** 异常处理新增/调整 | BP(certain：活动明细异常处理字段) / BF(likely) / UI(conditional：异常处理是否需新页面) |
| **FN-01** 功能新增 | BF(certain：功能清单/描述新增) / BP(certain：新功能通常对应新业务活动) / UI(certain：新功能必有页面承载) |
| **FN-02** 功能描述调整 | BF(certain) / BP(conditional：是否触达活动级变化) / UI(likely：操作步骤变化通常涉及页面) |
| **FN-03** 功能下线 | BF(certain：功能清单删除) / BP(certain：对应活动同步下线) / UI(certain：对应页面同步下线) |
| **FN-04** 业务权限变化 | BF(certain：业务权限矩阵) / BP(likely：角色清单可能调整) |
| **UI-01** 页面新增 | UI(certain：页面清单+流转+低保真) / BF(likely：新页面通常对应新功能或扩展) / BP(conditional：新页面是否对应新活动) |
| **UI-02** 页面字段/控件调整 | UI(certain：低保真) / BF(likely：字段变化常涉及功能描述) / BP(likely：输入信息可能同步变化) |
| **UI-03** 页面流转变化 | UI(certain：页面流转) / BP(likely：流转变化通常映射流程顺序) |
| **UI-04** 页面下线 | UI(certain：页面清单+流转) / BF(likely：对应功能可能下线) |
| **NF-01** 性能要求变化 | NFR(certain：性能要求子要素) |
| **NF-02** 安全/隐私要求变化 | NFR(certain：信息安全 + 个人隐私保护 子要素) |

---

## 第七章 always_affected 要素

FE 增量每次都需重新审视的要素，不依赖变化点识别结果：

```yaml
- id: "original-requirement"
  always_affected_in: ["incremental"]   # 每次都要追加新需求来源到矩阵
  
- id: "requirement-type"
  always_affected_in: ["incremental"]   # 每次都重判类型（关键词可能因新需求变化）
  
- id: "glossary"
  always_affected_in: ["incremental"]   # 每次都扫描新出现的术语
```

**为什么这三个**：

- **original-requirement**：FE 文档头部的"原始需求来源矩阵"必须追加本次新需求条目；这是文档的根，不追加会导致追溯链断裂
- **requirement-type**：新增需求可能引入新关键词、改变需求类型判定（TP / AP / AI / IT），必须重判
- **glossary**：新需求中可能出现基线未定义的业务术语，必须扫描入库

> 与 PRD 增量的差异：PRD 增量的 always_affected 是 `story-design`（出口收口）；FE 增量的 always_affected 是三个入口/过程要素（追溯入口 + 类型判定 + 术语扫描）。这是因为 FE 与 PRD 在文档结构上的天然差异——FE 起始于业务侧的原始描述，PRD 终结于实施级的 Story 设计。

---

## 第八章 依赖图（cascade 安全网）

FE 各要素之间存在上下游依赖关系。第六章的"原子变化点 → 要素影响映射"只覆盖了"变化点 → 要素"的直接关系，依赖图补充"要素 → 要素"的间接关系，作为 ChangeRouter 第 4 步的安全网。

### 8.1 完整依赖图

```yaml
impact_edges:
  - source: business-process
    targets:
      - { element: business-function, impact_type: direct,
          reason: "业务活动是功能清单的来源，活动变化必然影响功能" }
      - { element: user-interaction,  impact_type: indirect,
          reason: "流程变化可能引发页面调整" }

  - source: business-function
    targets:
      - { element: user-interaction,  impact_type: direct,
          reason: "功能必有页面承载，功能变化常对应页面变化" }
      - { element: business-process,  impact_type: indirect,
          reason: "功能下线时对应活动需同步处理" }

  - source: user-interaction
    targets:
      - { element: business-function, impact_type: indirect,
          reason: "页面字段/操作变化反推功能描述需要更新" }
      - { element: business-process,  impact_type: indirect,
          reason: "页面流转变化可能反映流程顺序调整" }
```

### 8.2 安全网用法

ChangeRouter 第 4 步对 effective_sequence 做闭包扩展：

1. 遍历 impact_edges
2. 对 effective_sequence 中已有的每个 element，取出该 element 作为 source 的所有 **direct** targets
3. 凡 target 不在 effective_sequence 中的，归入"安全网额外发现"清单
4. 输出警告并由用户确认是否加入

**注意**：indirect 边不强制加入，仅作提示。

---

## 第九章 六步执行流程详解

> 与 PRD 增量方案 V3.0 的七步在前六步对应（PRD 多了 Step 7 Story 全局收口）。FE 阶段不拆 Story，止于 Step 5 + 收尾。

### Step 0：环境准备

读取 `workspace/ongoing.md` 提取版本号、project_name；定位基线 FE 文档（status=completed）；创建增量 FE 输出文档（与基线 FE 同目录、新日期戳），写入初始 frontmatter。

> 基线 FE 是 incremental 模式的硬前置——不存在则 Skill 不应进入 incremental，应走 build 模式。本节不展开校验细节。

### Step 1.0：原始需求登记（RR）

**输入**：用户对增量诉求的业务描述

**处理**：

1. 引导用户输入：
   ```
   请用一句话或一段话描述本次增量诉求（业务语言即可）。
   若有多条需求，请分条列出，AI 会逐条编号。
   例如：
     1. 审批前增加部门预审环节
     2. 审批阈值从 5 万改为 10 万
     3. 增加批量导出功能
   ```
2. 把每条需求结构化为 RR：

   ```yaml
   - id: RR-01
     description: "原文一字不改"
     source: "对话输入"
     status: "待分析"
   ```

3. 把 RR 列表写入增量 FE 的 frontmatter `requirement_register` 字段（同时这也是 always_affected 要素 `original-requirement` 在本次增量中的追加内容）

**暂停触发**：

- 单条需求过于模糊（如"优化体验"无具体动作）→ 暂停询问"该需求具体改什么"

---

### Step 1.1：原子变化点识别（ChangeRouter Step 1）

**输入**：RR 列表 + 基线 FE 文档

**处理**：对**每条 RR** 单独识别其原子变化点：

1. **关键词初筛**：用每个变化点的 detection_keywords 在 RR.description 中做模糊匹配，得到候选集合
2. **语义匹配**：对候选集合按 description_zh + examples 做语义判断
3. **证据收集**：每个识别出的变化点必须能引用以下之一作为 evidence，并标注 evidence_source：
   - `baseline_fe`：基线 FE 文档对应章节 + 引用片段
   - `dialog`：用户当前 RR 描述原文片段
4. **用户确认**：同一 RR 命中多个变化点时列出选项让用户选

**暂停触发**：

- 同一 RR 命中多个变化点且无法消歧
- 命中置信度为"低"
- RR 描述明显超出 18 个变化点的覆盖范围

**输出**：AtomicChange 列表，每项含 source_requirement / evidence / evidence_source / confidence。

---

### Step 2：影响汇聚（ChangeRouter Step 2）

**输入**：Step 1.1 的 AtomicChange 列表

**处理**：对每个变化点，从第六章映射表读 affects 列表，按 impact_level 处理：

- `certain` → 直接加入 effective_sequence
- `likely` → 加入但标记 optional_skippable=true，最终在 Step 4 由用户决定
- `conditional` → 检查 condition：
  - 条件可从证据判断 → 按结果决定
  - 条件无法判断 → 暂停询问

**输出**：候选 effective_sequence，每项保留 source_changes 来源跟踪。

---

### Step 3：always_affected 强制补全（ChangeRouter Step 3）

**处理**：将 `original-requirement` / `requirement-type` / `glossary` 三个要素强制加入 effective_sequence（若已存在则不重复）。

> 这三个要素在每次 incremental 都执行，与变化点识别结果无关。它们与变化点驱动的要素并列加入循环，按 chapter_no 排序。

---

### Step 4：依赖图安全网校验（ChangeRouter Step 4）

**处理**：见第八章 8.2。

输出：

```
🛡️ 依赖图安全网校验
当前 effective_sequence：[要素列表]
依赖图发现可能受影响但未在路由结果中的要素：
  - <element_id> ← 来源 <source_element_id>，原因：<reason>
是否加入 effective_sequence？
  [Y] 全部加入  [N] 全部忽略  [S] 选择性加入
```

**输出**：最终 effective_sequence（按 chapter_no 升序排序）。

---

### Step 5：执行计划展示与确认 + 要素循环执行

#### 5.1 执行计划展示

```
✅ 增量影响域分析完成

原始需求：
  RR-01: "..."
  RR-02: "..."

触发原子变化点：
  RR-01 → PR-01（high）
  RR-02 → PR-07（high）

受影响要素（按章节顺序）：
  1. original-requirement（always_affected）
  2. requirement-type（always_affected）
  3. glossary（always_affected）
  4. business-process（primary，触发 PR-01 + PR-07）
  5. business-function（cascade）
  6. user-interaction（cascade，UI 承载需确认）

不涉及要素：non-functional-req

[C] 开始执行  [B] 重做 ChangeRouter  [Q] 退出
```

收到 C 后进入 5.2。

#### 5.2 要素循环执行 + ImpactPoint 累积

按 effective_sequence 顺序，对每个 element 执行：

1. **读基线 FE 对应章节**：定位 chapter_info.l1_no 的章节，作为 baseline_state 引用来源
2. **筛本要素相关变化点**：从 effective_sequence 中筛出 affects 包含本 element 的变化点
3. **针对每个变化点对话挖掘细节**：沿用 build 模式的提问内核，但聚焦变化部分
   - 对话挖掘的内容标注 target_state_evidence = dialog
   - 引用基线 FE 的部分标注 target_state_evidence = baseline_fe
4. **生成 ImpactPoint**：按第四章 4.3 的统一结构。**FE 阶段必填 out_of_scope 与 out_of_scope_reason**——若同类对象（如多个审批活动）中只有部分受影响，必须显式标注哪些不受影响及原因
5. **生成 DELTA 标注内容**：

   ```html
   <!-- DELTA: change=<change_id>, chapter=<element_id>, op=<add|modify|delete>, level=<certain|likely|conditional> -->
   <增量内容（保留 Markdown 格式，遵循本要素的输出骨架）>
   <!-- /DELTA -->
   ```

6. **累积 ImpactPoint 到 frontmatter.impact_points**

**特殊要素处理摘要**：

| 要素 | incremental 模式核心动作 |
|---|---|
| original-requirement | 在原始需求来源矩阵追加 RR-NN 行，不动既有行 |
| requirement-type | 重新评估需求类型；若变化记录变更原因，否则维持 |
| glossary | 扫描 RR 与本次产生的 IP 中的新术语，追加术语条目 |
| business-process | 流程图（增量重绘并 DELTA 包裹）/ 活动总览表（追加或修改行）/ 活动明细（详细字段更新）/ 角色清单 / 业务规则 |
| business-function | 功能清单 / 功能描述 / 业务权限矩阵 |
| user-interaction | 页面清单 / 页面流转 / 低保真说明 |
| non-functional-req | 性能要求 / 信息安全 / 隐私保护 子要素 |

---

### Step 6：影响点汇总 + 草案输出

要素循环完毕后输出草案给用户。详见第十章。

---

## 第十章 人工确认与最终交付物

### 10.1 草案输出格式

Step 5 全部要素执行完毕后输出：

```
=== 增量 FE 影响域分析草案 ===

【一、原始需求】
RR-01: 描述 / 状态：已分析
RR-02: 描述 / 状态：已分析

【二、原子变化点】
RR-01 → PR-01（high，evidence_source=dialog）
RR-02 → PR-07（high，evidence_source=baseline_fe）

【三、受影响 FE 要素总表】
要素 | 触发类型 | 触发变化点 | 改动摘要

【四、不涉及要素说明】
要素 | 不涉及原因 | 验证依据

【五、影响点清单】
（统一列表，含 baseline_state / target_state / target_state_evidence / in_scope / out_of_scope / out_of_scope_reason；
 边界约束作为 boundary_constraints 子字段嵌入相邻影响点中。
 evidence_source=dialog 的项以 ⚠️ 标记。）

IP-001 [primary, source_change=PR-07, source_requirement=RR-02]
  element: business-process
  baseline_ref: 基线 FE §4.4 业务规则 → BR-审批阈值-001
  baseline_state: "审批阈值 5 万元"
  action: 修改
  target_state: "审批阈值调整为 10 万元（target_state_evidence=dialog）"
  in_scope: 
    - "A02 采购申请审批活动"
    - "A05 出差申请审批活动"
  out_of_scope:
    - "A08 报销审批活动"
  out_of_scope_reason: "报销审批走的是 BR-报销阈值-002，不在本次调整范围（基线 FE §4.4 已明确）"
  boundary_constraints:
    - target: "BR-报销阈值-002"
      reason: "业务规则边界 — 报销与采购/出差是不同业务域"
      consequence: "误改会导致报销审批流程异常"
      evidence: "基线 FE §4.4 业务规则编号体系"

IP-002 [...]

【六、追溯链路】
RR-02 → PR-07 → IP-001（business-process，A02、A05 in_scope，A08 out_of_scope）

=== 待确认问题汇总 ===
（仍存在的低置信度判断，集中列出）

请确认：
1. 分析结论是否准确？有无遗漏或错误？
2. 影响点的 in_scope / out_of_scope 划分是否准确？是否漏掉了"应该不受影响但被错列入 in_scope"的对象？
3. 边界约束是否完整？
4. evidence_source=dialog 的项是否需要再补证据？
```

### 10.2 最终落盘

用户确认后写入增量 FE 文档：

```markdown
---
（frontmatter：含 base_fe / requirement_register / triggered_changes / impact_points / stepsCompleted ...）
---

## 0. 变更说明
- 基线 FE：<path>
- 原始需求：<RR-01, RR-02, ...>
- 触发变化点：<PR-01, PR-07, ...>
- 本次变更范围摘要：<一段话>

## 1. 原始需求清单
| RR 编号 | 描述 | 触发变化点 | 状态 |

## 2. 原子变化点清单
（AtomicChange 列表，含 evidence、evidence_source、confidence）

## 3. 受影响要素总表
（element 表）

## 4. 不涉及要素说明

## 5. 各要素增量改动详情
（按 effective_sequence 顺序，每个 element 一段，含 DELTA 块）

### <章节编号> <element_name>
- 触发类型：primary / cascade
- 触发变化点：PR-07 / ...
- 基线章节：FE §X.Y
- 影响点引用：IP-001 / IP-002

<!-- DELTA: change=PR-07, chapter=business-process, op=modify, level=certain -->
具体增量内容
<!-- /DELTA -->

## 6. 影响点清单
（IP-001、IP-002...统一列表，每条含 in_scope / out_of_scope / out_of_scope_reason / boundary_constraints）
```

---

## 第十一章 完整执行案例

### 11.1 案例 A：单需求触发多变化点 + 暂停澄清

**场景背景**：

> 用户启动 Skill，对话输入：
>
> "审批前增加部门预审环节，由部门助理负责。"

**基线 FE 摘录**：

- §4.1 业务流程图：采购申请 → 部门主管审批 → 总监审批 → 完成
- §4.2 活动总览表：A01 申请提交 / A02 部门主管审批 / A03 总监审批
- §4.3 活动明细：A02 输入字段=申请单 / 输出=审批结果 / 角色=部门主管 / 规则=BR-审批阈值-001
- §4.5 角色清单：申请人、部门主管、总监、财务（无"部门助理"）
- §6 用户交互-页面清单：PG01 申请提交页、PG02 审批待办页

---

**🤖 Step 0：环境准备**

定位基线 FE：`workspace/requirements/I20251201/FE-采购系统-20251201.md` ✅
创建增量 FE：`workspace/requirements/I20260509/FE-采购系统-20260509.md`

---

**🤖 Step 1.0：原始需求登记**

```yaml
requirement_register:
  - id: RR-01
    description: "审批前增加部门预审环节，由部门助理负责。"
    source: "对话输入"
    status: "待分析"
```

---

**🤖 Step 1.1：原子变化点识别**

关键词命中：PR-01（"增加…环节"）。语义判断：
- 增加一个审批流程节点 → PR-01
- 涉及新角色"部门助理"（基线角色清单无此角色）→ 隐含 PR-04？

需要暂停消歧：

```
⏸ 分析暂停 — 需要澄清以下问题，才能继续 Step 1.1：

Q1：本次新增的"部门预审"环节是否仅是"增加新节点"，还是同时也涉及现有审批节点的角色调整？
背景：基线角色清单中无"部门助理"角色。"增加预审"的最直接解释是 PR-01（节点新增）；
但若你期望部门助理也参与后续节点（替代部门主管）则同时涉及 PR-04（角色调整）。

Q2：部门预审的输入和输出是什么？是否有专门的预审规则？
背景：FE 活动明细要求每个活动有清晰的输入字段、输出结果、业务规则。
若不澄清，无法生成完整的 A02-pre（部门预审）活动明细行。

请回答以上问题后，我将继续分析。
```

**用户回答**：

> Q1：仅是新增预审节点，部门主管审批节点的角色不变。
> Q2：输入是申请单（与原 A02 相同）；输出是预审结论（通过/退回）；规则是 BR-预审-001：金额>1万的申请才走预审，<=1万的跳过。

---

**🤖 根据用户澄清继续 Step 1.1**

```yaml
RR-01:
  - id: PR-01
    name: "流程节点新增"
    evidence: "RR-01 原文 + 用户澄清 Q1 确认仅是节点新增"
    evidence_source: dialog
    confidence: high
```

---

**🤖 Step 2：影响汇聚**

PR-01 → BP(certain) / BF(likely) / UI(conditional：新节点是否需 UI 承载)

针对 conditional：

```
⏸ Step 2 暂停澄清

Q3：部门预审节点是否需要专门的预审操作页面？还是部门助理在已有"审批待办页"上完成预审？
背景：UI(conditional) 受影响等级取决于是否需要新页面。

请回答：
  [A] 复用已有"审批待办页"
  [B] 需要新增"预审待办页"
```

**用户**：选 A，复用已有审批待办页。

→ user-interaction 不进入 effective_sequence。

候选 effective_sequence：
- business-process（certain）
- business-function（likely）

---

**🤖 Step 3：always_affected 补全**

加入：original-requirement / requirement-type / glossary。

---

**🤖 Step 4：依赖图安全网**

- business-process → business-function（已在）/ user-interaction（未在，indirect）
- business-function → user-interaction（未在，direct）

```
🛡️ 依赖图安全网校验
依赖图发现可能受影响但未在路由结果中的要素：
  - user-interaction ← 来源 business-function（direct），原因：功能必有页面承载

是否加入？
```

**用户**：N，因为 Q3 已确认复用已有页面。

最终 effective_sequence：original-requirement / requirement-type / glossary / business-process / business-function

---

**🤖 Step 5：执行计划 + 要素循环**

执行计划展示并用户确认 C 后进入循环。

#### 关键 ImpactPoint（节选）

**IP-001（business-process）**

```yaml
source_requirement: RR-01
source_change: PR-01
trigger_type: primary
element: business-process

baseline_ref: 基线 FE §4 业务流程（流程图、活动总览、活动明细）
baseline_state: "采购申请流程：A01 申请提交 → A02 部门主管审批 → A03 总监审批"
action: 新增
target_state: |
  在 A01 与 A02 之间新增 A02-pre 部门预审活动，活动明细如下：
  - 输入：申请单（同 A01 输出）
  - 角色：部门助理（新增到角色清单）
  - 输出：预审结论（通过/退回）
  - 业务规则：BR-预审-001（金额>1万的申请才走预审，<=1万的跳过 A02-pre 直接进入 A02）
target_state_evidence: dialog

in_scope:
  - "流程图：A01 → A02-pre → A02 → A03（A02-pre 含金额分支）"
  - "活动总览表：追加 A02-pre 行"
  - "活动明细：追加 A02-pre 完整明细"
  - "角色清单：追加'部门助理'"
  - "业务规则：追加 BR-预审-001"

out_of_scope:
  - "A02 部门主管审批活动（含其角色、输入、输出、规则）"
  - "A03 总监审批活动"
  - "BR-审批阈值-001（既有规则）"

out_of_scope_reason: |
  本次仅是在 A01 与 A02 之间插入新节点，A02 与 A03 的活动明细本身不变；
  既有规则 BR-审批阈值-001 仍作用于 A02 节点，与 BR-预审-001 是并列规则，互不影响。
  用户在 Q1 明确确认"部门主管审批节点的角色不变"。

boundary_constraints:
  - target: "A02 部门主管审批活动的角色字段（部门主管）"
    reason: "用户 Q1 明确说明本次仅新增节点、不调整已有节点角色"
    consequence: "误改会引入未声明的 PR-04 变化"
    evidence: "用户澄清 Q1"
```

**IP-002（business-function）**

```yaml
source_requirement: RR-01
source_change: PR-01
trigger_type: cascade
cascade_rule: "新节点通常对应新功能（依赖图：business-process → business-function direct）"
element: business-function

baseline_ref: 基线 FE §5.1 功能清单
baseline_state: "功能清单含'采购申请提交'、'采购单审批'、'采购单查询'"
action: 新增
target_state: "新增功能'采购单部门预审'，描述：部门助理对部门内员工提交的采购申请进行金额合理性预审"
target_state_evidence: dialog

in_scope:
  - "功能清单追加'采购单部门预审'"
  - "业务权限矩阵：'部门助理'对'采购单部门预审'功能可操作"

out_of_scope:
  - "'采购单审批'功能（既有功能）"
  - "其他角色对'采购单部门预审'功能的权限（默认无权限）"

out_of_scope_reason: |
  本功能为新增功能，无需修改既有功能描述。
  其他角色的权限默认沿用基线 FE 的权限矩阵（其他角色对新功能默认无权限），无需在权限矩阵中显式列出"无"的行。
```

**IP-003（original-requirement，always_affected）**

```yaml
source_requirement: RR-01
source_change: ""           # always_affected 不来自变化点
trigger_type: always_affected
element: original-requirement

baseline_ref: 基线 FE §1.1 原始需求来源矩阵
baseline_state: "已有需求条目 N 行"
action: 新增
target_state: "在原始需求来源矩阵末尾追加 RR-01 行：'审批前增加部门预审环节，由部门助理负责'，关联本次产生的 IP-001 / IP-002"
target_state_evidence: baseline_fe

in_scope:
  - "原始需求来源矩阵 RR-01 行"

out_of_scope:
  - "既有原始需求条目（前 N 行）"

out_of_scope_reason: "本次为追加，不修改既有需求条目，避免破坏历史追溯链"
```

**IP-004（glossary，always_affected）**

```yaml
source_requirement: RR-01
trigger_type: always_affected
element: glossary

baseline_ref: 基线 FE §2 术语表
baseline_state: "术语表含'采购申请'、'部门主管审批'、'总监审批'等"
action: 新增
target_state: |
  追加术语：
    - 部门预审（一级业务概念）
    - 部门助理（角色）
    - BR-预审-001（业务规则编号）
target_state_evidence: dialog

in_scope: ["术语表新增 3 条"]
out_of_scope: ["既有术语条目"]
out_of_scope_reason: "本次术语为补充，不修订既有术语定义"
```

> 注：requirement-type 要素的 IP 略（本次重判结论：仍为 TP 类，无变化）。

---

**🤖 Step 6：影响点汇总 + 草案输出**

按第十章 10.1 格式输出（略）。

---

### 11.2 案例 B：多需求 + 无暂停（清晰场景）

**场景背景**：

> 用户对话输入：
>
> 1. 审批阈值从 5 万改为 10 万
> 2. 增加批量导出功能（在采购单查询页）

**基线 FE 摘录**：

- §4.4 业务规则：BR-审批阈值-001 = 5 万（用于 A02 部门主管审批、A05 出差申请审批；A08 报销审批用 BR-报销阈值-002）
- §5.1 功能清单：含"采购单查询"（PG03 采购单查询页承载）
- §5.2 业务权限矩阵：采购员、部门主管、财务对"采购单查询"均有权限

---

**🤖 Step 0~1.0**：环境准备 + RR 登记

```yaml
requirement_register:
  - id: RR-01
    description: "审批阈值从 5 万改为 10 万"
    source: "对话输入"
    status: "待分析"
  - id: RR-02
    description: "增加批量导出功能（在采购单查询页）"
    source: "对话输入"
    status: "待分析"
```

---

**🤖 Step 1.1：变化点识别（无暂停）**

```yaml
RR-01:
  - id: PR-07
    name: "业务规则新增/修改"
    evidence: "RR-01 原文 + 基线 FE §4.4 BR-审批阈值-001 当前值 5 万"
    evidence_source: baseline_fe
    confidence: high

RR-02:
  - id: FN-01
    name: "功能新增"
    evidence: "RR-02 原文：增加批量导出功能"
    evidence_source: dialog
    confidence: high
```

---

**🤖 Step 2：影响汇聚**

PR-07 → BP(certain) / BF(likely)
FN-01 → BF(certain) / BP(certain) / UI(certain)

**针对 PR-07 的 likely**：用户确认是否影响 BF？

```
⏸ Step 2 暂停澄清

Q1：BR-审批阈值-001 调整为 10 万后，是否影响 FE 第 5 章功能描述（如'采购单审批'功能描述中提到的金额条件描述）？
背景：PR-07 对 business-function 是 likely 等级——业务规则修改通常会触达功能描述。
若功能描述中已经引用了"5 万"作为说明，则需要同步修改。

  [Y] 是，需同步修改功能描述
  [N] 否，功能描述未引用具体阈值
```

**用户**：Y。

候选 effective_sequence：business-process / business-function / user-interaction

---

**🤖 Step 3-4**：略，加入 always_affected 三要素，依赖图无新增。

---

**🤖 Step 5：要素循环**

#### IP-001（business-process，PR-07 修改阈值）

```yaml
source_requirement: RR-01
source_change: PR-07
trigger_type: primary
element: business-process

baseline_ref: 基线 FE §4.4 业务规则 → BR-审批阈值-001
baseline_state: "BR-审批阈值-001 = 5 万元；适用范围：A02 部门主管审批、A05 出差申请审批"
action: 修改
target_state: "BR-审批阈值-001 = 10 万元；适用范围不变"
target_state_evidence: dialog

in_scope:
  - "BR-审批阈值-001 阈值字段（5 万 → 10 万）"
  - "A02 活动明细业务规则字段（自动同步阈值描述）"
  - "A05 活动明细业务规则字段（自动同步阈值描述）"

out_of_scope:
  - "A08 报销审批活动"
  - "BR-报销阈值-002"

out_of_scope_reason: |
  报销审批走的是 BR-报销阈值-002，不引用 BR-审批阈值-001，本次调整不波及。
  基线 FE §4.4 业务规则编号体系明确两条规则相互独立。

boundary_constraints:
  - target: "BR-报销阈值-002"
    reason: "业务规则边界 — 报销与采购/出差是不同业务域"
    consequence: "误改会引发非本次需求范围的报销审批变更"
    evidence: "基线 FE §4.4"
```

#### IP-002（business-process，FN-01 触发新活动）

```yaml
source_requirement: RR-02
source_change: FN-01
trigger_type: primary
cascade_rule: "新功能通常对应新业务活动（FN-01 → BP certain）"
element: business-process

baseline_ref: 基线 FE §4 业务流程
baseline_state: "采购单查询流程：用户进入查询页 → 输入条件 → 查看列表"
action: 新增
target_state: |
  在采购单查询流程末尾增加批量导出活动：
  - 输入：用户已选择的采购单列表
  - 输出：Excel 文件
  - 角色：所有具有'采购单查询'权限的角色（采购员、部门主管、财务）
  - 业务规则：单次导出≤1000 条
target_state_evidence: dialog

in_scope:
  - "采购单查询流程图：追加批量导出节点"
  - "活动总览表：追加批量导出活动"
  - "活动明细：追加完整明细"
  - "业务规则：追加 BR-导出限额-001"

out_of_scope:
  - "其他流程（采购申请、采购审批、报销审批）"

out_of_scope_reason: "批量导出仅承载于'采购单查询'流程，与申请/审批/报销流程无关"
```

#### IP-003（business-function，PR-07 cascade 修改功能描述）

```yaml
source_requirement: RR-01
source_change: PR-07
trigger_type: cascade
cascade_rule: "PR-07 → BF likely（用户在 Step 2 Q1 确认 Y）"
element: business-function

baseline_ref: 基线 FE §5.1 功能清单 → "采购单审批"功能
baseline_state: "功能描述：'对金额超过 5 万的采购申请进行审批'"
action: 修改
target_state: "功能描述：'对金额超过 10 万的采购申请进行审批'"
target_state_evidence: dialog

in_scope:
  - "'采购单审批'功能描述中的金额"

out_of_scope:
  - "'采购单审批'功能的其他描述（操作步骤、范围）"
  - "其他功能（采购申请提交、采购单查询）"

out_of_scope_reason: "本次仅调整规则阈值，不调整功能本身的操作步骤或范围"
```

#### IP-004（business-function，FN-01 新增功能）

```yaml
source_requirement: RR-02
source_change: FN-01
trigger_type: primary
element: business-function

baseline_ref: 基线 FE §5.1 功能清单
baseline_state: "采购单查询、采购单审批、采购申请提交"
action: 新增
target_state: |
  新增功能：批量导出
  - 描述：从采购单查询页选择多条记录，导出为 Excel
  - 业务权限矩阵：与'采购单查询'权限相同（采购员、部门主管、财务）
target_state_evidence: dialog

in_scope:
  - "功能清单追加'批量导出'"
  - "业务权限矩阵新增'批量导出'行"

out_of_scope:
  - "其他功能"
out_of_scope_reason: "本次仅新增一个功能"
```

#### IP-005（user-interaction，FN-01 cascade 页面承载）

```yaml
source_requirement: RR-02
source_change: FN-01
trigger_type: primary
cascade_rule: "新功能必有页面承载（FN-01 → UI certain）"
element: user-interaction

baseline_ref: 基线 FE §6 用户交互 → PG03 采购单查询页
baseline_state: "PG03 含查询条件区、列表区、分页区"
action: 修改
target_state: |
  PG03 列表区新增'批量导出'按钮：
  - 位置：列表上方操作栏右侧
  - 行为：点击后导出当前列表选中的记录为 Excel
target_state_evidence: dialog

in_scope:
  - "PG03 列表区按钮变更"

out_of_scope:
  - "PG03 其他区域（查询条件区、分页区）"
  - "其他页面"

out_of_scope_reason: "本次仅在 PG03 列表区新增按钮，不动其他页面或同页面其他区域"
```

> always_affected 三个要素的 IP 略。

---

**🤖 Step 6：影响点汇总 + 草案输出**

按第十章 10.1 格式输出（略）。

---

*案例 B 说明了"多需求一次性输入"的场景：两条 RR 各自触发自己的变化点链路，但都涉及 business-process 与 business-function 要素。要素循环时同一要素可能产生多个 ImpactPoint（例如 IP-001 与 IP-002 都属于 business-process），分别引用各自的 source_requirement，追溯链清晰。*

---

## 附录 A：编号与引用体系

FE 阶段的编号体系比 PRD 阶段宽松，**不强求精确编号**：

| 编号 | 含义 | 来源 | 是否强制 |
|---|---|---|---|
| RR-NN | 原始需求 | Step 1.0 用户输入 | 强制 |
| {CATEGORY-NN} | 原子变化点 | 第五章固定目录（PR/FN/UI/NF）| 强制 |
| 活动名称（如 A02） | 业务流程的活动总览 | 基线 FE | 推荐有，无则用名称 |
| 角色名称（如 ROLE-001） | 角色清单 | 基线 FE | 推荐有，无则用名称 |
| 业务规则编号（如 BR-审批阈值-001）| 业务规则字段 | 基线 FE | 推荐有，无则用业务名称 |
| 页面编号（如 PG03） | 用户交互-页面清单 | 基线 FE | 推荐有，无则用名称 |
| 功能编号（如 BF-NN） | 业务功能-功能清单 | 基线 FE | 推荐有，无则用名称 |
| IP-NNN | 影响点编号 | Step 5 全局递增 | 强制 |

**引用规则**：

- 基线 FE 已用编号 → 增量 FE 沿用编号（不新建编号体系）
- 基线 FE 未用编号 → 增量 FE 用业务名称引用（如"A02 部门主管审批活动"）

---

## 附录 B：与 PRD 增量方案的衔接

FE 增量方案与 PRD 增量方案 V3.0 是**前后衔接的两个阶段**，共享术语和数据结构：

| 衔接点 | FE 阶段产出 | PRD 阶段消费 |
|---|---|---|
| 原始需求 | 增量 FE.frontmatter.requirement_register（RR-01、RR-02…）| PRD 增量 Step 1.0 直接引用，免去重新登记 |
| 原子变化点 | FE 用业务语言变化点（PR/FN/UI/NF 共 18 个）| PRD 用工程语言变化点（UI/DA/LG/PR/IN/NF 共 26 个），两套独立但概念对齐 |
| ImpactPoint | FE 阶段产出业务影响点（业务流程、功能、交互角度）| PRD 阶段产出工程影响点（架构、信息、特性、权限、集成角度），两阶段叠加形成完整影响域 |
| baseline_ref | 章节路径或业务名称（宽松）| 严格编号引用（FR-、E-、PAGE-）|
| 追溯链 | RR → 变化点 → IP（终止于 FE 阶段）| RR → 变化点 → IP → Story（PRD 阶段延伸）|

**关键原则**：

1. **FE 阶段不读 PRD**：FE 与 PRD 切干净
2. **PRD 阶段读 FE**：PRD 增量需要新版 FE 文档作为业务事实基线（也可以缺位）
3. **RR 在两阶段通用**：同一个 RR-NN 编号在 FE 与 PRD 阶段保持一致，便于跨阶段追溯

---

## 附录 C：与 v1.2.0 规范的同步事项

V2.0 在数据结构上**简化**了 v1.2.0 规范第 3.7.3 节中的 ImpactPoint 定义。具体差异：

| 维度 | v1.2.0 规范 | V2.0 方案 |
|---|---|---|
| 顶层字段 | `kind: modify \| forbid` | 取消 kind |
| 表达方式 | 两类 ImpactPoint 并列存在 | 一个 ImpactPoint 同时承载修改与边界约束 |
| forbid 表达 | `IP-xxx-forbid` 独立条目，含 reason/consequence/adjacent_to | `boundary_constraints` 子字段嵌入相邻 ImpactPoint |
| 草案展示 | 按 kind 分组（modify 类、forbid 类）| 统一影响点清单 |
| out_of_scope 字段 | 通用字段 | FE 阶段升级为**强制必填**，并新增 out_of_scope_reason 子字段 |

**简化理由**：

1. **认知一致性**：边界约束与修改本就是同一上下文中的两面，分两个清单破坏认知模型
2. **业务评审友好**：1→N 阶段的实际评审中，业务侧的注意力在某个章节时，希望看到"这个章节改什么、连带什么不改、什么不能改"在同一个位置
3. **out_of_scope 在 FE 阶段更重要**：FE 阶段大量出现"同类活动只影响部分"的场景（如多个审批活动只影响 2 个），强制 out_of_scope_reason 可避免下游误读

**对规范的回写建议**：

V2.0 落地后，与 PRD 增量方案 V3.0 同步触发设计文档 Skill 构建规范第 3.7.3 节的修订。本次修订**只影响增量场景**，不破坏 build 模式。

---

*本方案 V2.0，遵循设计文档 Skill 构建规范 v1.2.0，与 PRD 增量方案 V3.0 保持术语对齐与同构。任何规则调整都应同时检查"是否需要 PRD 增量方案同步"。*
