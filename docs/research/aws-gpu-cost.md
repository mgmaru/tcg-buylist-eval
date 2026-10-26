# AWSの費用調査：ローカルのモデルのGPUサーバーとエージェントの実行環境

調査日：2026-10-26

## 結論

- **おすすめは us-east-1 の g6.xlarge（L4・24GB）を、オンデマンドで使うときだけ作る** こと。単価は0.8048ドル／時間（約127円／時間）。
- 東京は米国より **約35〜45%高い**。バッチの評価なので通信の遅さは問題にならず、東京にする理由は小さい。us-east-1 と us-west-2 は同じ価格。
- M0（GPUを月5〜10時間）なら、GPUサーバー・エージェントの実行環境・ディスク・S3を合わせて **月1,000〜2,000円程度** で収まる（g6.xlarge・us-east-1・税込）。
- GPUの予算（月5,000円）で使える時間は、us-east-1 の g6.xlarge で約36時間、g4dn.xlarge で約55時間（税込）。月60時間使うと g6.xlarge は GPU だけで予算を超える。
- **新しいアカウントでは、GPUの利用枠（G and VT インスタンスの vCPU 数）の初期値が0** で、申請しないとGPUサーバーを起動できない。早めに申請する。
- 無料利用枠（Free Tier）はGPUには効かない。クレジット（最大200ドル）が使える可能性はあるが要確認。
- 消し忘れ対策として、AWS Budgets のアラート（無料）と、サーバー自身の自動停止を最初に用意する。

## 前提

- 為替：**1ドル＝158円** と仮定する。欧州中央銀行の参照レートは2026-10-23で156.80円、10月15〜23日は156.8〜158.2円で推移した。少し円安側に丸めた。
- 消費税：日本の利用者には AWS の利用料に10%の消費税がかかる（AWS の消費税 FAQ）。表の「税込」は ×1.1 で計算した。実際の請求の扱いは要確認。
- 価格はすべて Linux、オンデマンド。オンデマンドの価格は AWS の公式の価格データ（2026-10-23 公開版）から取った。
- スポットの価格は変動する。AWS の公式のスポット価格の配信データは2024年から更新されていなかったため、第三者のサイト（Vantage）の平均値を参考として載せる。

## GPUインスタンスの単価

VRAM 16〜24GBで、7B〜12Bクラスの画像入力のモデルを動かせるものを比べる。xlarge はどれも vCPU 4・メモリ 16GiB（g6e.xlarge のみ 32GiB）。

### オンデマンド（ドル／時間）

| インスタンス | GPU・VRAM | us-east-1 | us-west-2 | 東京 | 東京の割高さ |
|---|---|---|---|---|---|
| g4dn.xlarge | T4・16GB | 0.526 | 0.526 | 0.710 | +35% |
| g4dn.2xlarge | T4・16GB（vCPU 8・32GiB） | 0.752 | 0.752 | 1.015 | +35% |
| g6.xlarge | L4・24GB | 0.8048 | 0.8048 | 1.1672 | +45% |
| g5.xlarge | A10G・24GB | 1.006 | 1.006 | 1.459 | +45% |
| g6e.xlarge | L40S・48GB | 1.861 | 1.861 | 2.699 | +45% |

円に直すと（1ドル＝158円、税抜）：

| インスタンス | us-east-1 | 東京 | 月5,000円で使える時間（us-east-1／東京、税込） |
|---|---|---|---|
| g4dn.xlarge | 83円 | 112円 | 約55時間／約40時間 |
| g4dn.2xlarge | 119円 | 160円 | 約38時間／約28時間 |
| g6.xlarge | 127円 | 184円 | 約36時間／約25時間 |
| g5.xlarge | 159円 | 231円 | 約29時間／約20時間 |
| g6e.xlarge | 294円 | 426円 | 約15時間／約11時間 |

### スポット（参考、ドル／時間）

Vantage の掲載値（調査日時点の平均）。中断される割合の目安も同じサイトの値。

| インスタンス | us-east-1 | us-west-2 | 東京 | 中断の目安 |
|---|---|---|---|---|
| g4dn.xlarge | 約0.32 | 約0.25 | 約0.34 | 10〜20%超 |
| g6.xlarge | 約0.63 | 約0.54 | 約0.57 | 20%超 |
| g5.xlarge | 約0.51 | 約0.56 | 約0.74 | 20%超 |
| g6e.xlarge | 約1.52 | 約1.85 | 約1.43 | 20%超 |

