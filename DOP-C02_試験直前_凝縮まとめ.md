# DOP-C02 試験直前 凝縮まとめ（マーク60問・分野別要点集約版）

> **対象**: 演習1-3(全226問)中、**マーク済60問**（不正解29問 + 正解だがマーク済31問）
> **凡例**: ✗=不正解 / 🏷=マーク済(正解したが自信が薄い)
> **本資料の狙い**: 類似問題を分野グループに集約し、各グループの **共通教訓** + 個別問題は **1行要約** に凝縮。
> **DOP特有の注意**: SAPが「どのサービスを選ぶか」なら、DOPは **「そのサービスのどの機能・どのフック・どの順序か」** を問う。選択肢は全部それらしく見えるので、**キーワード1個の正誤**で切る。

---

## 📊 苦手分野の俯瞰

| ドメイン | 試験比率 | 苦手数 | ✗不正解 | 🏷マーク済のみ | 誤答率 |
|---|---|---|---|---|---|
| **D1 SDLCの自動化** | 22% | 15問 | 7 | 8 | 47% |
| **D2 構成管理とIaC** | 17% | 13問 | 6 | 7 | 46% |
| **D3 回復力の高いクラウドソリューション** | 15% | 12問 | 4 | 8 | 33% |
| **D4 モニタリングとロギング** | 15% | 3問 | 2 | 1 | 67% |
| **D5 インシデント・イベント対応** | 14% | 3問 | 2 | 1 | 67% |
| **D6 セキュリティとコンプライアンス** | 17% | 14問 | 8 | 6 | 57% |
| **合計** | 100% | **60問** | 29 | 31 | 48% |

**優先度**: 「誤答率 × 試験比率」で **D6 > D1 > D2**。

- **D6(誤答8/14)** … 落とし方が「SCPが効く範囲」「信頼関係の向き」に集中。**型で覚え直せば一番伸びる**。
- **D1(誤答7/15)** … 最大配点ドメイン。フック名・デプロイ構成名の**丸暗記が効く**領域。
- **D4/D5** はマーク数自体が少ないが誤答率が高い。**出題は各15%/14%あるので油断禁物**。

---

## 🎯 全体キーワード早見表(頻出のみ)

### CI/CD（CodeDeploy / CodePipeline / CodeBuild）
- **`CodeDeployDefault.HalfAtATime`** = 一度に半数（「新インスタンスの半数を同時に」要件はこれ）
- **`BeforeInstall`** = リビジョン配置**前**（ライセンス/依存ファイルの先置き）
- **`BeforeAllowTraffic`** = 新環境がトラフィックを受ける**直前**（最終前提条件チェック、一時ファイル削除、DB変更待ち）
- **`Install` / `BlockTraffic` / `AllowTraffic` はCodeDeploy予約イベント** → スクリプト実行**不可**
- **Lambdaデプロイのフックは `BeforeAllowTraffic` / `AfterAllowTraffic` の2つだけ**
- **`AWS::CodeDeployBlueGreen` トランスフォーム + `AWS::CodeDeploy::BlueGreen` フック** = **CFnで**ECSブルー/グリーン(`TrafficRoutingConfig`)
- **SAM `AutoPublishAlias` + `DeploymentPreference`** = Lambdaカナリア(`LambdaCanary10Percent10Minutes`)
- **単一CodeDeployアプリ + 複数デプロイメントグループ** = 並列デプロイ + 環境ごとの個別ロールバック
- **CodeBuildでGitタグ** = ネイティブGit + `git-credential-helper: yes`（CodeCommit APIにタグ作成はない）
- **ECRへpush** = `aws ecr get-login-password` → `docker login` が**必須**（IAM権限だけでは通らない）
- **CodeArtifact** = **1リポジトリにつき外部接続は1つ**、パブリックリポジトリは**アップストリーム**として繋ぐ
- **オンプレCI/CDからAWSへ** = **IAM Roles Anywhere**（トラストアンカー + 証明書）

### CloudFormation / IaC
- **`cfn-init`** = 起動時に構成適用 / **`cfn-hup`** = メタデータ変更を検知して**再適用** / **`cfn-signal` + `WaitOnResourceSignals`** = UserData完了までスタックを待たせる
- **CFnサービスロール** = 開発者は ReadOnly + cfn操作 + `iam:PassRole`、ロールの信頼は `cloudformation.amazonaws.com`
- **ドリフトの通知** = Config `cloudformation-stack-drift-detection-check` + EventBridge(コンプライアンス変更) → SNS
- **IaCジェネレーター** = 既存リソースをスキャンして **CDKアプリを直接生成**（テンプレ経由の手作業は不要）
- **SAM `no changes to deploy`** = `CodeUri`がS3パスのままだと**コード変更を検知しない** → ローカルディレクトリを指す
- **AMI ID配布** = Image Builder → **Parameter Store** → CFnの**SSMパラメータ型** / 起動テンプレは `resolve:ssm:`

### Systems Manager
- **`aws:downloadContent`** = GitHub/S3/HTTP/SSMDocument から直接取得（S3を中継させる選択肢は不正解）
- **State Manager アソシエーション** = 全マネージドノードに**継続適用**、新規インスタンスにも自動適用
- **`AWS::SSM::Association` + タグターゲティング** = CFnからランブックを紐付け（`AWS-JoinDirectoryServiceDomain`）
- **ハイブリッドアクティベーション** = オンプレ/物理端末をマネージドノード化

