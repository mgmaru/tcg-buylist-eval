# 調査：環境の準備・リセットとエージェントの自動実行の仕組み

調査日：2026-10-26

[ADR-0008](../adr/0008-execution-framework.md) の判断材料。自作の実行スクリプト、Inspect AI、Harbor、coder_eval を比べる。

## 結論

- **M0（H0のみ）は、小さな自作の実行スクリプトで始める。** H0は「画像1枚とプロンプトをモデルのAPIに送り、JSONを受け取る」だけで、ツールもファイル操作もない。Anthropic SDKとOpenAI互換のクライアントの2つを呼び分ければ済み、300〜500行程度に収まる見込み。隔離は、実行スクリプトごとDockerコンテナに入れてEC2で動かせば足りる。
- **ただし、採点器は「出力JSONと正解CSVを受け取り、指標の辞書を返す純粋な関数」として作る。** これでInspect AIの採点関数にもHarborの検証スクリプトにも、ほぼそのまま移せる。
- **M2以降（H2・H3、エージェントCLI、繰り返し実行）は、Inspect AIを第一候補にする。** 理由は次の3つ。
  - モデルを直接呼ぶ評価（H0）と、Claude Code・Codex CLI・Gemini CLIを動かす評価を、**同じモデル呼び出しの層** で扱える。外部のエージェントCLIのAPI呼び出しも、Inspectのモデルの設定に中継される。
  - 1サンプルに複数の指標（正解・不明・誤答の件数など）を持たせる仕組み、繰り返し（epochs）、並列、時間・トークン・費用の上限が標準でそろっている。
  - MITライセンスで、更新が活発。
- **Harborは次点。** エージェントCLIの対応数とクラウドのサンドボックスの種類は最も多い。ただし、実行の単位が「エージェントがコンテナの中で作業する」ことなので、モデルを直接呼ぶH0の比較には向かない。
- **coder_evalは採用しない。** 登場して日が浅く（2026年7月に公開）、モデルを直接呼ぶ評価ができず、Langfuseとの連携も見当たらない。
- **Langfuseとの連携は、どの選択肢でも組み込みでは用意されていない。** Inspectはフック（サンプルの終了時などに呼ばれる関数）から、LangfuseのSDKでスコアを送る形になる。

## 前提の整理

| 項目 | 内容 | 実行の仕組みに求めること |
|---|---|---|
| 入力 | 買取表の画像1枚（設計書 2章） | 画像をモデルに渡せる |
| 出力 | JSON（設計書 5章） | 出力を回収し、スキーマを検証できる |
| M0 | H0のみ。ツールなし、プロンプト1回（設計書 6章、11章） | モデルを直接呼べれば足りる |
| M1以降 | H2でカードDB検索のMCPサーバー、H3で自己検証 | ツールやMCPを渡せる |
| 将来 | Claude Code（`-p`）、Codex（`exec`）、Gemini CLIを被験エージェントにする（設計書 7章） | 外部のエージェントCLIを隔離して動かせる |
| モデル | Claude Opus 5.5（APIキー）と、自分で立てるオープンウェイトのマルチモーダルモデル（[0002](../adr/0002-compare-cloud-and-local-models.md)、[0006](../adr/0006-cloud-model-authentication.md)） | Anthropic APIと、vLLM・Ollama・OpenAI互換のエンドポイントの両方 |
| 隔離 | 作業用のPCから切り離す（[0003](../adr/0003-isolated-execution-environment.md)）。AWS EC2の可能性が高い（[0005](../adr/0005-gpu-environment.md)） | Dockerで動く。EC2の上でも動く |
| 採点 | 自作の採点器。カードごとに正解・不明・誤答に分ける（設計書 8章） | 自作の採点を組み込める。1サンプルに複数の指標 |
| 記録 | Langfuse（[0007](../adr/0007-use-langfuse-for-scoring-and-tracing.md)） | スコアとトレースを送れる |

設計書 3章のとおり、このタスクのエージェントはリポジトリを書き換えない。**H0では、隔離は「実行スクリプトをコンテナの中で動かす」ことで足りる。** 1回ごとのサンドボックスが本当に必要になるのは、エージェントCLIがファイルやシェルを触る段階からである。

## 比較表

凡例：◎ 標準で十分にある／◯ ある／△ 工夫すれば可能／× ない／要確認 公式の資料で確かめられなかった

