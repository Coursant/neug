# GitHub Upload Workflow

这个文件记录将 `learning/` 学习笔记上传到 GitHub 的推荐方式。

## 当前状态

本地仓库远端为：

```bash
origin https://github.com/alibaba/neug
```

这通常是上游开源仓库，不建议直接推送个人学习笔记到 `origin/main`。推荐使用个人 fork 或独立笔记仓库。

## 推荐方案 A：推送到个人 fork

适合希望笔记和 NeuG 源码保持在同一个仓库上下文里。

```bash
git remote add myfork git@github.com:<your-user>/neug.git
git checkout -b learning/neug-notes
git add learning/
git commit -m "docs(learning): initialize neug learning notes"
git push -u myfork learning/neug-notes
```

优点：

- 笔记和源码在同一个 Git 历史上下文中。
- 后续可以持续同步上游 `alibaba/neug`。
- 适合做源码阅读笔记。

缺点：

- fork 里会包含完整 NeuG 项目，仓库较大。

## 推荐方案 B：推送到独立笔记仓库

适合只想保存学习笔记，不想维护完整 NeuG fork。

```bash
mkdir neug-learning-notes
cp -R learning/* neug-learning-notes/
cd neug-learning-notes
git init
git add .
git commit -m "docs: initialize neug learning notes"
git branch -M main
git remote add origin git@github.com:<your-user>/neug-learning-notes.git
git push -u origin main
```

优点：

- 仓库轻量。
- 适合公开展示学习过程。

缺点：

- 笔记中的源码路径需要依赖外部 NeuG 仓库。

## 推荐方案 C：当前仓库新分支

只有在你确认自己对远端有写权限，并且接受把学习笔记放在这个仓库中时使用。

```bash
git checkout -b learning/neug-notes
git add learning/
git commit -m "docs(learning): initialize neug learning notes"
git push -u origin learning/neug-notes
```

## 后续自动化约定

每次我更新学习笔记后，可以按以下步骤处理：

```bash
git diff -- learning/
git add learning/
git commit -m "docs(learning): <topic>"
git push
```

如果工作区存在与学习笔记无关的变更，只提交 `learning/`，避免把第三方库、构建产物或环境目录误提交。