- スポットはオンデマンドの4〜6割程度だが、中断が多い。M0は使う時間が短いので、節約額は月数百円にとどまる。
- スポットの利用枠はオンデマンドと別（後述）。M0はオンデマンドで十分。

### インスタンスの選び方

| インスタンス | 向いている使い方 | 注意点 |
|---|---|---|
| g4dn.xlarge | 量子化した7〜8Bクラス。最も安い | T4は世代が古く、bf16に対応しない。16GBでは7Bの重みをそのまま（16bit）載せると余裕がない |
| g6.xlarge | 7Bクラスを16bitのまま、12Bクラスを8bitで動かす。単価と性能のバランスが良い | 12Bクラスを16bitで載せるには24GBでは足りない |
| g5.xlarge | g6.xlarge と同じ24GB。速度はA10Gのほうが速い場面がある | g6.xlarge より約25%高い |
| g6e.xlarge | 12Bクラスを16bitで、またはより大きいモデル | 予算に対して高い。M0には不要 |
| g4dn.2xlarge | CPUとメモリを増やしたいとき | GPUは xlarge と同じ。VRAMは増えない |

ほかに、g6f（L4を分割したもの、VRAM 3〜12GB）、g5g（Arm のCPUとT4）、g7／g7e（新しい世代、2.5ドル／時間以上）も見たが、VRAMが足りない・ソフトウェアの対応が狭い・高い、のいずれかで今回の候補から外した。

## エージェントの実行環境（CPUのインスタンス）

被験エージェントと実行スクリプトを動かすVM（ADR-0003）。Docker とエージェントのCLIが動けばよい。

| インスタンス | vCPU・メモリ | us-east-1 | 東京 | 常時起動したときの月額（us-east-1／東京） |
|---|---|---|---|---|
| t4g.small（Arm） | 2・2GiB | 0.0168 | 0.0216 | 約1,940円／約2,490円 |
| t3.small | 2・2GiB | 0.0208 | 0.0272 | 約2,400円／約3,140円 |
| t4g.medium（Arm） | 2・4GiB | 0.0336 | 0.0432 | 約3,880円／約4,990円 |
| t3.medium | 2・4GiB | 0.0416 | 0.0544 | 約4,800円／約6,280円 |

（単価はドル／時間。月額は730時間、税抜。）

- **GPUサーバーと同じく、使うときだけ作れば月数十円** で済む。常時起動すると、それだけでほかのサーバー代の枠の大半を使う。
- メモリはコンテナを動かすので、2GiBでは足りない可能性がある。t3.small で試し、足りなければ t3.medium にする。
- Arm（t4g）は安いが、使うコンテナのイメージが Arm に対応している必要がある。
- 別案：GPUサーバーの中で、エージェント用のコンテナを別に動かす。サーバーが1台で済むが、ADR-0003の「エージェントの実行環境とモデルのサーバーを分ける」と、どこまで分けるかを決める必要がある。

## サーバーを消しても残る費用、その他の費用

| 項目 | us-east-1 | 東京 | メモ |
|---|---|---|---|
| EBS gp3（ディスク） | 0.08ドル／GB・月 | 0.096ドル／GB・月 | 3,000 IOPS・125MB/s は込み。**インスタンスを停止しても課金が続く**。削除すれば止まる |
| EBS スナップショット | 0.05ドル／GB・月 | 0.05ドル／GB・月 | ディスクを残したいときの保存先 |
| S3 標準（モデルの重み） | 0.023ドル／GB・月 | 0.025ドル／GB・月 | 同じリージョンのEC2との転送は無料 |
| 受信のデータ転送 | 無料 | 無料 | モデルの重みのダウンロードなど |
| インターネットへの送信 | 0.09ドル／GB | 0.114ドル／GB | 全リージョン合計で月100GBまで無料。評価の結果は小さいので実質0円 |
| 同じリージョンの別のゾーン間の転送 | 0.01ドル／GB（片方向ごと） | 0.01ドル／GB | GPUサーバーとエージェントのVMを同じゾーンに置けば0円 |
| パブリックIPv4（Elastic IP を含む） | 0.005ドル／時間・個 | 同じ | 使用中も未使用も同じ料金。常時1個で月約580円 |

