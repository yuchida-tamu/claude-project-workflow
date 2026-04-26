# post-session

Session-wrap-up plugin for Claude Code. Surveys the current session for durable findings (gotchas, runbooks, preferences, repeated patterns), classifies each as *new skill / skill update / memory entry / discard*, and — after a confirmation gate — writes the skill-worthy ones to disk so the next session is materially more efficient.

## What it does

- Scans the current session for friction, repeated patterns, user corrections/confirmations, and decisions
- Classifies each finding via a decision tree: skill-worthy, memory-worthy, or discard
- Drafts proposals to the user **before** writing anything (Step 3 confirmation gate is non-negotiable)
- On approval:
  - New project-local skills land under `.claude/skills/<name>/SKILL.md`
  - Cross-project skills land under `~/.claude/skills/<name>/SKILL.md`
  - Memory entries follow the auto-memory two-step process (write the memory file, then add a one-line pointer to `MEMORY.md`)
- Leaves working-tree changes for the user to review — never auto-commits

## When it triggers

- Slash command: `/post-session`
- Trigger phrases: "let's wrap up", "review the session", "extract findings", "make next time better", "what did we learn", "compound this", "capture lessons"
- Proactively at the end of any non-trivial session — especially after a multi-PR feature where the orchestration pattern is reusable, or after a session with real friction

## Pairing with the rest of the workflow

This plugin pairs cleanly with `init-project` and `exec-tasks`:

```
/exec-tasks                  # ship a batch of PRs
# ... PRs reviewed and merged ...
/post-session                # capture what slowed us down so the next batch is faster
```

`exec-tasks` produces the runbook-shaped friction (parallel worktrees, contract conformance, review-feedback triage). `post-session` is the place to compound those lessons into project-local skills or memory entries.

## Usage

```
/plugin marketplace add yuchida-tamu/claude-project-workflow
/plugin install post-session@claude-project-workflow
```

Then at the end of a session:

```
/post-session
```

The skill is read-only until Step 3 — it surveys, classifies, and drafts proposals before writing anything. You always get a chance to veto or course-correct.

---

# post-session (日本語)

Claude Code 用のセッション振り返りプラグイン。現在のセッションを走査して持続的な学び（ハマりどころ、手順書、好み、繰り返しパターン）を抽出し、それぞれを *新規スキル / スキル更新 / メモリエントリ / 破棄* に分類して、確認ゲートの後にスキル化すべきものをディスクに書き込みます。次のセッションが目に見えて効率化されることが目的です。

## 機能

- 現在のセッションを走査し、摩擦・繰り返しパターン・ユーザーからの修正/確認・意思決定を抽出
- 決定木で各候補を分類：スキル化すべきか、メモリにすべきか、破棄か
- 何かを書き込む**前に**ユーザーへ提案をドラフト提示（Step 3 の確認ゲートは省略不可）
- 承認後:
  - プロジェクト内スキルは `.claude/skills/<name>/SKILL.md` に配置
  - プロジェクト横断スキルは `~/.claude/skills/<name>/SKILL.md` に配置
  - メモリエントリは auto-memory の 2 ステップ手順に従う（メモリファイルを書く → `MEMORY.md` に 1 行ポインタを追加）
- 作業ツリーの変更はそのまま残し、ユーザーが確認してからコミット — 自動コミットはしない

## トリガー

- スラッシュコマンド: `/post-session`
- トリガーフレーズ: "let's wrap up"、"review the session"、"extract findings"、"make next time better"、"what did we learn"、"compound this"、"capture lessons"
- 非自明なセッションの終わりには能動的に発火 — 特に複数 PR を伴う機能開発の後や、明確な摩擦があったセッションの後

## 他プラグインとの組み合わせ

このプラグインは `init-project` および `exec-tasks` と自然に連携します:

```
/exec-tasks                  # 一連の PR をシップ
# ... PR をレビューしてマージ ...
/post-session                # 何が遅延要因だったかを取り込み、次回をより速くする
```

`exec-tasks` は手順書化に値する摩擦（並列ワークツリー、契約準拠、レビューフィードバックのトリアージ）を生み出します。`post-session` はそうした学びをプロジェクトローカルなスキルやメモリにまとめる場所です。

## 使い方

```
/plugin marketplace add yuchida-tamu/claude-project-workflow
/plugin install post-session@claude-project-workflow
```

セッション終わりに:

```
/post-session
```

このスキルは Step 3 までは読み取り専用です — 走査・分類・提案ドラフトを書き込み前に提示します。常に却下または軌道修正の機会があります。
