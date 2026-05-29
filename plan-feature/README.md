# plan-feature

The **Plan** phase of the design → plan → execute → review workflow. Design a feature before implementation, produce a structured plan with mermaid diagrams, and convert the plan into GitHub issues that `exec-tasks` can execute.

## Installation

```
/plugin install plan-feature@claude-project-workflow
```

## Commands

### `/plan-feature <feature-name>`

Design a feature end-to-end:

1. **Complexity assessment** — routes underspecified features to `/grill-me`, suggests breakdown for oversized ones
2. **Research** — reads `.memory/`, `docs/PRD.md`, `docs/glossary.md`, `docs/adr/`, `CLAUDE.md`, and the codebase
3. **Plan creation** — writes `.memory/plans/{feature-name}.md` with user flow, system flow, data model, implementation phases, and file manifest
4. **ADR extraction** — promotes architectural decisions from the plan to `docs/adr/`
5. **User approval** — presents the plan summary and waits for your sign-off
6. **Issue creation** — converts each implementation phase into a GitHub issue conforming to the workflow contract, with proper labels, milestone, and `Depends on` links
7. **Close-out** (called after all issues are merged) — runs `/doc-feature` to capture the actual implementation in `docs/features/`

### `/doc-feature <feature-name>`

Generate reference documentation from implemented code. Reads the actual codebase (not the plan) and produces `docs/features/{feature-name}.md` with:

- Mermaid user-flow and system-flow diagrams
- Data model (key types only — linked, not duplicated)
- State & side effects
- Key files table
- Design decisions (non-obvious choices and their rationale)

Can be called standalone to document existing features or after significant refactoring.

## Typical flow

```
/plan-feature user-authentication    # design → plan → issues created
/exec-tasks                          # pick up the new issues, implement, open PRs
# ... PRs reviewed and merged ...
/doc-feature user-authentication     # generate docs/features/user-authentication.md
```

## How it fits in the workflow

| Phase | Skill | What it produces |
|---|---|---|
| Design | `init-project` | Repo scaffold, PRD, ADRs, M1 seed issues |
| **Plan** | **`plan-feature`** | **`.memory/plans/*.md`, ADRs, GitHub issues** |
| Execute | `exec-tasks` | Implemented PRs |
| Review | `review-impl` *(planned)* | Retrospective, updated docs |

`plan-feature` sits between `init-project` and `exec-tasks`. It turns a feature idea into a set of well-specified GitHub issues that `exec-tasks` can pick up immediately.

## Workflow contract

Issues created by `plan-feature` conform to [workflow contract v3](../docs/WORKFLOW_CONTRACT.md):

- Body template: `## Summary` / `## Acceptance Criteria` / `## Priority` / `## Depends on`
- Priority labels: `P0`–`P3`
- Type labels: `type:feature`, `type:bug`, `type:chore`, `type:docs`
- Milestone: assigned from existing milestones
- Dependencies: `#N` tokens in the `Depends on` section

---

# plan-feature（日本語）

**design → plan → execute → review** ワークフローの **Plan** フェーズです。実装前に機能を設計し、ミームダイアグラム付きの構造化されたプランを作成して、`exec-tasks` が実行できる GitHub イシューに変換します。

## インストール

```
/plugin install plan-feature@claude-project-workflow
```

## コマンド

### `/plan-feature <feature-name>`

機能をエンドツーエンドで設計します：

1. **複雑さの評価** — 仕様が不明確な機能を `/grill-me` に誘導し、範囲が広すぎる場合は分割を提案
2. **リサーチ** — `.memory/`、`docs/PRD.md`、`docs/glossary.md`、`docs/adr/`、`CLAUDE.md`、コードベースを読み込み
3. **プラン作成** — ユーザーフロー、システムフロー、データモデル、実装フェーズ、ファイルマニフェストを含む `.memory/plans/{feature-name}.md` を作成
4. **ADR 抽出** — アーキテクチャ上の決定をプランから `docs/adr/` に昇格
5. **ユーザー承認** — プランのサマリーを提示し、承認を待つ
6. **イシュー作成** — 各実装フェーズをワークフロー契約に準拠した GitHub イシューに変換（ラベル、マイルストーン、`Depends on` リンク付き）
7. **クローズアウト**（全イシューのマージ後）— `/doc-feature` を実行して実際の実装を `docs/features/` にドキュメント化

### `/doc-feature <feature-name>`

実装済みコードから参照ドキュメントを生成します。実際のコードベース（プランではなく）を読み取り、`docs/features/{feature-name}.md` を作成します。既存機能のドキュメント化やリファクタリング後にスタンドアロンで呼び出すことも可能です。

## 典型的なフロー

```
/plan-feature user-authentication    # 設計 → プラン → イシュー作成
/exec-tasks                          # 新しいイシューを拾い、実装し、PR を開く
# ... PR がレビューされてマージされる ...
/doc-feature user-authentication     # docs/features/user-authentication.md を生成
```
