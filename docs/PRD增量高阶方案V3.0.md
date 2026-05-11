# PRD 增量高阶方案 V3.0

> 适用：`ia-fe-to-prd` Skill 的 `incremental` 模式
> 遵循：设计文档 Skill 构建规范 v1.2.0 / 引擎 v2.1.0
> 与 FE 增量方案在架构上同构、术语对齐
> 取代：v1.0（旧术语 AtomicScenario/ChangePoint/ForbiddenItem）、v2.0（结构残缺）
>
> **本方案是一份独立可读的产品方案文档**：自包含全部表格、数据结构、流程定义；不依赖任何 YAML 或 Skill 实现文件即可被讨论、评审、培训。

---

## 目录

- 第一章 为什么需要这个方案（Why）
- 第二章 整体流程（What）
- 第三章 两条铁律
- 第四章 关键概念定义
- 第五章 原子变化点完整清单（26 个）
- 第六章 原子变化点 → PRD 要素影响映射
- 第七章 always_affected 要素与 Story 全局收口机制
- 第八章 依赖图（cascade 安全网）
- 第九章 七步执行流程详解
- 第十章 增量 Story 拆分原则
- 第十一章 人工确认与最终交付物
- 第十二章 完整执行案例
- 附录 A 编号体系总览
- 附录 B 与 v1.2.0 规范的同步事项

---

## 第一章 为什么需要这个方案（Why）

### 1.1 问题背景

企业 IT 系统进入 1→N 阶段后，每次提出增量需求都要回答相同的问题：

- 这条需求究竟在产品上变了什么？
- 这个变化影响 PRD 的哪些章节？
- 哪些章节必须改、哪些可以不改、哪些**绝对不能动**（动了会破坏现有依赖）？
- 改完后开发怎么拆 Story、怎么验收？

这些问题有规律可循，但人工逐次分析效率低、遗漏多、结论不一致。本方案的目标是用一套结构化方法把"一段业务描述"转化为"精确的影响点 + 可执行的 Story + 完整追溯链"，让设计人员只做最终确认而不是从零分析。

### 1.2 方案解决的核心问题

**问题一：影响域不清**
业务说"加一个按钮"，实际可能影响应用架构、UI 原型、信息架构、功能特性、权限设计 5 个章节，且这些章节存在上下游依赖关系，改 A 必须同步检查 B。人工分析容易只看到直接影响、漏掉传导影响。

**问题二：边界约束不可控**
在 Coding Agent 参与开发的场景下，如果没有明确的"什么不能改"清单，Agent 可能在分析中自行扩展改动范围，改到不该改的地方（如修改既有接口契约、删除既有 AC）。

**问题三：验收标准不可执行**
传统 PRD 的 AC 是给人读的描述，无法直接转化为测试用例，也无法让 Agent 自我核查。本方案要求每个 Story 配 BDD 格式 AC（Given/When/Then），机器可断言。

**问题四：追溯链断裂**
1→N 阶段经常一次接收多条需求，最终交付时无法回答"这个 Story 是为了满足哪条原始需求"。本方案建立 **RR → 原子变化点 → ImpactPoint → Story** 完整追溯链。

**问题五：FE 缺位**
实际项目中常见"业务方一句话需求直接抛给 PRD 环节，没有增量 FE 文档"。方案必须能在没有 FE 的情况下兜底执行，通过对话挖掘补齐证据。

### 1.3 方案的价值

通过结构化分析，把"几句业务描述（含或不含 FE 文档）"转化为"精确的影响点 + 边界约束 + 可执行的 Story"，让人只做最终确认。

---

## 第二章 整体流程（What）

### 2.1 方案的角色定位

```
【输入层】
（A）原始需求 RR（用户提供的业务描述，可一条或多条）
（B）基线 PRD 文档（必备，已 status=completed）
（C）新版本 FE 文档（可选；缺位时由对话挖掘兜底）
              ↓
【ChangeRouter 路由层】
（D）原子变化点 AtomicChange（用户描述映射到 26 个工程语言变化点之一）
              ↓
【影响域分析层】
（E）受影响 PRD 要素清单
   - 主触发要素：原子变化点直接影响
   - 依赖传导要素：通过 dependency-graph 间接影响
   - always_affected 要素：每次增量必执行（仅 story-design）
              ↓
【影响点确认层】
（F）ImpactPoint 影响点清单（含变更落点 + 边界约束）
              ↓
【Story 全局收口层】
（G）增量 Story 列表（按合并规则统一拆分，含完整 AC）
              ↓
【输出层】
（H）增量 PRD 文档（DELTA 标注的增量内容）
（I）独立 Story 文件（与 PRD 同目录）
```

**分层说明**：

- **（A）原始需求 RR**：输入层，回答"业务想解决什么"。一次可输入多条 RR-1、RR-2…
- **（B）基线 PRD**：输入层，是增量的事实基线。每条结论必须能回到基线某章节做对照。
- **（C）新版本 FE**：可选输入。存在则作为业务事实证据；缺位则方案降级为对话挖掘。
- **（D）原子变化点**：路由层，回答"产品上到底变了什么"。共 26 个，覆盖 6 类业务域（UI/DA/LG/PR/IN/NF）。
- **（E）受影响 PRD 要素**：承接层，回答"哪些 PRD 章节需要变化"。
- **（F）ImpactPoint**：设计层，回答"每个章节具体怎么改、有什么边界约束"。
- **（G）增量 Story**：收口层，回答"开发怎么落地、怎么验收"。**全局收口产出**——所有 ImpactPoint 收齐后统一拆分。
- **（H）（I）输出层**：人工确认后落盘的最终交付物。

### 2.2 人做什么、AI 做什么

| 环节 | 执行者 | 内容 |
|---|---|---|
| 提供原始需求 | 人 | 业务描述（一句话或一段）；多条需求并存时分别给出 |
| 提供基线 PRD | 人 | 路径或在 ongoing.md 中已记录 |
| 提供新版本 FE（可选）| 人 | 路径；缺位时由 AI 通过对话挖掘补齐 |
| Step 1~7 分析 | AI | 原子变化点识别、要素推导、影响点分析、Story 全局拆分 |
| 暂停澄清 | 人 + AI | AI 暂停提问、人回答 |
| 审查草案 | 人 | 检查分析准确性、影响点完整性、Story 拆分合理性 |
| 确认/修改/批准 | 人 | 对结果负责的节点 |
| 增量 PRD + Story 文件 | AI | 按确认结论结构化输出 |

---

## 第三章 两条铁律

这两条规则优先级高于一切，所有 Step 都须严格遵守。

### 3.1 铁律一：有理有据，不猜测

每条结论必须有明确依据。依据只有三种：

**evidence_source 三档**：

| 档位 | 描述 | 标注示例 |
|---|---|---|
| `fe_doc` | 来自新版本 FE 文档原文 | "FE §4 业务流程 → 活动明细第 3 行" |
| `baseline_prd` | 来自基线 PRD 文档原文 | "基线 PRD §4.3 实体详情 PurchaseOrder" |
| `dialog` | 来自用户当次澄清回答 | "根据用户澄清（Q3）：默认 null，不补填" |

**当依据不足时唯一合法的处理是暂停询问，不得推断或假设。**

具体表现：
- 描述说"录入供应商信息"但没说具体字段 → 不能自行推断字段，必须询问
- 基线 PRD 的权限矩阵章节标注"待完善" → 不能假设权限规则，必须询问
- 描述说"历史数据需要处理" → 不能自行设计迁移策略，必须询问

### 3.2 铁律二：随时暂停，主动澄清

分析过程不是一次性线性执行，**允许在任何 Step 中途暂停询问**。下列情形必须暂停：

| 触发情形 | 暂停时机 |
|---|---|
| 一句描述同时命中多个原子变化点，无法区分 | Step 1.1 识别变化点时 |
| 变化点识别置信度为"低" | Step 1.1 之后 |
| `conditional` 影响项的 condition 无法判断 | Step 2 |
| 受影响要素在基线 PRD 中无对应章节 | Step 4 / Step 5 |
| ImpactPoint 的 target_state 无法从证据中推出 | Step 5 |
| 边界约束（boundary_constraints）依赖外部系统知识，PRD 无记录 | Step 5 |
| 基线 PRD 与新版 FE 存在事实矛盾 | 任意 Step |
| Story 合并判定不明确（多 FR 是否真的可合并） | Step 7 |

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
  id:               "{CATEGORY-NN}"   # 如 UI-01、DA-04
  source_requirement: "RR-{xx}"        # 引用的原始需求
  evidence:         "证据原文片段"
  evidence_source:  "fe_doc | baseline_prd | dialog"  # 证据来源档位
  confidence:       "high | medium | low"
  open_question:    "置信度非 high 时的待确认问题"