| 観点 | 自作 | Inspect AI | Harbor | coder_eval |
|---|---|---|---|---|
| 開発元 | 自分 | 英国AI Security Institute ほか | Terminal-Benchの開発者（harbor-framework） | UiPath |
| ライセンス | — | MIT | Apache-2.0 | Apache-2.0 |
| 版（PyPI） | — | 0.3.285 | 0.25.0 | 0.12.16 |
| GitHubのスター | — | 約3,100 | 約6,500 | 約150（2026年7月公開） |
| Anthropic API | ◯ SDKを直接呼ぶ | ◎ `anthropic/...` | ◯ エージェント経由 | ◯ エージェント経由 |
| 自前のモデル（vLLM・Ollama・OpenAI互換） | ◯ OpenAI互換のクライアント | ◎ `vllm/`、`ollama/`、`openai-api/` が組み込み | ◯ `--base-url` などで指定。Codexの例がある | △ OpenCode、Piを使う。Claude CodeはLiteLLMの中継が必要 |
| モデルを直接呼ぶ評価（H0） | ◎ | ◎ `generate()` だけで書ける | △ 自作のエージェントを書く必要がある | × エージェントの実行が前提 |
| 画像の入力 | ◯ base64で送る | ◎ `ContentImage`。ただし自前のモデルで通るかは要確認 | △ 画像は環境にファイルとして置き、エージェントに読ませる | △ 同左 |
| Docker | ◯ 実行スクリプトをコンテナに入れる | ◎ サンプルごとにサンドボックス | ◎ 試行ごとにコンテナ | ◯ `docker` ドライバー |
| EC2で動かす | ◯ | ◯ EC2のVMをサンドボックスにする拡張もある | ◯ `ec2` のサンドボックスがある | 要確認 |
| エージェントCLI（Claude Code、Codex、Gemini CLI） | △ 自分でアダプタを書く | ◎ inspect_swe に3つともある | ◎ 40種類以上が組み込み | ◯ Claude Code、Codex、Antigravity（Gemini）、OpenCode、Pi |
| 自作の採点器 | ◎ | ◎ 採点関数を書く。辞書で複数の指標を返せる | ◯ `test.sh` が `reward.json` に複数の数値を書く | ◯ 判定の種類を追加する拡張の仕組み、またはコマンドを実行する判定 |
| MCP（H2・H3） | △ 自分で組む | ◯ エージェントにMCPサーバーを渡せる | ◯ タスクにMCPの設定を書ける | ◯ 要確認（エージェントの設定経由） |
| ログの閲覧 | × 自分で作る（CSV・Markdown） | ◎ `inspect view` | ◎ Harbor Viewer | ◯ evalboard |
| Langfuse | ◯ SDKで直接送る | △ フックからSDKで送る | △ OpenTelemetryのプラグイン経由（要確認） | 要確認（見当たらない） |
| 繰り返し | △ 自分で書く | ◎ `--epochs` と集計方法 | ◯ `n_attempts` | ◯ `--repeats` |
| 並列 | △ 自分で書く | ◎ 同時接続数、サンプル数、サンドボックス数 | ◎ `-n` | ◯ `--max-parallel` |
| 時間制限 | △ 自分で書く | ◎ 時間、作業時間、トークン、メッセージ数、費用 | ◎ エージェント・検証・構築ごと | ◯ 上限の設定がある |
| トークン・費用 | △ APIの応答から自分で集計 | ◎ トークンを記録。単価を設定すれば費用も | ◯ ビューアに費用とトークン | ◯ ツールごとの費用 |
| 覚えること | 少ない | 多い | 中程度 | 中程度 |

## 選択肢ごとの詳細

### 自作の実行スクリプト

**M0のH0に必要な範囲**

