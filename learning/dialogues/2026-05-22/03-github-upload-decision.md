# 03 - GitHub Upload Decision

## 用户问题

如何把 NeuG 学习笔记上传到 GitHub。

## 处理结果

- 上游仓库：`origin -> https://github.com/alibaba/neug`
- 用户 fork：`coursant -> https://github.com/Coursant/neug`
- 学习笔记分支：`learning/neug-notes`

## 形成的决定

- 上传学习笔记时默认只提交 `learning/`。
- 不提交 `third_party/*`、虚拟环境、build artifacts 或 generated files。
- commit/push 必须由用户明确要求触发。
