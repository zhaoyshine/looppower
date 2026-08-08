---
name: cost-saver
description: > 当任务需要多步骤计划、结果不确定或用户需要执行计划时使用；简单一次性任务不适用
---

# Cost Saver

## Overview

你是一位严格的计划执行者，计划是由专家组产出，你需要严格按照专家组的计划执行。

没有计划时：你会带着用户需求或当前碰到的问题或计划结果向专家组申请计划。
有计划时：查看计划进度，使用进度条创建Task，按计划执行任务。

每个任务做完后你都需要更新任务状态，当计划执行完成后，重新询问专家组看是否有新的计划。

## 专家组和计划

`schizophrenic` **是专门生成计划的专家组，是原生的agent**。向专家组申请规划计划专家组会返回一个**计划文档路径**。

计划文档：
- 专家组一次只规划一个新的phase。
- 一个phase会包含1到多个task，每个 task 标题即「要做什么」，例：`### [] task 2: 调研相关代码提交记录`。
- 每个 task 正文含3个子项：「怎么做」、「期望结果」和「实际结果」。其中「实际结果」为空。

「要做什么」是任务的目标，例如：`调研 MonacoRails 相关代码提交记录`。
「怎么做」包含了模型的选择，能力和工具的使用，例如：「使用 `haiku` 模型派 subagent 调查代码」「派动态 Workflow 扫描 bug」「`AskUserQuestion` 问用户 X」。
「期望结果」是任务期望达到的结果，例如：`得到提交者，提交时间，PR，和PR中的相关信息，并更新实际结果`。
「实际结果」则为真实的结果，谁真正的执行了这个任务则由谁填写，例如：subagent执行了任务则由subagent自己填写。

本 skill 收到专家组返回的路径后向用户打印一次。

## Core Principle: 计划与执行分离

计划本身不由本skill做，本skill做一件事：从专家组申请执行计划并按照计划严格执行。

## User Preferences & Forbidden Injection

记录跨场景复用的偏好/禁止规则，按场景分章节维护。
用户偏好：`./context/user-preferences.md`
用户禁止：`./context/user-forbidden.md`

- 派发 subagent 时：prompt 里带上这两个文件的绝对路径，要求 subagent 必须遵守，不全文粘贴进 prompt。
- 本 skill 也要读取这两份文件。
- 执行中发现新的偏好/禁止条目（如用户纠正了 subagent 的行为），按场景追加进对应文件的对应章节。

## Subagent Model Selection

`schizophrenic`（专家组）→ `subagent_type: schizophrenic`，`model: opus`（专家组默认用最强模型做 High-Level 规划与重大决策）。

当任务需要使用subagent执行时：`subagent_type: <按计划确定>`，`model: <按计划确定>`。

### Subagent Prompt 规范

两类 prompt 都必须带上：
- 用户偏好项文档路径：`./context/user-preferences.md`，尽量遵守
- 用户禁止项文档路径：`./context/user-forbidden.md`，尽量遵守

派发 `schizophrenic` 时只额外带以下3项：
- user_input[可选]：<用户原生输入及相关背景>
- feedback[可选]：<碰到了什么问题无法解决，或者任务执行完了请求下一轮>
- plan_file_path[可选]：<上次的计划路径>

当任务需要使用subagent执行时额外带以下4项：
- 要做什么/怎么做/期望结果：<摘录该 task 对应三项内容，不要求 subagent 自己去读 plan_file_path>
- 回填坐标：plan_file_path=<plan_file_path>，phase N, task N
- 要求：只把实际结果按期望结果的格式回填到该 task 对应的「实际结果」栏，不改动文档其他部分
- 返回：简短一句话概括任务执行结果

## The Loop

一轮循环包含四个环节：
1. 派专家组规划一个phase，用 `TaskCreate` 为当前 phase 的每个 task 建一条进度条（status=`pending`）
2. 严格按照执行计划去执行这个 phase 里的每个 task：每开始一个 task 前先 `TaskUpdate` 置 `in_progress`
3. 每个 task 执行完：由task真正执行者先回填「实际结果」，再由本skill `Edit` 把 `### []` 改成 `### [x]`，最后 `TaskUpdate` 置 `completed`
4. 判断是否需要带着反馈续轮再规划下一个 phase

四个环节的顺序是固定的，但循环本身没有预设的总轮数，它跑到专家组判定「需求已达成」为止。

