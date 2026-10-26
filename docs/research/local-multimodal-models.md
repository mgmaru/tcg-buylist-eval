# 調査：自分で動かすマルチモーダルモデルの候補

調査日：2026-10-26

[ADR-0002](../adr/0002-compare-cloud-and-local-models.md) の「ローカルのマルチモーダルモデル1つ」と、[ADR-0005](../adr/0005-gpu-environment.md) のGPU環境を決めるための調査。タスクは設計書 2章のとおり、買取表画像1枚（カード20枚以上の格子）から、カード名・HP・ワザ名・価格を読み取り、設計書 5.3のJSONで返すこと。

## 結論

- **第一候補はQwen系**。日本語の文字読み取りの第三者評価で、同じクラスの他のモデルより明らかに強い（下の「日本語の読み取り」）。ライセンスはApache 2.0、Ollama公式ライブラリとvLLMの両方に対応。
- GPUごとのおすすめ：

| GPU | 第一候補 | 予備 |
|---|---|---|
| 10GB（RTX 3080） | Qwen3.5-9B（4bit、Ollama） | Qwen3-VL-8B-Instruct（4bit、Ollama） |
| 16GB（T4） | Qwen3.5-9B（8bit、Ollama） | Gemma 4 12B（8bit、Ollama） |
| 24GB（L4／A10G） | Qwen3.8-27B（4bit、Ollama） | Qwen3.5-9B（BF16、vLLM） |

- **M0はOllamaで始める**。量子化済みの重みがタグ1つで手に入り、OpenAI互換のAPIとJSONスキーマの出力指定がある。vLLMは24GBで量子化しないモデルを動かす場合や、再現性を詰める段階で使う。
- **24GB（L4／A10G）が一番よい**。27Bクラスが4bitで載り、9Bなら量子化せずに動かせる。10GBと16GBは9Bクラスまで。
- 注意点：
  - Qwen3.5以降は、初期設定で考える過程（thinking）を出力する。JSONだけを返させるには、thinkingを切るか、考える過程と回答を分けて受け取る設定が要る。
  - Gemma 4は1枚の画像に使えるトークンが最大1,120。カード20枚以上の買取表では1枚あたりの情報量が足りない恐れがある。
  - **Qwen系の学習データの時期は公表されていない（要確認）**。設計書の「学習後に出たカードを含める」には、実際に聞いて確かめる必要がある。

## 候補の一覧

2025年後半〜2026年10月に出た、自分で動かせる大きさ（おおむね35B以下）のモデルを中心に調べた。

### 主な候補

| モデル | 公開 | パラメータ | コンテキスト | ライセンス | Ollama公式 | vLLM |
|---|---|---|---|---|---|---|
| Qwen3.5-9B | 2026年2〜3月 | 9B（密） | 262K | Apache 2.0 | あり（`qwen3.5:9b`） | あり |
| Qwen3.5-4B | 2026年2〜3月 | 4B（密） | 262K | Apache 2.0 | あり（`qwen3.5:4b`） | あり |
| Qwen3.6-27B／35B-A3B | 2026年4月 | 27B（密）／35B（MoE、動くのは3B） | 262K | Apache 2.0 | あり（`qwen3.6`） | あり（0.19以上を推奨） |
| Qwen3.8-27B | 2026年8月 | 27B（密） | 262K | Apache 2.0 | あり（`qwen3.8:27b`） | あり |
| Qwen3-VL-8B-Instruct | 2025年10月（要確認） | 8B（密） | 256K | Apache 2.0 | あり（`qwen3-vl:8b`） | あり |
| Gemma 4 E4B／12B | 2026年4月／6月 | 実効4.5B／12B | 128K／256K | Apache 2.0 | あり（`gemma4`） | あり（要確認） |
| Gemma 4 26B-A4B／31B | 2026年4月 | 25B（MoE、動くのは3.8B）／31B（密） | 256K | Apache 2.0 | あり（`gemma4`） | あり（要確認） |
| Ministral 3 8B／14B | 2025年12月 | 8.4B＋画像0.4B／13.5B＋画像0.4B | 256K | Apache 2.0 | あり（`ministral-3`） | あり（0.12以上） |
| MiniCPM-V 4.5 | 2025年8月 | 8B（Qwen3-8Bがベース） | 要確認 | Apache 2.0 | あり（`minicpm-v4.5`） | あり |
| MiniCPM-V 4.7 35B-A3B | 2026年10月 | 35B（MoE、動くのは3B） | 要確認 | Apache 2.0 | なし（調査日時点） | あり |
| InternVL3.5-8B など | 2025年8月 | 1B〜241B | 要確認 | 要確認 | なし | あり |
| LLM-jp-4-VL 9B | 2026年8月 | 9B（日本語のモデルがベース） | 要確認 | Apache 2.0 | なし | 要確認 |

