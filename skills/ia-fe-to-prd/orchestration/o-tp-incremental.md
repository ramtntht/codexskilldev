# TP 1-n 增量PRD创建 编排文件
# workflow_id: tp-incremental
# 对应 workflow-registry 中 id: tp-incremental

## 前置说明

本编排文件由 workflow-engine 在命中 `tp-incremental` 后调用。
核心差异：在要素循环前执行**影响域分析（Impact Analysis）**，仅对受影响要素调用 element-runner（incremental 模式），未受影响要素直接从基线复制章节内容。
实际要素内容生成完全交由 element-runner 执行，本文件只控制宏观流程。

---

## ⚠️ 单一文档强制约束

> **本编排文件执行期间，有且只有一个新版PRD文档存在。**
> - 在 Action 0 Step 8 创建唯一新版PRD文档，路径写入 `context.output_doc_path`。
> - 基线PRD（`context.base_doc_path`）只读，禁止对其执行任何写操作。
> - 所有要素的执行结果由 element-runner Phase 6 追加写入新版PRD，**不得另建任何过程稿、临时稿**。
> - 违反以上原则，立即停止当前操作，删除多余文档，重新走 Action 0。

---

## 初始化阶段

### Action 0：验证环境与准备上下文

**Step 1**：读取 `workspace/ongoing.md`，提取：
- `current_version`
- `project_name`
- `requirement_nature`
- `requirement_type`
- `workflow_hint`

**Step 2**：定位新版FE文档：
- 查找 `workspace/requirements/{current_version}/FE-*.md`
- 验证 frontmatter：`status == completed`，`requirement_type == TP`
- 将路径记入 `context.input_doc_path`

**Step 3**：定位基线PRD文档：
- 扫描 `workspace/design/` 下所有版本目录
- 收集 `status == completed` 的 PRD 文档
- 过滤出 `project_name` 与 `ongoing.md` 匹配的文件
- 若找到多个，列出供用户选择：

```
检测到以下已完成的基线PRD文档：
  1. {PRD文件名}（版本: {version}，完成时间: {completed_at}）
  2. ...
请选择作为基线的PRD（输入序号）：
  [Q] 退出
```

- 若只有一个，自动选定并提示用户确认
- 将基线PRD路径记入 `context.base_doc_path`

**Step 4**：提取基线PRD元数据：
- 读取基线PRD frontmatter，提取：
  - `base_version`（原版本号）
  - `stepsCompleted`（已有章节列表）
  - `effective_sequence`（原执行序列）
  - `requirement_type`（验证与当前一致）

**Step 5**：若 `docs/biz_kl/` 存在，作为可选知识库上下文加载。

**Step 6**：推导新版本号：
- 读取新FE frontmatter 中的版本信息
- 生成新版PRD的版本标识（如原为 `I20260101`，新为 `I20260420`）

**Step 7**：生成新版PRD文件名，格式：`PRD-{project_name}-{date}.md`（新日期）

**Step 8**：创建新版PRD文档，写入初始 frontmatter，路径存入 `context.output_doc_path` 并同步更新 `ongoing.md.current_prd_path`：

```yaml
---
doc_type: "PRD"
project_name: "{project_name}"
requirement_type: "TP"
workflow_id: "tp-incremental"
execution_mode: "incremental"
base_prd_path: "{context.base_doc_path}"
base_version: "{base_version}"
current_version: "{current_version}"
status: "in_progress"
created_at: "{today}"
last_updated: "{today}"
effective_sequence: []        # Action 1 填入
stepsCompleted: []
affected_elements: []         # Impact Analysis 填入
unaffected_elements: []       # Impact Analysis 填入
last_element: ""
quality_score: ""
---
```

---

### Action 1：影响域分析（Impact Analysis）

> 本 Action 是 1-n 场景的核心差异环节，在要素循环前执行。引擎不执行任何要素，只做变更识别。

**Step 1**：读取基线PRD各章节标题与摘要（提取结构骨架即可，不需要全文）

**Step 2**：读取新FE文档，与基线PRD对比，从以下维度识别变更信号：

| 变更信号 | 受影响要素候选 |
|----------|---------------|
| 新增/变更业务目标或项目范围描述 | product-positioning |
| 新增/变更/删除用户角色或权限描述 | permission-design |
| 新增/变更/删除业务流程或流程步骤 | scenario-solution, feature-spec |
| 新增/变更/删除功能点或用户故事 | feature-spec |
| 新增/变更实体、属性或关系 | info-architecture |
| 新增/变更/删除子特性模块或服务依赖 | app-architecture |
| 新增/变更界面或交互流程 | ui-prototype |
| 新增/变更/删除外部系统集成 | integration-design |
| 新增/变更配置项或参数 | config-design |
| 变更性能、安全、可用性等要求 | nfr |

**Step 3**：生成影响域分析报告，展示给用户确认：

```
📊 影响域分析报告
─────────────────────────────────────────────────
基线PRD：{base_prd_path}（{base_version}）
新FE文档：{input_doc_path}（{current_version}）
─────────────────────────────────────────────────
🟡 受影响要素（将进入增量执行）：
  - {要素名称}：{变更原因摘要，1-2句话}
  - {要素名称}：{变更原因摘要}
  ...

⬜ 未受影响要素（将从基线直接复制）：
  - {要素名称}
  - {要素名称}
  ...
─────────────────────────────────────────────────
[C] 认以上分析，开始执行
[E] 手动调整（输入要素编号切换状态）
[Q] 退出
```