目安の金額（1ドル＝158円、税抜）：

- GPUサーバーのディスク100GBを1か月残すと、us-east-1 で約1,260円、東京で約1,520円。**ディスクを残す運用は避け、毎回作り直す** ほうがよい。
- モデルの重み30GBをS3に置くと月約110〜120円。Hugging Face から毎回ダウンロードするなら0円（受信は無料）。ただしダウンロードの時間もGPUの課金時間になる。
- GPUサーバーの起動用のイメージ（Deep Learning AMI など）が必要とするディスクの最小サイズは要確認。ここでは100GBと仮定した。
- パブリックIPv4をなくすためにNATゲートウェイを置くと、それ自体が時間課金になり高くつく。M0ではパブリックIPv4を付け、セキュリティグループで接続元を絞る。

## GPUの利用枠（Service Quotas）

- 枠の単位は **vCPU の数**。インスタンスの台数ではない。
- 新しいアカウントの初期値（AWS の公式ドキュメントの表）：

| 利用枠の名前 | 初期値 |
|---|---|
| Running On-Demand G and VT instances | 0 |
| All G and VT Spot Instance Requests | 0 |
| Running On-Demand Standard (A, C, D, H, I, M, R, T, Z) instances | 5 |

- g4dn.xlarge／g6.xlarge／g5.xlarge は1台で4 vCPU。**8 vCPU を申請** しておくと、2xlarge への変更や作り直しの重なりにも対応できる。
- 申請の方法：Service Quotas のコンソールで、**使うリージョンを選んでから**「Amazon EC2」→「Running On-Demand G and VT instances」→「クォータの引き上げをリクエスト」。スポットを使うなら、別の枠（All G and VT Spot Instance Requests）も申請する。
- 枠はリージョンごと。東京で申請しても us-east-1 では使えない。
- 承認までの時間：AWS の公式の目安は見つからなかった（要確認）。利用者の報告では数時間〜数日。新しいアカウントでは断られた例もあり、用途（評価のために短時間だけ使う、など）を具体的に書くとよいという助言がある。
- AWS は、利用実績に応じてオンデマンドの枠を自動で引き上げるとしている。
- なお、エージェントの実行環境（t3など）は Standard の枠（初期値5 vCPU）に入るので、申請は不要。

## 無料利用枠（Free Tier）

2025年7月15日以降に作ったアカウントは、新しい無料利用枠の対象になる。

- 作成時に100ドルのクレジット、所定の操作（EC2の起動、AWS Budgets の設定など）で最大100ドルを追加でもらえる。
- **Free plan**：料金はかからないが、使えるサービスが限られる。6か月かクレジットを使い切るまでで終わり、その後は有料プランに切り替えないとアカウントが閉じられる。
- **Paid plan**：すべてのサービスを使える。クレジットは対象の利用に自動で充てられ、超えた分を払う。
- EC2の無料枠の対象のインスタンスは t3.micro／t3.small／t4g.micro／t4g.small／c7i-flex.large／m7i-flex.large で、**GPUインスタンスは対象外**。
- Free plan のままGPUインスタンスを起動できるかは公式で確認できなかった（要確認）。Paid plan に切り替える前提で考えるのが安全。
- クレジットがGPUインスタンスの料金に充てられるかも要確認。充てられれば、M0の費用は実質0円になりうる。
- 既存のアカウント（2025年7月15日より前に作成）の場合は、この特典は使えない。

## 費用の見積もり

### 仮定

- GPUサーバー：オンデマンド。ディスク100GB（gp3）。使う時間だけ存在し、使い終わったら Terraform で削除する。
- エージェントの実行環境：t3.small。準備や後片付けを含めて、GPUの時間の2倍だけ存在する。ディスク30GB。
- パブリックIPv4：両方のサーバーに1個ずつ、存在する時間だけ。
- S3：モデルの重み30GBを1か月保存。
- データ転送：送信は月100GBの無料枠に収まる。2台は同じゾーンに置く。
- AWS Budgets のアラート：無料。

### M0の使用時間

- 実行回数：2モデル × 1〜3タスク × 3回 ＝ 6〜18回。GPUを使うのはローカルのモデルの分（3〜9回）だけ。
- 推論そのものは1回数分の見込み。GPUの時間の大半は、サーバーの起動、モデルのダウンロードと読み込み、試行錯誤になる。
- 月に **5時間**（順調な場合）〜 **10時間**（作り直しや試行錯誤が多い場合）と見込む。