```

### 4.3 ImpactPoint（影响点，统一结构）

> **重要**：v3.0 取消 v1.2.0 规范第 3.7.3 节中的 `kind` 字段，所有影响点统一为一个完整结构。"边界约束"作为可选子字段嵌入。这是给设计人员减负——**改什么、不改什么、为什么不能改，都是同一个影响点的不同侧面**。

```yaml
ImpactPoint:
  id:                  "IP-{全局序号}"
  source_requirement:  "RR-{xx}"        # 来源原始需求（可多个，合并影响点时）
  source_change:       "{change_id}"    # 来源原子变化点
  trigger_type:        "primary | cascade"  # 主触发 / 依赖传导
  cascade_rule:        ""               # cascade 时填依赖图边的 reason
  element:             "{element_id}"   # 受影响的 PRD 要素
  
  # ─── 变更落点 ───────────────────────
  baseline_ref:        "PRD §X.Y 章节引用"
  baseline_state:      "基线现状（直接引用基线内容；基线无此内容时写'基线无对应章节/字段'）"
  action:              "新增 | 修改 | 删除 | 复用 | 不涉及"
  target_state:        "变更后目标状态"
  target_state_evidence: "fe_doc | baseline_prd | dialog"  # 目标状态的证据来源
  in_scope:            ["明确包含的对象列表"]
  out_of_scope:        ["明确排除的对象列表"]
  
  # ─── 边界约束（可选子字段）─────────
  boundary_constraints:
    - target:       "禁止改动的对象（实体名/字段名/接口名/编号等）"
      reason:       "禁止原因（现有依赖 | 合规约束 | 架构边界 | 历史数据风险）"
      consequence:  "若违反会发生什么"
      evidence:     "依据来源（基线 PRD 章节 / 用户澄清说明）"
```

**与旧术语映射**：

| 旧概念（v1.0 / v1.2.0 规范）| 新概念（v3.0）|
|---|---|
| ChangePoint | ImpactPoint 主体 |
| ForbiddenItem | ImpactPoint.boundary_constraints[] |
| 改动点清单 + 禁止改动项清单 | 影响点清单（一份完整列表） |

**为什么这样设计**：边界约束总是与某个具体的修改点相邻产生（例如"修改 PurchaseOrder 实体加 supplier_code 字段"必然伴随"order_no 主键禁止修改"这条约束）。强行分两个清单会让评审时反复跳读、对应关系易错。一个 IP 表达一个修改连同它的禁区，是设计人员最自然的认知模型。

### 4.4 增量 Story

```yaml
Story:
  id:                   "S-FR-{特性分类}-{特性}-{序号}-{Story序号}"  # Functional Story
                        # 或 "S-NFR-{NFR编号}-{Story序号}"          # NFR Story
  name:                 "Story 名称"
  story_type:           "Functional | NFR"
  
  # ─── 追溯链 ─────────────────────────
  source_requirements:  ["RR-01", "RR-02"]    # 原始需求（一对多：合并时多个）
  source_changes:       ["UI-03", "DA-02"]    # 原子变化点（一对多）
  source_anchors:       ["FR-01-01-001"]      # FR 或 NFR 锚点（一对多：合并时多个）
  source_impact_points: ["IP-001", "IP-005"]  # 影响点（一对多）
  
  # ─── 合并标记（仅合并 Story 需要）────
  merge_info:
    pattern:   "merged-by-same-change | merged-by-same-anchor | merged-by-shared-ac"
    rationale: "合并理由说明"
  
  # ─── Story 描述 ─────────────────────
  description:
    role:    "作为 [角色]"
    action:  "我希望 [做什么]"
    value:   "以便 [达到什么目的]"
  
  # ─── 验收标准 ───────────────────────
  acceptance_criteria:
    - id:       "AC-1"
      type:     "功能验证 | 数据验证 | 权限验证 | 集成验证 | 边界验证 | 性能验证 | 安全验证"
      given:    "前置条件"
      when:     "触发动作"
      then:     "期望结果（必须可机器断言）"
      negative: "反例（可空）"
  
  # ─── 关联设计 ───────────────────────
  related_design:
    - "PRD §3.2 PAGE-001 采购申请表单"
    - "PRD §4.3 E-001 PurchaseOrder.supplier_code"
