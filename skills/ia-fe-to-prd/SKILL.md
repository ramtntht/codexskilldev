---
name: ia-fe-to-prd
description: 支持从FE文档生成PRD文档，且支持多模式工作流程（新建、增量设计、评审修改、续接），支持0-1专题需求(新建系统)和1-n优化需求(现有系统改进)，包括TP/AP/AI类需求。适用于IT产品设计场景中FE→PRD转换工作。场景包括：（1）FE文档已完成需生成PRD，（2）历史PRD评审修改，（3）增量PRD设计，（4）续接未完成PRD。触发词：创建PRD、生成PRD、产品方案设计、需求转PRD、FE转PRD、ia-fe-to-prd、根据评审意见修改、修改PRD
disable-model-invocation: false
version: 2.0.0
---

# ia-fe-to-prd

启动声明：

`我正在使用 ia-fe-to-prd，将已完成的 FE 文档转化为可实施的 PRD，或对现有 PRD 执行评审修改 / 增量设计。`

## 全局执行约束

- 先路由、后执行：先完整读取 `engine/workflow-engine.md`，确认 workflow 后再进入具体编排。
- 所有要素都必须经过 `engine/element-runner.md` 的 6 个阶段，禁止 orchestration 直接绕过要素执行。
- 只在 `Phase 6` 更新 PRD frontmatter，禁止在其他位置私自写入 `stepsCompleted`、`last_element`、`status`。
- 始终使用中文输出，并优先基于 FE / PRD / ReviewItem 的已有事实，不凭空补设业务细节。

## 启动序列

1. 读取 `config.yaml`，建立路径、文档类型、registry 与 standards 的挂载点。
2. 读取 `engine/workflow-engine.md`，把用户原始输入作为 `user_message` 传入。
3. 由 `workflow-engine` 构建 Input Inventory、执行 SceneRouter、确认当前 workflow。
4. 仅在 workflow 确认后加载对应的 orchestration 文件。
5. orchestration 只负责初始化、顺序编排、结果收尾；实际章节生成与修改都交给 `element-runner`。

## 完成提示模板

```text
✅ ia-fe-to-prd 已完成

输出文件：
  {output_folder_base}/{current_version}/{output_filename}

当前模式：
  {workflow_id} / {execution_mode}

建议下一步：
  ia-prd-to-design {current_version}
```
