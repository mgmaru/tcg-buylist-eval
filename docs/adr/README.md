# ADR（Architecture Decision Record）

このプロジェクトで決めたことと、**なぜそう決めたか** を記録する。

- 設計書（[tcg-buylist-eval-design.md](../tcg-buylist-eval-design.md)）は「何をどう作るか」、ADRは「なぜそう決めたか」を書く。
- 1つの決定につき1ファイル。ファイル名は `NNNN-<タイトル>.md`（番号は連番）。
- 新しく書くときは [template.md](template.md) をコピーする。
- 採用したADRの本文は書き換えない。決定を変えるときは新しいADRを書き、古いADRのステータスを「置き換え」にする。

## ステータス

| ステータス | 意味 |
|---|---|
| 提案 | 検討中。調査や判断が終わっていない |
| 採用 | 決定済み。この内容に従って進める |
| 却下 | 検討したが採用しなかった |
| 廃止 | 前提がなくなり、もう従わない |
| 置き換え | 新しいADRに置き換えられた（置き換え先を本文に書く） |

## 一覧

| 番号 | タイトル | ステータス | 日付 |
|---|---|---|---|
| [0001](0001-record-decisions-in-adr.md) | 決定はADRで記録する | 採用 | 2026-10-26 |
| [0002](0002-compare-cloud-and-local-models.md) | 比較するモデル：クラウドのモデル × ローカルのマルチモーダルモデル | 採用 | 2026-10-26 |
| [0003](0003-isolated-execution-environment.md) | 実行基盤：隔離された環境で動かし、モデルのサーバーと分ける | 採用 | 2026-10-26 |
| [0004](0004-infrastructure-as-code.md) | インフラをIaC化する（Terraform） | 採用 | 2026-10-26 |
| [0005](0005-gpu-environment.md) | ローカルモデルを動かすGPU環境：AWSのg6.xlarge（米国東部） | 採用 | 2026-10-26 |
| [0006](0006-cloud-model-authentication.md) | クラウドのモデルの認証：APIキー | 採用 | 2026-10-26 |
| [0007](0007-use-langfuse-for-scoring-and-tracing.md) | 記録・採点・比較・観測にLangfuseを使う | 採用 | 2026-10-26 |
| [0008](0008-execution-framework.md) | 環境の準備と実行：M0は自作の実行スクリプト | 採用 | 2026-10-26 |
| [0009](0009-models-for-m0.md) | M0で使うモデル：Claude Opus 5.5 と Qwen3.8-27B | 採用 | 2026-10-26 |
| [0010](0010-self-host-langfuse.md) | Langfuseの運用：AWS上に自分で立てる | 採用 | 2026-10-26 |
| [0011](0011-immutable-infrastructure.md) | インフラはイミュータブルにする | 採用 | 2026-10-26 |

調査の結果は [docs/research/](../research/) にある。
