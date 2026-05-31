# AGENTS.md

This file is for assistants/agents working inside `learning/`. The human-facing overview is `README.md`.

## Scope

- The user is studying the NeuG C++ graph database repository at `/home/lcc/cpp/neug`.
- Treat `learning/` as the user's personal learning knowledge base, not official NeuG documentation.
- Use mixed Chinese and English in this directory: Chinese for explanations and reflections; English for code terms, APIs, class/function names, commands, and standard technical vocabulary.

## Default Workflow

1. For every NeuG learning, source-reading, or C++ learning question, update `learning/` by default.
2. Preserve the original question before summarizing conclusions.
3. Separate `我的理解`, `代码事实`, `修正后的结论`, `未验证假设`, and `后续问题` when the user gives their own interpretation.
4. Bind conclusions to source paths, command outputs, or existing documentation whenever possible.
5. Do not write unverified guesses as facts.
6. Update `learning/learning-log.md` after each answered learning question.
7. Update a topic note under `learning/topics/` when the answer is reusable.

## Source Reading Rules

- Inspect local source code and existing notes before explaining NeuG internals.
- Explain with concrete file paths, classes, functions, and commands.
- For C++ learning tasks, prefer incremental, project-shaped examples over isolated snippets.
- Keep edits scoped and consistent with the existing repository structure.
- Run local build/test/format commands when feasible; if not run, say so clearly.

## Note Locations

- `learning/learning-log.md`: chronological learning records.
- `learning/questions.md`: unresolved questions.
- `learning/topics/*.md`: reusable topic notes.
- `learning/dialogues/<date>/`: structured dialogue summaries.
- `learning/templates/question-note.md`: template for single-question notes.

## Git And Upload Rules

- Do not commit, push, or upload unless the user explicitly asks.
- The NeuG learning branch is `learning/neug-notes`.
- The user's fork remote is `coursant`, pointing to `https://github.com/Coursant/neug`.
- When committing learning notes, stage only `learning/` unless the user explicitly asks otherwise.
- Avoid committing unrelated local changes such as `third_party/*`, virtual environments, build artifacts, or generated files.