| 部品 | 内容 | 規模の目安 |
|---|---|---|
| 構成の読み込み | `configs/*.yaml` からモデル、プロンプト、`repeats`、`timeout_sec` を読む | 小 |
| モデルのアダプタ | Anthropic SDK（Opus 5.5）と、OpenAI互換のクライアント（vLLM・Ollama）の2つ。画像はbase64で送る | 100〜150行 |
| 実行のループ | タスク × 構成 × 繰り返し。時間制限、失敗時の再試行、並列は `asyncio` の同時実行数で抑える | 100行前後 |
| 回収と形式チェック | 応答からJSONを取り出し、`output.schema.json` で検証する。壊れていれば形式エラー | 50行前後 |
| 記録 | `results/<run_id>/` に `run.yaml`、出力、トークン数、所要時間を保存する | 50行前後 |
| 採点器 | 位置で対応付け、カードごとに正解・不明・誤答を判定、指標を計算、CSVに出す（設計書 8章） | 実行の仕組みとは別。どの選択肢でも必要 |
| Langfuse（任意） | SDKでトレースとスコアを送る | 50行前後 |

- 合計で300〜500行程度（採点器を除く）の見込み。
- 隔離は、この実行スクリプトをDockerイメージに入れ、EC2で `docker run` する形で足りる。H0はファイルを書き換えるエージェントを動かさないため。
- Langfuseには、データセットの各項目に関数を当てて評価を送る実験用のSDK（`run_experiment`）がある。同時実行数の上限も指定できる。自作の実行ループの代わりにこれを使えば、さらに小さくできる。

**良い点**：中身をすべて把握できる。依存が少ない。モデル2つの呼び方の差を自分で揃えられる。

**弱い点**：エージェントCLIの起動、1回ごとのサンドボックス、ログの閲覧画面、MCPの受け渡しは、M2以降で自分で作ることになる。ここが膨らむと、評価フレームワークの再発明になる。

### Inspect AI

**モデル**

- Anthropicは `anthropic/<モデル名>` で呼ぶ（Opus 5.5なら `anthropic/claude-opus-5-5`）。
- vLLMは既存のサーバーに `VLLM_BASE_URL` でつなげる。Ollamaは `ollama/<モデル名>`。それ以外のOpenAI互換のエンドポイントは `openai-api/<名前>/<モデル名>` と、`<名前>_BASE_URL`・`<名前>_API_KEY` の環境変数で指定する。
- クラウドのモデルと自前のモデルを、同じタスクのコードでモデルの指定だけ変えて比べられる。ADR-0008の「同じ仕組みで呼べれば、ランタイムの差が混ざりにくい」に合う。

**画像**

- サンプルの入力に `ContentImage(image="picture.png")` を入れる。データセットのJSONで `{"type": "image", "image": "..."}` とも書ける。
- 公式の対応表に、画像の入力ができるプロバイダーとしてAnthropicは載っている。**vLLMとOllamaは表に載っていない（要確認）。** どちらもOpenAI互換のAPI経由なので通る可能性は高いが、試作で確かめる。
- `--no-log-images` でログに画像を残さないようにできる。買取表の画像の扱い（[0007](../adr/0007-use-langfuse-for-scoring-and-tracing.md)、設計書 13章）に関係する。

**サンドボックス**

- 標準は `docker` と `local`。拡張でKubernetes、Daytona、Modal、EC2のVM、Proxmoxなどがある。
- サンプルごとに別のサンドボックスが作られ、終了時に自動で片付けられる。
- H0はサンドボックスなしで動く。モデルを呼ぶだけなので。
- **Inspect自体をコンテナの中で動かし、その中からDockerのサンドボックスを作る形（Dockerのソケットを渡すなど）は、公式の資料に記載がない（要確認）。** EC2では、ホストに直接Inspectを入れてDockerのサンドボックスを使うのが素直と考えられる。

**エージェントCLI**

- inspect_swe パッケージに `claude_code()`、`codex_cli()`、`gemini_cli()` がある。サンドボックスの中で動く。
- エージェントのAPI呼び出しは、サンドボックス内の中継サーバーを通ってInspectのモデルの設定に送られる。そのため、**Claude Codeを自前のモデルで動かす** といった組み合わせも原理上できる（公式の資料は「どのモデルでも使える」とする。自前のモデルでの動作は要確認）。
- `claude_code()` には、MCPサーバーの追加、版の固定、システムプロンプトの追記、使わせないツール（Web検索など）の指定がある。H2・H3と、設計書 6章の `web_search: false` に対応できる。
- 画像はサンドボックスにファイルとして置き、パスをプロンプトで伝える形になる見込み。入力メッセージの画像がエージェントに直接渡るかは要確認。

**採点**