### 監視・イベント
- **ECSタスク終了の追跡** = EventBridge `ECS Task State Change` → CloudWatch Logs → **Logs Insights**
- **VPCフローログ** = **ACCEPTのみ**に絞ってコスト削減 → メトリクスフィルタ → アラーム → SNS
- **AMP(Prometheus)アラート** = アラートルール + **Alert Manager(`sns_configs`)** + **SNS側のリソースポリシー**で `aps.amazonaws.com` 許可
- **`s3:Replication:OperationFailedReplication` は EventBridge 経由で来ない** → SQS/SNS/Lambda へ**直接**配信
- **クロスアカウントイベント** = 送信側ルールのターゲットに相手の**デフォルトイベントバス** + 受信側バスの**リソースポリシー**

### セキュリティ・マルチアカウント
- **SCPは管理アカウントには効かない** → パーミッションセット側で **Deny + `aws:PrincipalAccount`**
- **「アカウント内で特定APIを禁止」= SCP**（IAMポリシー/パーミッションセットの個別修正より確実・効率的）
- **信頼ポリシーは「引き受けられる側」に置く**。Assumeする側には `sts:AssumeRole` 許可（**方向を間違えない**）
- **Organizations API(`CreateAccount`等)は管理アカウントのみ** → 管理アカウントにロールを作りAssume
- **予防+検出の両方 & 将来アカウントにも自動適用 = Control Tower**（Configコンフォーマンスパックは検出のみ）
- **検出 + 自動修復 = Config マネージドルール + SSM Automation ランブック**
- **Security Hub 組織運用** = 信頼されたアクセス + **委任管理者** + **自動登録**
- **API Gateway**: IAM認証 + **SigV4署名** / **mTLS** = ACM Private CA + **S3上のCA証明書をトラストストア**
- **OIDC連携** = IAM IDプロバイダー作成 → **`aud`条件**付き信頼ポリシー → `AssumeRoleWithWebIdentity`

### 回復力・DR
- **RPO/RTOが厳しい×マルチリージョン** = **Aurora Global Database** + **Route 53フェイルオーバー**（両方必要）
- **AZ障害の局所化** = **ターゲットグループ**でクロスゾーン無効 + **ARC ゾーナルシフト**
- **S3で15分SLA = S3 RTC** / **自動フェイルオーバー = MRAP アクティブ-パッシブ + `SubmitMultiRegionAccessPointRoutes`**
- **既存オブジェクトの複製 = S3 Batch Operations**（レプリケーションは設定後の新規のみ）
- **SMB + NFS 両対応の共有ストレージ = FSx for NetApp ONTAP**（EFSはNFSのみ、EBSは同時マウント不可）
- **起動が遅い = ウォームプール(Stopped) + `EC2_INSTANCE_LAUNCHING` ライフサイクルフック**
- **SQS標準 + Lambda = `ReportBatchItemFailures`** で部分再処理
- **非同期Lambdaの取りこぼし** = 失敗時 **Destination(SQS)** / **SNS→SQS→Lambda(ポーリング)** に変更

---

## 🚀 D1: SDLCの自動化 (15問) ⚠️ 最大配点22%

### G1-1. CodeDeploy: ライフサイクルフックとデプロイ構成 (3問) ⚠️ 頻出

**共通教訓**:
- フックの選択は **「何の直前/直後にやりたいか」を文章から拾う** だけ。
  - 「アプリ配置前に必要なファイルを置く」→ **`BeforeInstall`**
  - 「トラフィックを受け始める前に〜」→ **`BeforeAllowTraffic`**
- **`Install` / `BlockTraffic` / `AllowTraffic` はCodeDeploy予約**でスクリプト実行不可。選択肢に出たら罠。
- **`BeforeBlockTraffic` / `AfterBlockTraffic` は「元(ブルー)環境」側**の処理。新環境の準備には使えない。
- **Lambdaプラットフォームのフックは `BeforeAllowTraffic` / `AfterAllowTraffic` のみ**。`BeforeInstall`はEC2/オンプレ用。
- デプロイ構成は **標準を優先**。「半数ずつ」= `CodeDeployDefault.HalfAtATime`（カスタム構成でminimum healthy hosts 50%を作る選択肢は"より複雑"で不正解）。

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [✗] 演習1-問題35 | ALB+ブルー/グリーン+ASG自動コピー+**`HalfAtATime`**+**`BeforeAllowTraffic`**で一時ファイル削除 |
| [🏷] 演習2-問題73 | ライセンスファイルの先置き = **`BeforeInstall`**（Install前に必要なので`BeforeAllowTraffic`では遅い） |
| [✗] 演習3-問題37 | Lambda: DB変更完了までトラフィックを流さない = **`BeforeAllowTraffic`** |

---

### G1-2. デプロイ戦略のIaC実装（SAM / CFn） (3問)