### 外した候補

| モデル | 外した理由 |
|---|---|
| Qwen3.8-Flash-Next | 125B（MoE）。24GBに載らない。ライセンスもApache 2.0ではない独自のもの |
| GLM-5.3-Flash | 320B（MoE）。大きすぎる |
| GLM-4.6V-Flash | 9B、MITライセンス。言語の表記が英語と中国語のみで、日本語の評価が見つからない。予備の予備 |
| Mistral Small 4 | 119B（MoE）。大きすぎる |
| Mistral Small 3.2（24B） | 2025年6月と古い。後継のMinistral 3 14Bが同程度とされる |
| Llama 3.2 Vision（11B） | 2024年と古い。画像とテキストの組み合わせは英語のみの対応と記憶している（要確認。モデルカードは要ログイン） |
| Llama 4 Scout、Muse Spark Open | 109B、816B。大きすぎる |
| Phi-4-multimodal（5.6B） | 画像の対応言語が英語のみとモデルカードに書かれている。学習データは2024年6月まで |
| Nemotron 3.5 Bolt 74B-A9B 24GB | 2bitで24GBに収める特殊な版。動く環境がA100／H100／Blackwellと書かれ、L4／A10Gは対象外。参考 |
| Stockmark JapanDocReader | 日本語の文書をJSONに構造化する特化モデル。決まったプロンプトと決まったJSONの形でしか使えない。30B（MoE）でBF16のみ |

## 必要なVRAM

重みの大きさの目安。Ollamaのタグに書かれたダウンロードの大きさ（画像を読む部分を含む）。実際には、これに会話の記憶（KVキャッシュ）と画像の処理の分として、1〜3GB以上が上乗せになる。

| モデル | BF16 | 8bit（Q8_0） | 4bit（Q4_K_M） |
|---|---|---|---|
| Qwen3.5-4B | 9.3GB | 5.2GB | 3.3GB |
| Qwen3.5-9B | 19GB | 10GB | 6.6GB |
| Qwen3-VL-8B | 18GB | 9.9GB | 6.2GB |
| Qwen3.6-27B／Qwen3.8-27B | 56GB | 30GB | 17〜18GB |
| Qwen3.6-35B-A3B | 72GB | 39GB | 24GB |
| Gemma 4 E4B | 16GB | 9.0GB | 6.6GB（QATは6.1GB） |
| Gemma 4 12B | 24GB | 13GB | 8.0GB（QATは7.2GB） |
| Gemma 4 26B-A4B | 52GB | 28GB | 18GB（QATは16GB） |
| Gemma 4 31B | 63GB | 34GB | 20GB（QATは19GB） |
| Ministral 3 8B | 18GB（FP16） | 9.9GB | 6.0GB |
| Ministral 3 14B | 28GB（FP16） | 15GB | 9.1GB |
| MiniCPM-V 4.7 35B-A3B | 要確認 | 要確認 | 22.6GB（int4／AWQの重みの合計） |

QATは、量子化を前提に学習した版。同じ4bitでも精度が落ちにくい。

vLLM向けの公式の量子化版の大きさ（Hugging Faceの重みの合計）：

| モデル | 大きさ | 24GBに載るか |
|---|---|---|
| Qwen3.5-27B-GPTQ-Int4 | 30.2GB | 載らない |
| Qwen3.5-35B-A3B-GPTQ-Int4 | 24.4GB | 載らない |
| Gemma 4 31B QAT（w4a16） | 23.3GB | 実質載らない |
| Gemma 4 12B QAT（w4a16） | 10.3GB | 載る（16GBにも載る） |

→ **27Bクラスを24GBで動かすなら、OllamaのGGUF（4bit）が現実的**。vLLMで24GBに載るのは、9B〜12Bクラスまで。

## GPUごとの当てはまり

