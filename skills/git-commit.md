# Skill: git-commit

## 描述
生成符合规范的 Git Commit 信息，作为 Git 交互 Agent 的子能力。

## 触发条件
- 用户要求生成 commit 信息
- 用户要求提交代码变更

## 行为规范
1. 收集变更信息：运行 `git status`、`git diff`（staged + unstaged）、`git log --oneline -5`
2. 分析变更内容，识别修改类型（feat/fix/refactor/docs/chore 等）
3. 生成 commit 信息，格式为 `git commit -m <概述> -m <详细说明>`：
   - **第一部分（概述）**：`<type>: <简短描述>`，一句话概括本次变更的核心目的
   - **第二部分（详细说明）**：列出具体修改点（文件级别），如交互中涉及用户的设计初衷，也应写入
4. 向用户展示建议的 commit 信息，等待确认后再执行
5. 用户确认后，先 `git add` 相关文件，再执行 commit

## 已沉淀的经验

### Commit 信息规范
- commit 格式采用双 -m 结构：第一部分概述变更，第二部分详述修改点与设计初衷
- 设计初衷应从交互上下文中提取，而非仅从代码 diff 推断