**共通教訓**:
- **CFnでECSブルー/グリーン** → **`AWS::CodeDeployBlueGreen` トランスフォーム + `AWS::CodeDeploy::BlueGreen` フックパラメータ + `TrafficRoutingConfig`**。
  - 「AppSpecに`CodeDeployDefault.ECSLinear10PercentEvery1Minutes`を書く」は**CFnテンプレートの問い**に対する答えではない（相当する設定は`TimeBasedLinear`）。
- **Lambdaカナリア** → SAMの **`AutoPublishAlias` + `DeploymentPreference`**（`LambdaCanary10Percent10Minutes`）+ **関数ごとの個別CloudWatchアラーム**（複合アラームは粒度が粗く不適）。
- **SAMの`no changes to deploy`** → `CodeUri`がS3パス直書きだと**テンプレート差分がなく変更を検知しない**。**ローカルディレクトリに変更**して`sam deploy`（または`cloudformation package`+`deploy`）。

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [✗] 演習1-問題62 | ECSブルー/グリーンをCFnで = **CodeDeployBlueGreenトランスフォーム + TimeBasedLinear(10%/1分)** |
| [🏷] 演習1-問題30 | `CodeUri`をローカルディレクトリへ → `sam deploy` / `cfn package`+`deploy` の2択が正解 |
| [🏷] 演習3-問題69 | SAM(`AutoPublishAlias`+`DeploymentPreference`) + 関数別CWアラーム + CodeCommit/Pipeline/Build |

---

### G1-3. パイプライン構成（マルチ環境・並列・クロスアカウント） (4問)

**共通教訓**:
- **環境の分離はリポジトリではなく「ブランチ」**。単一リポジトリ + 環境別ブランチ + 環境別パイプライン、本番は**手動承認**。
  - リポジトリを環境ごとに分けるのは同期・マージが増える**アンチパターン**。
- **複数の独立環境へ並列デプロイ + 個別ロールバック** → **単一CodeDeployアプリ + 環境ごとのデプロイメントグループ**（アプリやパイプラインを増やさない）。
- **クロスアカウントデプロイの403** → **KMS(カスタマーマネージドキー)** と **S3バケットポリシー** の**両方**を疑う。
  - AWSマネージドキーは**ポリシーを編集できない**ので、クロスアカウントでは**CMKが必須**。
  - IAMロール側の許可だけでは不十分（**リソースベースポリシーとの両側許可**）。
- パイプラインの後処理（SDK生成/アップロード/キャッシュ無効化）は**Lambdaアクションでパイプラインに組み込む**。

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [✗] 演習2-問題13 | **単一リポジトリ + 環境別ブランチ** + 環境別パイプライン + 本番に手動承認 |
| [🏷] 演習3-問題29 | 単一パイプライン + **単一CodeDeployアプリ + 複数デプロイメントグループ**で並列デプロイ |
| [🏷] 演習3-問題36 | クロスアカウント: **CMK作成+kms:Decrypt許可** と **S3バケットポリシー**で許可（両方） |
| [🏷] 演習3-問題1 | API更新後のSDK配布 = デプロイ直後にLambdaアクション(SDK取得→S3→CloudFront無効化) |

---

### G1-4. アーティファクト・レジストリ管理 (4問) ⚠️ 誤答多い

**共通教訓**:
- **ECRのdocker操作にはログインが要る**: `aws ecr get-login-password` → `docker login`。**IAM権限の付与だけでは失敗する**（S3と違いDockerクライアントはSigV4署名しない）。
- **クロスアカウントでのイメージ取得** → **元アカウントのリポジトリにリポジトリポリシー**を書く。**新アカウントにリポジトリを作り直さない**（CI/CDの整合性が壊れる）。
  - **完全プライベート化には ECR VPCエンドポイント + S3ゲートウェイエンドポイントの両方**（レイヤー実体はS3にある）。
- **スキャン前後のイメージ分離** → 共有サービスアカウントに**リポジトリを2つ**（前=書き込み許可、後=読み取り許可）+ **EventBridgeのスキャン完了イベント → Lambdaで評価・移動**。リポジトリごとにパイプラインを作るのは管理過多。
- **CodeArtifact**: **1リポジトリ = 1外部接続**。パブリックリポジトリごとにCodeArtifactリポジトリを作り、**アップストリーム**として接続（ダウンストリームではない）。

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [🏷] 演習3-問題58 | ECRアクセス失敗 = **`aws ecr get-login-password` + `docker login`** が抜けている |
| [✗] 演習2-問題50 | クロスアカウントpull = **元アカウント側のリポジトリポリシー** + ECR/S3エンドポイント + タスク実行ロール |
| [✗] 演習1-問題65 | 共有アカウントにECR前/後リポジトリ+リソースポリシー、**EventBridge(スキャン完了)+Lambda**で移動 |
| [✗] 演習2-問題62 | パブリックごとにCodeArtifactリポジトリ+**外部接続をアップストリーム** + **IAM Roles Anywhere** |

---

### G1-5. CodeBuildの実装テクニック (1問)

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [🏷] 演習1-問題70 | Gitタグ付け = **ネイティブGitでclone→tag→push**（`git-credential-helper: yes`）。CodeCommit APIにタグ作成はない |

---

## 🏗️ D2: 構成管理とIaC (13問)

