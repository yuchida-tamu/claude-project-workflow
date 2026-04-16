# claude-project-workflow

A Claude Code marketplace for the **design → plan → execute → review** project workflow. Install once, then move a project from "empty directory" to "working codebase with PRs merging" entirely through Claude Code slash commands.

## Installation

Add this marketplace to Claude Code:

```
/plugin marketplace add yuchida-tamu/claude-project-workflow
```

Then install the plugins you need:

```
/plugin install init-project@claude-project-workflow
/plugin install exec-tasks@claude-project-workflow
```

## The four-skill workflow

This marketplace is organized around four phases of project development. Each phase is a slash command that you run when you need it.

| Phase | Skill | Command | What it does |
|---|---|---|---|
| **Design** | `init-project` | `/init-project` | Interviews you about a new project, then scaffolds `CLAUDE.md`, `docs/PRD.md`, `docs/glossary.md`, ADRs, GitHub repo, labels, milestones, and seed issues — ready for `exec-tasks` on day one. |
| **Plan** | `plan-feature` | `/plan-feature` | *Planned.* Walks you through designing a feature before implementation — produces a phased plan in `.memory/plans/` and opens GitHub issues linked to the plan. |
| **Execute** | `exec-tasks` | `/exec-tasks` | Reads the GitHub project board, selects unblocked high-priority issues, spawns coding agents in isolated worktrees (parallel when safe), and opens PRs that trigger automated `@claude` review. |
| **Review** | `review-impl` | `/review-impl` | *Planned.* Reviews merged implementations against the original plan and PRD, produces a retrospective, and updates project docs. |

**Currently implemented:** `init-project`, `exec-tasks`.
**Currently planned (not yet built):** `plan-feature`, `review-impl`.

The two implemented skills share a **workflow contract** — a shared definition of how issues, labels, milestones, and project docs are structured — documented in [`docs/WORKFLOW_CONTRACT.md`](./docs/WORKFLOW_CONTRACT.md). The contract is the reason these skills live in the same repo: they evolve in lockstep.

## Plugins

### [init-project](./init-project/)

Interview-driven project scaffolder. Creates the file layout, git repo, GitHub repo, labels, milestones, and seed issues that `exec-tasks` expects — with a single command. Emphasizes the Tidy First principle (separate structural from behavioral changes) in every `CLAUDE.md` it generates.

```
/plugin install init-project@claude-project-workflow
```

### [exec-tasks](./exec-tasks/)

Task orchestrator. Reviews a GitHub project board, selects unblocked high-priority issues, spawns coding agents in isolated worktrees (parallel when safe), and opens PRs that trigger automated `@claude` code review.

```
/plugin install exec-tasks@claude-project-workflow
```

## Typical workflow

```
mkdir ~/Projects/my-new-app && cd ~/Projects/my-new-app
/init-project                  # interview + scaffold + GitHub repo + seeded M1 issues
/exec-tasks                    # pick up M1 issues, spawn agents, open PRs
# ... review and merge PRs ...
/exec-tasks                    # continue with next unblocked M1 issues
# ... M1 ships ...
/plan-feature                  # (when implemented) design next feature for M2
/exec-tasks                    # execute M2 issues
```

## Workflow contract

The shared contract between `init-project` and `exec-tasks` is documented in [`docs/WORKFLOW_CONTRACT.md`](./docs/WORKFLOW_CONTRACT.md). It specifies:

- Issue body template
- Label vocabulary (`P0`–`P3`, `type:feature|bug|chore|docs`)
- Milestone naming (`M1`, `M2`, …)
- Dependency syntax (`Depends on: #N`)
- Priority sort order
- Project documentation paths

Any change that affects the contract requires a version bump and synchronized updates to both plugins.

---

# claude-project-workflow (日本語)

**design → plan → execute → review** プロジェクトワークフローのための Claude Code マーケットプレイス。一度インストールするだけで、Claude Code のスラッシュコマンドから空のディレクトリを PR がマージされる稼働中のコードベースに変えることができます。

## インストール

Claude Code にマーケットプレイスを追加：

```
/plugin marketplace add yuchida-tamu/claude-project-workflow
```

