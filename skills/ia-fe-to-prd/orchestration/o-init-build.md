# o-init-build

## 职责

负责 0→1 PRD 新建与 `prd-resume` 续接的完整编排，覆盖初始化、要素执行循环、完成阶段三部分。

---

## ⚠️ 单一文档强制约束

> **本编排文件执行期间，有且只有一个 PRD 文档存在。**  
> - 在 Action 0 Step 7 创建唯一 PRD 文档，路径写入 `context.prd_doc_path`。  
> - 所有要素的执行结果由 element-runner Phase 6 追加写入该文档，**不得另建任何初始稿、过程稿、阶段稿、临时稿**。  
> - 要素执行循环是一个连续过程，**禁止宣告"第X阶段完成，进入第X+1阶段"或中途另起新文档**。  
> - 违反以上原则，立即停止当前操作，删除多余文档，重新走 Action 0。

---

## 初始化阶段

### Action 0：验证环境与准备上下文

1. 读取 `workspace/ongoing.md`，提取：
   - `current_version`
   - `project_name`
   - `requirement_nature`
   - `requirement_type`
   - `current_prd_path`（可选，若存在则表示已有确定的工作 PRD 路径）
   - `workflow_hint`（可选，用户预设的 workflow_id）

   > **SceneRouter 消歧后同步**：若用户在路由层已明确选择 workflow_id，则 `workflow_hint` 已记录该选择：
   > - 用户选择"新建"：清除 `current_prd_path`
   > - 用户选择"增量"或"继续"：保留 `current_prd_path`

2. 查找 `workspace/requirements/{current_version}/FE-*.md`。
3. 验证 FE frontmatter：
   - `status == completed`
   - `requirement_type` 与 workflow 匹配
4. **检查 design 目录并交互确认（多项目共存安全检查）**：
   - 定位 `workspace/design/{current_version}/` 目录
   - 遍历所有 `PRD-*.md` 文件，提取 frontmatter 的 `project_name` 和 `status`
   - 若找到与当前 `ongoing.md` 的 `project_name` 匹配的 PRD 文件（一个或多个）：
     - 列出所有匹配的 PRD 文件及其状态
     - **交互提示**：
       ```
       检测到以下与当前项目「{project_name}」匹配的 PRD 文件：
         1. {PRD文件名}（状态: {status}，最后更新: {last_updated}）
         2. ...
       请选择：
         [1/2/...] 续接已有 PRD（输入序号）
         [N] 新建 PRD
         [Q] 退出
       ```
     - 根据用户选择：
       - 选"续接"：加载对应 PRD，更新 `ongoing.md.current_prd_path`，进入续接恢复流程（跳转到本文件末尾"续接恢复"章节）
       - 选"新建"：继续执行后续步骤
   - 若无匹配 PRD：直接创建新 PRD
5. 若 `docs/biz_kl/` 存在，作为可选知识库上下文加载。
6. 自动推导项目属性，至少展示：
   - 用户规模
   - 流程复杂度
   - 集成复杂度
   - 安全级别
7. 生成建议文件名（格式：`PRD-{project_name}-{date}.md`），**在此创建唯一 PRD 文档**，写入初始 frontmatter，将文档路径存入 `context.output_doc_path` 并同步更新 `ongoing.md.current_prd_path`。

> **路径动态生成规则**:  
> - 完整路径:`{output_folder_base}/{current_version}/{default_filename}`  
> - 示例:`workspace/design/I20260419/PRD-项目名称-20260419.md`
> - 创建完成后,将此路径赋值给 `context.output_doc_path`,用于后续 element-runner 写入

### Action 1：确认执行序列

1. 读取 `registry/element-type-registry.yaml`。
2. 过滤 `belongs_to` 包含当前 workflow 的 `requirement_type` 的要素。
3. 按 `chapter_no` 排序，生成要素执行序列。
4. 标注 `optional: true` 的要素（`integration-design`、`config-design`）。
5. 向用户展示执行序列并询问确认：
   ```
   📋 执行序列已生成（基于 requirement_type={requirement_type}）
   ────────────────────────────────────
   必选要素：
     1. {element_id} - {element_name}
     ...
   可选要素（可跳过）：
     {element_id} - {element_name} [可选]
     ...
   ────────────────────────────────────
   是否跳过可选要素？
     [Y] 执行全部要素（包含可选）
     [S] 选择性跳过可选要素
     [N] 跳过所有可选要素
   ```
