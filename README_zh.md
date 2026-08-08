# looppower

一个基于「计划与执行严格分离」的 Claude Code 插件：专家组负责规划，循环负责执行。两侧各司其职——**schizophrenic** 专家组产出 High-Level 计划，**dynamic-loop** skill 按计划逐 phase 执行，直到专家组判定目标达成。

## 快速开始

在 Claude Code 中安装：

```
/plugin marketplace add zhaoyshine/looppower
/plugin install looppower
```

安装后用 `/plugin` 确认 `looppower` 已出现。

需要 Claude Code v2.1.196 及以上版本（见[环境要求](#环境要求)）。

## 使用示例

**场景：发布前检查项目。** 用户在 Claude Code 中输入：

```
/looppower:dynamic-loop 帮我把这个项目发布到 GitHub 前检查一遍
```

循环随即开始：专家组规划一个 phase，skill 并行派 subagent 执行各检查 task 并回填结果，专家组评审后判定继续或停止，直到需求达成。期间你可以随时打断，补充信息或调整方向。

## 工作原理

多数 agentic 工作流让同一个 agent 既决策又执行，计划因此悄悄过期。looppower 把职责拆给两个角色，各有限定：

- **规划是专家组的事。** **schizophrenic** agent 以专家组形态工作，一次只规划一个 phase，产出一份 High-Level 执行计划文档，绝不碰执行。phase 完成后，它评审反馈并规划下一个 phase——直到判定目标达成。
- **执行是循环的事。** **dynamic-loop** skill 严格按专家组计划执行。每个 task 都有进度跟踪，跑完一个 phase 就回到专家组申请下一轮。不临场发挥，不偏离计划。

## 基本工作流

1. 任务需要多步计划，或已有一份执行计划待执行时，**dynamic-loop** skill 接管。
2. skill 向专家组（**schizophrenic**）申请 High-Level 执行计划。
3. skill 用 `TaskCreate`/`TaskUpdate` 逐 task 执行计划并跟踪进度。
4. phase 完成后，专家组评审结果，判定是否还需要下一 phase。
5. 循环反复，直到专家组宣布目标达成。

简单的一次性任务不需要这个 skill。

## 包含内容

- **schizophrenic** — 专家组 agent，一次只产出一个 phase 的 High-Level 执行计划。
- **dynamic-loop** — 执行 skill，循环运行计划、跟踪进度、申请下一 phase，直到专家组宣布目标达成。

skill 刻意命名为 **dynamic-loop** 而非 looppower：插件是整体，skill 是插件里的一个组件，组件按职责命名，不跟随插件名。

## 设计理念

**模型分级。** 专家组默认使用最强的 AI 模型（opus），只承担一件事：High-Level 规划与重大决策。它不读代码、不执行任务。调研代码、跑 lint、机械执行这些 token 大头，都被拆成小任务派给更便宜的模型（haiku/sonnet）。模型选择遵循「重大决策使用 opus，简单的事情使用 haiku，sonnet 作为 haiku 的备用」，并通过减小 Task 复杂度来减少 opus 的使用。Claude Code 的 plan 模式由主模型边探索边规划，探索本身很贵；本工作流把最贵的模型完全挡在执行之外，只做决策，对大多数任务，同样的结果整体开销更低。

**工具自动扩展。** 本项目的 skill 与 agent 位于插件根目录下的 `skills/`、`agents/` 目录，由 Claude Code 自动发现并触发。Claude 升级带来的新能力与新工具不需要改动本工作流；你自己下载安装的 skill 和 agent 也会被同样自动发现、纳入执行。

**流程不固定。** 随 AI 升级、按你的偏好动态调整。规划按 phase 分批产出，一次只规划当前信息能支撑的一个 phase，下一个 phase 等真实执行结果回填后再规划——计划始终建立在事实上。循环没有预设总轮数，它跑到专家组判定「需求已达成」为止。模型选择、偏好与禁止规则、计划文档目录，都按你的环境与偏好配置。

## 环境要求

- Claude Code v2.1.196 及以上。**schizophrenic** 的计划文档路径模板使用 `${CLAUDE_PROJECT_DIR}` 占位符，该占位符自 v2.1.196 起才会展开。旧版本上计划路径无法解析。

## 许可证

[MIT](LICENSE)