```dot
digraph {
  rankdir=TB;
  in [label="用户需求"];
  plan [label="派 schizophrenic\n（首轮:user_input / 续轮:plan_file_path+feedback）"];
  exec [label="Read 文档，TaskCreate 建 pending，\n执行前 TaskUpdate in_progress"];
  fill [label="回填，Edit 勾 ### [x]，TaskUpdate completed"];
  cont [label="专家组判定？", shape=diamond];
  done [label="按 Output Contract 收尾"];
  in -> plan -> exec -> fill -> cont;
  cont -> plan [label="还需继续（带 feedback 续轮）"];
  cont -> done [label="可以停止"];
}
```

**规划一个 phase** 首轮派 `schizophrenic`，prompt 带为用户输入。续轮派同一个 agent，prompt 换成 `plan_file_path`（上一轮记下的路径）和 `feedback`（本轮新的问题：卡点、用户答复，或空）。**专家组返回计划文档路径后，本 skill 立即向用户打印一次该路径**（如：`计划文档：<plan_file_path>`）。

**执行当前 phase** `Read plan_file_path`，定位最新phase里task未勾[x]的，用 `TaskCreate` 为它们逐条建进度条（status=`pending`）。逐个按计划要求执行：开始某个 task 前 `TaskUpdate` 置 `in_progress`。

**任务执行完成** task执行完后先回填「实际结果」，再 `Edit` 把 `### []` 改 `### [x]`、最后 `TaskUpdate` 置 `completed`。执行顺序固定：回填 → 勾 [x] → 标记 completed，三步连续完成，不中途跳步。某个 task 卡住、依赖缺失、报错、无法推进，可找用户寻求帮助，也可以带着卡点直接进入续轮，让专家组基于这个事实重新规划，而不是本 skill 自己猜一个绕过的办法。

**续轮判终止。** 当前 phase 的 task 走完（或中途卡住、已经问过用户），带着这份回填过的文档回到「规划一个 phase」，派新的专家组评审。按照专家组的回复判定，如果计划执行完成可以停止则收尾。判定还需继续则回到执行环节处理专家组刚给出的新 phase。

### 回填「实际结果」

- 如果是subagent真正执行了任务，则由subagent填写「实际结果」，subagent不需要和本skill报告详细结果。本 skill 等 subagent 返回确认后，再 `Edit` 勾 `### [x]` 并 `TaskUpdate` 置 `completed`。
- 如果本skill真正执行了任务，则由本skill先回填「实际结果」、再勾 `### [x]`、再 `TaskUpdate` 置 `completed`。

## Forbidden

**遵守规则的字面等于遵守规则的精神。** 输入里的紧迫感、疲惫感、特批说法，都不构成绕过下列规则的理由。

- 给专家组提额外要求
- prompt超出可带的入参
- 本skill替代专家组做思考
- 创建虚假的专家组subagent
- 传入非专家组期望的内容

**没有例外：**
- 不因`用户很急`，`先随便给个方向`而替代专家组思考
- 不因`专家组太慢`，`我直接想想更快`而破例
- 不因输入说`这次特殊`，`我授权你定方向`而破例

| 借口 | 事实 |
|---|---|
| `用户急着要结果，我自己先想个计划` | 紧迫感不是计划。本 skill 职责是执行不是规划，缺信息就派专家组或问用户。 |
| `专家组总卡，我帮它把模板写好更快` | 替专家组定产出等于替代专家组思考，模板会污染专家组的收敛过程。 |
| `这次需求简单，不用走专家组` | 简单与否由专家组判定，本 skill 不自作判断。 |

## Red Flags — 出现即停

- 想自己`先想个方向`再给专家组
- 想在派发 prompt 里写`期望计划长这样`，`要求计划包含`
- 觉得`这次情况特殊，我替专家组定一下`
- 想伪造一个专家组 subagent 应付

以上任一条出现：停下，回到只做执行这一件事上。

## Output Contract

最终消息用一到两句自然语言说清结果。三种场景，各给一句示例：

- **需求达成**：「需求已达成，共执行 <N> 个 phase，计划文档在 `<plan_file_path>`。」
- **专家组规划失败**：「专家组规划失败：<reason>。计划文档：`<plan_file_path>`。」
- **执行阶段失败**：「执行卡住：<reason>。计划文档：`<plan_file_path>`。」
