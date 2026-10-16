# kdskills / startproject 実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Claude Code プラグイン `kdskills` を作り、最初のスキル `/kdskills:startproject <name>` でプロジェクトの土台と `プロジェクト概要書.md` を作れるようにする。

**Architecture:** `~/kdskills` は自分自身をマーケットプレイスとして登録する単一プラグインのリポジトリ。スキルは `skills/startproject/SKILL.md`(日本語の手順書)と `templates/`(固定出力)で構成し、手順書はテンプレートをコピーして埋める。コードは書かない。成果物はすべて Markdown と JSON。

**Tech Stack:** Claude Code plugin(`.claude-plugin/plugin.json`、`marketplace.json`)、Skills(`SKILL.md` frontmatter)、git、`claude plugin validate`

**Spec:** `docs/superpowers/specs/2026-10-16-startproject-design.md`

## Global Constraints

- スキル名は英小文字ハイフン区切り。呼び出し名は `/kdskills:startproject`
- 生成物は日本語名(`プロジェクト概要書.md`)。仕組み側(フォルダ、JSON のキー)は英語
- テンプレートは `templates/` に置き、SKILL.md 本文に埋め込まない
- 状態語は `決定` / `検討中` / `未着手` の 3 つのみ。各項目の見出し直下に `**状態**: <語>` の 1 行
- 概要書の項目は `## 1.` 〜 `## 11.` の 11 個。状態を持つのは 1〜9
- 日付は `date +%Y-%m-%d` で取得。推測で書かない
- `git init -b main`。初回コミットメッセージは `init`、概要書更新は `docs: 概要書を更新`
- スキルの自動テストは作らない。検証は `claude plugin validate` と実走

## Review Focus

1. `<name>` に日本語やスペースが含まれる(例 `/kdskills:startproject 揚重 予約`)。止まって理由を示す。フォルダを作らない → Task 3 の手順書に明記、Task 4 で実走確認
2. `<name>` のフォルダは存在するが概要書が無い(無関係のフォルダ)。「kdskills の概要書ではないようです」と止め、何も書かない → Task 3、Task 4
3. 概要書を手で編集して `**状態**:` の語が 3 語以外になっている(例 `済`)。その項目を `検討中` 扱いで聞き直し、勝手に書き換えない → Task 3 に明記
4. `git` の user.name / user.email が未設定でコミットに失敗する。概要書は残り、設定コマンドを示して止まる → Task 3 の失敗時表
5. 現在のディレクトリが既に Git リポジトリの中(入れ子になる)。止めずに進めるが、入れ子になる旨を 1 行で伝える → Task 3 に明記

---

### Task 1: プラグインの骨格とマニフェスト

**Files:**
- Create: `.claude-plugin/plugin.json`
- Create: `.claude-plugin/marketplace.json`
- Create: `.gitignore`
- Create: `README.md`(骨格のみ。インストール手順は Task 4 で確定させる)

**Interfaces:**
- Produces: プラグイン名 `kdskills`、マーケットプレイス名 `kdskills`。以降のタスクは `claude plugin validate ~/kdskills` が通ることを前提にする

- [ ] **Step 1: 検証コマンドが失敗することを確認する**

Run: `claude plugin validate ~/kdskills`
Expected: マニフェストが無い旨のエラー(終了コード 0 以外)

- [ ] **Step 2: `.claude-plugin/plugin.json` を書く**

```json
{
  "name": "kdskills",
  "version": "0.1.0",
  "description": "AI-DLC 独自版。プロジェクト開始から要件定義・DB設計・画面設計までの工程をスキルとして束ねる",
  "author": { "name": "ichio kon" },
  "keywords": ["ai-dlc", "project-kickoff", "requirements", "japanese"]
}
```

- [ ] **Step 3: `.claude-plugin/marketplace.json` を書く**

```json
{
  "name": "kdskills",
  "description": "kdskills を配布する自前のマーケットプレイス",
  "owner": { "name": "ichio kon" },
  "plugins": [
    {
      "name": "kdskills",
      "description": "AI-DLC 独自版の工程スキル集",
      "source": "./"
    }
  ]
}
```

`source` は「このリポジトリ自身」を指す相対パス。validate で拒否されたら `{ "source": "local", "path": "./" }` 形式に切り替える(どちらが通ったかを README に書く)。

- [ ] **Step 4: `.gitignore` と `README.md` の骨格を書く**

`.gitignore` はリポジトリ自身用(`.DS_Store`、`.idea/`、`.vscode/`、`*.swp`、`.claude/settings.local.json`)。
`README.md` は見出しのみ: `# kdskills`、`## 何をするか`(spec 1 節を 3 行に要約)、`## スキル一覧`(表: スキル / 役割。`startproject` の 1 行)、`## インストール`(「Task 4 で確定」と書かず、暫定の手順を書いておく)、`## 開発`(`claude --plugin-dir ~/kdskills`)。