- 採点関数は `Score(value={...})` で辞書を返せる。キーごとに集計方法（平均など）を指定できる。キー全体を受け取る集計関数も書ける。
- 「正解・不明・誤答の件数」「価格の一致件数」などを1サンプルの辞書にまとめ、誤答率や適合率を集計関数で出す形が素直。
- 生成と採点を分けられる（`--no-score` で実行し、あとで `inspect score` で採点し直す）。採点器を直したときに、モデルを呼び直さずに済む。

**実行の制御**

- `--epochs` で繰り返し、集計方法（平均、中央値など）も選べる。設計書 8章の「最悪値」を見るには、自作の集計方法を書くか、繰り返しごとの点数をそのまま使う。
- 同時接続数、同時サンプル数、同時サンドボックス数を指定できる。
- 1サンプルごとの上限：時間、作業時間、トークン、メッセージ数、費用（費用は単価の設定が必要）。

**ログとLangfuse**

- ログは独自の形式で保存され、`inspect view` で閲覧できる。繰り返しごとの結果も並べて見られる。
- Langfuseとの組み込みの連携はない（検索で見つからなかった）。
- フックの仕組みがあり、実行・タスク・サンプルの開始と終了、モデルの使用量などで自作の処理を呼べる。公式の例はWeights & BiasesとMLflow。**サンプルの終了時のフックで、LangfuseのSDKからスコアを送る** のが現実的。
- トレースまで送るなら、Anthropic SDKやOpenAI SDKを計装するOpenTelemetryのライブラリを併用する方法が考えられる（要確認）。

**弱い点**：覚えることが多い（タスク、ソルバー、採点関数、サンドボックス、エージェントの中継）。版が0.3系のままで更新が速く、版を固定して使う必要がある。

### Harbor

- タスクは「指示（`instruction.md`）」「環境（Dockerfile）」「検証スクリプト（`tests/test.sh`）」の組。検証スクリプトが `/logs/verifier/reward.json` に数値を書く。
- `reward.json` には名前付きの数値を複数書ける。集計は既定で各キーの平均。`metric.py` で自作の集計も書ける。検証は別のサンドボックスで行うのが既定で、検証だけやり直す機能（regrade）もある。
- エージェントは40種類以上が組み込み（`claude-code`、`codex`、`gemini-cli` を含む）。MCPの設定、Web検索の無効化、Claude Codeの `settings.json` の受け渡しにも対応する。
- モデルの名前はLiteLLMの書式で解釈される。自前のモデルは `--base-url` などで接続先を指定する。vLLMにつなぐCodexの例が公式にある。**Claude CodeやGemini CLIを自前のモデルにつなぐ方法は要確認。**
- サンドボックスはDocker（既定）のほか、`ec2`、Daytona、Modal、E2Bなど30種類近い。同時実行数は `-n`。繰り返しは `n_attempts`。時間制限はエージェント・検証・環境の構築ごとに倍率や上限を指定できる。
- 結果はHarbor Viewerで閲覧でき、報酬、費用、トークン、所要時間、エージェントの軌跡が見られる。
- 外部連携のプラグインは、公式にLangSmithと、軌跡をOpenTelemetryに変換するもの（`atif2otel`）がある。後者でLangfuseのOpenTelemetryの受け口に送れる可能性はあるが、MLflow向けの設定が中心で、**Langfuseでの表示は要確認。**

**このタスクとの相性**

- 指示は文字列で、エージェントに渡される。**画像を入力として直接渡す仕組みはなく**、環境の中にファイルとして置いてエージェントに読ませる形になる。
- **モデルを直接呼ぶH0の比較には向かない。** やるなら、モデルAPIを1回呼ぶだけの自作のエージェント（`BaseAgent` を継承）を書くことになる（要確認）。
- エージェントCLIをたくさん並べて比べる段階（M2以降）では強い。

### coder_eval