```

**`then` 写法要求**：

✅ 正确示例：
- "页面显示字段`供应商名称`，值与实体 Supplier.name 一致"
- "返回 HTTP 200，响应体包含字段 order_no，类型为 string，不为空"
- "角色`采购员`看不到按钮`审批通过`，DOM 中不存在该元素"

❌ 错误示例：
- "界面展示正确"（无法断言）
- "系统正常运行"（无法断言）
- "用户体验良好"（主观）

---

## 第五章 原子变化点完整清单（26 个）

按 6 类业务域组织。每个变化点有唯一 ID、名称、用户视角描述、识别关键词、典型例子。

### 5.1 UI 类——用户界面变更（5 个）

| id | name | description_zh | detection_keywords | 例子 |
|---|---|---|---|---|
| UI-01 | 新增按钮/操作入口 | 在某页面增加按钮或操作入口 | 增加按钮、新增按钮、加一个按钮、添加操作、新增入口、增加导出、批量操作 | "在订单列表页加'批量导出'按钮" |
| UI-02 | 新增页面 | 增加一个全新的页面 | 新增页面、加一个页面、新页面、增加页面 | "增加订单详情页" |
| UI-03 | 页面字段增减 | 在某页面增加或减少表单字段、显示字段 | 增加字段、新增字段、去掉字段、页面增加、表单加、字段调整 | "申请表单增加'优先级'字段" |
| UI-04 | 页面布局/样式调整 | 调整布局、样式、交互方式 | 布局调整、样式调整、改版、界面优化 | "申请表单改为分步骤填写" |
| UI-05 | 页面流转关系变更 | 调整页面之间的跳转关系 | 页面跳转、流转调整、跳转关系 | "提交后直接跳详情页" |

### 5.2 DA 类——数据变更（5 个）

| id | name | description_zh | detection_keywords | 例子 |
|---|---|---|---|---|
| DA-01 | 新增实体 | 增加全新业务对象 | 新增实体、增加实体、新增对象、新业务对象 | "增加供应商实体" |
| DA-02 | 实体字段新增 | 已有实体增加属性字段 | 增加字段、新增字段、实体加、属性增加 | "订单实体增加'紧急程度'字段" |
| DA-03 | 实体字段修改/删除 | 修改字段类型、改名或删除 | 字段改、改字段类型、删除字段、去掉字段 | "把'金额'字段从 Integer 改为 Decimal" |
| DA-04 | 实体关系变更 | 调整实体间关联关系 | 关系调整、关联变化、多对多、一对多 | "用户和角色从一对多改为多对多" |
| DA-05 | 实体状态流转变更 | 调整状态机 | 状态流转、新增状态、状态变化、状态机 | "订单增加'待发货'状态" |

### 5.3 LG 类——业务逻辑变更（4 个）

| id | name | description_zh | detection_keywords | 例子 |
|---|---|---|---|---|
| LG-01 | 业务规则新增 | 增加新业务规则 | 新增规则、增加规则、新规则、规则补充 | "增加'金额>10万必须二级审批'规则" |
| LG-02 | 业务规则修改 | 调整已有规则的阈值或条件 | 改规则、规则调整、阈值调整 | "审批阈值从 5 万改为 10 万" |
| LG-03 | 计算逻辑变更 | 调整金额、数量、日期等计算公式 | 计算逻辑、公式调整、算法变化 | "折扣计算改为按累计金额阶梯" |
| LG-04 | 权限规则变更 | 调整角色对功能的访问权限 | 权限调整、角色权限、访问权限 | "增加'部门主管'查全部数据的权限" |

### 5.4 PR 类——流程变更（5 个）

| id | name | description_zh | detection_keywords | 例子 |
|---|---|---|---|---|
| PR-01 | 新增流程节点 | 业务流程中增加新环节 | 新增环节、增加节点、增加步骤、增加预审 | "审批前增加'部门预审'环节" |
| PR-02 | 流程节点删除/合并 | 去掉或合并环节 | 去掉环节、合并步骤、简化流程 | "去掉'部门预审'，直接走总监审批" |
| PR-03 | 流程顺序调整 | 调整节点执行顺序 | 顺序调整、流程顺序、节点顺序 | "把'付款'放到'发货'之前" |
| PR-04 | 异常分支新增 | 增加异常处理分支 | 异常处理、驳回、撤回、超时、退回 | "审批超时 24 小时自动转交" |
| PR-05 | 角色变更/职责调整 | 调整执行某节点的角色 | 换角色、改角色、职责调整、角色变化 | "审批人从'部门主管'改为'项目经理'" |

### 5.5 IN 类——集成变更（4 个）

| id | name | description_zh | detection_keywords | 例子 |
|---|---|---|---|---|
| IN-01 | 新增外部系统集成 | 与新外部系统对接 | 新增集成、对接、集成新系统、新增接口 | "增加与 SAP 对接，同步订单数据" |
| IN-02 | 集成方式调整 | API/MQS 等方式调整 | 集成方式、改为接口、改为消息队列、改异步 | "实时 API 调用改为 MQ 异步" |
| IN-03 | 接口字段/参数变更 | 调整外部接口的入参出参 | 接口字段、参数调整、接口字段变化 | "订单接口增加'渠道'入参" |
| IN-04 | 集成异常处理变更 | 降级、重试策略 | 重试、降级、异常处理、失败处理 | "外部失败由阻断改为降级处理" |

### 5.6 NF 类——非功能变更（3 个）

| id | name | description_zh | detection_keywords | 例子 |
|---|---|---|---|---|
| NF-01 | 性能要求变更 | 调整响应时间、并发、吞吐 | 性能、响应时间、并发、吞吐量 | "响应时间从 3 秒收紧到 1 秒" |
| NF-02 | 安全/合规要求变更 | 调整加密、脱敏、合规 | 安全、加密、脱敏、合规、GDPR | "增加 GDPR 合规要求" |
| NF-03 | 可用性/可靠性要求变更 | 调整 SLA / RTO / RPO | 可用性、SLA、RTO、RPO、宕机 | "可用性从 99.9% 提升到 99.99%" |

---

## 第六章 原子变化点 → PRD 要素影响映射

`impact_level` 三档：

- **certain**：一定影响，自动加入 effective_sequence
- **likely**：通常影响，默认加入但可由用户跳过
- **conditional**：条件影响，附 `condition`，按条件判断或交由用户决定

> 表头说明：每行展示一个变化点对各 PRD 要素的影响等级与原因。
> PRD 要素简称：APP=应用架构 / UI=界面原型 / INFO=信息架构 / FEAT=功能特性 / PERM=权限设计 / INT=集成设计 / CFG=配置设计 / SCEN=场景解决方案 / NFR=非功能需求 / PPO=定位与目标。

| 变化点 | 受影响要素与等级 / 原因 |
|---|---|
| **UI-01** 新增按钮 | UI(certain：直接影响页面规格) / FEAT(certain：新按钮对应新子特性或子特性扩展) / PERM(likely：新按钮通常需权限控制) / SCEN(likely：新按钮对应操作可能形成新场景) |
| **UI-02** 新增页面 | UI(certain：直接影响页面清单和流转图) / APP(certain：新页面对应新子特性，影响架构) / FEAT(certain：新页面需新子特性详细规格) / PERM(certain：新页面需权限分配) / SCEN(likely：新页面可能引入新场景) |
| **UI-03** 字段增减 | UI(certain：字段增减影响页面规格) / INFO(certain：字段增减通常对应实体属性增减) / FEAT(likely：可能影响子特性字段说明) |
| **UI-04** 布局调整 | UI(certain：布局直接影响页面规格) |
| **UI-05** 流转变更 | UI(certain：流转关系调整影响 Pageflow) / SCEN(likely：页面流转影响场景串联) |
| **DA-01** 新增实体 | INFO(certain：直接影响实体清单和 ER 图) / FEAT(certain：新实体需 CRUD 子特性) / SCEN(likely：新实体可能引入新场景) |
| **DA-02** 字段新增 | INFO(certain：影响实体详情表) / UI(likely：新字段通常对应页面显示) / FEAT(likely：新字段可能影响 AC) |
| **DA-03** 字段修改/删除 | INFO(certain：影响实体详情) / UI(likely) / FEAT(likely) / INT(conditional：若该字段曾用于外部接口参数映射) |
| **DA-04** 关系变更 | INFO(certain：直接影响 ER 图和外键标注) / FEAT(likely) / PERM(conditional：若涉及数据权限范围) |
| **DA-05** 状态流转变更 | INFO(certain：影响状态流转图) / FEAT(certain：新状态通常需要新功能或扩展) / UI(likely) |
| **LG-01** 规则新增 | FEAT(certain：新规则写入子特性业务规则列表) / SCEN(likely) |
| **LG-02** 规则修改 | FEAT(certain) / CFG(conditional：若调整阈值是可配置项) |
| **LG-03** 计算变更 | FEAT(certain) / INFO(conditional：若新计算逻辑需要新字段) |
| **LG-04** 权限变更 | PERM(certain) / FEAT(likely) / SCEN(likely) |
| **PR-01** 新增节点 | FEAT(certain：新节点对应新子特性) / SCEN(certain：流程节点新增直接影响场景串联) / PERM(likely) / UI(likely) |
| **PR-02** 节点删除/合并 | FEAT(certain) / SCEN(certain) / UI(likely) |
| **PR-03** 顺序调整 | SCEN(certain) / FEAT(likely) |
| **PR-04** 异常分支新增 | FEAT(certain：异常分支增加新 AC) / SCEN(certain) |
| **PR-05** 角色变更 | PERM(certain) / FEAT(likely) / SCEN(likely) |
| **IN-01** 新增集成 | INT(certain) / APP(certain：新增外部依赖影响系统边界表) / FEAT(likely：可能引入新集成型子特性) / SCEN(likely) |
| **IN-02** 集成方式调整 | INT(certain) |
| **IN-03** 接口字段变更 | INT(certain) |
| **IN-04** 异常处理变更 | INT(certain) / NFR(likely：降级策略可能涉及可用性指标) |
| **NF-01** 性能变更 | NFR(certain) / APP(conditional：若性能变化导致架构方案调整，如增加缓存层) |
| **NF-02** 安全/合规变更 | NFR(certain) / INFO(likely：脱敏规则可能新增字段标注) / PERM(likely) |
| **NF-03** 可用性变更 | NFR(certain) / INT(conditional：若 SLA 变更影响外部依赖处理策略) |

> **关于 PPO（定位与目标）**：26 个变化点中**没有任何一个**直接将其列为 affects 项。PPO 仅在用户明确说"调整产品范围/目标"时才进入 effective_sequence，这种诉求超出 26 个变化点目录，触发暂停询问后由用户主动加入。

---

## 第七章 always_affected 要素与 Story 全局收口机制

### 7.1 always_affected 设计

`element-type-registry` 中通过 `always_affected_in` 字段声明在哪些模式下永远受影响：

```yaml
- id: "story-design"
  always_affected_in: ["modify", "incremental"]
```

**只有 `story-design` 一个**。

> 与 FE 不同：FE 增量有 `original-requirement / requirement-type / glossary` 三个 always-affected，因为 FE 起始于业务侧的原始描述；PRD 起始于已成型的 FE 与基线 PRD，**不需要重新提取需求来源或重判类型**。

### 7.2 Story 不参与按章节顺序的要素循环

**这是 V3.0 的关键设计调整**：

传统做法（v2.0 草案）：把 story-design 视作普通要素，按 chapter_no=11 排在要素循环末尾。

V3.0 做法：**Story 设计单独成阶段，在所有其它要素分析完毕、所有 ImpactPoint 收齐后再启动**。理由：

1. **合并规则需要全局视野**：合并 Story 必须看到所有变化点对所有 FR/NFR 的影响才能做出"哪几个 FR 应该合并到一个 Story"的决策。要素循环中即时拆 Story 会导致后期发现可合并时返工。
2. **追溯链需要 ImpactPoint 全集**：Story 的 source_impact_points 字段可能引用跨多个要素的 IP，必须全部生成完毕。
3. **NFR Story 与 Functional Story 隔离**：两类 Story 验收方式根本不同（NFR 走专项测试），分类必须基于全局视图。

### 7.3 执行流程上的体现

要素循环阶段（Step 6）只执行**除 story-design 以外**的要素，按 chapter_no 升序逐一处理；Step 7 单独是 Story 全局收口阶段。详见第九章。

---

## 第八章 依赖图（cascade 安全网）

PRD 各要素之间存在上下游依赖关系。当某个要素发生变化时，其下游要素也可能需要同步更新。第六章的"原子变化点 → 要素影响映射"只覆盖了"变化点 → 要素"的直接关系，依赖图补充"要素 → 要素"的间接关系，作为 ChangeRouter 第 4 步的安全网。

### 8.1 完整依赖图

```yaml
impact_edges:
  - source: product-positioning
    targets:
      - { element: app-architecture,  impact_type: direct,   reason: "产品目标与范围变更直接影响模块边界和特性划分" }
      - { element: nfr,               impact_type: indirect, reason: "成功标准变化可能影响性能、可用性等目标" }

  - source: app-architecture
    targets:
      - { element: info-architecture, impact_type: direct,   reason: "特性边界变化影响实体归属" }
      - { element: feature-spec,      impact_type: direct,   reason: "子特性编号体系来源于应用架构" }
      - { element: permission-design, impact_type: direct,   reason: "权限矩阵基于子特性清单" }
      - { element: integration-design,impact_type: direct,   reason: "系统依赖表是集成设计的输入" }
      - { element: scenario-solution, impact_type: indirect, reason: "子特性分布变化影响场景串联" }

  - source: info-architecture
    targets:
      - { element: feature-spec,      impact_type: direct,   reason: "实体操作说明依赖实体定义" }
      - { element: scenario-solution, impact_type: direct,   reason: "场景表中的操作实体来自信息架构" }
      - { element: integration-design,impact_type: indirect, reason: "字段调整可能影响接口参数映射" }

  - source: feature-spec
    targets:
      - { element: permission-design, impact_type: direct,   reason: "权限矩阵需要与子特性同步" }
      - { element: scenario-solution, impact_type: direct,   reason: "场景串联依赖子特性清单" }
      - { element: story-design,      impact_type: direct,   reason: "Story 直接从功能特性与 AC 拆分" }

  - source: permission-design
    targets:
      - { element: scenario-solution, impact_type: indirect, reason: "角色权限变更影响场景中的角色边界" }

  - source: integration-design
    targets:
      - { element: scenario-solution, impact_type: indirect, reason: "外部系统调用步骤影响场景链路" }

  - source: nfr
    targets:
      - { element: story-design,      impact_type: direct,   reason: "非功能特性是 NFR Story 的直接来源" }
