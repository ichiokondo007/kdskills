# kdskills

## 何をするか

プロダクト開発の工程(プロジェクト開始 → 要件定義 → DB 設計 → 画面設計 → …)を Claude Code のスキルとして束ねた、AI-DLC(AI-Driven Development Lifecycle)の独自版。
1 プロジェクト = 1 リポジトリで、要件定義・DDL・画面設計などの成果物をそのリポジトリに蓄積する。
各工程は `/kdskills:<スキル名>` で呼び出す。

## スキル一覧

| スキル | 役割 |
|---|---|
| `startproject` | `<name>/` を作り、`docs/` と `プロジェクト概要書.md` を用意して main ブランチで初回コミット。概要書を対話で埋め、未決の項目から再開できる |

## インストール

```bash
git clone <このリポジトリ> ~/kdskills
claude plugin marketplace add ~/kdskills
claude plugin install kdskills@kdskills
```

## 開発

push 前でも、このリポジトリを直接読み込んで試せる。

```bash
claude --plugin-dir ~/kdskills
```

設計書は `docs/superpowers/specs/`、実装計画は `docs/superpowers/plans/` にある。
