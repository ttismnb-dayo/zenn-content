---
title: "GitHub Copilot Premium Requests におけるトークン消費の仕組み"
emoji: "🔢"
type: "tech"
topics: ["githubcopilot", "premiumrequests", "openai", "token", "llm"]
published: false
---

# GitHub Copilot Premium Requests におけるトークン消費の仕組み

この記事は、**GitHub Copilot Premium Requests のトークン消費が、どのように発生するのか、その内部メカニズムを理解したい人**向けの記事です。

OpenAI および GitHub が公開している公式ドキュメントと、日常的な GitHub Copilot Chat 利用における実運用上の挙動を突き合わせて整理しました。

:::message alert
GitHub Copilot Chat の内部実装や詳細な仕様は公式にすべて公開されているわけではないため、一部は「公式情報から合理的に推測できる範囲」および「多くの利用者が観測している挙動」に基づいて記載しています。
:::

## TL;DR（先に結論）

- トークン消費は「入力トークン + 出力トークン」の合算
- 入力トークンには、ユーザーに見えない IDE / リポジトリ文脈が含まれる
- instructions / skills / AGENTS 系ファイルは、条件次第で参照されやすい
- Copilot Chat に永続的な常駐メモリは存在しない
- 設計と使い方次第で、トークン消費と精度は十分にコントロール可能

## 前提

- GitHub Copilot Chat を利用している
- Premium Requests の概念を理解したい
- 「なぜ短い入力でも多くのトークンが消費されるのか」を知りたい

---

## 1. トークン消費の基本構造

ChatGPT や GitHub Copilot Chat は、内部的には LLM（大規模言語モデル）をトークン単位で利用しており、OpenAI が公式に説明しているトークンの考え方と同じ枠組みで処理されているとされています。

トークン消費は、次の合算で決まります。

```
消費トークン = 入力トークン + 出力トークン
```

### トークンの処理単位（目安）

- トークン = AI の処理単位
- 文字・単語だけで構成されるとは限らない
- 記号・改行・スペースもトークンになり得る
- 日本語やコードは分割されやすく、トークン数が増えやすい
- 分割ルールはモデルや実装によって変わる

:::message
日本語はスペースで単語が区切られないため、モデルのトークナイザーがより細かい単位（文字やサブワード）で分割しやすい。結果として、同じ意味量でも英語よりトークン数が増えやすい傾向がある。
:::

:::details 日本語のトークン分割の例
例えば「この資料を要約してください。」は、概念的には以下のような単位に分割されやすい。

```
この / 資料 / を / 要約 / して / ください / 。
```

さらに細かく分割される場合もある。

```
こ / の / 資 / 料 / を / 要 / 約 / し / て / く / だ / さ / い / 。
```
:::

### 具体例（短文/英語/日本語/コードのトークン数比較・概算）

※ 実際のトークン数はモデルや実装で変わるため、あくまで目安。

| 種類 | 例 | ざっくりの傾向 |
|---|---|---|
| 短文（英語） | "Hello, world!" | 2〜4 トークン程度 |
| 英語（1文） | "Please summarize this document." | 5〜8 トークン程度 |
| 日本語（1文） | 「この資料を要約してください。」 | 10前後〜十数トークンになりやすい（目安） |
| コード（1行） | `const sum = (a, b) => a + b;` | 記号が多く、増えやすい |

### 入力トークンに含まれるもの

- ユーザーが入力したプロンプト
- チャット履歴（同一スレッド内の過去のやり取り）
- システム指示（UI には表示されない内部指示）
- IDE / リポジトリ文脈（Copilot Chat 特有）
- instructions / skills / AGENTS などの指示書（必要と判断された場合）

### 出力トークンに含まれるもの

- AI が生成した回答全文
- コード、説明文、コメント等すべて

---

## 2. なぜ「プロンプト入力時点」でトークン消費が増えるのか

Copilot Chat では、ユーザーが入力を送信したタイミングで、LLM へのリクエストが行われると考えられます。

その際、モデルに渡される情報は、単純に「今回の入力」だけではありません。

```
[system]
Copilot としての振る舞い・制約

[context]
- IDE の状態
- アクティブファイル
- 関連ファイルの抜粋
- README / 設定 / 指示書（必要と判断されたもの）

[user]
今回の入力
```

