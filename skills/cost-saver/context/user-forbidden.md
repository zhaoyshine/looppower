# 用户禁止项

## 开发流程

- 注释就是代码实现的重复说明/对读代码的人无帮助（复述代码已表达的 what）
- 注释冗长啰嗦/超过3行（可以不写注释就不写，要写也只写重点）
- 顺手重构无关代码，夹带需求外功能
- 开发全流程（代码、注释、变量名、commit message、Jira 任务、PR 标题与正文）一律用英文，不写中文

## Commit 与 PR

- 尝试绕过 pre-commit hook
- 用 git commit -m 不带 -S（必须签名）
- 使用 --no-gpg-sign 绕过签名
- git add 时用 git add -A / 一把全加，须列出具体文件
- 在注释、代码、PR body、commit 信息、ticket 中使用中文

## Git 与 Worktree

- 在原 repo 工作目录 commit/checkout/改文件/切分支/pull（所有改动隔离在 git worktree 内）
- 使用 git worktree remove / rm -rf / 删除 worktree 目录（结束时原样保留交下游）
- 使用 git push --force / --force-with-lease（仅新增 commit，不改写历史）
- 使用 git reset --hard、强推
- 在非 repo 目录中使用 git 命令
- 写死主分支名为 main/master（须从 repos 记忆中读取）
- working tree 不干净时自行 stash/reset/clean/checkout
- 使用非 fast-forward 合并（git pull 须带 --ff-only）

## 测试

- 跑本地测试前不看 local-setup-guide
- 使用 bin/docker_bundle rspec 跑测试
- 写无法触发的测试（错误路径若无可靠注入手段则不写测试，不为防御性代码配 documentation-only 测试）
- 运行全量测试套件（只跑改动相关的测试/Lint/Check）

## 调研

- 给出结论但无法提供对应证据
- 知道答案就在代码里面但不去查
- 结论/洞察用空泛套话（如"这很重要""值得学习""AI是趋势"）
- 没有发现/结果时编造或注水（应如实说明"未发现问题"/"仅有N条"）