| GPU | 載る大きさ | 注意点 |
|---|---|---|
| 10GB（RTX 3080） | 9Bの4bit、4Bの8bit、12BのQAT | 9Bの8bit（10GB）は載らない。12BのQAT（7.2GB）は残りが3GBほどで、画像が大きいと厳しい。Windowsでは、vLLMはWSL2かDocker経由になる |
| 16GB（T4） | 9Bの8bit、12Bの8bit、14Bの4bit | **T4はBF16とFP8に対応していない**（vLLMではFP16を使う）。FlashAttention 2も使えない。古い世代なので、新しいモデルの部品が動かないことがある（要確認） |
| 24GB（L4／A10G） | 27Bの4bit、9Bの量子化なし、26B-A4BのQAT | L4はFP8に対応、A10Gは対応していない。27Bの4bit（18GB）は残りが6GBほどで、コンテキストを絞る必要がある |

Qwen3.5以降は、注意（attention）の層の4分の3を軽い仕組み（Gated DeltaNet）に置き換えている。そのため、長い入力でもKVキャッシュが増えにくい。一方、T4のような古いGPUでこの仕組みが速く動くかは要確認。

## 日本語の読み取り

公式の数値は英語中心で、日本語の文字読み取りをそのまま示すものは少ない。LLM-jp（国立情報学研究所の大規模言語モデル研究開発センター）が、日本語の評価を同じ条件で比べた表を公開している。第三者による比較として一番参考になる。

| 評価（日本語） | Qwen3.5-9B | Gemma 4 12B | InternVL3.5-8B | LLM-jp-4-VL 9B |
|---|---|---|---|---|
| CC-OCR-JA（日本語の文字読み取り） | **79.8** | 47.8 | 51.5 | 70.3 |
| JDocQA（日本語の文書の質問応答） | **93.3** | 69.4 | 59.5 | 74.0 |
| JMMMU（日本語の総合問題） | **69.6** | 63.8 | 50.9 | 47.5 |
| 日本語の平均（8種類） | **80.1** | 68.9 | 56.5 | 65.6 |

- Qwen3.5-9Bが、日本語の文字読み取りではっきり強い。Gemma 4 12Bは、日本語の文字読み取りが弱い。
- 公式の文字読み取りの数値（OCRBench）：Qwen3.5-9Bは89.2、Qwen3.5-4Bは85.0、Qwen3.6-27Bは89.4。
- 文書の読み取りの数値（OmniDocBench 1.5）：Qwen3.5-9Bは87.7、Qwen3.6-35B-A3Bは89.9、Qwen3.8-27Bは91.1。Gemma 4は別の指標（誤りの少なさ）で、31Bが0.131、12Bが0.164（小さいほど良い）。
- Qwen3-VLは、文字を読める言語を32に増やしたとしている。Ministral 3は、対応言語に日本語を含むと書いている。MiniCPM-V 4.5は文字読み取りに強いとしているが、日本語の数値は見つからない（要確認）。
- **ポケモンカードのような、小さく、飾りの多い文字（カード名・ワザ名）を読めるかは、どの評価にも含まれていない**。実際に動かして確かめる。

## JSONで出力させる

| 方法 | 内容 |
|---|---|
| Ollama | `format` にJSONスキーマを渡せる。画像を入力するモデルでも使える。OpenAI互換のAPIでは `response_format` で指定できる |
| vLLM | OpenAI互換のAPIの `response_format`（`json_schema`）で指定できる。中では文法に沿ってトークンを絞る（xgrammar など） |
| モデル側 | Gemma 4は関数呼び出しに対応。Qwen系とMinistral 3も道具の呼び出しに対応 |

注意点：

- **thinkingとの組み合わせ**。Qwen3.5／3.6／3.8は、初期設定で考える過程を `<think>` の中に出してから回答する。vLLMでは、考える過程を分ける設定（reasoning parser）を使えば、JSONの指定は回答の部分だけにかかる。設定を間違えると、JSONの指定がかからないことがある。thinkingを切るなら `enable_thinking: false` を渡す。
- thinkingを入れるか切るかは、精度と時間の両方に効く。比べる構成の1つとして扱う（設計書 6.5の「部品は仮説として扱う」）。
- **出力の長さ**。カード20枚分のJSONは、1枚あたり150トークンほどとして、3,000トークン以上になる。thinkingを入れると、さらに数千〜数万トークン増える。出力の上限を十分に取る。
- 文法でトークンを絞ると、形は必ず正しくなる。ただし、中身の読み間違いは減らない。形の正しさと中身の正しさは分けて採点する。

## 高解像度の画像の入力

買取表は、カード20枚以上が詰まった1枚の画像。カード1枚あたりに使える画像のトークンが少ないと、小さな文字が読めない。