### 月額（円、1ドル＝158円）

| GPUの時間 | リージョン | インスタンス | GPUサーバー分（税抜） | 全体（税抜） | 全体（税込） |
|---|---|---|---|---|---|
| 5時間 | us-east-1 | g4dn.xlarge | 430 | 580 | 640 |
| 5時間 | us-east-1 | g6.xlarge | 650 | 800 | 880 |
| 5時間 | 東京 | g6.xlarge | 940 | 1,110 | 1,220 |
| 10時間 | us-east-1 | g4dn.xlarge | 860 | 1,060 | 1,160 |
| 10時間 | us-east-1 | g6.xlarge | 1,300 | 1,500 | 1,650 |
| 10時間 | 東京 | g6.xlarge | 1,870 | 2,110 | 2,320 |
| 30時間 | us-east-1 | g4dn.xlarge | 2,570 | 2,950 | 3,250 |
| 30時間 | us-east-1 | g6.xlarge | 3,890 | 4,280 | 4,700 |
| 30時間 | us-east-1 | g5.xlarge | 4,840 | 5,230 | 5,750 |
| 30時間 | 東京 | g6.xlarge | 5,620 | 6,080 | 6,690 |
| 60時間 | us-east-1 | g4dn.xlarge | 5,140 | 5,800 | 6,380 |
| 60時間 | us-east-1 | g6.xlarge | 7,780 | 8,440 | 9,290 |
| 60時間 | us-east-1 | g5.xlarge | 9,690 | 10,350 | 11,380 |
| 60時間 | 東京 | g6.xlarge | 11,240 | 12,040 | 13,250 |

「GPUサーバー分」は、GPUインスタンス・そのディスク・パブリックIPv4の合計。「全体」は、それにエージェントの実行環境・S3を足したもの。

### 予算との比較

- **M0（5〜10時間）**：どの組み合わせでも、GPUの予算（月5,000円）、全体の予算（月1万円）に大きく余裕がある。
- **月30時間**：us-east-1 なら g6.xlarge でもGPUの予算内。東京の g6.xlarge や us-east-1 の g5.xlarge は、税込でGPUの予算を少し超える。
- **月60時間**：GPUの予算に近いのは us-east-1 の g4dn.xlarge だけ（GPUサーバー分が税抜で約5,140円、税込で約5,650円と、わずかに超える）。g6.xlarge は全体の予算（1万円）にはぎりぎり収まるが、GPUの予算は超える。スポットを使えば半分程度になる。
- エージェントの実行環境を **常時起動** すると、t3.small・ディスク30GB・パブリックIPv4で月約3,400円（us-east-1）〜4,200円（東京）が別にかかる。使うときだけ作れば避けられる。
- クラウドのモデルのAPI代（ADR-0006）は、この見積もりに含めていない。

## 代替案：手元のWindows PC（RTX 3080・VRAM 10GB）

### 電気代

- RTX 3080 の消費電力の仕様は320W。推論中のPC全体で350〜450W程度と仮定する。
- 電気代の目安単価31円／kWh（全国家庭電気製品公正取引協議会の目安）で計算すると、**1時間あたり約11〜14円**。月60時間使っても約840円。
- 実際の単価は契約によって違う。PCの購入費や、使っていない時間の待機電力は含めていない。

### 動かせるモデルの大きさ

| モデルの例 | 量子化 | 必要なVRAMの目安 | 10GBに収まるか |
|---|---|---|---|
| Qwen2.5-VL 7B | 4bit（Q4_K_M） | 約6GB（文脈8k込み） | 収まる |
| 7〜8Bクラス | 8bit | 約8〜9GB | ぎりぎり。文脈を短くする必要がある |
| Gemma 3 12B | 4bit | 約10GB以上（推定） | ほぼ余裕がない。実測が必要 |
| 7Bクラス | 16bit（量子化なし） | 約16GB以上 | 収まらない |

- 画像を入力するモデルは、画像を処理する部分（llama.cpp では mmproj という別のファイル）の分のVRAMも追加で要る。上の目安は第三者の計算値で、実測ではない（要確認）。
- AWS の24GB（g6.xlarge）なら、7Bクラスを量子化せずに動かせる。量子化の影響を評価から外せるのが利点。