### G2-1. cfnヘルパースクリプト (3問) ⚠️ 3つの役割を混同しない

**共通教訓**:
- **`cfn-init`** = スタックの**メタデータを読んで構成を適用**（起動時1回）
- **`cfn-hup`** = メタデータの**変更を検知して再実行**（実行中インスタンスの更新）
- **`cfn-signal` + `UpdatePolicy: AutoScalingRollingUpdate(WaitOnResourceSignals)`** = **UserDataの成否をCFnに通知**し、失敗ならスタック更新を失敗/ロールバック
- **「スタックはUPDATE_COMPLETEなのにアプリが壊れている」= cfn-signalが無い**のサイン
- **「UserDataを変えてもスタック更新で既存インスタンスに反映されない」** → cfn-init/cfn-hup **または** SSM State Manager（**インスタンス置換なしで**適用できる2択）

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [✗] 演習2-問題24 | 既存インスタンスへ反映 = **cfn-init+cfn-hup** / **SSMドキュメント+State Manager** の2つ |
| [🏷] 演習3-問題67 | 構成ファイルをテンプレのinitメタデータに埋め込み → cfn-init + cfn-hupで追従（ソース管理要件も満たす） |
| [🏷] 演習3-問題70 | UserData失敗時にデプロイを失敗させる = **cfn-signal + WaitOnResourceSignals** |

---

### G2-2. Systems Managerによる構成管理 (3問)

**共通教訓**:
- **外部リポジトリからの取得は `aws:downloadContent` + `sourceType: GitHub` + `sourceInfo`**。**S3を中継させる選択肢は「不要に複雑」で不正解**。
  - `aws:configurePackage`=Distributorのパッケージ用 / `aws:softwareInventory`=インベントリ収集用（ダウンロード不可）。
- **「全インスタンスに適用し、追加された新規インスタンスにも自動で」= State Manager アソシエーション**。Config/Inspector/CloudTrailは検出はできても**インストールはできない**。
- **CFnからのSSM連携** = **`AWS::SSM::Association`** リソース + **タグでターゲット指定** + **AWSマネージドランブック**（`AWS-JoinDirectoryServiceDomain`）。
  - ドメイン参加には **`AmazonSSMManagedInstanceCore` + `AmazonSSMDirectoryServiceAccess`** の2つ。

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [🏷] 演習1-問題3 | GitHubのスクリプト適用 = **`aws:downloadContent` + sourceType=GitHub + sourceInfo** |
| [🏷] 演習1-問題5 | 全ノードへウイルス対策を継続適用 = **State Manager アソシエーション** |
| [✗] 演習2-問題35 | AD参加 = 起動テンプレにタグ + **`AWS::SSM::Association`** + `AWS-JoinDirectoryServiceDomain` |

---

### G2-3. AMIのライフサイクルと配布 (2問)

**共通教訓**:
- **Image Builderの配布設定(Distribution Settings)** で **起動テンプレートを直接更新**できる。EventBridge+Lambdaの自作は「標準機能の迂回」で不正解。
- **AMI IDの共有は Parameter Store が正解ルート**。CFn側は **SSMパラメータ型**（または`resolve:ssm:`）で参照 → **テンプレート変更なしで常に最新**。
  - SNS通知 + 各チームがLambdaでスタック更新、はスケールしない。

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [🏷] 演習2-問題8 | 最新AMIを起動テンプレへ = **Image Builderの配布設定で直接更新** |
| [✗] 演習3-問題53 | AMI ID配布 = **Image Builder → Parameter Store → CFnのSSMパラメータ型で参照** |

---

### G2-4. CloudFormation運用（ドリフト・サービスロール・UserData） (4問)

**共通教訓**:
- **ドリフトの「検知して通知」** = Lambdaを2個作るのではなく **AWS Config `cloudformation-stack-drift-detection-check` + EventBridge(コンプライアンス変更) → SNS**。**マネージド機能で置き換えられないか**を最初に疑う。
- **「変更はCloudFormation経由のみ」の実装3点セット**:
  1. 開発者ロール: **AdministratorAccess剥奪 → ReadOnlyAccess + cfnスタック操作権限 + `iam:PassRole`**
  2. デプロイ用ロールの**信頼ポリシー**: `cloudformation.amazonaws.com`
  3. デプロイ用ロールに `cloudformation:*` + **`iam:PassRole`(`iam:PassedToService`=cloudformation)**
- **ECSインスタンスが誤ったクラスタに入る** = 起動時にクラスタ名が渡っていない → **LaunchConfiguration/起動テンプレの`UserData`に`!Ref`でクラスタ名**を書き込む。

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [✗] 演習2-問題21 | ドリフト通知 = **Config管理ルール + EventBridge → SNS**（自作Lambdaは不要） |
| [✗] 演習1-問題45 | 開発者ReadOnly+cfn権限+PassRole / 信頼は`cloudformation.amazonaws.com` / PassedToService条件 |
| [🏷] 演習2-問題67 | ECSクラスタ誤登録 = **UserDataにクラスタ参照**を書く |

---

### G2-5. 既存リソースのIaC化 (1問)

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [🏷] 演習3-問題57 | 手動作成Lambdaの取り込み = **IaCジェネレーターの部分スキャン → CDKアプリを直接生成 → `from_asset`** |

