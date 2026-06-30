# pnc-agnet

面向 `Autoware / ROS 2 / 场景排障 / 调试规划` 的自定义 Agent 与 Skill 集合。

这个仓库主要用于沉淀一套可复用的 VS Code / Copilot Agent 工作流，帮助在自动驾驶研发场景中更高效地完成以下工作：
- 根据问题场景快速整理专业的 `场景描述.md`
- 从场景现象定位到关键函数、调用链和代码节点
- 基于问题场景规划低噪声、可落地的调试日志方案
- 对 Autoware 功能改动生成最小侵入式的实施计划与验证步骤
- 复用经过整理的调试、规划、代码评审与技能编写方法论

## 仓库定位

`pnc-agnet` 不是业务代码仓库，而是一个 **Agent / Skill 工作流仓库**。

它的核心目标是：
- 把高频使用的分析代理沉淀成可复用的 `.agent.md`
- 把通用的方法论沉淀成可复用的 `SKILL.md`
- 提高场景分析、日志规划、实现交接与研发协作的一致性

## 目录结构

```text
agnet/
├── agents/     # 自定义 Agent 定义
├── skills/     # 可复用 Skill 方法论
└── scripts/    # 预留脚本目录（当前为空）
```

## Agents

当前仓库包含的主要 Agent：

- `Autoware Feature Delivery Agent.agent.md`
  - 面向 Autoware ROS 2 单仓的最小侵入式实施规划代理
- `FunctionLocator.agent.md`
  - 根据问题场景定位关键函数、调用链和代码位置
- `Scenario Debug Manager.agent.md`
  - 场景调试编排中心，串联函数定位与日志规划
- `Scenario Description Writer.agent.md`
  - 根据用户自然语言描述生成专业的 `场景描述.md`
- `Scenario Root Cause Iteration Manager.agent.md`
  - 面向根因定位的多轮场景分析管理代理
- `Scenario Solution Iteration Manager.agent.md`
  - 面向问题解决方案迭代的管理代理
- `debug-log-cleanup.agent.md`
  - 调试日志清理与收敛辅助代理
- `scenario-implementation-agent.agent.md`
  - 承接规划结果并执行实现、验证与交付
- `scenario-investigation-planner.agent.md`
  - 场景问题调查与证据驱动计划输出代理
- `scenario-log-cause-planner.agent.md`
  - 基于场景描述与日志证据判断问题定位层级
- `scenario-node-debug-planner.agent.md`
  - 从问题场景出发规划节点级调试日志方案
- `scenario-simulation-iteration-manager.agent.md`
  - 场景仿真迭代流程管理代理
- `scenario-simulation-launcher.agent.md`
  - 场景仿真启动与运行流程辅助代理
- `CodeReadingLearningGuide.agent.md`
  - 代码阅读与学习导向型辅助代理

## Skills

当前仓库收录的主要 Skill 方向：

- `brainstorming/`
- `dispatching-parallel-agents/`
- `executing-plans/`
- `finishing-a-development-branch/`
- `receiving-code-review/`
- `requesting-code-review/`
- `subagent-driven-development/`
- `systematic-debugging/`
- `test/`
- `test-driven-development/`
- `using-git-worktrees/`
- `using-superpowers/`
- `verification-before-completion/`
- `writing-plans/`
- `writing-skills/`

这些 Skill 更偏向于方法论沉淀，用来支持 Agent 在真实研发任务中的稳定输出。

## 典型使用场景

### 1. 场景描述沉淀
当你拿到一个 bag、视频或现场现象描述时，可以用 `Scenario Description Writer` 把口语化描述整理成研发可读的 `场景描述.md`。

### 2. 场景问题定位
当你已经明确症状，但还不知道从哪个函数入手时，可以用 `FunctionLocator` 或 `Scenario Debug Manager` 做函数映射和路径梳理。

### 3. 调试日志规划
当你需要针对问题场景打日志，又不希望日志泛滥时，可以用 `scenario-node-debug-planner` 规划基于场景门控、对象门控的低噪声日志插点。

### 4. 实施交接
当方案已经明确，需要有人落代码、跑验证时，可以将规划结果 handoff 到 `scenario-implementation-agent`。

## 推荐工作流

一个典型的场景排障链路如下：

1. 先用 `Scenario Description Writer` 整理场景描述
2. 再用 `FunctionLocator` 定位关键函数与调用链
3. 再用 `Scenario Node Debug Planner` 规划节点级日志方案
4. 必要时通过 `Scenario Debug Manager` 统一编排上述步骤
5. 最后将确认后的计划交给 `scenario-implementation-agent` 执行实现与验证

## 适用范围

本仓库尤其适合以下类型工作：
- Autoware / ROS 2 功能排障
- 自动驾驶规划控制链路问题分析
- 场景包、bag、日志、视频联合分析
- 调试日志设计与收敛
- 研发方法论沉淀与团队复用

## 当前状态

- 仓库已初始化并推送到 GitHub
- `agents/` 与 `skills/` 已具备基础可用内容
- `scripts/` 目录当前为空，可按需补充自动化辅助脚本

## 后续建议

可以继续补充以下内容：
- 一个更详细的 `CONTRIBUTING.md`，说明如何新增 Agent / Skill
- 一个 `examples/` 目录，放典型输入输出示例
- 一个统一的命名规范文档，约束 Agent / Skill 的 frontmatter 与描述风格
- 一个最小 `.gitignore`，避免后续把临时文件、缓存或大包误提交
