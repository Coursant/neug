# GitHub Upload

精简版上传说明。默认不自动 commit/push，只有用户明确要求时执行。

## 当前约定

- 上游：`origin -> https://github.com/alibaba/neug`
- 用户 fork：`coursant -> https://github.com/Coursant/neug`
- 学习笔记分支：`learning/neug-notes`
- 默认只提交 `learning/`

## 常用命令

```bash
git status --short -- learning/
git diff -- learning/
git add learning/
git commit -m "docs(learning): <topic>"
git push coursant learning/neug-notes
```

## 注意

不要把无关本地变更带入提交，例如 `third_party/*`、`.venv/`、build artifacts、generated files。