| モデル | 画像の扱い | 1枚の上限 |
|---|---|---|
| Qwen3.5／3.6／3.8 | 32×32ピクセルごとに1トークン。縦横比は自由 | 最大 約1,680万ピクセル（4,096×4,096相当）＝約16,000トークン。最小は約6.5万ピクセル |
| Qwen3-VL | 同じ系統の仕組み | 要確認 |
| Gemma 4 | トークンの数を5段階から選ぶ。縦横比は自由 | 70／140／280／560／**1,120**トークン |
| MiniCPM-V 4.5 | 画像を切り分けて読む | 約180万ピクセル（1,344×1,344相当） |
| MiniCPM-V 4.7 | 圧縮率を4倍か16倍から選ぶ。切り分けの数も指定できる | 要確認 |
| Ministral 3 | 縦横比を1：1に近づけることを推奨 | 要確認 |

- 例：2,048×1,536ピクセルの買取表を、Qwen系にそのまま渡すと約3,000トークン。カード21枚なら1枚あたり約150トークン。
- Gemma 4は最大でも1,120トークン。カード21枚なら1枚あたり約50トークンしかない。文字の多いカードには足りない恐れがある。
- どのモデルでも、**画像を段ごとに切って渡す**と、1枚あたりのトークンを増やせる。これは被験ハーネスの部品として比べる（H0は切らずに渡す）。
- Ollamaが画像を内部で縮めるかは要確認。縮める場合、vLLMとの差の原因になる。
- **Ollamaのコンテキストの初期値は小さい**（VRAM 24GB未満では4K、という記事がある。要確認）。画像と出力で数千トークンを超えるので、`num_ctx` を必ず明示する。

## モデルを動かす仕組み（サーバー）

| | Ollama | vLLM | llama.cpp（llama-server） |
|---|---|---|---|
| 始めやすさ | 一番簡単。`ollama pull` で量子化済みの重みが手に入る | 重みの選択、量子化、メモリの設定が要る | 重み（GGUF）と画像用のファイルを自分で揃える |
| OpenAI互換のAPI | あり | あり | あり |
| JSONスキーマの指定 | あり（`format`、`response_format`） | あり（`response_format`） | あり（文法の指定） |
| 量子化 | GGUF（4bit、8bitなど）。QAT版もある | AWQ、GPTQ、FP8（L4のみ）、bitsandbytes、GGUF（試験的） | GGUF |
| 新しいモデルへの対応 | 早いが、モデルによってはない（MiniCPM-V 4.7、InternVLなど） | 一番早い。公式のモデルカードがvLLMを推奨していることが多い | Ollamaと同じくらい。画像の対応はモデルごとに差がある |
| Docker | 公式のイメージあり | 公式のイメージあり（`vllm/vllm-openai`） | 公式のイメージあり |
| 再現性 | タグの中身が後で変わることがある。重みのハッシュを記録する | Hugging Faceのリビジョンを固定できる。設定項目が多く、記録すべきことも多い | ファイルを固定すれば一番固い |
| 弱い点 | 画像の前処理やコンテキストの初期値が隠れている | 量子化の選択肢が限られ、24GBに27Bを載せにくい。起動が遅い | 自分で揃える手間が多い |

- 速さは重要でない（M0は18回程度）。**手軽さと、24GBに27Bを載せられることを重視して、Ollamaを第一候補にする**。
- 再現性のため、`run.yaml`（設計書 5.5）に、サーバーの版、モデルのタグと重みのハッシュ、量子化、`num_ctx`、thinkingの有無、画像の扱いを記録する。
- vLLMは、量子化しない9Bクラスを24GBで動かすときや、画像の前処理を細かく決めたいときに使う。Ollamaとの差（同じモデルでもサーバーで結果が変わるか）は、余裕があれば確かめる。

## 学習データの時期

設計書では、モデルの学習後に出たカードを含めたい。各モデルの学習データの時期：

| モデル | 学習データの時期 | 出典 |
|---|---|---|
| Gemma 4 | 事前学習のデータは2025年1月まで | 公式のモデルカード |
| Qwen3.5／3.6／3.8 | **公表なし（要確認）**。公開は2026年2月〜8月 | 公式のモデルカードに記載なし |
| Qwen3-VL | 公表なし（要確認） | |
| Ministral 3 | 公表なし（要確認） | |
| MiniCPM-V 4.5 | 公表なし（要確認）。ベースのQwen3は2025年4月公開 | |
| Phi-4-multimodal（参考） | 2024年6月まで | 公式のモデルカード |
| Nemotron 3.5 Bolt（参考） | 事後学習のデータは2026年5月まで | 公式のモデルカード |