必要なプラグインをインストール：

```
/plugin install init-project@claude-project-workflow
/plugin install exec-tasks@claude-project-workflow
```

## 4 スキルワークフロー

このマーケットプレイスはプロジェクト開発の 4 つのフェーズを中心に構成されています。各フェーズは必要なときに実行するスラッシュコマンドです。

| フェーズ | スキル | コマンド | 機能 |
|---|---|---|---|
| **設計** | `init-project` | `/init-project` | 新しいプロジェクトについてインタビューし、`CLAUDE.md`、`docs/PRD.md`、`docs/glossary.md`、ADR、GitHub リポジトリ、ラベル、マイルストーン、初期イシューを作成します — 初日から `exec-tasks` が使えます。 |
| **計画** | `plan-feature` | `/plan-feature` | *計画中。* 実装前に機能を設計するためのウォークスルーを提供します。`.memory/plans/` にフェーズ分けされた計画を作成し、計画にリンクした GitHub イシューを開きます。 |
| **実行** | `exec-tasks` | `/exec-tasks` | GitHub プロジェクトボードをレビューし、ブロックされていない優先イシューを選択し、分離ワークツリーでコーディングエージェントを（安全な場合は並列で）起動し、`@claude` 自動レビューをトリガーする PR を開きます。 |
| **レビュー** | `review-impl` | `/review-impl` | *計画中。* マージされた実装を元の計画と PRD と照らし合わせてレビューし、振り返りを作成してプロジェクトドキュメントを更新します。 |

**現在実装済み：** `init-project`, `exec-tasks`
**計画中（未実装）：** `plan-feature`, `review-impl`

実装済みの 2 スキルは **ワークフロー契約** を共有しています — イシュー、ラベル、マイルストーン、プロジェクトドキュメントの構造に関する共有定義で、[`docs/WORKFLOW_CONTRACT.md`](./docs/WORKFLOW_CONTRACT.md) にまとめられています。この契約こそが、これらのスキルを同じリポジトリに置いている理由です：ロックステップで進化させる必要があるためです。

## プラグイン

### [init-project](./init-project/)

インタビュー駆動のプロジェクトスキャフォルダ。`exec-tasks` が期待するファイルレイアウト、git リポジトリ、GitHub リポジトリ、ラベル、マイルストーン、初期イシューを単一のコマンドで作成します。生成するすべての `CLAUDE.md` で Tidy First 原則（構造的変更と振る舞いの変更を分離する）を強調します。

```
/plugin install init-project@claude-project-workflow
```

### [exec-tasks](./exec-tasks/)

タスクオーケストレーター。GitHub プロジェクトボードをレビューし、ブロックされていない優先イシューを選択し、分離ワークツリーでコーディングエージェントを（安全な場合は並列で）起動し、`@claude` 自動コードレビューをトリガーする PR を開きます。

```
/plugin install exec-tasks@claude-project-workflow
```

## 典型的なワークフロー

```
mkdir ~/Projects/my-new-app && cd ~/Projects/my-new-app
/init-project                  # インタビュー + スキャフォールド + GitHub リポジトリ + 初期 M1 イシュー
/exec-tasks                    # M1 イシューを拾い、エージェントを起動し、PR を開く
# ... PR をレビューしてマージ ...
/exec-tasks                    # 次のブロックされていない M1 イシューを続ける
# ... M1 リリース ...
/plan-feature                  # （実装されたら）M2 の次の機能を設計
/exec-tasks                    # M2 イシューを実行
```

## ワークフロー契約

`init-project` と `exec-tasks` の間の共有契約は [`docs/WORKFLOW_CONTRACT.md`](./docs/WORKFLOW_CONTRACT.md) に記載されています。以下を規定しています：

- イシュー本文テンプレート
- ラベル語彙（`P0`–`P3`, `type:feature|bug|chore|docs`）
- マイルストーン命名規則（`M1`, `M2`, …）
- 依存関係の構文（`Depends on: #N`）
- 優先度ソート順
- プロジェクトドキュメントのパス

契約に影響する変更はバージョンの更新と両プラグインの同期更新が必要です。