- [ ] **Step 5: 検証が通ることを確認する**

Run: `claude plugin validate ~/kdskills`
Expected: エラーなし。警告があれば内容を読み、スキーマ上の不備なら直す

- [ ] **Step 6: Commit**

```bash
git add .claude-plugin .gitignore README.md
git commit -m "feat: kdskills プラグインの骨格を追加"
```

---

### Task 2: テンプレート(概要書と .gitignore)

**Files:**
- Create: `skills/startproject/templates/プロジェクト概要書.md`
- Create: `skills/startproject/templates/gitignore`

**Interfaces:**
- Produces: 概要書テンプレート。`<name>` と `YYYY-MM-DD` の 2 種類のプレースホルダだけを持つ。Task 3 の手順書はこの 2 つを置換してコピーする
- Produces: `gitignore`(ドットなし)。コピー先で `.gitignore` に改名する

- [ ] **Step 1: 概要書テンプレートを spec 5 節のとおりに書く**

spec 5 節のコードブロックを **そのまま** ファイルにする。先頭行は `# <name> プロジェクト概要書`、表の日付は `YYYY-MM-DD` のまま。

- [ ] **Step 2: 構造を grep で検証する**

Run:
```bash
T=skills/startproject/templates/プロジェクト概要書.md
grep -c '^## [0-9]*\. ' "$T"            # 期待: 11
grep -c '^\*\*状態\*\*: 未着手$' "$T"   # 期待: 9
grep -c '| 未着手 |' "$T"               # 期待: 5(技術スタックの 5 行)
grep -c '<name>' "$T"                   # 期待: 1
grep -c 'YYYY-MM-DD' "$T"               # 期待: 2
grep -c '<!--' "$T"                     # 期待: 0
```
Expected: コメントの数字と一致する。ずれていればテンプレートを直す

- [ ] **Step 3: `gitignore` テンプレートを spec 6 節のとおりに書く**

- [ ] **Step 4: Commit**

```bash
git add skills/startproject/templates
git commit -m "feat: startproject のテンプレートを追加"
```

---

### Task 3: `startproject` の手順書(SKILL.md)

**Files:**
- Create: `skills/startproject/SKILL.md`

**Interfaces:**
- Consumes: Task 2 のテンプレート 2 つ(`templates/プロジェクト概要書.md`、`templates/gitignore`)
- Produces: `/kdskills:startproject <name>` の振る舞い。spec 4 節が仕様

- [ ] **Step 1: frontmatter を書く**

```yaml
---
name: startproject
description: >
  プロジェクトの土台を作り、プロジェクト概要書を対話で埋める。
  `/kdskills:startproject <name>` で <name>/ フォルダ、docs/、プロジェクト概要書.md を作り、
  main ブランチで git init と初回コミットを行う。既存の <name>/ に対して実行すると、
  概要書の「検討中」「未着手」の項目から再開する。トリガー: 「プロジェクトを始めたい」
  「新しいプロジェクト」「概要書を書きたい」「/startproject」。
---
```

- [ ] **Step 2: 本文を spec 4 節の構成で書く**

見出しと、各見出しに必ず入れる決定事項:

1. `## 目的`: 2 行。土台作りと概要書の明文化。それ以外はしない
2. `## 引数の確認`: 未指定なら聞いて止まる。許可文字は `^[A-Za-z0-9_-]+$`。違反時は「英数字、ハイフン、アンダースコアのみ使えます」と示して止まる。現在のディレクトリが Git リポジトリ内なら「入れ子のリポジトリになります」と 1 行伝えて続行
3. `## モード判定`: `./<name>` の有無で新規 / 再開
4. `## 新規モード`: spec 4.2 の 3 手順。コピー元はこの SKILL.md と同じディレクトリの `templates/`(スキル起動時に示される Base directory を使う)。`<name>` と `YYYY-MM-DD`(`date +%Y-%m-%d` の値)を置換。`docs/.gitkeep`、`.gitignore`(改名)。`git init -b main && git add -A && git commit -m init`
5. `## 再開モード`: spec 4.3。概要書の有無と `## 1.` 〜 `## 11.` の見出しが揃っているかを確認。揃っていなければ「kdskills の概要書ではないようです」と止める。`**状態**:` の語が 3 語以外の項目は `検討中` として扱い、本文は書き換えずに聞き直す
6. `## 埋め込みフェーズ`: spec 4.4 の 7 手順をそのまま。ブレインダンプの問い文は spec の文言を使う。1 メッセージ 1 項目。答えの 3 分類(決定 / 検討中 / 飛ばす)。`検討中` には「候補:」「判断軸:」の 2 行。1 項目ごとに書き込む。「ここまでで終わり」で終了処理へ
7. `## 終了処理`: 項目 10 の表を `検討中`・`未着手` で書き直す。項目 11 が空なら 1〜3 個提案して確認後に書く。先頭の表(最終更新、件数)を更新。項目 7 の状態は全行 `決定` なら `決定`、そうでなければ `未着手` > `検討中` の重い方。`git add プロジェクト概要書.md && git commit -m "docs: 概要書を更新"`
8. `## やらないこと`: spec 4.5 の 4 行
9. `## 失敗時`: spec 4.6 の表をそのまま。identity 未設定時は `git config --global user.name` と `user.email` の設定例を示す