### 比較

| 観点 | AWS（us-east-1・g6.xlarge） | Windows PC（RTX 3080） |
|---|---|---|
| M0の費用 | 月1,000〜2,000円程度 | 電気代で月100円未満 |
| 月60時間の費用 | 約9,300円（全体・税込） | 約840円 |
| VRAM | 24GB | 10GB |
| 動かせるモデル | 7Bを量子化なし、12Bを8bit | 量子化した7〜8B（12Bはぎりぎり） |
| 準備 | GPUの利用枠の申請が要る | すぐ使える。エージェントの実行環境からの接続と安全性を考える必要がある |
| IaCの経験 | 多く積める | WSL2の構成管理が中心 |

## 消し忘れを防ぐ仕組み

| 仕組み | 費用 | できること | 注意点 |
|---|---|---|---|
| AWS Budgets のアラート | 無料 | 実績や予測の金額がしきい値を超えたらメールやSNSで通知 | 請求のデータの更新は1日1回以上で、リアルタイムではない。予測のアラートは約5週間の利用データが必要 |
| AWS Budgets のアクション | 月に2つまで無料。3つ目からは1つ0.10ドル／日 | しきい値を超えたら、指定したEC2インスタンスを停止する、IAMのポリシーを付けて新しいリソースの作成を禁止する | 停止ではディスクの課金は止まらない。反応は請求のデータの更新に左右される |
| サーバー自身の自動停止 | 0円 | 起動時の設定（user data）で一定時間後に `shutdown` する。停止時の動作を「終了（terminate）」にすれば、ディスクごと消える | 長い実行の途中で止まらないよう、時間に余裕を持たせる |
| CloudWatch のアラーム | 10個まで無料、以降1個0.10ドル／月 | CPU使用率などが一定時間低いままならインスタンスを停止・終了する | GPU使用率で判定するには、別にメトリクスを送る設定が要る |
| 定時の削除 | ほぼ0円 | EventBridge Scheduler と Lambda で、毎晩タグの付いたGPUインスタンスを終了する | Terraform の状態と実際のリソースがずれるので、次回の `terraform apply` で作り直す前提にする |

おすすめの組み合わせ：

- Budgets のアラートを月1,000円・5,000円・10,000円の3段で作る（Terraform で作る、ADR-0004）。
- GPUサーバーは、停止時の動作を「終了」にし、起動から一定時間（例：3時間）で自動で `shutdown` する。
- 作業の終わりに `terraform destroy` を必ず実行し、手順をREADMEに書く。

## おすすめ

**us-east-1 の g6.xlarge を、オンデマンドで使うときだけ Terraform で作る。** エージェントの実行環境は同じゾーンの t3.small（足りなければ t3.medium）で、これも使うときだけ作る。

理由：

- M0の費用は月1,000〜2,000円程度で、予算に十分収まる。
- 24GBのVRAMで、7Bクラスを量子化せずに動かせる。12Bクラスも8bitなら載る。モデルの選択肢を狭めない。
- 東京より約30%安い。評価はバッチで動かすので、米国との通信の遅さは問題にならない。
- us-east-1 は利用者が多く、インスタンスの在庫が多い。us-west-2 も同じ価格なので、在庫がないときの予備にする。
- 費用を最優先するなら g4dn.xlarge（約2/3の価格）。ただし16GBでbf16に対応しないため、量子化したモデルが前提になる。

Windows PC は、利用枠の申請が通るまでの間や、モデルを手元で試すときの予備とする（ADR-0005の提案のとおり）。

## まだ確認が必要なこと

- GPUの利用枠の申請が、このアカウントで通るか。通るまでの時間（公式の目安は見つからなかった）。
- アカウントが新しい無料利用枠の対象か。Free plan でGPUインスタンスを起動できるか。クレジットがGPUの料金に充てられるか。
- 消費税の扱い（請求書で確認）。カード会社の為替の手数料。
- GPUサーバーの起動用のイメージ（Deep Learning AMI など）が必要とするディスクのサイズ。
- 選んだモデルの実際のVRAMの使用量と、1回の推論の時間（GPUの時間の見積もりに影響する）。
- エージェントの実行環境に必要なメモリ（2GiBで足りるか）。
- スポットの価格は変動するため、使う場合はその時点の価格を確認する。

