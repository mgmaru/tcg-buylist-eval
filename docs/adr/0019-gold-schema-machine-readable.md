# 0019. 正解データの列の定義は、機械で読めるYAMLで書く

- ステータス：採用
- 日付：2026-10-28
- 決定者：kajiwara
- 関係するADR：[0016](0016-task-folder-per-task.md)（タスクごとのフォルダ）

## 背景

- 正解データ（`gold.csv`）は、モデルの成績を測る物差しである（設計書 10.2）。列の名前や型が間違っていると、採点そのものが狂う。
- 最初の案では、正解データの列の定義を `gold.md`（Markdown）に書くことにしていた。人には読みやすいが、機械では確かめられない。
- 出力のスキーマは、すでに JSON Schema（`output.schema.json`）で、機械で確かめられる形にしている。

## 検討した選択肢

| 選択肢 | 書き方 | 良い点 | 弱い点 |
|---|---|---|---|
| A. Markdown（最初の案） | `gold.md` に、表で列の意味を書く | 人が読みやすい | 機械で確かめられない。`gold.csv` と定義がずれても気づけない |
| B. 独自の形のYAML | 自分で決めた項目で、列の名前・型・必須かどうか・説明を書く | 機械で読める。自由に決められる | 項目の決め方を自分で考え、文書にする必要がある。ほかのツールでは読めない |
| **C. Table Schema に沿ったYAML** | CSVの列の定義の決まった書き方（Table Schema。Frictionless という仕組みの一部）に沿って書く | 機械で読める。決まった書き方なので、項目を自分で考えなくてよい。ほかのツールでも読める | Table Schema の書き方を覚える必要がある |

## 決定

**C にする。** 正解データの列の定義は、`tasks/<タスク名>/schemas/gold.schema.yaml` に、Table Schema に沿ったYAMLで書く。人向けの説明は、各列の `description` に書く。

```yaml
# tasks/buylist-card-id/schemas/gold.schema.yaml
fields:
  - name: row
    type: integer
    constraints: {required: true, minimum: 1}
    description: 段（上から1始まり）
  - name: col
    type: integer
    constraints: {required: true, minimum: 1}
    description: 列（左から1始まり）
  - name: name
    type: string
    constraints: {required: true}
    description: 画像に印刷されているカード名
  - name: price_jpy
    type: integer
    constraints: {required: true, minimum: 0}
    description: 買取価格
  - name: provisional_key
    type: string
    constraints: {required: true}
    description: 仮のキー（M0で使用。設計書 2.2）
  # ほかの列（設計書 5.2）も同じように書く
```

## 理由

- **物差しの誤りを、機械で見つけられるようにするため。** 正解データは採点の基準なので、列の名前や型の誤り、必須の値の抜けは、人の目ではなく機械で確かめたい。定義を機械で読める形にすれば、`gold.csv` を定義どおりか確かめる処理を、採点器やCIに組み込める。
- **出力のスキーマと扱いを揃えるため。** 出力はすでに JSON Schema で確かめる。正解データも同じように、定義に沿っているかを確かめられるようにする。
- **人向けの説明も、1つのファイルで済ませるため。** `description` に説明を書けば、Markdown を別に持つ必要がなく、定義と説明がずれない。A を選ばなかったのはこのため。
- **B ではなく C にしたのは、** CSVの列の定義には決まった書き方（Table Schema）がすでにあり、項目を自分で考える必要がないため。決まった書き方に沿っておけば、ほかのタスクでも同じ書き方を使い回せ、ほかのツールでも読める。

## 影響

- 設計書 4章・5.2 の `schemas/gold.md` を、`tasks/<タスク名>/schemas/gold.schema.yaml` に直す。
- TODO 7章の「`schemas/gold.md`：正解データの列定義を書く」を、この形に直す。
- 採点器は、`gold.csv` を読むときに、この定義に沿っているかを確かめる。沿っていなければ、採点器そのものの失敗として扱う（被験エージェントの成績にはしない）。
- Table Schema の書き方のうち、どこまでを使うか（型、必須、値の範囲など）は、作るときに決める。
