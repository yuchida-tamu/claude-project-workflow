# exec-tasks

Task orchestrator plugin for Claude Code. Reviews a GitHub project board, selects unblocked high-priority issues, spawns coding agents in isolated worktrees (parallel when safe), and opens PRs that trigger automated `@claude` review.

## What it does

- Determines the current repo from `git remote`
- Lists open issues, open PRs, and recently closed issues via `gh`
- Filters to the earliest incomplete milestone, skips blocked tasks, sorts by priority (P0 > P1 > P2 > P3)
- Identifies parallelizable tasks (no overlapping files, no mutual dependencies)
- Spawns coding agents in isolated worktrees with a full context package (CLAUDE.md, ADRs, PRD, glossary, `.memory/plans/`)
- Creates PRs with `Closes #N` and posts a `@claude` review comment to trigger the review workflow

## Usage

```
/plugin marketplace add yuchida-tamu/claude-project-workflow
/plugin install exec-tasks@claude-project-workflow
```

Then invoke with `/exec-tasks` from inside any repo that conforms to the workflow contract.

## Workflow contract

This skill only works on repositories that conform to [`docs/WORKFLOW_CONTRACT.md`](../docs/WORKFLOW_CONTRACT.md) (contract version 2). The contract specifies the issue body template, label vocabulary (`P0`–`P3`, `type:...`), milestone naming (`M1`, `M2`, …), dependency syntax (`Depends on: #N`), and required project documentation paths.

Projects scaffolded by `init-project` conform to the contract automatically. For existing repos, either migrate them to the contract format or use a different tool — `exec-tasks` will stop and report if it encounters non-conforming issues rather than guessing.

## Pairing with init-project

```
/init-project my-new-app          # scaffold
/exec-tasks                        # start executing M1 issues
```

The two plugins share the workflow contract and are designed to be used together.

---

# exec-tasks (日本語)

Claude Code 用のタスクオーケストレータープラグイン。GitHub プロジェクトボードをレビューし、ブロックされていない優先イシューを選択し、分離ワークツリーでコーディングエージェントを（安全な場合は並列で）起動し、`@claude` 自動レビューをトリガーする PR を開きます。

## 機能

- `git remote` から現在のリポジトリを特定
- `gh` でオープンイシュー・オープン PR・最近クローズされたイシューを取得
- 最も早い未完了マイルストーンにフィルタリングし、ブロックされたタスクをスキップし、優先度 (P0 > P1 > P2 > P3) でソート
- 並列実行可能なタスク（ファイル重複なし・相互依存なし）を特定
- 完全なコンテキストパッケージ（CLAUDE.md・ADR・PRD・用語集・`.memory/plans/`）を持たせてエージェントを分離ワークツリーで起動
- `Closes #N` を含む PR を作成し、`@claude` レビューコメントを投稿してレビューワークフローを起動

## 使い方

```
/plugin marketplace add yuchida-tamu/claude-project-workflow
/plugin install exec-tasks@claude-project-workflow
```

その後、ワークフロー契約に準拠するリポジトリ内で `/exec-tasks` を実行します。

## ワークフロー契約

このスキルは [`docs/WORKFLOW_CONTRACT.md`](../docs/WORKFLOW_CONTRACT.md)（契約バージョン 1）に準拠するリポジトリでのみ動作します。契約はイシュー本文テンプレート、ラベル語彙 (`P0`–`P3`、`type:...`)、マイルストーン命名規則 (`M1`、`M2`、…)、依存関係の構文 (`Depends on: #N`)、プロジェクトドキュメントの必須パスを規定しています。

`init-project` でスキャフォールドされたプロジェクトは自動的に契約に準拠します。既存リポジトリの場合は契約形式に移行するか、別のツールを使用してください — `exec-tasks` は契約に準拠しないイシューを検出した場合、推測せずに停止して報告します。

## init-project との組み合わせ

```
/init-project my-new-app          # スキャフォールド
/exec-tasks                        # M1 イシューの実行を開始
```

2 つのプラグインはワークフロー契約を共有しており、一緒に使うことを想定しています。