- Gemma 4は、2025年以降のカードを知らない前提で比べられる。
- Qwen系は時期が分からない。公開の時期から見て、2025年後半のカードまで知っている可能性がある。実際に、発売時期の違うパックのカードについて聞いて、どこから知らないかを確かめる（要確認）。
- Claude Opus 5.5との比較では、学習データの時期の差が結果に混ざる。時期より後に出たカードの割合を、タスクセットごとに記録する。

## おすすめ

### 10GB（RTX 3080、Windows PC）

| | モデル | 理由 |
|---|---|---|
| 第一候補 | Qwen3.5-9B（`qwen3.5:9b`、4bit、6.6GB） | 日本語の文字読み取りが同じクラスで一番強い。画像の解像度の上限も高い |
| 予備 | Qwen3-VL-8B-Instruct（`qwen3-vl:8b-instruct`、4bit、6.2GB） | thinkingのない版があり、JSONだけを返させやすい。古い分、サーバーの対応が安定している |

- 9Bの8bit（10GB）は載らない。4bitでの読み間違いが増えないかを確かめる。
- 別の系統も試すなら、Ministral 3 8B（6.0GB）。

### 16GB（AWS T4）

| | モデル | 理由 |
|---|---|---|
| 第一候補 | Qwen3.5-9B（`qwen3.5:9b-q8_0`、8bit、10GB） | 10GBと同じモデルを、量子化を弱くして動かせる。10GBの結果と比べれば、量子化の影響が分かる |
| 予備 | Gemma 4 12B（`gemma4:12b-it-q8_0`、8bit、13GB） | 別の系統で、学習データの時期（2025年1月）がはっきりしている。ただし日本語の文字読み取りは弱く、画像のトークンも最大1,120 |

- T4はBF16に対応していない。vLLMでFP16にしたときに数値があふれないかは要確認（Ollamaなら気にしなくてよい見込み）。

### 24GB（AWS L4／A10G）

| | モデル | 理由 |
|---|---|---|
| 第一候補 | Qwen3.8-27B（`qwen3.8:27b`、4bit、18GB） | 調査日時点で一番新しいQwenの密なモデル。文書の読み取りの数値が一番高い（OmniDocBench 1.5で91.1） |
| 予備 | Qwen3.5-9B（BF16、vLLM、19GB） | 量子化なし。10GB／16GBの結果と並べれば、量子化と大きさの影響を分けられる |

- 27Bの4bitは残りが6GBほど。コンテキストを16K〜32Kに絞れば動く見込み（要確認）。
- 別の系統も試すなら、Gemma 4 26B-A4BのQAT（16GB）。動くパラメータが3.8Bと少なく速い。
- MiniCPM-V 4.7 35B-A3B（2026年10月公開）は、int4で22.6GBと24GBぎりぎり。Ollamaの公式ライブラリにまだない。様子を見る。

### 全体として

- **M0は24GB（g6.xlargeのL4）で、Qwen3.8-27Bを第一候補にする**のがよい。GPUの利用枠が取れないときは、Windows PCでQwen3.5-9B（4bit）にする。
- 同じQwen3.5-9Bを、4bit（10GB）、8bit（16GB）、BF16（24GB）で動かせる。量子化の影響を見る対照になる。

## 実際に動かして確かめること

1. **カード名・ワザ名・HP・価格が読めるか**：サンプルの買取表1枚で、文字の読み取りの正しさを見る。小さい文字、飾りのある文字、「ー」と「一」のような似た文字
2. **格子の位置が合うか**：段・列（`position`）が正しいか。カードの数え漏れや重複がないか
3. **JSONの形**：スキーマの指定ありとなしで、形の崩れと中身の正しさがどう変わるか
4. **thinkingの有無**：精度、出力のトークン数、時間の差
5. **画像の大きさ**：そのまま渡す場合と、段ごとに切る場合の差。Ollamaが画像を縮めていないか
6. **VRAMの実際の使用量**：画像1枚＋出力の上限で、メモリが足りるか。27Bの4bitが24GBで動くか
7. **学習データの時期**：発売時期の違うパックのカードについて聞き、どこから知らないかを調べる
8. **同じ条件での揺れ**：温度0で3回動かして、結果がどれだけ変わるか
9. **T4で動くか**：使うなら、vLLMとOllamaで新しいモデルが動くか（BF16なし、FlashAttention 2なし）