**補足**: `Code.ZipFile`(インライン)は**4,096バイト制限**、`ImageUri`は**コンテナ形式専用**。zip形式のLambdaには使えない。

---

### G2-6. Control Towerのカスタムオプション提供 (1問)

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [✗] 演習2-問題49 | **CFnテンプレート + Service Catalog製品 + `AWSControlTowerBlueprintAccess`ロール**（信頼: `AWSControlTowerAdmin` / 権限: `AWSServiceCatalogAdminFullAccess`）※StackSetではない |

---

## 🛡️ D3: 回復力の高いクラウドソリューション (12問)

### G3-1. S3レプリケーション (3問) ⚠️ 頻出

**共通教訓**:
- **レプリケーションは「設定後の新規オブジェクト」だけ**。**既存オブジェクトは S3 Batch Operations(バッチレプリケーション)** で別途複製する。
- 両リージョンで読み書きするなら **双方向レプリケーションルール**（片方向の選択肢は要件を満たさない）。
- **クロスアカウントCRRの3点セット**:
  1. **ソースアカウント**でレプリケーション用**IAMロール**作成
  2. **ソースバケット**にレプリケーションルール
  3. **ターゲットバケットのバケットポリシー**でそのIAMロールを許可
- **15分以内のSLA = S3 RTC**（99.99%が15分以内、CloudWatchで遅延監視）。
- **リージョン障害時の自動切替 = MRAP アクティブ-パッシブ + `SubmitMultiRegionAccessPointRoutes` API**。Transfer Accelerationは**転送高速化であってフェイルオーバーではない**。

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [🏷] 演習1-問題6 | IAMロール(S3+S3 Batch Operationsプリンシパル) + **Batch Operationsで既存複製** + 双方向ルール |
| [✗] 演習1-問題29 | **S3 RTC(15分)** + **MRAPアクティブ-パッシブ** + `SubmitMultiRegionAccessPointRoutes` |
| [🏷] 演習3-問題8 | クロスアカウント: ソースにIAMロール+ルール、**ターゲットのバケットポリシー**で許可 |

---

### G3-2. マルチリージョンDR (2問)

**共通教訓**:
- **RPO≦2h / RTO≦10min / リージョン障害対応** → **Aurora Global Database(遅延1秒未満・数分で昇格)** ＋ **Route 53フェイルオーバールーティング + ヘルスチェック + 各リージョンのASG**。**DB層とアプリ層の両方**が要る。
- **NLBでリージョン間のDBリクエストを分散**、のような選択肢は**成立しない**（NLBはリージョナル）。
- **リードレプリカ昇格の自動化** = **EventBridgeで障害検知 → Lambdaで昇格 + Parameter Storeのエンドポイント更新 → アプリは接続エラー時に再取得**。

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [✗] 演習1-問題22 | **Aurora Global Database** + **Route 53フェイルオーバー+ヘルスチェック+ASG**（2つセットで正解） |
| [🏷] 演習1-問題31 | 昇格自動化 = **EventBridge + Lambda + Parameter Store**、アプリはエンドポイントをリロード |

---

### G3-3. AZ障害の局所化とスケーリング最適化 (3問)

**共通教訓**:
- **ALBのクロスゾーン負荷分散は「ロードバランサーレベルでは常時オン・変更不可」**。無効化は **ターゲットグループレベル**でのみ可能（この1点で選択肢が割れる）。
- **特定AZの切り離し = ARC(Application Recovery Controller)の ゾーナルシフト**。
- **起動が遅い問題** = **ウォームプール(Stopped)** で事前初期化 + **`autoscaling:EC2_INSTANCE_LAUNCHING` ライフサイクルフック**で準備完了後に`CompleteLifecycleAction`。
  - Stoppedは**EBS課金のみ**でコンピュート課金なし → コスト効率の要件に合う。
- **Windows(SMB) + Linux(NFS) 両対応の共有ストレージ = FSx for NetApp ONTAP**。EFSはNFSのみ、gp3は複数インスタンス同時アタッチ不可。
  - 新規インスタンスへは**起動テンプレのUserDataでマウント**、既存へは**インスタンスリフレッシュ**。

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [🏷] 演習1-問題26 | **ターゲットグループ**でクロスゾーン無効 + **ARC ゾーナルシフト** |
| [🏷] 演習3-問題45 | **ウォームプール(Stopped)** + `EC2_INSTANCE_LAUNCHING`フックで起動時間短縮とコスト両立 |
| [🏷] 演習1-問題33 | **FSx for NetApp ONTAP Multi-AZ** + 起動テンプレUserDataでマウント + **インスタンスリフレッシュ** |

---

### G3-4. 非同期処理のメッセージ消失対策 (2問)

**共通教訓**:
- **SNS→Lambdaは非同期呼び出し**。**関数コード内の失敗**は既定で**2回再試行(計3回)** した後**破棄**される。
  - **SNSサブスクリプションのDLQは「呼び出し自体の失敗」しか守らない** → コード内失敗には効かない（頻出の引っかけ）。
  - 対策は **①Lambdaの失敗時Destination(SQS)** と **②SNS→SQS→Lambdaのポーリング構成に変更** の2本立て。
