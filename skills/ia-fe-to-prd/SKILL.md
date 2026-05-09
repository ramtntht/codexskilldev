---
name: ia-fe-to-prd
description: >
  支持从 FE 文档生成 PRD 文档（产品需求文档/产品方案）。支持多模式工作流（新建、
  增量设计、评审修改、续接），覆盖 0-1 专题需求（新建系统）和 1-n 优化需求
  （现有系统改进），包括 TP/AP/AI 类需求。
  适用场景：（1）FE 文档已完成需生成 PRD；（2）历史 PRD 评审修改；
  （3）增量 PRD 设计；（4）续接未完成的 PRD。
  
  即使用户没有明确说"创建 PRD"，只要他们想：
  - 把已经整理好的业务需求（FE）转化为可实施的产品方案
  - 基于评审意见对已完成 PRD 做局部修改
  - 在历史 PRD 基础上做增量设计
  - 续接之前没写完的 PRD
  也应优先使用本 Skill。
  
  触发词：创建PRD、生成PRD、产品方案设计、需求转PRD、FE转PRD、ia-fe-to-prd、
  根据评审意见修改、修改PRD、PRD评审、写产品方案、把需求转成产品方案、
  做产品设计、PRD续接、继续做PRD
disable-model-invocation: false
version: 2.0.0
spec_compliance: "v1.2.0"
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
2. 引擎版本自检（容错执行）：
   - 读取 `engine/ENGINE-VERSION`，记录当前引擎版本号
   - 尝试读取 `docs/engine-canonical/ENGINE-VERSION`：
     - 若文件存在且内容一致 → 继续，不输出
     - 若文件存在但内容不一致 → 输出警告（不阻断）：
       ```
       ⚠️ 引擎版本与权威源不一致：
         当前: {engine/ENGINE-VERSION}
         权威源: {docs/engine-canonical/ENGINE-VERSION}
       建议执行同步：
         cp docs/engine-canonical/* skills/ia-fe-to-prd/engine/
       ```
     - 若 `docs/engine-canonical/` 不可访问（如 Skill 已安装到运行环境，
       无开发仓库上下文）→ 静默跳过，不输出任何信息
   - 无论结果如何，本步骤不阻断启动序列
3. 读取 `engine/workflow-engine.md`，把用户原始输入作为 `user_message` 传入。
4. 由 `workflow-engine` 构建 Input Inventory、执行 SceneRouter、确认当前 workflow。
5. 仅在 workflow 确认后加载对应的 orchestration 文件。
6. orchestration 只负责初始化、顺序编排、结果收尾；实际章节生成与修改都交给 `element-runner`。

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