- タスクをYAML 1ファイル（プロンプト、エージェントの設定、サンドボックス、成功条件）で書く。成功条件は重み付きで、0.0〜1.0の連続値になる。
- エージェントはClaude Code、Codex、Antigravity（Gemini）、OpenCode、Pi。オープンウェイトのモデルはOpenCodeやPiで使う想定。
- サンドボックスは一時ディレクトリが既定で、Dockerのドライバーもある。繰り返しは `--repeats`、並列は `--max-parallel`。費用とトークンも記録する。
- 判定の種類は拡張の仕組みで追加できる。コマンドを実行する判定で自作の採点器を呼ぶこともできる。
- **合わない点**
  - モデルを直接呼ぶ評価ができない。H0もエージェント経由になる。
  - 1サンプルの点数が重み付きの合計1つにまとまる設計で、正解・不明・誤答を別々の指標として持つのに向くかは要確認。
  - Langfuseとの連携は資料に見当たらない。
  - 2026年7月公開で、スター数も少ない。Python 3.13以上が必要。
  - 利用状況の匿名の送信が既定で有効（`TELEMETRY_ENABLED=false` で止める）。

### その他の候補（簡単に）

| 候補 | 内容 | 評価 |
|---|---|---|
| Langfuseの実験用のSDK（`run_experiment`） | データセットの各項目に関数を当て、評価関数の結果をスコアとしてLangfuseに送る。同時実行数の上限あり | 自作の実行スクリプトの土台として有力。隔離やエージェントCLIの起動は持たない |
| promptfoo | 設定ファイルでプロンプトとモデルを並べて比べる。Claude Agent SDKを呼ぶプロバイダーがあり、MCPの設定も渡せる | H0のモデル比較には使えるが、カードごとの判定を返す採点やDockerでの隔離はInspectほど素直でない（詳細は要確認） |

## M2以降に移るときのコスト

| 移す対象 | 自作 → Inspect AI | 自作 → Harbor |
|---|---|---|
| 採点器 | 純粋な関数なら、採点関数で包むだけ。辞書で返す形はそのまま使える | `test.sh` から呼び、結果を `reward.json` に書く |
| タスクセット | `tasks/<id>/` の画像と正解CSVを、サンプル（入力に画像、メタ情報に正解CSVのパス）に変換する関数を書く | タスクごとにディレクトリ、Dockerfile、指示を作る。数が増えると生成用のスクリプトが要る |
| モデルのアダプタ | 捨てる。Inspectのモデルの指定に置き換わる | 捨てる。エージェントの指定に置き換わる |
| 実行のループ・記録 | 捨てる。ログはInspectの形式になる | 捨てる。ログはHarborの形式になる |
| 過去の結果との比較 | Langfuseにスコアを送っておけば、そこで並べられる | 同左 |

- 採点器とタスクセットの形式を先に決めておけば、捨てるのはモデルのアダプタと実行のループ（数百行）だけで済む。
- 移行のコストを抑えるために、M0の自作でも次を守る。
  - 採点器は、実行の仕組みに依存しない関数にする（入力：出力JSONと正解CSV、出力：指標の辞書）。
  - 1回の実行の記録に、モデル名、プロンプトの版、トークン数、所要時間、終了理由（正常・時間切れ・形式エラー）を残す。Inspectのログと項目を揃えやすい。
  - スコアは最初からLangfuseに同じ名前で送る（[0007](../adr/0007-use-langfuse-for-scoring-and-tracing.md)）。移行の前後で比較が途切れない。

## 推奨

### M0：自作の実行スクリプト

- H0はモデルを直接呼ぶだけで、評価フレームワークの強み（サンドボックス、エージェントの中継、MCP）をまだ使わない。
- 設計書のM0の目安は「表が1枚出る」こと。依存を増やさずに早く結果を出すほうが優先される。
- 評価の仕組みを作る練習になる（ADR-0008の背景）。
- 実行スクリプトごとDockerイメージに入れ、EC2で動かす。これで [0003](../adr/0003-isolated-execution-environment.md) の隔離を満たす。

### M2以降：Inspect AIに移る（第一候補）

- モデルを直接呼ぶ評価とエージェントCLIの評価を、同じモデルの層で扱える。
- 複数の指標、繰り返し、並列、上限、ログの閲覧がそろっていて、自作で作る量が減る。
- 採点器を純粋な関数にしておけば、移行は小さい。
- エージェントCLIを大量に並べることが主になり、Inspectで足りない場合はHarborを見直す。

### 試作（スパイク）で確かめること

半日〜1日程度。M0の採点器ができた時点で行う。