- **SQS標準 + Lambda の重複処理**: **イベントソースマッピングに `FunctionResponseTypes: ReportBatchItemFailures`** を追加 → 失敗メッセージだけ再配信、成功分は削除。大きなバッチサイズが安全に使える。

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [✗] 演習2-問題10 | **Lambda失敗時Destination(SQS)** + **SNS→SQS→Lambda**（SNSのDLQでは守れない） |
| [🏷] 演習2-問題9 | **`ReportBatchItemFailures`** で部分的失敗のみ再処理 |

---

### G3-5. ネットワーク・データ整合性 (2問)

**共通教訓**:
- **多数VPC(CIDR重複あり)のサービス間通信** = **各VPCにNLB + PrivateLink(エンドポイントサービス)**。VPCピアリング/TGWは20VPC規模やCIDR重複で破綻。
- **S3アップロードの完全性検証** = **`Content-MD5`ヘッダーで送信時に検証** + **レスポンスの`ETag`とMD5を比較**（二重検証）。

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [🏷] 演習3-問題38 | 20VPC間のプライベート通信 = **NLB + PrivateLink**、DNS名で参照（変更が最小） |
| [✗] 演習3-問題10 | **`Content-MD5`送信** + **`ETag`比較** の2つ |

---

## 📈 D4: モニタリングとロギング (3問) ※誤答率高

**共通教訓**:
- **「タスク/リソースの状態変化を漏れなく記録」= EventBridge → CloudWatch Logs → Logs Insights**。
  - EC2にエージェントを入れてログを送る案は**タスクレベルの詳細（終了理由・終了コード）が取れない**。
  - **Contributor Insights は「上位貢献者の分析」**であって障害調査の詳細取得用ではない。
- **VPCフローログのコスト最適化** = **必要なaction(ACCEPT)だけキャプチャ** → **メトリクスフィルタ** → **アラーム(5分/1データポイント)** → SNS。
- **AMP(Prometheus)のSNSアラート3点セット**:
  1. **アラートルール**(PromQL)を作る
  2. **Alert Manager**の設定でSNSトピックを**レシーバー**に指定
  3. **SNSトピックのアクセスポリシー**で **`aps.amazonaws.com`** に `sns:Publish` / `sns:GetTopicAttributes` を許可（**AMP側のIAMロールではない**）
  - リモート書き込みURLは**メトリクス受信用**でアラート転送には使わない。

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [✗] 演習1-問題63 | ECSタスク終了の調査 = **EventBridge(Task State Change) → CloudWatch Logs → Logs Insights** |
| [✗] 演習2-問題70 | AMP = アラートルール + **Alert ManagerのSNSレシーバー** + **SNS側リソースポリシー(aps.amazonaws.com)** |
| [🏷] 演習3-問題72 | フローログ(ACCEPTのみ) → メトリクスフィルタ → アラーム → SNS（低コストで実現） |

---

## 🔄 D5: インシデント・イベント対応 (3問) ※誤答率高

**共通教訓**:
- **S3レプリケーション失敗の再試行** = **S3イベント通知 `s3:Replication:OperationFailedReplication` → Lambda → S3 Batch Operations**。
  - **このイベントタイプは EventBridge 経由では配信されない**（SQS/SNS/Lambdaへの直接配信のみ）。大規模なら**個別PutObjectではなくバッチ**。
- **クロスアカウントのイベント共有** = **発生側アカウントのEventBridgeルール**のターゲットに **相手アカウントのデフォルトイベントバス** を指定 + **受信側バスのリソースポリシー**で送信元を許可。
  - CloudTrailイベントは **API を実行したアカウント** に記録される、という前提の理解が問われる。
- **CodeDeployが「スキップ」** = デプロイ指示は出たが**エージェント側が動けていない**。
  - **①NAT/IGW欠如でCodeDeployエンドポイントに到達不可** **②インスタンスプロファイルの権限不足**
  - appspec.yml欠如などは「スキップ」ではなく**明確なエラー**になる（症状で切り分ける）。

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [✗] 演習1-問題12 | 失敗レプリカの再試行 = **S3イベント通知 → Lambda → S3 Batch Operations** |
| [🏷] 演習2-問題31 | クロスアカウント配信 = **相手のデフォルトイベントバスをターゲット + 受信側リソースポリシー** |
| [✗] 演習3-問題73 | 「スキップ」= **NAT/IGW不足** と **インスタンスプロファイル権限不足** |

---

## 🔐 D6: セキュリティとコンプライアンス (14問) ⚠️ 最重要・誤答8問

### G6-1. SCPと権限の境界 (3問) ⚠️ 最頻出の落とし穴

**共通教訓**:
- **SCPは「管理アカウント」には適用されない**。管理アカウント内での制限は **パーミッションセット/IAMポリシー側で明示的Deny** + **`aws:PrincipalAccount`（管理アカウントIDと比較）**。
  - 「管理アカウント用のOUを作ってSCPを当てる」は**無効**（OUに移しても管理アカウントは対象外）。