```

### 8.2 安全网用法

ChangeRouter 第 4 步对 effective_sequence 做闭包扩展：

1. 遍历 impact_edges
2. 对 effective_sequence 中已有的每个 element，取出该 element 作为 source 的所有 **direct** targets
3. 凡 target 不在 effective_sequence 中的，归入"安全网额外发现"清单
4. 输出警告并由用户确认是否加入

**注意**：indirect 边不强制加入，仅作提示。

---

## 第九章 七步执行流程详解

> 与 PRD 分析 v1.0 的"5 Step"语义同构、与 FE 增量方案同构，但 V3.0 把 Step 1 拆为 1.0/1.1（适配多需求场景）、Step 7 独立（Story 全局收口）。

### Step 0：环境准备与文档定位

**输入**：用户启动 Skill

**处理**：
1. 读取 `workspace/ongoing.md`，提取当前版本号、project_name、requirement_type
2. 定位**基线 PRD**：必须存在 status=completed 的历史 PRD；多个匹配时让用户选择
3. 定位**新版本 FE**（可选）：扫描当前版本目录下的 FE 文档；找到则路径写入 context；找不到则记录"FE 缺位"标志
4. 校验环境一致性：基线 PRD.requirement_type 必须与 ongoing.md 一致；存在新版 FE 时其 project_name 必须与基线 PRD 一致

**FE 缺位时的兜底声明**：

```
ℹ️ 未找到本版本对应的新版 FE 文档。
本次增量将依赖：
  - 基线 PRD（已加载）
  - 用户描述
  - 对话挖掘补齐缺失证据
所有 target_state 的证据来源会标注为 dialog（对话挖掘），
评审时请重点核查这些影响点。
```

5. 创建增量 PRD 输出文档（与基线 PRD 同目录、新日期戳），写入初始 frontmatter（含 `fe_doc_available: true|false` 字段）

---

### Step 1.0：原始需求登记（RR）

**输入**：用户对增量诉求的业务描述

**处理**：

1. 引导用户输入：
   ```
   请用一句话或一段话描述本次增量诉求（业务语言即可）。
   若有多条需求，请分条列出，AI 会逐条编号。
   例如：
     1. 在订单列表页加'批量导出'按钮
     2. 审批超时 24 小时自动转交
     3. 增加'紧急程度'字段
   ```
2. 把每条需求结构化为 RR：

   ```yaml
   - id: RR-01
     description: "原文一字不改"
     source: "对话输入"
     status: "待分析"
   ```

3. 把 RR 列表写入增量 PRD 的 frontmatter `requirement_register` 字段

**暂停触发**：
- 单条需求过于模糊（如只说"优化体验"而无具体动作描述）→ 暂停询问"该需求具体改什么"

---

### Step 1.1：原子变化点识别（ChangeRouter Step 1）

**输入**：RR 列表 + 新版 FE 文档（若存在）

**处理**：对**每条 RR** 单独识别其原子变化点：

1. **关键词初筛**：用每个变化点的 detection_keywords 在 RR.description 中做模糊匹配，得到候选集合
2. **语义匹配**：对候选集合按 description_zh + examples 做语义判断
3. **证据收集**：每个识别出的变化点必须能引用以下之一作为 evidence，并标注 evidence_source：
   - `fe_doc`：新版 FE 文档对应章节 + 引用片段
   - `baseline_prd`：基线 PRD 对应章节 + 引用片段
   - `dialog`：用户当前 RR 描述原文片段
4. **用户确认**：同一 RR 命中多个变化点时列出选项让用户选

**暂停触发**：
- 同一 RR 命中多个变化点且无法消歧
- 命中置信度为"低"
- RR 描述明显超出 26 个变化点的覆盖范围

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

### Step 3：always_affected 标注（ChangeRouter Step 3）

**处理**：扫描 element-type-registry，凡 always_affected_in 含 incremental 的 element_id，标注为 always_affected=true。

**注意**：只有 `story-design` 命中。**story-design 不加入 Step 6 的要素循环**，而是单独保留给 Step 7 处理。其余 always_affected 要素（若未来扩展）按 chapter_no 加入要素循环。

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

### Step 5：执行计划展示与确认

输出：

```
✅ 增量影响域分析完成

原始需求：
  RR-01: "..."
  RR-02: "..."

触发原子变化点：
  RR-01 → UI-03（high）, DA-02（high）
  RR-02 → PR-04（high）

受影响要素（按章节顺序）：
  1. ui-prototype - 界面原型（primary，触发 UI-03）
  2. info-architecture - 信息架构（primary，触发 DA-02）
  3. feature-spec - 功能特性（primary，触发 PR-04 + 多 cascade）
  4. permission-design - 权限设计（cascade）
  ...
不涉及要素：app-architecture, integration-design, config-design, ...

