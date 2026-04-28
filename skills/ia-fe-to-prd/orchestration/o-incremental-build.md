# o-incremental-build

## 职责

负责在历史 PRD 基线之上，根据新的 FE 文档执行增量设计、影响分析与 DELTA 合并。

## Action 1：加载历史 PRD 基线

1. 定位 `workspace/design/{prev_version}/` 目录下的所有 `PRD-*.md` 文件。
2. **多项目共存场景处理**：
   - 读取每个 PRD 文件的 frontmatter，提取 `project_name` 和 `status`
   - 匹配当前 `ongoing.md` 的 `project_name`，找到所有匹配的目标 PRD 文件
   - 如果找到多个匹配的PRD（同项目不同版本）：
     - 列出所有版本及其状态、最后更新时间
     - **交互提示**："检测到项目{project_name}存在多个历史PRD版本：
       1. {PRD文件名} (状态: {status}, 最后更新: {last_updated})
       2. {PRD文件名} (状态: {status}, 最后更新: {last_updated})...
       请选择作为基线的版本（输入序号1/2/...）或 [Q] 退出"
     - 根据用户选择加载对应PRD作为基线
     - 同步更新 `ongoing.md` 的 `current_prd_path` 字段指向该基线PRD路径
   - 如果目录下存在多个不同项目的 PRD 文件（多项目并存）：
     - 仅列出 `project_name` 匹配的PRD版本供用户选择
     - 不显示其他项目的PRD（避免混淆）
   - 如果不存在匹配项目的历史 PRD，则提示切换到 `o-init-build`
3. 读取基线 PRD 正文与 frontmatter，尤其是：
   - `fe_reference_snapshot`
   - `incremental_history`
   - `project_name`（验证匹配）

## Action 2：识别改动类型 A-F

1. 读取新的完成态 FE 文档。
2. 与 `base_prd.frontmatter.fe_reference_snapshot` 指向的旧 FE 做差异对比。
3. 按 `registry/dependency-graph.yaml.incremental_impact_rules.change_types` 分类：
   - A 新增实体
   - B 实体字段变更
   - C 新增或修改功能点
   - D 权限规则变更
   - E 性能或非功能优化
   - F 多改动点组合
4. 展示分类结果并等待用户确认。

## Action 3：影响分析

1. 按 `dimension_mapping` 计算受影响章节。
2. 输出影响分析报告：
   - 受影响章节清单
   - 每个章节的影响原因
   - 建议执行顺序
3. 允许用户调整范围。

## Action 4：执行增量设计

```text
FOR each impacted_element IN confirmed_impact_list by chapter_no:
  调用 element-runner(
    element_id=impacted_element,
    mode="incremental",
    context.base_prd=base_prd,
    context.impact_analysis=impact_report,
    context.change_type=change_type
  )
```

要求 `element-runner` 输出 DELTA 标注块：

```html
<!-- DELTA: type={A-F}, chapter={element_id}, op={add|modify|delete} -->
...内容...
<!-- /DELTA -->
```

## Action 5：合并 DELTA

1. 解析 DELTA 标注块。
2. 按 `op` 类型合并到对应章节：
   - `add`
   - `modify`
   - `delete`
3. 保证未受影响章节保持与基线一致。

更新 frontmatter：

```yaml
version: "{new_version}"
last_updated: "{today}"
fe_reference_snapshot: "{new_fe_path}"
incremental_history:
  - version: "{new_version}"
    date: "{today}"
    changes_summary: ""
```

## 完成阶段

与 `o-init-build` 保持一致：

- 独立审查
- 质量检查
- 知识沉淀建议
- 完成态 frontmatter 更新
