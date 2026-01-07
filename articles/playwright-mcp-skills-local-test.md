---
title: "Playwright MCP × Markdown指示書で、ローカルテストが一気に速くなった"
emoji: "🧪"
type: "tech"
topics: ["playwright", "testing", "copilot", "mcp", "automation"]
published: true
---

# Playwright MCP × Markdownで、ローカルテストが一気に速くなった

この記事では、**ローカルでUI確認を回すことが多いフロントエンド担当向け**に、
毎回の長いプロンプト運用をやめて、Markdown の指示書に寄せて修正サイクルを短縮できた話をまとめます。
本当は Claude Code を使いたかったけれど、社内の都合で Copilot Chat しか使えない人向けです。

## TL;DR（先に結論）
- その回のテスト内容は Markdown の指示書にだけ書く
- 実行時にその md を指定して Playwright MCP を回す

## 前提
- Playwright MCP を使える環境
- テスト内容を Markdown にまとめる方針

<!-- TOC -->

---

## はじめに

Playwright MCP でローカルテストを回していると、UI の仕様変更が頻繁に入って、同じテストを何度も回すことになります。そのたびに、テスト内容をプロンプトへ貼り付け直すのが地味にしんどかったです。

そこで、
- その回のテスト内容は Markdown の指示書に書く
- 実行時はその md を指定して Playwright MCP を回す

という運用にしました。
結果、プロンプトがかなり短くなって、修正→確認のサイクルが体感で半分くらいに短縮されました。

---

## 全体構成（シンプル版）

### ディレクトリ構成

今回の構成は、できるだけ最小・役割分離にしています。

```text
frontend-repo/
├─ tests/specs/
│  ├─ _template.spec.md # 指示書(Markdown)のテンプレ
│  └─ login.spec.md     # 機能ごとの指示書（基準）
└─ playwright/README.md # 実行方法・環境・CI設定の入口
```

- WHAT（その回の指示書）：tests/specs/*.spec.md
- 実行方法：playwright/README.md

### 構成図

以下が、今回の構成を 1枚で表した図です。

```mermaid
flowchart LR
  Spec["tests/specs/*.spec.md<br/>（指示書・ここが基準）"] --> Code["Playwright テスト実装"]
  Code --> MCP["Playwright MCP<br/>（実行・再実行）"]
```

### この構成のポイント（ざっくり）

- 指示内容は Markdown にだけ書く
- 実行時にその md を MCP に投げる

---

## なぜ Markdown を指示書にしたのか

UI の仕様変更が多いと、同じテストを何度もやり直すことになります。
そのたびにプロンプトへ貼り付けるのが面倒で、地味にミスります。

そこで、テスト内容は Markdown の指示書にまとめて、
プロンプトには「このファイル指定で Playwright MCP 実行して」とだけ書く形にしました。
これだけで、毎回の指示コストがほぼゼロになります。
なお、この記事はローカル確認を前提にしています（本番運用やCIは対象外）。

---

## 指示書（Markdown）の例

```markdown
# Login Feature IT Test

## Preconditions
- ユーザーは既存アカウントを持っている

## Scenario 1: 正常ログイン
1. ログインページにアクセスする
2. 正しいメールアドレスを入力する
3. 正しいパスワードを入力する
4. ログインボタンを押す
5. ダッシュボードに遷移すること

## Expected Results
- ダッシュボードが表示される
- URL が /dashboard になる
```

この内容を `tests/specs/login.spec.md` に書きます。
Gherkin でなくても問題ありません。
自然文 + 箇条書き で十分でした。

---

## Playwright MCP × Copilot の流れ

1. Plan
   - spec(Markdown) を読み、テスト実装計画を立てる
2. Edit
   - spec を指定して MCP を実行し、結果に合わせて実装
3. Fix
   - 失敗理由を切り分け（selector / wait / 仕様ズレ）
   - 最小修正して再実行

このループがわりと安定して回ります。

---

## Before / After（プロンプトの差分）

### Before
毎回プロンプトにテスト内容を貼っていました。

```text
# Login Feature IT Test

## Preconditions
- ユーザーは既存アカウントを持っている

## Scenario 1: 正常ログイン
1. ログインページにアクセスする
2. 正しいメールアドレスを入力する
3. 正しいパスワードを入力する
4. ログインボタンを押す
5. ダッシュボードに遷移すること

## Expected Results
- ダッシュボードが表示される
- URL が /dashboard になる
```

### After
指示書は Markdown に置いて、プロンプトはこれだけにしました。

```text
`tests/specs/login.spec.md` を指定して Playwright MCP 実行して
```

---

## このやり方の良かった点

- プロンプトが短くなって回転が速い
- テスト内容はファイルで管理できる
- Copilot が暴走しにくい
- 修正は「仕様なのか実装なのか」で判断しやすい

---

## 注意点

- Markdown が曖昧だと、そのまま曖昧に実装される
- 「よしなに」は書かない
- UI 文言や遷移はなるべく明示する
- E2Eまでやる場合は、テストコードをきちんと書く（別途運用を分ける）

仕様を書く力が、そのままテスト品質に出ます。

---

## まとめ

- Playwright MCP は強力
- ただし、仕様が曖昧だと不安定
- Markdown を基準にすると安定する