[C] 开始执行  [B] 重做 ChangeRouter  [Q] 退出
```

收到 C 后进入 Step 6。

---

### Step 6：要素循环（生成 ImpactPoint 与 DELTA）

**循环对象**：除 story-design 外的所有要素，按 chapter_no 升序。

**对每个 element**：

1. **读基线 PRD 对应章节**：定位 chapter_info.l1_no 的章节，作为 baseline_state 的引用来源
2. **筛本要素相关变化点**：从 effective_sequence 中筛出 affects 包含本 element 的变化点
3. **针对每个变化点对话挖掘细节**：
   - 引用基线 PRD 内容作为 baseline_state
   - 若新版 FE 存在，交叉验证；不存在则通过对话挖掘
   - 标注 target_state_evidence 为 fe_doc / baseline_prd / dialog 之一
4. **生成 ImpactPoint**：按第四章 4.3 的统一结构，每条 IP 同时填变更落点与可选的 boundary_constraints
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
| product-positioning | 极少触发；若触发则更新指标/范围条目 |
| app-architecture | 新增/废弃子特性节点；调整系统边界表；架构图整图重绘并 DELTA 包裹。**FR 编号必须沿用基线** |
| ui-prototype | 新增/调整页面、按钮、字段、Pageflow；DELTA 块包页面/Pageflow 子段。**PAGE 编号沿用基线** |
| info-architecture | 新增/调整实体、字段、关系、状态；**实体详情严格 9 列**；boundary_constraints 必检：主键/外键禁止修改 |
| feature-spec | 新增 FR 时完整生成 5 项必填规格；既有 FR 扩展时只追加规则/AC；boundary_constraints 必检：既有 AC 禁止删除 |
| permission-design | 矩阵 FR 行 × 角色列增减 |
| integration-design | 新增/调整 INT 条目；boundary_constraints 必检：既有接口契约禁止修改 |
| config-design | 通常 SKIP；触发时新增可配置项行 |
| scenario-solution | 新增异常场景或修改既有场景步骤 |
| nfr | 直接改对应子表条目（性能/安全/隐私/易用性/可维护性）|

---

### Step 7：Story 全局收口（**V3.0 新增独立阶段**）

**前置条件**：Step 6 全部完成，所有 ImpactPoint 已收齐。

**处理流程**：

#### 7.1 收集 Story 拆分锚点

扫描所有 ImpactPoint：

- **Functional 锚点**：从 element=feature-spec 的 IP 中提取受影响的 FR 编号
- **NFR 锚点**：从 element=nfr 的 IP 中提取 NFR 子条目（如"性能-响应时间-1秒"）

每个锚点产出 1 个 Story 是默认起点。

#### 7.2 应用合并规则

详见第十章 10.2，对所有锚点做合并判定。

#### 7.3 拆分（不主动做）

详见第十章 10.3，**默认不主动拆分**——Story 只要满足 INVEST 即可，不强求小粒度。

#### 7.4 生成 Story 完整数据结构

按第四章 4.4 填充每个 Story 的所有字段，特别是 source_requirements / source_changes / source_anchors / source_impact_points 四条追溯链。

#### 7.5 写入两个文件

- **PRD 第十一章索引**：用 DELTA 块包裹 Story 清单表格的新增行
- **独立 Story 文件**：与基线 PRD 同目录，文件名将 PRD 替换为 Story；每个 Story 一个三级标题段，包含描述、AC、关联设计

---

## 第十章 增量 Story 拆分原则

> Story 拆分是 V3.0 最具决策性的章节。本章给出**默认锚点 + 合并规则**两层判定，不做主动拆分。

### 10.1 默认锚点

**锚点定义**：每个被影响的 FR 或 NFR 子条目默认产出 1 个 Story。

| Story 类型 | 锚点来源 |
|---|---|
| Functional Story | feature-spec 影响点中受影响的 FR 编号 |
| NFR Story | nfr 影响点中受影响的 NFR 子条目 |

**为什么选 FR/NFR 作为默认锚点**：

- **FR 是 PRD 编号体系的根**，从设计、权限到 AC 都围绕 FR 展开，与 Story 的"独立可验证"语义最契合
- **NFR 验收方式独立**（性能压测、安全扫描），不能与 Functional Story 混合
- **CodingAgent 友好**：Agent 接到 Story 后可以直接定位到 PRD §5.x 看 5 项规格，不需要二次推理

### 10.2 合并规则（核心）

**合并判定**：满足**任一**条件即合并多个锚点为 1 个 Story。

| 合并模式 | 触发条件 | 例子 |
|---|---|---|
| **merged-by-same-change** | 同一原子变化点跨多 FR，且这些 FR 的修改是"对同一规则/字段的同步生效" | LG-02 把审批阈值从 5 万改成 10 万，影响"采购申请"和"出差申请"两个 FR — 1 个 Story |
| **merged-by-same-anchor** | 多个原子变化点收敛到同一 FR/NFR | UI-03 加字段 + LG-01 加规则都落到 FR-01-01-001 — 1 个 Story |
| **merged-by-shared-ac** | 一组 BDD AC 同时验证多 FR/NFR 的变更，且 AC 不可拆 | 一组 AC 同时跨"提交"和"审批"才能验证 — 1 个 Story |

**合并约束**（不可合并的情况）：

| 不可合并的情形 | 说明 |
|---|---|
| Functional Story 与 NFR Story | 验收方式根本不同，必须分开 |
| 涉及不同业务领域 | 即使共享某个变化点，跨业务域的变更应分别交付（如同一变化点同时影响"采购"和"销售"领域） |
| 跨完全不同的角色 | 同一变化对不同角色的可见性差异巨大时分别交付 |

### 10.3 拆分原则（V3.0 不做主动拆分）

> **重要决策**：CodingAgent 主导开发场景下，3 人天与 30 人天的 Story 在执行成本上没有本质差异。**强行拆分反而引入 Story 间的依赖管理负担**，违背 INVEST 中的 Independent 原则。

**因此 V3.0 不设主动拆分规则**，唯一的"拆"行为是：

- 上述合并约束（10.2 末尾）触发时被动分开
- 设计人员评审时主观判定 Story 过大、必须拆 → 此时由人工决策，方案不强制规则

终止条件：**满足 INVEST 即停**，特别是 Valuable（有用户可见价值）和 Testable（可机器断言 AC）。

### 10.4 编号规则

**Functional Story 编号**：`S-FR-{特性分类}-{特性}-{序号}-{Story序号}`

- 与 FR 编号 1:1 对应时：`S-FR-01-01-001-01`
- 合并多个 FR 时：用第一个 FR 作为编号锚点，其余在 source_anchors 字段引用

**NFR Story 编号**：`S-NFR-{NFR章节号}-{Story序号}`

- 例如：`S-NFR-10-1-01`（性能要求-第1条 Story）

**追溯字段必填**：

每个 Story 必须填齐四条追溯链：

```yaml
source_requirements:  ["RR-01", "RR-02"]    # 哪些原始需求催生本 Story
source_changes:       ["UI-03", "DA-02"]    # 哪些原子变化点
source_anchors:       ["FR-01-01-001"]      # 哪些 FR/NFR
source_impact_points: ["IP-001", "IP-005"]  # 哪些影响点
```

**合并 Story 必填 merge_info**：

```yaml
merge_info:
  pattern:   "merged-by-same-change"
  rationale: "LG-02 把审批阈值改为 10 万，FR-01-01-001 和 FR-02-01-001 都需同步生效，AC 共享，故合并"
```

非合并 Story 不需要 merge_info 字段。

### 10.5 拆分流程示意

```
所有 ImpactPoint
       ↓
┌─────────────────────────────────┐
│ 7.1 收集锚点                     │
│ Functional 锚点 = 受影响 FR 集合 │
│ NFR 锚点 = 受影响 NFR 子条目集合 │
└─────────────────────────────────┘
       ↓
默认每锚点 1 个 Story
       ↓
┌─────────────────────────────────┐
│ 7.2 应用合并规则                 │
│ - merged-by-same-change         │
│ - merged-by-same-anchor         │
│ - merged-by-shared-ac           │
└─────────────────────────────────┘
       ↓
┌─────────────────────────────────┐
│ 7.3 应用合并约束反向校验          │
│ - Functional 与 NFR 不可合       │
│ - 跨业务域不可合                 │
│ - 跨角色差异巨大不可合           │
└─────────────────────────────────┘
       ↓
最终 Story 列表
       ↓
┌─────────────────────────────────┐
│ 7.4 填充完整数据结构              │
│ 含 4 条追溯链 + merge_info       │
└─────────────────────────────────┘
       ↓
┌─────────────────────────────────┐
│ 7.5 双文件写入                   │
│ - PRD 第十一章索引               │
│ - 独立 Story 文件                │
└─────────────────────────────────┘
```

---

## 第十一章 人工确认与最终交付物

### 11.1 草案输出格式

Step 7 完成后输出草案给用户：

```
=== 增量 PRD 影响域分析草案 ===

【一、原始需求】
RR-01: 描述 / 状态：已分析
RR-02: 描述 / 状态：已分析

【二、原子变化点】
RR-01 → UI-03（high，evidence_source=fe_doc）, DA-02（high，evidence_source=dialog）
RR-02 → PR-04（high，evidence_source=fe_doc）
（注：evidence_source=dialog 的项请重点核查）

【三、受影响 PRD 要素总表】
要素 | 触发类型 | 触发变化点 | 改动摘要

【四、不涉及要素说明】
要素 | 不涉及原因 | 验证依据

【五、影响点清单】
（统一列表，含 baseline_state / target_state / target_state_evidence / in_scope / out_of_scope；
 边界约束作为 boundary_constraints 子字段嵌入相邻影响点中。
 evidence_source=dialog 的项以 ⚠️ 标记。）

IP-001 [primary, source_change=UI-03, source_requirement=RR-01]
  element: ui-prototype
  baseline_ref: PRD §3.1
  baseline_state: 操作列含查看、编辑、删除
  action: 修改
  target_state: 操作列新增"批量导出"按钮（target_state_evidence=fe_doc）
  in_scope: §3.1 操作列定义
  out_of_scope: 列表页其他列、查询条件区域
  boundary_constraints:
    - target: 列表页其他列（包括查询条件区域）
      reason: 架构边界，本次仅限操作列变化
      consequence: 修改其他列可能影响其他用户使用中的功能
      evidence: 业务需求原文仅描述"加批量导出按钮"

IP-002 [...]

【六、Story 增量】
S-FR-01-01-001-01 / Functional / 合并模式：merged-by-same-anchor
  名称：采购申请增加优先级字段
  source_requirements: [RR-01]
  source_changes: [UI-03, DA-02]
  source_anchors: [FR-01-01-001]
  source_impact_points: [IP-001, IP-005]
  描述: As 采购员 / I want 在申请表单填写优先级 / So that 审批人识别紧急程度
  AC:
    AC-1（功能验证）: Given... When... Then...
  关联设计: PRD §3.2 PAGE-001, §4.3 E-001

=== 待确认问题汇总 ===
（仍存在的低置信度判断，集中列出）

