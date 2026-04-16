# init-project

Project scaffolder plugin for Claude Code. Interviews the user about a new project, then writes a contract-conforming local scaffold (CLAUDE.md, PRD, ADRs, glossary, `.memory/`, GitHub Actions, post-edit hooks) and creates the GitHub repo with labels, milestones, and seeded M1 issues. The output is immediately consumable by `exec-tasks`.

## What it does

- Runs a 13-question interview (one question at a time) covering pitch, platform, tech stack, core loop, non-goals, risks, milestones, M1 issues, ADR-0002 constraints, glossary terms, repo conventions, and GitHub org/visibility
- Offers a `grill-me` escape hatch on underspecified answers
- Shows a preview gate (file list + key excerpts + GitHub operations) and waits for `y/N`
- Writes `CLAUDE.md` with a verbatim Principles spine (Think Before Coding, Simplicity First, Surgical Changes, Tidy First, Goal-Driven Execution) plus generated Architecture / Tech Stack / Commands / Conventions / Workflow sections
- Writes `docs/PRD.md`, `docs/glossary.md`, `docs/adr/0001-record-architecture-decisions-in-adrs.md`, and optionally `docs/adr/0002-<slug>.md`
- Seeds `.memory/plans/.gitkeep`, `.github/workflows/claude.yml`, `.github/workflows/ci.yml`, `.claude/settings.json` (post-edit hook), `.gitignore`, `LICENSE`, and `README.md`
- `git init`, initial commit, `gh repo create`, label + milestone creation (in parallel), and 3–5 seeded M1 issues that conform to the workflow contract
- Phased execution with a `.init-project-state.json` marker — any failure preserves state and can be continued with `--resume`

## Usage

```
/plugin marketplace add yuchida-tamu/claude-project-workflow
/plugin install init-project@claude-project-workflow
```

Then invoke:

```
/init-project                    # scaffold the current (empty-ish) directory
/init-project my-new-app         # create ./my-new-app/ and scaffold there
/init-project my-app --dry-run   # print the plan without writing anything
/init-project my-app --skip-github   # local files + git init only, no GitHub
/init-project --resume           # continue an interrupted scaffold
```

### Flags

- `--skip-github` — stop after local files + git init + initial commit
- `--no-hook` — don't install the `.claude/settings.json` PostToolUse hook
- `--dry-run` — print the full plan without executing
- `--resume` — read `.init-project-state.json` and continue from the first incomplete phase
- `--name=<name>` — with `--resume`, override the target repo name (e.g., after a name collision)
- `--license=mit|apache-2.0|none` — default `mit`

## Workflow contract

Every file and issue produced by this skill conforms to [`docs/WORKFLOW_CONTRACT.md`](../docs/WORKFLOW_CONTRACT.md) (contract version 2). Issue bodies follow the fixed `## Summary` / `## Acceptance Criteria` / `## Priority` / `## Depends on` template, labels are exactly `P0`–`P3` and `type:feature|bug|chore|docs`, and milestones are named `M1`, `M2`, `M3`. `exec-tasks` reads the same contract, so the two plugins hand off cleanly.

## Pairing with exec-tasks

```
/init-project my-new-app          # scaffold
/exec-tasks                        # start executing M1 issues
```

After scaffolding, the skill prints a `gh secret set ANTHROPIC_API_KEY` command — run it once to enable `@claude` PR review via the installed `claude.yml` workflow.

---

# init-project (日本語)

Claude Code 用のプロジェクトスキャフォールドプラグイン。新規プロジェクトについてユーザーにインタビューし、ワークフロー契約に準拠したローカルスキャフォールド（CLAUDE.md、PRD、ADR、用語集、`.memory/`、GitHub Actions、post-edit フック）を書き出した上で、ラベル・マイルストーン・M1 イシュー付きで GitHub リポジトリを作成します。出力はそのまま `exec-tasks` で消費可能です。

## 機能

- 13 個の質問（1 つずつ）でインタビュー: ピッチ、プラットフォーム、技術スタック、コアループ、非目標、リスク、マイルストーン、M1 イシュー、ADR-0002 制約、用語、リポジトリ規約、GitHub org/可視性
- 回答が曖昧な場合、`grill-me` エスケープハッチを提示
- プレビューゲート（ファイル一覧・主要抜粋・GitHub 操作）を表示し `y/N` で確認
- `CLAUDE.md` を生成（Principles セクション — Think Before Coding / Simplicity First / Surgical Changes / Tidy First / Goal-Driven Execution — は逐語コピー、Architecture / Tech Stack / Commands / Conventions / Workflow は生成）
- `docs/PRD.md`、`docs/glossary.md`、`docs/adr/0001-record-architecture-decisions-in-adrs.md`、必要に応じて `docs/adr/0002-<slug>.md` を生成
- `.memory/plans/.gitkeep`、`.github/workflows/claude.yml`、`.github/workflows/ci.yml`、`.claude/settings.json`（post-edit フック）、`.gitignore`、`LICENSE`、`README.md` をシード
- `git init`、初期コミット、`gh repo create`、ラベル + マイルストーン作成（並列）、契約に準拠した 3〜5 個の M1 イシュー作成
- フェーズ実行と `.init-project-state.json` マーカー — 失敗時は状態を保持し `--resume` で再開可能

## 使い方

```
/plugin marketplace add yuchida-tamu/claude-project-workflow
/plugin install init-project@claude-project-workflow
```

起動:

```
/init-project                    # カレントディレクトリ（空に近い）をスキャフォールド
/init-project my-new-app         # ./my-new-app/ を作成してスキャフォールド
/init-project my-app --dry-run   # 書き込まずに計画を表示
/init-project my-app --skip-github   # ローカル + git init のみ、GitHub 操作なし
/init-project --resume           # 中断したスキャフォールドを再開
```

### フラグ

- `--skip-github` — ローカルファイル + git init + 初期コミットで停止
- `--no-hook` — `.claude/settings.json` の PostToolUse フックをインストールしない
- `--dry-run` — 実行せず計画のみ表示
- `--resume` — `.init-project-state.json` を読み、最初の未完了フェーズから再開
- `--name=<name>` — `--resume` と併用で、リポジトリ名を上書き（名前衝突後など）
- `--license=mit|apache-2.0|none` — デフォルトは `mit`

## ワークフロー契約

このスキルが生成するすべてのファイルとイシューは [`docs/WORKFLOW_CONTRACT.md`](../docs/WORKFLOW_CONTRACT.md)（契約バージョン 1）に準拠します。イシュー本文は固定の `## Summary` / `## Acceptance Criteria` / `## Priority` / `## Depends on` テンプレートに従い、ラベルは `P0`〜`P3` と `type:feature|bug|chore|docs` のみ、マイルストーンは `M1`、`M2`、`M3` という命名です。`exec-tasks` は同じ契約を読み取るため、2 つのプラグインはクリーンに連携します。

## exec-tasks との組み合わせ

```
/init-project my-new-app          # スキャフォールド
/exec-tasks                        # M1 イシューの実行を開始
```

スキャフォールド完了時に `gh secret set ANTHROPIC_API_KEY` コマンドが表示されます。一度実行すれば、インストール済みの `claude.yml` ワークフロー経由で `@claude` PR レビューが有効になります。