そのため、ユーザーが短い一文しか入力していなくても、背後では複数の文脈情報がまとめて送信され、入力トークンが増加する場合があります。

---

## 3. Copilot Chat が参照する IDE / リポジトリ文脈

GitHub の公式ドキュメントでは、Copilot が「relevant context from your workspace」を利用すると説明されています。

実運用上、参照対象になりやすいものとして、以下が挙げられます。

- アクティブなファイル
- 選択中のコード（選択されている場合は特に優先）
- import / require 先
- 同一ディレクトリ内の関連ファイル
- package.json / go.mod / pyproject.toml などの設定ファイル
- README.md

:::message alert
- リポジトリ全体を常に読むわけではありません
- 毎回「関連性が高いと推測された断片」が選ばれます
- 差分のみを送信するのではなく、文脈はリクエストごとに再構築されます
:::

---

## 4. instructions / skills / AGENTS 系ファイルの影響

### 4.1 ファイル名・内容による影響

GitHub Copilot では、

- instructions
- AGENTS
- skills

といった語を含むファイル名や、行動ルール・方針を列挙した Markdown 文書が、「指示書的な文脈」として扱われる可能性が高いと考えられます。

:::message
これらのファイル名や挙動は、公式仕様として明文化されているものではありません。
:::

### 4.2 トークン消費への影響（実運用上の傾向）

| 条件 | トークン消費への影響 |
|---|---|
| ルート直下に配置 | 高くなりやすい |
| ルール調（must / should 等） | 高くなりやすい |
| 設計・方針系の質問 | 参照されやすい |
| 巨大なファイル | 非常に高くなりやすい |

:::message alert
Copilot Chat に常駐メモリは存在しないため、これらの指示書は「一度読まれたら記憶される」のではなく、リクエストごとに参照されるかどうかが再判定されると考えられます。
:::

---

## 5. 参考：トークン消費を意識した設計

### 推奨される工夫

- instructions / skills は最小限の内容にする
- 詳細なルールは docs/ai/ などに分離する
- 必要な場合のみ、チャット内で明示的に参照させる

```
この質問では AGENTS.md / skills.md は参照せず、
src/api/user.ts のみを対象にしてください
```

### 避けたいパターン

- 巨大な instructions.md を常設する
- 曖昧な質問を繰り返し投げる

:::message alert
これらは、トークン消費の増加だけでなく、回答精度のブレにもつながりやすくなります。
:::

---

## まとめ：GitHub Copilot Chat のトークン消費の仕組み

### 核となる理解

1. **トークン消費の基本式**
   - トークン消費 = 入力トークン + 出力トークン
   - ユーザーが短い入力をしても、背後では大量の文脈がリクエストに含まれうる

2. **入力トークンに含まれるもの（ユーザーに見えない部分）**
   - システム指示（Copilot の振る舞い定義）
   - IDE の状態（アクティブファイル、選択中のコード）
   - リポジトリ文脈（package.json、README.md、関連ファイルの断片）
   - instructions / skills / AGENTS 等の指示書（参照と判定された場合）

3. **文脈の再構築**
   - Copilot Chat に永続的なメモリは存在しない
   - チャット履歴は保持される が、IDE 文脈はリクエストごとに再判定される
   - 同じ質問をしても、IDE の状態が異なれば、入力トークンが変わる可能性がある

4. **instructions / skills ファイルの参照判定**
   - これらのファイルが常に参照されるわけではない
   - ファイルのサイズ、ファイル名、質問の種類などに基づいて、参照するかどうかが判定される
   - 判定ロジックは公開されておらず、実運用上の観測に基づいている

---

## 参考

- [What are tokens and how to count them - OpenAI](https://help.openai.com/en/articles/4936856-what-are-tokens-and-how-to-count-them)
- [GitHub Copilot Premium Requests - GitHub Docs](https://docs.github.com/ja/billing/concepts/product-billing/github-copilot-premium-requests)
- [Copilot requests and billing - GitHub Docs](https://docs.github.com/en/copilot/concepts/billing/copilot-requests)
- [GitHub Models billing - GitHub Docs](https://docs.github.com/ja/billing/concepts/product-billing/github-models)