请确认：
1. 分析结论是否准确？有无遗漏或错误？
2. 影响点的边界约束是否完整？
3. Story 合并/拆分粒度是否合适？
4. evidence_source=dialog 的项是否需要再补证据？
```

### 11.2 最终落盘

用户确认后写入两个文件：

#### 增量 PRD 文档结构

```markdown
---
（frontmatter：含 base_prd / new_fe（可选）/ fe_doc_available / requirement_register / triggered_changes / impact_points / stepsCompleted ...）
---

## 0. 变更说明
- 基线 PRD：<path>
- 新版 FE：<path 或 "未提供，本次依赖对话挖掘">
- 原始需求：<RR-01, RR-02, ...>
- 触发变化点：<UI-01, DA-02, ...>
- 本次变更范围摘要：<一段话>

## 1. 原始需求清单
| RR 编号 | 描述 | 触发变化点 | 状态 |
|---|---|---|---|

## 2. 原子变化点清单
（AtomicChange 列表，含 evidence、evidence_source、confidence）

## 3. 受影响要素总表
（element 表）

## 4. 不涉及要素说明
（明确排除的要素及原因）

## 5. 各要素增量改动详情
（按 effective_sequence 顺序，每个 element 一段，含 DELTA 块）

### <章节编号> <element_name>
- 触发类型：primary / cascade
- 触发变化点：UI-01 / ...
- 基线章节：PRD §X.Y
- 影响点引用：IP-001 / IP-002

<!-- DELTA: change=UI-01, chapter=ui-prototype, op=add, level=certain -->
具体增量内容
<!-- /DELTA -->

## 6. 影响点清单
（IP-001、IP-002...统一列表，每条含 boundary_constraints 子字段）

## 7. 增量 Story 索引
| Story 编号 | Story 名称 | 类型 | 合并模式 | 来源 RR | 来源 FR/NFR | 独立文件路径 |
|---|---|---|---|---|---|---|
（说明：详细 Story 见独立 Story 文件）
```

#### 独立 Story 文件结构

```markdown
---
story_doc_id: "Story-{project_name}-{date}"
source_prd: "PRD-{project_name}-{date}.md"
total_stories: N
functional_count: N1
nfr_count: N2
---

# 增量 Story 设计 - {project_name}

## 一、Story 清单
（完整表格）

## 二、Story 概述
（按类型分组的概述表）

## 三、Story 详细规格
### S-FR-01-01-001-01 采购申请增加优先级字段
（完整数据结构 4.4 节填充）

### S-FR-01-02-002-01 ...
```

---

## 第十二章 完整执行案例

### 12.1 案例 A：FE 缺位 + 多需求 + 暂停澄清

**场景背景**：

> 用户启动 Skill，对话输入：
>
> 1. 在采购单页面增加供应商指定功能，采购员可以录入供应商相关信息并保存
> 2. 审批超时 24 小时自动转交给上级
>
> 当前版本目录下没有新版 FE 文档。

**基线 PRD 摘录**：

- §2.3 子特性清单：包含 FR-01-01-001（采购申请提交）、FR-01-02-001（采购单审批），无供应商指定相关子特性
- §3.1 采购单列表页：操作列含查看、编辑、删除
- §4.3 PurchaseOrder 实体：字段含 order_no（主键）、status、amount、creator_id、created_at
- §6.1 权限矩阵：包含采购员、采购主管、财务三个角色，权限矩阵章节标注"待完善"
- §7.1 集成：GET /api/purchase-orders 接口供外部报表系统调用
- §10.1 性能要求：审批响应 ≤3s

---

**🤖 Step 0：环境准备**

定位基线 PRD：`workspace/design/I20251201/PRD-采购系统-20251201.md` ✅
定位新版 FE：未找到 ❌

输出：

```
ℹ️ 未找到本版本对应的新版 FE 文档。
本次增量将依赖：
  - 基线 PRD（已加载）
  - 用户描述
  - 对话挖掘补齐缺失证据
所有 target_state 的证据来源会标注为 dialog（对话挖掘）。
```

创建增量 PRD：`workspace/design/I20260509/PRD-采购系统-20260509.md`，frontmatter 中 `fe_doc_available: false`。

---

**🤖 Step 1.0：原始需求登记**

```yaml
requirement_register:
  - id: RR-01
    description: "在采购单页面增加供应商指定功能，采购员可以录入供应商相关信息并保存"
    source: "对话输入"
    status: "待分析"
  - id: RR-02
    description: "审批超时 24 小时自动转交给上级"
    source: "对话输入"
    status: "待分析"
```

---

**🤖 Step 1.1：原子变化点识别**

**RR-01 分析**：

- 关键词命中：UI-01（"增加…功能"模糊匹配）、DA-02（"录入信息"暗示字段）、UI-02（"页面"出现）、UI-03（字段调整）
- 多变化点命中，置信度低，必须暂停

```
⏸ 分析暂停 — 需要澄清以下问题，才能继续 Step 1.1（RR-01 变化点识别）：

Q1：供应商指定功能的入口形式是什么？
背景：RR-01 说"增加供应商指定功能"，可以理解为：
  (a) 在采购单列表页加一个'指定供应商'按钮，点击后弹窗录入（→ UI-01 + UI-02）
  (b) 在采购单编辑页直接新增供应商相关字段（→ UI-03）
两种方式对应不同变化点，影响要素也不同，无法推断。

Q2：需要录入哪些供应商信息？请列出字段名称、类型和必填性。
背景：RR-01 提到"录入供应商相关信息"但未说明具体字段。
字段定义直接决定 info-architecture 是否需要变更，无法推断。

Q3：历史采购单如何处理？
背景：未提及历史数据处理策略。
字段新增的历史数据策略影响实体设计。

请回答以上问题后，我将继续分析。
```

**用户回答**：

> Q1：在列表页操作列加一个"指定供应商"按钮，点击后弹窗录入。
> Q2：字段是 supplier_code（varchar 50，必填）、supplier_remark（varchar 200，选填）。
> Q3：历史数据两个字段默认 null，不补填。

---

**🤖 根据用户澄清继续 Step 1.1**

```yaml
RR-01:
  - id: UI-01
    name: "新增按钮/操作入口"
    evidence: "用户澄清 Q1：列表页操作列加'指定供应商'按钮"
    evidence_source: "dialog"
    confidence: "high"
  - id: UI-02
    name: "新增页面"
    evidence: "用户澄清 Q1：点击后弹窗录入"
    evidence_source: "dialog"
    confidence: "high"
  - id: DA-02
    name: "实体字段新增"
    evidence: "用户澄清 Q2：supplier_code、supplier_remark"
    evidence_source: "dialog"
    confidence: "high"

RR-02:
  - id: PR-04
    name: "异常分支新增"
    evidence: "RR-02 原文：审批超时 24 小时自动转交"
    evidence_source: "dialog"
    confidence: "high"
```

---

**🤖 Step 2：影响汇聚**

| 变化点 | 受影响要素 |
|---|---|
| UI-01 | ui-prototype(certain) / feature-spec(certain) / permission-design(likely) / scenario-solution(likely) |
| UI-02 | ui-prototype(certain) / app-architecture(certain) / feature-spec(certain) / permission-design(certain) / scenario-solution(likely) |
| DA-02 | info-architecture(certain) / ui-prototype(likely) / feature-spec(likely) |
| PR-04 | feature-spec(certain) / scenario-solution(certain) |

候选 effective_sequence：
- ui-prototype（来自 UI-01/UI-02/DA-02）
- info-architecture（来自 DA-02）
- app-architecture（来自 UI-02）
- feature-spec（来自 UI-01/UI-02/DA-02/PR-04）
- permission-design（来自 UI-02 certain；UI-01 likely）
- scenario-solution（来自 PR-04 certain；UI-01/UI-02 likely）

---

**🤖 Step 3：always_affected 标注**

`story-design` 标注为 always_affected=true，但**不加入 Step 6 要素循环**，留给 Step 7。

---

**🤖 Step 4：依赖图安全网**

遍历 effective_sequence 中每个要素的下游：
- app-architecture → info-architecture（已在）/ feature-spec（已在）/ permission-design（已在）/ integration-design（未在）
- info-architecture → feature-spec（已在）/ scenario-solution（已在）

```
🛡️ 依赖图安全网校验
依赖图发现可能受影响但未在路由结果中的要素：
  - integration-design ← 来源 app-architecture，原因：系统依赖表是集成设计的输入

是否加入？
  [Y] 全部加入  [N] 全部忽略  [S] 选择性加入
```

**用户选择**：N（本次不涉及外部系统集成调整）。

---

**🤖 Step 5：执行计划展示**

```
✅ 增量影响域分析完成

原始需求：RR-01, RR-02
触发变化点：UI-01, UI-02, DA-02, PR-04