## 出典

- Qwen3.5-9B のモデルカード：https://huggingface.co/Qwen/Qwen3.5-9B
- Qwen3.5-4B のモデルカード：https://huggingface.co/Qwen/Qwen3.5-4B
- Qwen3.5-9B の画像の前処理の設定：https://huggingface.co/Qwen/Qwen3.5-9B/blob/main/preprocessor_config.json
- Qwen3.6-35B-A3B のモデルカード：https://huggingface.co/Qwen/Qwen3.6-35B-A3B
- Qwen3.6-27B のモデルカード：https://huggingface.co/Qwen/Qwen3.6-27B
- Qwen3.8-27B のモデルカード：https://huggingface.co/Qwen/Qwen3.8-27B
- Qwen3.8-Flash-Next のモデルカード：https://huggingface.co/Qwen/Qwen3.8-Flash-Next
- Qwen3.5-27B-GPTQ-Int4：https://huggingface.co/Qwen/Qwen3.5-27B-GPTQ-Int4
- Qwen3-VL-8B-Instruct のモデルカード：https://huggingface.co/Qwen/Qwen3-VL-8B-Instruct
- Gemma 4 のモデルカード：https://ai.google.dev/gemma/docs/core/model_card_4
- Gemma 4 12B のモデルカード：https://huggingface.co/google/gemma-4-12B-it
- Gemma 4（Wikipedia、公開日の確認）：https://en.wikipedia.org/wiki/Gemma_(language_model)
- Ministral 3 8B のモデルカード：https://huggingface.co/mistralai/Ministral-3-8B-Instruct-2512
- Ministral 3 14B のモデルカード：https://huggingface.co/mistralai/Ministral-3-14B-Instruct-2512
- MiniCPM-V 4.5 のモデルカード：https://huggingface.co/openbmb/MiniCPM-V-4_5
- MiniCPM-V 4.7 35B-A3B のモデルカード：https://huggingface.co/openbmb/MiniCPM-V-4.7-35B-A3B
- MiniCPM-V（GitHub）：https://github.com/openbmb/MiniCPM-V
- InternVL3.5 の論文：https://arxiv.org/abs/2508.18265
- LLM-jp-4-VL 9B のモデルカード（日本語の評価の比較表）：https://huggingface.co/llm-jp/llm-jp-4-vl-9b
- GLM-4.6V-Flash のモデルカード：https://huggingface.co/zai-org/GLM-4.6V-Flash
- GLM-5.3-Flash のモデルカード：https://huggingface.co/zai-org/GLM-5.3-Flash
- Phi-4-multimodal のモデルカード：https://huggingface.co/microsoft/Phi-4-multimodal-instruct
- Nemotron 3.5 Bolt 74B-A9B 24GB のモデルカード：https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Bolt-74B-A9B-24GB
- Stockmark JapanDocReader のモデルカード：https://huggingface.co/stockmark/Stockmark-Nemotron-3-Nano-Omni-JapanDocReader
- Mistral Small 4 の紹介記事：https://letsdatascience.com/news/mistral-releases-mistral-small-4-model-8731b156
- Muse Spark Open のモデルカード：https://huggingface.co/meta-models/Muse-Spark-Open-816B-A42B
- Ollama：qwen3.5：https://ollama.com/library/qwen3.5/tags
- Ollama：qwen3.6：https://ollama.com/library/qwen3.6/tags
- Ollama：qwen3.8：https://ollama.com/library/qwen3.8/tags
- Ollama：qwen3-vl：https://ollama.com/library/qwen3-vl/tags
- Ollama：gemma4：https://ollama.com/library/gemma4/tags
- Ollama：ministral-3：https://ollama.com/library/ministral-3/tags
- Ollama：minicpm-v4.5：https://ollama.com/library/minicpm-v4.5
- Ollama の構造化出力：https://docs.ollama.com/capabilities/structured-outputs
- Ollama のコンテキストの初期値（解説記事）：https://jacar.es/en/gemma-4-locally-with-ollama/
- vLLM のGPUの要件：https://docs.vllm.ai/en/latest/getting_started/installation/gpu.html
- vLLM の量子化とGPUの対応：https://docs.vllm.ai/en/latest/features/quantization/
- vLLM の構造化出力：https://docs.vllm.ai/en/latest/features/structured_outputs.html
- T4でのvLLM（解説記事）：https://markaicode.com/benchmarks/cuda-production-benchmark-latency/
