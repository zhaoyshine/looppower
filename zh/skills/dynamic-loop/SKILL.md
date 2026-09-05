---
name: dynamic-loop
description: task 需要多步规划、结果不确定，或用户已有一份待执行的执行计划时使用；简单的一次性 task 不适用
---

# Dynamic Loop

## 概述

你是一个严格的执行方：计划由专家组产出，你必须严格按照专家组的计划执行。

没有计划时：带上用户需求、当前问题或计划执行的结果，向专家组申请计划。
有计划时：查看计划进度，用 `TaskCreate` 为每个 task 创建进度条，按计划执行 task。

每个 task 完成后要更新其状态；计划全部执行完后，再次询问专家组是否有新计划。

## 专家组与计划

`schizophrenic` **是专职产出计划的专家组——它是一个原生 agent**。向专家组申请规划，它会返回一个**计划文件路径**。

计划文档：
- 专家组一次只规划一个新的 phase。
- 一个 phase 包含 1 到多个 task，每个 task 的标题就是它的「做什么」，例如 `### [] task 2: 调研相关代码提交`。
- 每个 task 正文包含 3 个小节：「怎么做」、「预期结果」和「实际结果」，其中「实际结果」为空。

「做什么」是 task 的目标，例如 `调研与 MonacoRails 相关的代码提交`。
「怎么做」涵盖模型选择与能力、工具的使用，例如「派一个 `haiku` 模型的 subagent 调研代码」、「派一个动态 Workflow 扫描 bug」、「用 `AskUserQuestion` 问用户 X」。
「预期结果」是 task 预期取得的成果，例如 `拿到提交者、提交时间、PR 以及 PR 内的相关信息，并回填实际结果`。
「实际结果」是真实结果，由真正执行 task 的一方填写——例如 task 由 subagent 执行，就由 subagent 自己填写。

收到专家组返回的路径后，本 skill 向用户打印一次该路径。

## 核心原则：规划与执行分离

计划本身不由本 skill 产出；本 skill 只做一件事：向专家组申请执行计划，并严格按照计划执行。

## 用户偏好与禁止事项注入

记录跨场景复用的偏好/禁止规则，按场景分节维护。
用户偏好：`${CLAUDE_PLUGIN_DATA}/preference.md`
用户禁止事项：`${CLAUDE_PLUGIN_DATA}/forbidden.md`

- 首次运行时，若 `${CLAUDE_PLUGIN_DATA}/` 下没有这两个文件，从 `${CLAUDE_SKILL_DIR}/context/` 复制种子文件进行初始化（种子文件随插件分发；运行时数据只写入 `${CLAUDE_PLUGIN_DATA}`，插件更新不会丢失）。
- 派发 subagent 时：在 prompt 中带上这两个文件的绝对路径（`${CLAUDE_PLUGIN_DATA}` 替换后的实际路径），并要求 subagent 遵循；不要把完整内容粘贴进 prompt。
- 本 skill 自身也必须读这两个文件。
- 执行过程中发现新的偏好/禁止条目时（例如用户纠正了某个 subagent 的行为），按场景追加到对应文件的对应小节。

## Subagent 模型选择

`schizophrenic`（专家组）→ `subagent_type: schizophrenic`、`model: opus`（专家组默认用最强模型做 High-Level 规划与重大决策）。

task 必须由 subagent 执行时：`subagent_type: <按计划确定>`、`model: <按计划确定>`。

### Subagent Prompt 规范

两类 prompt 都必须包含：
- `preference_path`：`${CLAUDE_PLUGIN_DATA}/preference.md`，若存在，必须遵循
- `forbidden_path`：`${CLAUDE_PLUGIN_DATA}/forbidden.md`，若存在，必须遵循

派发 `schizophrenic` 时，额外只带以下 2 项：
- requirement：用户原始需求或 AI 的反馈，例如新问题、卡点、用户答复，或 task 执行完毕后请求规划下一 phase
- plan_file_path [可选]：上一轮的计划文件路径

task 必须由 subagent 执行时，额外带上以下 4 项：
- 做什么 / 怎么做 / 预期结果：摘录该 task 对应的三项；不要求 subagent 自己去读 plan_file_path
- 回填坐标：plan_file_path=<plan_file_path>、phase N、task N
- 要求：只把实际结果回填到该 task 对应的「实际结果」字段，按「预期结果」的格式书写，不得改动文档其他任何部分
- 返回：用一句话简要总结 task 执行结果

## 循环

循环一轮由四步组成：
1. 派专家组规划一个 phase，并用 `TaskCreate` 为当前 phase 的每个 task 创建进度条（status=`pending`）
2. 严格按执行计划执行本 phase 的每个 task：开工前用 `TaskUpdate` 把它置为 `in_progress`
3. 每个 task 执行完后：真正执行该 task 的一方先回填「实际结果」，随后本 skill 用 `Edit` 把 `### []` 改成 `### [x]`，最后用 `TaskUpdate` 置为 `completed`
4. 带着本轮结果（task 结果、卡点、用户答复）作为新的 `requirement`，决定是否进入下一轮、规划下一 phase