受影响要素（按章节顺序）：
  1. app-architecture（primary，UI-02 触发）
  2. ui-prototype（primary，UI-01/UI-02/DA-02 触发）
  3. info-architecture（primary，DA-02 触发）
  4. feature-spec（primary，UI-01/UI-02/DA-02/PR-04 触发）
  5. permission-design（primary，UI-02 触发）
  6. scenario-solution（primary，PR-04 触发）

不涉及要素：product-positioning, integration-design, config-design, nfr

[C] 开始执行
```

用户选 C。

---

**🤖 Step 6：要素循环（节选关键 IP）**

> 完整执行涉及 6 个要素，篇幅所限只摘录关键 IP。

**IP-001（primary, source_change=UI-02, source_requirement=RR-01）**

```yaml
element: app-architecture
baseline_ref: 基线 PRD §2.3 子特性清单
baseline_state: "包含'采购申请提交'、'采购单查询'、'采购单审批'，无'供应商指定'相关子特性"
action: 修改
target_state: "在'采购单管理'特性下扩展'采购单创建/编辑'子特性范围，增加'指定供应商'能力描述（操作对象仍是采购单，无独立业务边界）"
target_state_evidence: dialog
in_scope: ["§2.3 '采购单创建/编辑'子特性描述"]
out_of_scope: ["特性分类结构不变", "不新增子特性节点"]
```

**IP-002（primary, source_change=DA-02, source_requirement=RR-01）**

```yaml
element: info-architecture
baseline_ref: 基线 PRD §4.3 PurchaseOrder 实体
baseline_state: "字段：order_no（主键）、status、amount、creator_id、created_at"
action: 修改
target_state: "新增 supplier_code（varchar 50，可空，默认 null）、supplier_remark（varchar 200，可空，默认 null）；历史数据默认 null，不补填"
target_state_evidence: dialog
in_scope: ["PurchaseOrder 实体字段定义", "数据样例更新"]
out_of_scope: ["其他实体不变", "实体关系图不变"]
boundary_constraints:
  - target: "PurchaseOrder.order_no 主键"
    reason: "数据库主键，禁止修改"
    consequence: "主键变更将导致外键引用断裂，影响全系统数据完整性"
    evidence: "基线 PRD §4.3 标注 order_no 为主键"
```

**IP-003（primary, source_change=UI-01, source_requirement=RR-01）**

```yaml
element: ui-prototype
baseline_ref: 基线 PRD §3.1 采购单列表页操作列
baseline_state: "操作列含：查看、编辑、删除"
action: 修改
target_state: "操作列新增'指定供应商'按钮，位于'编辑'之后；仅采购员角色可见"
target_state_evidence: dialog
in_scope: ["§3.1 操作列定义"]
out_of_scope: ["列表页其他列", "查询条件区域"]
boundary_constraints:
  - target: "采购单列表页其他列与查询条件区域"
    reason: "架构边界，本次改动仅限操作列"
    consequence: "修改其他列可能影响其他使用中的功能"
    evidence: "用户描述仅说'增加指定供应商功能'，无其他列变更意图"
```

**IP-004（primary, source_change=UI-02, source_requirement=RR-01）**

```yaml
element: ui-prototype
baseline_ref: 基线 PRD §3.4 Pageflow
baseline_state: "无'指定供应商'弹窗节点"
action: 新增
target_state: "新增'指定供应商'弹窗：含 supplier_code（必填）、supplier_remark（选填）、确认按钮、取消按钮；Pageflow 新增：列表页按钮 → 弹窗 → 确认保存 → 刷新当前行；取消 → 关闭不保存"
target_state_evidence: dialog
in_scope: ["新弹窗页面定义", "Pageflow 新增节点"]
out_of_scope: ["列表页其他 Pageflow 路径不变"]
```

**IP-005（primary, source_change=PR-04, source_requirement=RR-02）**

```yaml
element: feature-spec
baseline_ref: 基线 PRD §5.x FR-01-02-001（采购单审批）
baseline_state: "审批流程：通过 / 驳回，无超时处理"
action: 修改
target_state: "新增异常分支：审批超时 24 小时自动转交给上级；新增 BR-审批超时-001 业务规则；新增 AC：超时未审批时记录 audit_log 并触发 transfer_to_upper"
target_state_evidence: dialog
in_scope: ["FR-01-02-001 业务规则、AC 列表"]
out_of_scope: ["FR-01-02-001 的 UIUX 操作步骤不变"]
boundary_constraints:
  - target: "既有 AC（FR-01-02-001 的 Happy Path AC）"
    reason: "既有测试用例可能依赖"
    consequence: "删除既有 AC 会导致回归测试失败"
    evidence: "基线 PRD §5.x FR-01-02-001 的 AC 已被实现"
```

**IP-006（primary, source_change=UI-02, source_requirement=RR-01）**

```yaml
element: permission-design
baseline_ref: 基线 PRD §6.1 权限矩阵（待完善）
baseline_state: "权限矩阵章节标注'待完善'"
action: 修改
target_state: "为'指定供应商'按钮和弹窗分配权限：采购员=可见可操作；采购主管=不可见；财务=不可见"
target_state_evidence: dialog
in_scope: ["'指定供应商'按钮的角色权限行"]
out_of_scope: ["其他按钮的权限不变"]
```

> 注：本案例还会产生 IP-007（feature-spec / FR-01-01-002 新增"指定供应商"子特性）等，篇幅所限省略。

---

**🤖 Step 7：Story 全局收口**

#### 7.1 收集锚点

- Functional 锚点：FR-01-01-002（新增"指定供应商"子特性）/ FR-01-02-001（审批+超时分支）
- NFR 锚点：无

#### 7.2 应用合并规则

判定：

- FR-01-01-002 涉及 IP-001/IP-003/IP-004/IP-006，全部由 RR-01 + UI-01/UI-02/DA-02 触发，落到同一 FR → **merged-by-same-anchor**
- FR-01-02-001 涉及 IP-005，由 RR-02 + PR-04 触发 → 独立 Story
- 是否跨 FR 合并？FR-01-01-002 与 FR-01-02-001 来自不同 RR、不同变化点、共享无 AC → **不合并**

#### 7.3 拆分

不主动拆分。

#### 7.4 最终 Story 列表

**S-FR-01-01-002-01：采购单指定供应商功能**

```yaml
story_type: Functional
source_requirements: [RR-01]
source_changes: [UI-01, UI-02, DA-02]
source_anchors: [FR-01-01-002]
source_impact_points: [IP-001, IP-002, IP-003, IP-004, IP-006]
merge_info:
  pattern: merged-by-same-anchor
  rationale: "UI-01/UI-02/DA-02 三个变化点都指向 FR-01-01-002 子特性，且 AC 共享（按钮入口 → 弹窗录入 → 字段持久化是同一业务流），故合并"
description:
  role: 采购员
  action: 在采购单列表页通过'指定供应商'按钮录入供应商信息
  value: 完成采购单的供应商绑定
acceptance_criteria:
  - id: AC-1
    type: 功能验证
    given: 采购单列表已加载，登录用户为采购员
    when: 点击某行'指定供应商'按钮，弹窗中输入 supplier_code='SUP-001'，点击确认
    then: 弹窗关闭；该行 supplier_code 持久化为 'SUP-001'；列表当前行刷新；DOM 中该行 supplier_code 列显示 'SUP-001'
  - id: AC-2
    type: 边界验证
    given: 弹窗已打开，supplier_code 输入框为空
    when: 点击确认
    then: 表单不提交；supplier_code 输入框下方显示必填提示；弹窗保持打开
  - id: AC-3
    type: 权限验证
    given: 登录用户为采购主管
    when: 查看采购单列表
    then: 操作列不显示'指定供应商'按钮；DOM 不存在该按钮元素
  - id: AC-4
    type: 数据验证（历史数据）
    given: 系统升级后查询升级前已存在的任意采购单
    when: 查询 purchase_order 表
    then: supplier_code = null；supplier_remark = null；其余字段值与升级前一致
related_design:
  - PRD §3.1 列表页操作列
  - PRD §3.4 Pageflow（'指定供应商'弹窗节点）
  - PRD §4.3 PurchaseOrder 实体（supplier_code、supplier_remark）
  - PRD §6.1 权限矩阵（'指定供应商'按钮行）
```

**S-FR-01-02-001-02：审批超时自动转交**

```yaml
story_type: Functional
source_requirements: [RR-02]
source_changes: [PR-04]
source_anchors: [FR-01-02-001]
source_impact_points: [IP-005]
# 非合并 Story，无 merge_info
description:
  role: 系统
  action: 检测到审批超时 24 小时未处理时自动将审批任务转交给上级
  value: 避免审批阻塞业务，提升处理效率