6. 根据用户选择更新实际序列，写入 PRD frontmatter 的 `effective_sequence`。
7. 确认后向用户输出执行计划摘要：
   ```
   ✅ 执行计划已确认
   目标文档：{context.prd_doc_path}
   将执行要素（{总数}个）：{要素列表}
   输入文档：{fe_doc_path}
   ────────────────────────────────────
   [C] 开始执行
   [Q] 退出
   ```

---

## 要素执行循环

> **循环执行约束**：
> - 本循环是一个连续的单一过程，所有要素的执行结果均写入 `context.prd_doc_path` 指向的唯一文档。
> - 禁止在循环中途宣告"阶段完成"或创建任何中间文档。
> - 循环未完成时，只有 Q（保存并退出）和 B（重跑当前要素）可中断，C 继续执行下一要素。

```text
FOR each element_id IN effective_sequence:
  IF element_id == "integration-design":
    检查 FE 是否存在外部依赖；若无，则自动 SKIP，记录跳过日志，继续下一要素
  调用 element-runner(element_id, mode="build", {
    workflow_id       : context.workflow.id,
    requirement_type  : context.requirement_type,
    input_doc_path    : context.paths.input_doc,    # FE 文档路径
    output_doc_path   : context.output_doc_path,    # orchestration 创建的 PRD 路径
    base_doc_path     : context.paths.base_doc,     # 增量基线文档路径
    modify_focus      : [],
    impact_analysis   : {},
    change_type       : "",
    chapter_info      : {
                         l1_no: element对应的chapter_no（从element-type-registry读取）,
                         element_name: element的name字段,
                         output_doc_path: context.output_doc_path
                       }
  })
  
  **强制等待 element-runner 返回控制信号**：
  - element-runner 必须在 Phase 6 步骤3 使用 AskUserQuestion 工具等待用户选择
  - orchestration 挂起，等待用户选择并返回控制信号
  - 未收到控制信号前，禁止执行下一个要素
  
  处理 element-runner Phase 6 返回的控制信号：
    C    → 继续循环，执行下一个要素
    B    → 重跑当前要素（重新调用 element-runner）
    Q    → 保存 PRD 文档当前状态，更新 frontmatter.status="in_progress"，退出循环
    SKIP → 记录跳过日志，继续下一个要素
END FOR

---

## 续接恢复

> 由 `prd-resume` workflow 触发，或在 Action 0 Step 4 用户选择"续接已有PRD"时进入。

1. 读取目标 PRD 的 frontmatter：
   - `effective_sequence`（原始执行序列）
   - `stepsCompleted`（已完成要素列表）
2. 计算未完成要素列表：`effective_sequence` - `stepsCompleted`。
3. 向用户展示续接信息：
   ```
   🔄 续接恢复模式
   目标文档：{prd_doc_path}
   已完成：{stepsCompleted}
   待执行：{remaining_elements}
   ────────────────────────────────────
   [C] 从「{first_remaining_element}」继续
   [Q] 退出
   ```
4. 用户确认后，以 `remaining_elements` 作为 `effective_sequence`，进入要素执行循环（参见上方）。

---

## 完成阶段

> 要素执行循环全部 `C` 走完后，进入完成阶段。

### Action A：独立审查

从以下维度进行独立审查：

- 逻辑自洽性
- 章节完整性
- 可实施性
- 跨章节一致性
- Mermaid 语法正确性
- 缺失点与风险

输出：

- 评分 `A/B/C/D`
- 问题列表：`致命 / 重要 / 次要`
- 修正建议

### Action B：质量检查

至少检查：

- FE→PRD 追溯完整性
- 功能点与实体一致性
- 功能点与场景一致性
- Mermaid 语法
- BDD AC 格式

### Action C：知识沉淀建议

如果出现以下内容，建议沉淀到知识库或扩展规则：

- 可复用业务术语
- 可复用实体模型
- 跨项目通用规则
- 经用户确认的格式偏好

### Action D：最终状态更新

由 element-runner Phase 6 完成最终状态字段更新（status → completed、completed_at、quality_score、last_updated）。

清理 `ongoing.md`（可选）：删除 `current_prd_path` 字段，保留元信息用于历史追溯。

输出完成提示，并建议进入 `ia-prd-to-design`：

```text
✅ ia-fe-to-prd 已完成

输出文件：{context.prd_doc_path}
当前模式：{workflow_id} / build
质量评分：{quality_score}

建议下一步：
  ia-prd-to-design {current_version}
  