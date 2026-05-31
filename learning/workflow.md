# Learning Workflow

精简版流程说明。详细 agent 执行规则见 [AGENTS.md](AGENTS.md)。

## 默认处理

1. NeuG 学习、源码阅读、C++ 学习问题默认记录到 `learning/`。
2. 小问题追加到 `learning-log.md`。
3. 可复用主题更新 `topics/*.md`。
4. 未解决问题放入 `questions.md`。

## 记录格式

- 保留原始问题。
- 区分：我的理解、代码事实、修正后的结论、未验证假设、后续问题。
- 结论尽量绑定源码路径、命令输出或文档路径。

## 提交

提交或上传必须等用户明确要求。

```bash
git diff -- learning/
git add learning/
git commit -m "docs(learning): <topic>"
git push
```