- **メンバーアカウントで「特定APIをそもそも実行させない」= SCP**。パーミッションセットにDenyポリシーを追加する案は、**そのセットを使う人にしか効かず、他の経路を塞げない**ので「最も効率的」ではない。
- **IPアドレスでAWS APIの実行を制限** = **SCP + `aws:SourceIp`**（許可ではなく**明示的Deny**）。
  - Firewall Manager/Network Firewall は**ネットワーク層**、GuardDutyは**検出専用**で要件を満たさない。「アクションの実行をブロック」= API/認可層と読み替える。

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [✗] 演習1-問題4 | 範囲外IPの拒否 = **SCP + `aws:SourceIp`のDeny をルートにアタッチ** |
| [✗] 演習1-問題34 | **SCPは管理アカウントに効かない** → パーミッションセットで`sso:*`/`sso-directory:*`をDeny + `aws:PrincipalAccount` |
| [✗] 演習1-問題61 | `iam:CreateUser`禁止 = **SCPをそのアカウントにアタッチ**（パーミッションセット修正より確実） |

---

### G6-2. クロスアカウントのロール設計 (4問) ⚠️ 「向き」で落とす

**共通教訓**:
- **鉄則**: **信頼ポリシー(誰に引き受けさせるか)は「リソースを持つ側=引き受けられるロール」に置く。`sts:AssumeRole`許可は「呼ぶ側」に置く。**
  - 管理アカウントのLambdaがメンバーの情報を読む → **メンバー側にロール(+信頼:管理アカウント)** / **管理側ロールに`sts:AssumeRole`**。
  - オペレーションチームが各ワークロードを管理 → **各ワークロードにロール(信頼:オペレーションアカウント)** / オペレーション側は**IAMグループにAssumeRoleポリシー**。
- **Organizations API(`CreateAccount`等)は管理アカウントのみ実行可**。別アカウントのLambdaからは **管理アカウントのロールをAssumeRole**（信頼ポリシーで**Lambda実行ロールだけ**に限定）。
- **EKSは2段階**: ①IAMのクロスアカウント信頼関係 ②**`aws-auth` ConfigMap**でIAMロールをKubernetesのグループにマッピング。**IAM権限だけでは`Unauthorized`**。

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [🏷] 演習1-問題24 | Organizations API = **管理アカウントにロール** + 信頼は専用アカウントのLambda実行ロールのみ |
| [✗] 演習2-問題41 | **各ワークロードにSysAdminロール(信頼:オペレーション)** + オペレーション側はIAMユーザー+グループ |
| [✗] 演習3-問題66 | **メンバー側にEC2ReadOnlyロール+信頼(管理)** + **管理側ロールに`sts:AssumeRole`** |
| [🏷] 演習3-問題20 | EKSの`Unauthorized` = 信頼関係 + **`aws-auth` ConfigMap**のマッピング |

---

### G6-3. 組織全体のガバナンス (4問)

**共通教訓**:
- **「予防的 + 検出的の両方」「将来のアカウントにも自動適用」 = AWS Control Tower**。
  - Configコンフォーマンスパックは**検出のみ**、StackSetsは**新規アカウントへの手動追従**が必要。
- **Control Towerの検出型コントロール(ガードレール)の非準拠 → EventBridge → SNS(監査アカウント)** が最小構成。カスタムLambdaは不要。
- **Security Hub の組織運用3点**: **信頼されたアクセスの有効化** + **専用セキュリティアカウントを委任管理者** + **自動登録の有効化**。
  - アクセス範囲の出し分けは **IAM Identity Center のアクセス許可セット**（SCPで「Security Hubを見せない」は粗すぎて不正解）。
- **「即座に適用」の要件 = 検出 + 自動修復**: **Config マネージドルール + SSM Automation ランブック(`AWS-EnableS3BucketEncryption`)**。

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [✗] 演習1-問題59 | 予防+検出+将来のアカウント = **Control Tower + OU + コントロール** |
| [🏷] 演習2-問題28 | RDS暗号化の検出 = **Control Tower検出型ガードレール + EventBridge → SNS(監査アカウント)** |
| [✗] 演習3-問題12 | **信頼されたアクセス + 委任管理者 + 自動登録** + IAM Identity Centerの許可セット |
| [🏷] 演習3-問題18 | S3暗号化強制 = **Config管理ルール + SSM Automationランブックで自動修復** |

---

### G6-4. API・アプリケーションの認証 (3問)

**共通教訓**:
- **`User: anonymous is not authorized`** = リクエストが**SigV4署名されていない**。対策は **①全メソッドで認証方式にAWS_IAMを設定** ＋ **②クライアントがSigV4で署名**（両方セット）。
- **mTLS on API Gateway** = **ACM Private CAでクライアント証明書を発行** + **ルートCA証明書をS3に置き、トラストストアとして参照**。個別のクライアント証明書や秘密鍵を信頼基点にはできない。
- **OIDCのみ対応のIdP → AWS** の3手順: **①IAM OIDC IDプロバイダー作成(URL/audience/署名)** → **②IAMロールの信頼ポリシーに`<provider>:aud`条件** → **③アプリが`AssumeRoleWithWebIdentity`**。
  - IAM Identity Centerは**SAMLベースの人間のSSO用**で、アプリのAPI呼び出しには不適。

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [🏷] 演習1-問題44 | anonymousエラー = **全メソッドにIAM認証** + **SigV4署名** |
| [🏷] 演習2-問題30 | mTLS = **ACM Private CAでクライアント証明書** + **S3のCA証明書をトラストストア参照** |
| [✗] 演習2-問題37 | OIDC = **IAM IDプロバイダー作成** + **aud条件の信頼ポリシー** + **AssumeRoleWithWebIdentity** |