| 確かめること | 合格の目安 |
|---|---|
| Inspectで、Opus 5.5と自前のモデル（vLLMまたはOllama）に同じ画像付きのサンプルを送れるか | 両方で画像が認識され、JSONが返る |
| 自作の採点器を、辞書を返す採点関数として組み込めるか。`--epochs 3` で集計できるか | 正解・不明・誤答の率が `inspect view` に出る |
| サンプルの終了時のフックから、LangfuseにスコアをSDKで送れるか | Langfuseの画面でスコアが見える |
| `--no-log-images` で画像をログから除けるか | ログに画像が残らない |
| EC2の上で、Inspectとinspect_sweの `claude_code()` をDockerのサンドボックスで動かせるか。画像ファイルを読ませられるか | 出力JSONが回収できる |
| inspect_sweの `claude_code()` を自前のモデルで動かせるか | 中継経由で応答が返る |
| （比較用）Harborで、画像を置いたタスクをClaude Codeに解かせ、`reward.json` に複数の指標を書けるか | Harbor Viewerで指標が見える |

## 要確認の一覧

- Inspectで、vLLM・Ollama・OpenAI互換のプロバイダーに画像を送れるか（公式の対応表に載っていない）
- Inspect自体をコンテナの中で動かし、そこからDockerのサンドボックスを使えるか
- inspect_sweのエージェントに、入力メッセージの画像が直接渡るか
- inspect_sweのエージェントを自前のモデルで動かしたときの挙動
- InspectからLangfuseへトレースを送る方法（OpenTelemetryの計装を併用できるか）
- HarborのOpenTelemetryのプラグインで、Langfuseに軌跡を表示できるか
- HarborでClaude Code・Gemini CLIを自前のモデルにつなぐ方法。モデルを直接呼ぶ自作のエージェントの書き方
- coder_evalのEC2での動作、MCPの渡し方、複数の指標の扱い、Langfuseとの連携
- promptfooの採点の自由度と隔離

## 出典

Inspect AI
- https://inspect.aisi.org.uk/providers.html
- https://inspect.aisi.org.uk/multimodal.html
- https://inspect.aisi.org.uk/sandboxing.html
- https://inspect.aisi.org.uk/agent-bridge.html
- https://inspect.aisi.org.uk/multiple-scorers.html
- https://inspect.aisi.org.uk/scoring-workflow.html
- https://inspect.aisi.org.uk/options.html
- https://inspect.aisi.org.uk/extensions-hooks.html
- https://inspect.aisi.org.uk/log-viewer.html
- https://github.com/UKGovernmentBEIS/inspect_ai
- https://pypi.org/project/inspect-ai/
- https://meridianlabs-ai.github.io/inspect_swe/
- https://meridianlabs-ai.github.io/inspect_swe/claude_code.html
- https://github.com/meridianlabs-ai/inspect_swe

Harbor
- https://docs.harborframework.com/
- https://docs.harborframework.com/llms.txt
- https://docs.harborframework.com/agents/pre-integrated-agents.md
- https://docs.harborframework.com/agents/custom-agents.md
- https://docs.harborframework.com/agents/agent-auth-resolution.md
- https://docs.harborframework.com/tasks/instruction.md
- https://docs.harborframework.com/tasks/verifier.md
- https://docs.harborframework.com/datasets/metrics.md
- https://docs.harborframework.com/sandboxes/pre-integrated-sandboxes.md
- https://docs.harborframework.com/jobs/configs.md
- https://docs.harborframework.com/plugins/existing-plugins.md
- https://docs.harborframework.com/results/view-job-results.md
- https://github.com/harbor-framework/harbor
- https://pypi.org/project/harbor/

coder_eval
- https://github.com/UiPath/coder_eval
- https://github.com/UiPath/coder_eval/blob/main/docs/USER_GUIDE.md
- https://github.com/UiPath/coder_eval/blob/main/docs/TASK_DEFINITION_GUIDE.md
- https://github.com/UiPath/coder_eval/blob/main/docs/DOCKER_ISOLATION.md
- https://github.com/UiPath/coder_eval/blob/main/docs/EXTENDING.md
- https://github.com/UiPath/coder_eval/blob/main/docs/agents/OPENCODE.md
- https://pypi.org/project/coder-eval/

その他
- https://langfuse.com/docs/evaluation/experiments/experiments-via-sdk
- https://langfuse.com/integrations/other/openinference
- https://www.promptfoo.dev/docs/providers/claude-agent-sdk/
