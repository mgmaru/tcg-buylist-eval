# 0047. 出力のカードを、中身の名前で reading（読んだもの）と answer（答え）に分ける

- ステータス：採用
- 日付：2026-10-31
- 決定者：kajiwara
- 関係するADR：[0030](0030-task-spec-in-yaml.md)（タスクの仕様と伝え方）、[0039](0039-deliverables-and-intermediates.md)（成果物と中間結果）、[0040](0040-intermediate-results-for-diagnosis.md)（段階ごとの中間結果）、[0049](0049-subject-gets-answering-instructions-only.md)（被験エージェントに採点の仕組みを伝えない）

## 背景

- 成績は成果物（何のカードか・価格）で決め、中間結果（印刷名・HP・ワザ・価格の文字など）は診断に使う（[0039](0039-deliverables-and-intermediates.md)）。
- それまでの出力では、カードごとに次のように項目が並んでいた。

```
position, printed（name, hp, attacks, price_text）, price_jpy, identification（status, card, candidates, confidence）
```

- この形では、成果物が `price_jpy` と `identification.card` の2か所に分かれ、中間結果の `printed` と並んでいた。どの項目が読んだもので、どれが答えなのかが、プロパティからは読み取りにくかった。実際、レビューの中でも取り違えが起きた。
- 出力のスキーマは、モデルと採点器のほか、人も読む。人は、ハーネスを設計・見直すときにスキーマを読み、診断で気になったカードについて出力のJSONそのものを読む。

## 検討した選択肢

| 選択肢 | 良い点 | 弱い点 |
|---|---|---|
| A. 今の形のまま | 変更がいらない | どの項目が答えかが、プロパティから読み取りにくい |
| B. 採点での役割の名前で、`intermediate`（中間結果）と `final`（最終結果）に分ける | 成績に使う項目が、名前でそのままわかる | 名前が採点の仕組みをモデルに教える（[0049](0049-subject-gets-answering-instructions-only.md) に反する）。分類が出力の形に固定され、分類を変えると項目を移すことになる。候補・確信度が `status` から離れる。段階を区別しない寄せ集めになる |
| **C. 中身の名前で、`reading`（画像から読んだもの）と `answer`（答え）に分ける** | 読んだものと答えが、プロパティで分かれる。名前は中身を表し、採点の仕組みを表さない。候補・確信度は答えの一部として `status` のそばに置ける | 入れ子が1段増える。分類は、まだある程度出力の形に表れる |

## 決定

**C にする。** カードごとの出力を、次の形にする。

```json
{
  "position": {"row": 4, "col": 1},
  "reading": {"name": "ブラッキー", "hp": 90, "attacks": ["つきかげのキバ", "クイックアタック"], "price_text": "17万円"},
  "answer": {
    "status": "identified",
    "card": {"pack": "よみがえる伝説", "number": "037/080",
             "variant": {"edition": "指定なし", "finish": "指定なし", "stamp": "なし"}},
    "price_jpy": 170000,
    "candidates": [],
    "confidence": 0.9
  }
}
```

| まとまり | 中身 |
|---|---|
| `position` | 段・列。出力と正解を対応付ける目印 |
| `reading` | 画像に書かれている通りに読んだもの：カード名、HP、ワザ、価格の文字 |
| `answer` | このカードについての答え：特定したかどうか（`status`）、どのカードか（`card`）、価格（`price_jpy`）、特定できないときの候補（`candidates`）、確信度（`confidence`） |

- 出力全体の `grid`（格子の段・列の数）は、これまで通り出力の一番上に置く。
- 採点器は、成績を `answer` のうち断定した値（`status` が `identified` のときの `card`、null でない `price_jpy`）で決める。`reading` と、`answer` のうち候補・確信度は、診断に使う（[0039](0039-deliverables-and-intermediates.md)）。この使い分けは採点器の決まりで、出力の形には書かない。

## 理由

- **読んだものと答えを、プロパティで見分けられるようにするため。** 人が出力のJSONを読むとき、`reading` を見れば読み取りの段階、`answer` を見れば答えがわかる。A を選ばなかったのはこのため。
- **名前を中身で付け、採点の仕組みをモデルに教えないため。** `intermediate`／`final` は、どこが成績に使われるかを表す名前で、モデルの答え方を変えるおそれがある（[0049](0049-subject-gets-answering-instructions-only.md)）。B を選ばなかった一番の理由。
- **分類を変えても、項目を動かさずに済むようにするため。** どの項目を成績に使うかは、採点器の決まりとして持つ。将来、成果物にする項目が変わっても、出力の形は変えなくてよい。
- **関係のある情報を、近くに置くため。** 候補と確信度は答えの一部なので、`status` と同じ `answer` に置く。「なぜ不明なのか」を調べるとき、1か所を見れば済む。
- **入れ子の増え方を小さく抑えたため。** 今の形からの違いは、`printed` を `reading` にし、`price_jpy` と `identification` の中身を `answer` にまとめる程度で、入れ子は増えない（`identification` が `answer` になる）。小さいモデルの形の誤りが増えるおそれは小さい。

## 影響

- `output.schema.json` と出力の例を、この形にする。
- [0040](0040-intermediate-results-for-diagnosis.md) などで `printed.price_text` と書いていた場所は、`reading.price_text` とする。
- 正解データ（`gold.csv`）は平らな表のままとする。`gold.schema.yaml` の説明で、どの列が `reading` の正解（カード名、HP、価格の文字）で、どの列が `answer` の正解（価格、パック、番号、仕様）かを分けて書く。
- 洗い出したデメリットのうち、次の点は残る。分類がある程度出力の形に表れること。入れ子が深いこと自体による形の誤り。形の誤りが多ければ、診断で見て、見直す。