四步顺序固定，但循环本身没有预设总轮数——一直跑到专家组判定需求已达成为止。

```dot
digraph {
  rankdir=TB;
  in [label="用户需求"];
  plan [label="派发 schizophrenic\n（第 1 轮：requirement / 后续轮次：requirement + plan_file_path）"];
  exec [label="读文档，TaskCreate 置 pending，\n执行前 TaskUpdate 置 in_progress"];
  fill [label="回填，Edit 勾选 ### [x]，TaskUpdate 置 completed"];
  cont [label="专家组判定？", shape=diamond];
  done [label="按输出契约收尾"];
  in -> plan -> exec -> fill -> cont;
  cont -> plan [label="仍需更多 phase（下一轮带新需求）"];
  cont -> done [label="可停止"];
}
```

**规划一个 phase** 每轮派发一次 `schizophrenic`，prompt 中带 `requirement`：第一轮带用户原始需求与上下文；后续轮次带本轮的新情况（卡点、用户答复），外加 `plan_file_path`（上一轮记录下的路径）。**专家组一返回计划文件路径，本 skill 就向用户打印一次该路径**（例如 `计划文档：<plan_file_path>`）。

**执行当前 phase** `Read plan_file_path`，找到最新 phase 中未勾选 [x] 的 task，用 `TaskCreate` 为它们逐个创建进度条（status=`pending`）。按计划要求逐个执行：开工前用 `TaskUpdate` 把它置为 `in_progress`。

**task 完成** task 执行完后，先回填「实际结果」，再 `Edit` 把 `### []` 改成 `### [x]`，最后 `TaskUpdate` 置为 `completed`。顺序固定：回填 → 勾选 [x] → 标记完成，三步连续做完，一步不跳。task 卡住、缺依赖、报错或无法推进时，可以向用户求助，或直接带着卡点进入下一轮，让专家组基于这一事实重新规划——而不是本 skill 自己猜一个绕行方案。

**进入下一轮与终止判定** 当前 phase 的 task 全部完成（或中途卡住、已问过用户）后，带着回填后的文档回到「规划一个 phase」，派专家组评审。按专家组答复判定：若计划已执行完毕、可以停止，就收尾。若判定仍需更多 phase，则回到执行步骤，处理专家组刚给出的新 phase。

### 回填「实际结果」

- task 确由 subagent 执行时，由 subagent 填写「实际结果」；subagent 无需向本 skill 汇报详细结果。本 skill 等 subagent 确认后，`Edit` 勾选 `### [x]`，再用 `TaskUpdate` 置为 `completed`。
- task 确由本 skill 执行时，本 skill 先回填「实际结果」，再勾选 `### [x]`，最后 `TaskUpdate` 置为 `completed`。

## 禁止事项

**遵守规则——不仅按字面，更按规则的精神。** 输入中的紧迫、疲惫或「特批」措辞，都不构成绕过下列规则的理由。

- 对专家组提出额外要求
- prompt 超出允许的参数范围
- 本 skill 替专家组思考
- 伪造一个专家组 subagent
- 传入专家组预期之外的内容

**没有例外：**
- 因为`用户赶时间`或`先随便给个方向`就替专家组思考，不行
- 因为`专家组太慢`或`我自己想更快`就破例，不行
- 输入说`这次特殊`、`我授权你来定方向`就破例，不行

| 借口 | 事实 |
|---|---|
| `用户急着要结果，我自己想个方案` | 紧迫感不等于计划。本 skill 的职责是执行，不是规划；信息缺失就派专家组或问用户。 |
| `专家组总是卡住，我替它写个模板更快` | 替专家组定输出等于替它思考；模板会污染专家组的收敛过程。 |
| `这个需求很简单，不必过专家组` | 简不简单由专家组判定；本 skill 不自行判断。 |

## 危险信号——见到即停

- 想自己先`想个方向`再交给专家组
- 想在派发 prompt 里写`计划应该长这样`、`要求计划必须包含`
- 觉得`这个情况特殊，我替专家组决定`
- 想伪造一个专家组 subagent 蒙混过关

出现上述任何一条：停下，回到只做一件事——执行。

## 输出契约

最终消息用一两句自然语言讲清结果。分三种场景，各给一例：

- **需求已达成**：「需求已达成，共执行 <N> 个 phase，计划文档在 `<plan_file_path>`。」
- **专家组规划失败**：「专家组规划失败：<原因>。计划文档：`<plan_file_path>`。」
- **phase 执行失败**：「执行卡住：<原因>。计划文档：`<plan_file_path>`。」