## 出典

- AWS 公式の価格データ（オンデマンドのEC2、us-east-1）：https://b0.p.awsstatic.com/pricing/2.0/meteredUnitMaps/ec2/USD/current/ec2-ondemand-without-sec-sel/US%20East%20(N.%20Virginia)/Linux/index.json
- AWS 公式の価格データ（オンデマンドのEC2、東京）：https://b0.p.awsstatic.com/pricing/2.0/meteredUnitMaps/ec2/USD/current/ec2-ondemand-without-sec-sel/Asia%20Pacific%20(Tokyo)/Linux/index.json
- AWS 公式の価格データ（オンデマンドのEC2、us-west-2）：https://b0.p.awsstatic.com/pricing/2.0/meteredUnitMaps/ec2/USD/current/ec2-ondemand-without-sec-sel/US%20West%20(Oregon)/Linux/index.json
- Amazon EC2 オンデマンド料金：https://aws.amazon.com/ec2/pricing/on-demand/
- AWS 公式の価格データ（EBS）：https://b0.p.awsstatic.com/pricing/2.0/meteredUnitMaps/ec2/USD/current/ebs.json
- Amazon EBS の料金：https://aws.amazon.com/ebs/pricing/
- AWS Price List（S3、東京）：https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonS3/current/ap-northeast-1/index.json
- AWS Price List（S3、us-east-1）：https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonS3/current/us-east-1/index.json
- AWS Price List（データ転送、東京）：https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AWSDataTransfer/current/ap-northeast-1/index.json
- AWS Price List（データ転送、us-east-1）：https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AWSDataTransfer/current/us-east-1/index.json
- Amazon VPC の料金（パブリックIPv4）：https://aws.amazon.com/vpc/pricing/
- Vantage（スポットの価格の参考）：https://instances.vantage.sh/aws/ec2/g4dn.xlarge 、https://instances.vantage.sh/aws/ec2/g6.xlarge 、https://instances.vantage.sh/aws/ec2/g5.xlarge 、https://instances.vantage.sh/aws/ec2/g6e.xlarge
- Amazon EC2 のインスタンスタイプの利用枠：https://docs.aws.amazon.com/ec2/latest/instancetypes/ec2-instance-quotas.html
- 利用枠の申請についての利用者の質問と回答：https://repost.aws/questions/QUSPetOgBLTQuxHpy_ffhiAg/instance-launch-failed 、https://www.repost.aws/questions/QUvN3JWV2IRamSM5FQv7yCig/problem-launching-instance-of-ec2-server-for-gpu-intensive-applications
- AWS Free Tier のプラン：https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/free-tier-plans.html
- AWS Free Tier の更新（2025年7月）：https://aws.amazon.com/about-aws/whats-new/2025/07/aws-free-tier-credits-month-free-plan/
- EC2 の無料枠の対象のインスタンス：https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-free-tier-usage.html
- AWS Budgets の料金：https://aws.amazon.com/aws-cost-management/aws-budgets/pricing/
- AWS Budgets のアクション：https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-controls.html
- AWS Budgets のベストプラクティス（更新の頻度）：https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-best-practices.html
- Amazon CloudWatch の料金：https://aws.amazon.com/cloudwatch/pricing/
- AWS の消費税 FAQ：https://aws.amazon.com/jp/c-tax-faqs/
- 為替（欧州中央銀行の参照レート、Frankfurter 経由）：https://api.frankfurter.app/2026-10-15..?from=USD&to=JPY
- 電気代の目安単価：https://www.jlma.or.jp/siryo/pdf/kokai/guideA139-2024.pdf 、https://bestcalendar.jp/tools/denki-ryokin
- RTX 3080 の仕様（消費電力）：https://www.gamingonlinux.com/2020/09/the-nvidia-geforce-rtx-3080-is-out-today-some-details-for-you/page=1
- Qwen2.5-VL 7B の GGUF：https://huggingface.co/ggml-org/Qwen2.5-VL-7B-Instruct-GGUF
- Qwen2.5-VL 7B の必要なVRAMの目安：https://bestllmfor.com/catalog/qwen25-vl-7b/
- Gemma 3 12B の必要なVRAMの目安：https://hardwarepedia.com/ai/gemma-3-12b
