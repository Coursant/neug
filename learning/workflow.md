# Learning Workflow

这个工作流用于把零散提问逐步整理成结构化的 NeuG 学习笔记。

## 每次对话的处理流程

1. 收集输入
   - 用户问题
   - 用户自己的理解
   - 相关代码路径、报错、命令输出或上下文

2. 查证
   - 优先阅读本仓库源码和已有文档。
   - 对涉及构建、测试、依赖版本或 GitHub 状态的问题，以本地命令结果为准。
   - 对外部信息只在必要时查询，并标注来源。

3. 整理
   - 把问题放入 `learning-log.md`。
   - 如果是可复用主题，新增或更新 `learning/topics/*.md`。
   - 如果仍未解决，追加到 `questions.md`。

4. 校正
   - 将用户原始理解拆成正确部分、需要修正部分、未验证部分。
   - 用源码路径和函数/类名支撑结论。

5. 上传准备
   - 检查 `git diff -- learning/`。
   - 提供提交信息建议。
   - 在远端仓库可写时，提交并推送到指定分支。

## 笔记粒度

每个问题不一定都新建文件。默认规则：

- 小问题：只追加到 `learning-log.md`。
- 同一模块的连续学习：更新 `learning/topics/<module>.md`。
- 独立、复杂、可复习的问题：从 `templates/question-note.md` 复制结构，创建单独笔记。

## 推荐提交节奏

- 每完成一个主题或一组相关问题提交一次。
- 提交信息格式：

```text
docs(learning): add notes for <topic>
```

示例：

```text
docs(learning): map neug query pipeline
docs(learning): summarize csr storage layout
docs(learning): record python binding build notes
```
