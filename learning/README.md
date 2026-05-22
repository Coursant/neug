# NeuG Learning Notes

这个目录用于沉淀学习 NeuG 时产生的问题、理解、纠偏和结论。它不是 NeuG 官方文档的一部分，而是个人学习知识库，目标是让每次提问都能逐步积累成可检索、可复习、可上传到 GitHub 的 Markdown 材料。

## 目录

- [workflow.md](workflow.md): 每次学习对话如何记录、整理和提交。
- [learning-log.md](learning-log.md): 按时间顺序记录每次学习问题和产出。
- [questions.md](questions.md): 待解决问题池。
- [topics/project-map.md](topics/project-map.md): NeuG 第一版项目地图。
- [templates/question-note.md](templates/question-note.md): 单个问题笔记模板。
- [github-upload.md](github-upload.md): 上传到 GitHub 的推荐流程。

## 记录原则

1. 先保留原始问题，再整理结论。
2. 明确区分「我的理解」「代码事实」「修正后的结论」。
3. 每条结论尽量绑定源码位置、命令输出或官方文档路径。
4. 不把尚未验证的猜测写成确定事实。
5. 每次学习结束后更新 `learning-log.md` 和必要的 topic 笔记。

## 后续协作方式

你之后可以直接问：

```text
我对 src/compiler 的理解是 ... 对吗？请帮我整理到学习笔记。
```

我会按这个目录结构做三件事：

1. 阅读相关源码和已有笔记。
2. 校正你的理解，补充代码证据。
3. 更新对应 Markdown 文件，并给出可提交到 GitHub 的变更摘要。