**Step 4**：用户确认后，将分析结果写入新版PRD frontmatter：
- `affected_elements`: 受影响要素列表（进入要素循环）
- `unaffected_elements`: 未受影响要素列表（直接复制基线内容）
- `effective_sequence`: 完整执行序列（affected + unaffected，按章节顺序排列）

---

### Action 2：确认执行计划

读取 `affected_elements` 和 `unaffected_elements`，向用户输出执行计划摘要：

```
✅ 执行计划已确认
目标文档（新版）：{context.output_doc_path}
基线文档：{context.base_doc_path}

🟡 增量执行要素（{n}个）：{受影响要素列表}
⬜ 直接复制要素（{m}个）：{未受影响要素列表}

输入文档：{input_doc_path}
───────────────────────────────────
[C] 开始执行
[Q] 退出
```

将 `effective_sequence` 写入PRD frontmatter。

---

## 要素执行循环

> **循环执行约束**：
> - 本循环面向 `effective_sequence` 中的所有要素，**顺序执行**（保持章节顺序）。
> - 受影响要素：调用 element-runner（incremental 模式）
> - 未受影响要素：直接从基线PRD复制对应章节内容到新PRD，记录到 stepsCompleted，不调用 element-runner
> - 禁止在循环中途创建任何中间文档。

```text
FOR each element_id IN effective_sequence:

  IF element_id IN unaffected_elements:
    # 直接复制基线章节，不调用 element-runner
    从 context.base_doc_path 提取 element_id 对应章节的完整内容
    追加写入 context.output_doc_path 对应章节位置
    在 PRD frontmatter.stepsCompleted 中追加 element_id
    输出提示：⬜ [{要素名称}] 已从基线复制，无需修改
    continue

  IF element_id == "integration-design":
    # 检查新FE是否存在外部依赖变更；若新FE中无外部依赖且基线无集成设计，则跳过
    IF 新FE无外部依赖 AND 基线PRD无integration-design章节:
      记录跳过日志
      continue

  # 受影响要素：调用 element-runner（incremental 模式）
  调用 element-runner(element_id, mode="incremental", {
    workflow_id       : "tp-incremental",
    requirement_type  : "TP",
    input_doc_path    : context.input_doc_path,    # 新FE文档路径
    output_doc_path   : context.output_doc_path,   # 新版PRD路径
    base_doc_path     : context.base_doc_path,     # 基线PRD路径（只读）
    modify_focus      : [],
    impact_analysis   : context.impact_analysis[element_id],  # 该要素的变更摘要
    change_type       : "incremental",
    chapter_info      : {
                         l1_no: element对应的chapter_no（从element-type-registry读取）,
                         element_name: element的name字段,
                         output_doc_path: context.output_doc_path
                       }
  })

  处理 element-runner Phase 6 返回的控制信号：
    C    → 继续循环，执行下一个要素
    B    → 重跑当前要素（重新调用 element-runner）
    Q    → 保存新版PRD文档当前状态，更新 frontmatter.status="in_progress"，退出循环
    SKIP → 记录跳过日志（将基线内容复制到新PRD），继续下一个要素

END FOR
```

---

## 完成阶段

> 要素执行循环全部走完后，进入完成阶段。

### Action A：增量一致性审查

针对增量场景特有的风险点进行独立审查：

- **变更一致性**：受影响要素之间的变更是否互相一致（如 feature-spec 新增功能点，info-architecture 是否同步新增了对应实体）
- **基线复制完整性**：未受影响要素的章节是否完整复制，无内容缺失
- **章节顺序正确性**：新版PRD章节顺序与基线一致
- **跨章节引用正确性**：新增/变更的内容在其他章节中的引用是否已更新
- **变更标注规范性**：所有变更处是否正确使用了变更标注格式
- **Mermaid 语法正确性**：修改后的流程图/ER图语法是否有效

输出：
- 评分 `A/B/C/D`
- 问题列表：`致命 / 重要 / 次要`
- 修正建议

### Action B：变更完整性检查

至少检查：
- 新FE中每个变更需求点，在新版PRD中是否有对应体现
- 受影响要素的变更明细表是否完整
- 变更前后对比是否清晰（每个修改项有"原始内容→更新内容"的说明）
- 未受影响要素的章节内容是否与基线一致（无意外修改）

### Action C：知识沉淀建议

如果出现以下内容，建议沉淀到知识库或扩展规则：
- 本次变更中识别出的可复用业务模式
- 用户确认的变更判断逻辑（哪类FE变更会影响哪类PRD章节）
- 经用户确认的格式偏好或规范调整

### Action D：最终状态更新

更新新版PRD frontmatter（通过 element-runner Phase 6 机制完成，此处为说明）：

```yaml
status: "completed"
completed_at: "{today}"
quality_score: "A|B|C|D"
last_updated: "{today}"
```

同步更新 `ongoing.md`：
- `current_prd_path` 更新为新版PRD路径
- 保留 `base_version` 字段用于历史追溯

输出完成提示：

```
✅ ia-fe-to-prd 增量执行已完成

基线文档：{context.base_doc_path}
新版文档：{context.output_doc_path}
当前模式：tp-incremental / incremental
质量评分：{quality_score}

变更摘要：
  🟡 增量执行要素：{n}个
  ⬜ 基线复制要素：{m}个
  📝 主要变更：{top 3 变更摘要}

建议下一步：
  ia-prd-to-design {current_version}
```