各手順は「何をするか」だけを書く。説明や背景は書かない。既存の自作スキル `notion-log` の密度を目安にする(300 行を超えたら削る)。

- [ ] **Step 3: 検証が通ることを確認する**

Run: `claude plugin validate ~/kdskills`
Expected: エラーなし。skills の frontmatter が認識されている

- [ ] **Step 4: Commit**

```bash
git add skills/startproject/SKILL.md
git commit -m "feat: startproject スキルの手順書を追加"
```

---

### Task 4: 実走確認とインストール手順の確定

**Files:**
- Modify: `README.md`(`## インストール` を実際に通った手順に書き換える)

**Interfaces:**
- Consumes: Task 1〜3 のすべて

- [ ] **Step 1: 作業用ディレクトリを用意する**

Run: `mkdir -p /tmp/kdskills-try && cd /tmp/kdskills-try && git status 2>&1 | head -1`
Expected: `fatal: not a git repository`(入れ子にならない場所であること)

- [ ] **Step 2: 新規モードを実走する**

Run(別端末で): `cd /tmp/kdskills-try && claude --plugin-dir ~/kdskills`
プロンプト: `/kdskills:startproject sample` → ブレインダンプに 3〜4 行答える → 項目 1 と 2 に答える → 項目 3 を「後で」 → 「ここまでで終わり」

- [ ] **Step 3: 新規モードの結果を検証する**

Run:
```bash
cd /tmp/kdskills-try/sample
ls -a                                   # 期待: .git .gitignore docs プロジェクト概要書.md
ls -a docs                              # 期待: .gitkeep
git branch --show-current               # 期待: main
git log --oneline                       # 期待: 2 行。下が init、上が docs: 概要書を更新
grep -c '^\*\*状態\*\*: 決定$' プロジェクト概要書.md     # 期待: 2
grep -c '^\*\*状態\*\*: 未着手$' プロジェクト概要書.md   # 期待: 7
head -5 プロジェクト概要書.md           # 期待: 1 行目が「# sample プロジェクト概要書」、表の日付が今日、件数が 2 / 0 / 7
```
Expected: すべて一致。ずれていれば SKILL.md の該当手順を直し、`rm -rf /tmp/kdskills-try/sample` してやり直す

- [ ] **Step 4: 再開モードを実走する**

プロンプト: `/kdskills:startproject sample` → 状態一覧が出て、ブレインダンプの質問が**出ない**ことを確認 → 項目 3 に答える → 「ここまでで終わり」
Run: `cd /tmp/kdskills-try/sample && git log --oneline | wc -l && grep -c '^\*\*状態\*\*: 決定$' プロジェクト概要書.md`
Expected: `3` と `3`

- [ ] **Step 5: 失敗系を実走する(Review Focus 1、2)**

プロンプト: `/kdskills:startproject 揚重 予約` → 止まり、フォルダができない(`ls /tmp/kdskills-try` に変化なし)
Run: `mkdir /tmp/kdskills-try/plain` → プロンプト: `/kdskills:startproject plain` → 「kdskills の概要書ではないようです」で止まり、`ls -a /tmp/kdskills-try/plain` が空のまま

- [ ] **Step 6: マーケットプレイス経由のインストールを通す**

Run:
```bash
claude plugin marketplace add ~/kdskills
claude plugin install kdskills@kdskills
claude plugin list 2>&1 | grep kdskills
```
Expected: 最後の行に `kdskills` が出る。コマンド名や引数が違えば `claude plugin --help` で正しい形を調べ、通った形を README に書く

- [ ] **Step 7: README の `## インストール` を通った手順に書き換えて Commit**

```bash
git add README.md
git commit -m "docs: インストール手順を実走結果で確定"
```

- [ ] **Step 8: 後片付け**

Run: `rm -rf /tmp/kdskills-try`