acceptance_criteria:
  - id: AC-1
    type: 功能验证
    given: 审批人 A 收到审批任务 PA-001，超过 24 小时未处理
    when: 系统定时任务运行
    then: 审批任务 PA-001 的 assignee 字段更新为 A 的上级 B；audit_log 增加一条 transfer_to_upper 记录
  - id: AC-2
    type: 边界验证
    given: 审批人 A 收到审批任务 PA-002，第 23 小时进行处理
    when: 提交审批意见
    then: 审批任务 PA-002 状态正常更新为 approved 或 rejected；不触发自动转交
related_design:
  - PRD §5.x FR-01-02-001 业务规则（BR-审批超时-001）
  - PRD §5.x FR-01-02-001 AC 列表
```

#### 7.5 双文件写入

略，按规范写入 PRD 第十一章索引和独立 Story 文件。

---

**🤖 输出草案**

（按第十一章 11.1 格式输出，等用户确认。）

---

### 12.2 案例 B：FE 完整 + 单需求 + 无暂停

**场景背景**：

> 用户：基于本月新版 FE，做增量 PRD。
> 新版 FE 比基线版本多了"采购单列表展示创建人部门"。

**新版 FE 摘录**：

- §3 用户交互 → 页面清单：采购单列表新增"创建人部门"列

**基线 PRD 摘录**：

- §3.1 采购单列表：列含 order_no、status、amount、creator_id、created_at
- §4.3 PurchaseOrder 实体：字段含 creator_dept（varchar 100，可空）— **已存在**
- §5.x 功能特性 FR-01-03-001（采购单查询）：列表展示字段清单未包含 creator_dept

---

**🤖 全程无暂停**

Step 0：基线 PRD ✅，新版 FE ✅，fe_doc_available=true。

Step 1.0：

```yaml
requirement_register:
  - id: RR-01
    description: "采购单列表展示创建人部门"
    source: "新版 FE §3"
    status: "待分析"
```

Step 1.1：

```yaml
RR-01:
  - id: UI-03
    name: "页面字段增减"
    evidence: "新版 FE §3 用户交互-页面清单：采购单列表新增'创建人部门'列"
    evidence_source: fe_doc
    confidence: high
```

Step 2：UI-03 → ui-prototype(certain) / info-architecture(certain) / feature-spec(likely)

Step 3：always_affected = story-design（留给 Step 7）

Step 4：依赖图扩展无新增。

Step 5：执行计划

```
受影响要素：
  1. ui-prototype（primary，UI-03）
  2. info-architecture（primary，UI-03）
  3. feature-spec（primary likely，UI-03）
不涉及：app-architecture, permission-design, scenario-solution, integration-design, ...
```

Step 6：

**IP-001（ui-prototype）**

```yaml
baseline_ref: PRD §3.1 列表列定义
baseline_state: "列含 order_no、status、amount、creator_id、created_at（5 列）"
action: 修改
target_state: "在 created_at 之后新增'创建人部门'列，展示 PurchaseOrder.creator_dept；为空时显示'—'"
target_state_evidence: fe_doc
in_scope: ["§3.1 列定义"]
out_of_scope: ["其他列顺序与定义不变", "查询条件不变"]
boundary_constraints:
  - target: "采购单列表其他列定义"
    reason: "本次仅说'加一列'，其他列无变更意图"
    consequence: "改动其他列可能影响使用中的功能"
    evidence: "新版 FE §3 仅描述新增'创建人部门'列"
```

**IP-002（info-architecture）**

```yaml
baseline_ref: PRD §4.3 PurchaseOrder 实体
baseline_state: "creator_dept 字段已存在（varchar 100，可空）"
action: 不涉及
target_state: "实体不变，仅在功能特性的列表展示字段清单中引用 creator_dept"
target_state_evidence: baseline_prd
in_scope: []
out_of_scope: ["实体定义不变"]
```

**IP-003（feature-spec）**

```yaml
baseline_ref: PRD §5.x FR-01-03-001 列表展示字段
baseline_state: "未包含 creator_dept"
action: 修改
target_state: "列表展示字段清单新增 creator_dept；列表查询接口出参需包含 creator_dept"
target_state_evidence: fe_doc
in_scope: ["FR-01-03-001 列表展示字段说明"]
out_of_scope: ["其他功能特性不变"]
```

Step 7：Story 全局收口

- Functional 锚点：FR-01-03-001
- 应用合并规则：单 FR 单变化点 → 默认 1 个 Story
- 输出：

```yaml
S-FR-01-03-001-01:
  story_type: Functional
  source_requirements: [RR-01]
  source_changes: [UI-03]
  source_anchors: [FR-01-03-001]
  source_impact_points: [IP-001, IP-002, IP-003]
  description:
    role: 采购员
    action: 在采购单列表中查看每条记录的创建人部门
    value: 了解采购单来源部门，辅助审批和统计
  acceptance_criteria:
    - id: AC-1
      type: 功能验证
      given: 采购单列表已加载
      when: 查看列表表头
      then: 表头最后一列为'创建人部门'，位于'创建时间'之后
    - id: AC-2
      type: 数据验证
      given: 数据库中某采购单 creator_dept = '采购一部'
      when: 该记录在列表中展示
      then: 该行'创建人部门'列显示'采购一部'
    - id: AC-3
      type: 边界验证
      given: 数据库中某采购单 creator_dept = null
      when: 该记录在列表中展示
      then: 该行'创建人部门'列显示'—'
  related_design:
    - PRD §3.1 列表列定义
    - PRD §5.x FR-01-03-001 列表展示字段
```

输出草案。

---

*案例 B 说明了"有理有据"在无歧义、有 FE 文档场景下的表现：每条结论都引用 FE 或基线 PRD 章节，info-architecture 的"不涉及"结论有明确依据（字段已存在），不需要猜测或澄清。*

---

## 附录 A：编号体系总览

| 编号 | 含义 | 来源 | 例 |
|---|---|---|---|
| RR-NN | 原始需求 | Step 1.0 用户输入 | RR-01 |
| {CATEGORY-NN} | 原子变化点 | 第五章固定目录 | UI-01 |
| FR-{特性分类}-{特性}-{序号} | 子特性编号（功能特性锚点） | 基线 PRD app-architecture | FR-01-01-001 |
| E-NNN | 实体编号 | 基线 PRD info-architecture | E-001 |
| PAGE-NNN | 页面编号 | 基线 PRD ui-prototype | PAGE-001 |
| INT-NNN | 集成点编号 | 基线 PRD integration-design | INT-001 |
| ROLE-NNN | 角色编号 | 基线 PRD permission-design | ROLE-001 |
| SCEN-NNN | 场景编号 | 基线 PRD scenario-solution | SCEN-001 |
| IP-NNN | 影响点编号 | Step 6 全局递增 | IP-001 |
| S-FR-...-{Story序号} | Functional Story | Step 7 | S-FR-01-01-001-01 |
| S-NFR-{NFR章节号}-{Story序号} | NFR Story | Step 7 | S-NFR-10-1-01 |
| AC-N | 验收标准 | Story 内部递增 | AC-1 |

---

## 附录 B：与 v1.2.0 规范的同步事项

V3.0 在数据结构上**简化**了 v1.2.0 规范第 3.7.3 节中的 ImpactPoint 定义。具体差异：

| 维度 | v1.2.0 规范 | V3.0 方案 |
|---|---|---|
| 顶层字段 | `kind: modify \| forbid` | 取消 kind |
| 表达方式 | 两类 ImpactPoint 并列存在 | 一个 ImpactPoint 同时承载修改与边界约束 |
| forbid 表达 | `IP-xxx-forbid` 独立条目，含 reason/consequence/adjacent_to | `boundary_constraints` 子字段嵌入相邻 ImpactPoint |
| 草案展示 | 按 kind 分组（modify 类、forbid 类）| 统一影响点清单 |

**简化理由**：

1. **认知一致性**：边界约束与修改本就是同一上下文中的两面（"加字段时主键不能动"），分两个清单破坏认知模型。
2. **设计人员友好**：1→N 阶段的实际评审中，设计人员的注意力在某个章节时，希望看到"这个章节改什么、连带不能改什么"在同一个位置。
3. **新增字段统一性**：V3.0 还为 ImpactPoint 增加了 `target_state_evidence` 字段（FE 缺位场景下区分证据来源），都收敛到一个数据结构里更稳。

**对规范的回写建议**：

V3.0 落地后，设计文档 Skill 构建规范应同步更新第 3.7.3 节的 ImpactPoint 数据结构定义。这次修订**只影响尚未实现的增量场景**（PRD 增量在 V3.0 后才进入实现），不破坏已实现的 FE 增量（FE 增量方案中的 ImpactPoint 用法可同步迁移到新结构，本就同构）。

---

*本方案 v3.0，遵循设计文档 Skill 构建规范 v1.2.0，与 FE 增量方案保持术语对齐与同构。任何规则调整都应同时检查"是否需要 FE 增量方案同步"。*