---

## 🎯 試験前日にこれだけ見る(超要点)

### D1(22%)で最頻出の判断ポイント
1. **`BeforeInstall`=配置前 / `BeforeAllowTraffic`=トラフィック前**、`Install`/`BlockTraffic`/`AllowTraffic`は**予約**
2. **Lambdaのフックは2つだけ**(`BeforeAllowTraffic`/`AfterAllowTraffic`)
3. **半数ずつ = `CodeDeployDefault.HalfAtATime`**（カスタム構成を作る選択肢は疑う）
4. **CFnでECSブルー/グリーン = `AWS::CodeDeployBlueGreen`トランスフォーム**
5. **SAMカナリア = `AutoPublishAlias` + `DeploymentPreference`** + 関数別アラーム
6. **環境分離はブランチ**、本番は手動承認、**リポジトリは分けない**
7. **ECRは`get-login-password`→`docker login`**、クロスアカウントは**元アカウントのリポジトリポリシー**
8. **CodeArtifactは1リポジトリ1外部接続 + アップストリーム**

### D2(17%)で最頻出の判断ポイント
1. **cfn-init=適用 / cfn-hup=追従 / cfn-signal=成否通知(WaitOnResourceSignals)**
2. **既存インスタンスの構成更新 = cfn-hup または State Manager**
3. **`aws:downloadContent`でGitHubから直接**（S3中継は不正解）
4. **新規インスタンスにも自動適用 = State Manager アソシエーション**
5. **AMI配布 = Image Builder配布設定 / Parameter Store + SSMパラメータ型**
6. **ドリフト通知 = Config管理ルール + EventBridge**
7. **既存リソースのIaC化 = IaCジェネレーター（CDKを直接生成）**

### D3(15%)で最頻出の判断ポイント
1. **既存オブジェクトは S3 Batch Operations**、新規のみがレプリケーション
2. **15分SLA = S3 RTC / 自動切替 = MRAP + SubmitMultiRegionAccessPointRoutes**
3. **RPO/RTO厳しい = Aurora Global DB + Route 53フェイルオーバー(両方)**
4. **ALBのクロスゾーンはターゲットグループでのみ無効化 + ARCゾーナルシフト**
5. **起動が遅い = ウォームプール(Stopped) + ライフサイクルフック**
6. **SMB+NFS = FSx for NetApp ONTAP**
7. **SQS部分失敗 = `ReportBatchItemFailures`** / **非同期Lambda = 失敗時Destination**

### D4(15%)で最頻出の判断ポイント
1. **状態変化の記録 = EventBridge → CloudWatch Logs → Logs Insights**
2. **フローログはactionを絞ってコスト削減 → メトリクスフィルタ → アラーム**
3. **AMP → SNS は「SNS側のリソースポリシーで`aps.amazonaws.com`」**

### D5(14%)で最頻出の判断ポイント
1. **`s3:Replication:OperationFailedReplication`はEventBridge非対応**、Lambda直呼び + **Batch Operations**
2. **クロスアカウントイベント = 相手のデフォルトバス + 受信側リソースポリシー**
3. **CodeDeploy「スキップ」= 到達性(NAT/IGW)とインスタンスプロファイル**

### D6(17%)で最頻出の判断ポイント
1. **SCPは管理アカウントに効かない** → パーミッションセットでDeny + `aws:PrincipalAccount`
2. **アカウント単位でAPIを封じる = SCP**
3. **信頼ポリシーは引き受けられる側、`sts:AssumeRole`は呼ぶ側**（向きを間違えない）
4. **予防+検出+将来アカウント = Control Tower**
5. **検出+自動修復 = Config + SSM Automation ランブック**
6. **Security Hub = 信頼されたアクセス + 委任管理者 + 自動登録**
7. **anonymousエラー = IAM認証 + SigV4** / **mTLS = ACM Private CA + S3のCA証明書**
8. **OIDC = IDプロバイダー + aud条件 + AssumeRoleWithWebIdentity**

---

## ⚡ 「迷ったらこう切る」DOP共通の判断軸

1. **マネージド標準機能 > 自作Lambda/EventBridge**（"最も効率的""運用負荷最小"はほぼこれ）
2. **中間ステップの少ない方**（S3を挟む、リポジトリを増やす、テンプレを経由する…は大体不正解）
3. **クロスアカウントは常に両側許可**（アイデンティティ側 + リソース側）
4. **「新しく追加されるリソースにも自動で」= State Manager / Control Tower / Config / 自動登録**
5. **症状から原因を逆算する問題**は、その症状**でしか起きない**原因を選ぶ（例: 「スキップ」=エージェントが動いていない）
6. **技術的に不可能な記述が混ざっていないか**を先に見る（ALBのクロスゾーンをLBレベルで無効化、NLBでリージョン間分散、ZipFileで大きなコード…）

---

**頑張ってください 🍀**
