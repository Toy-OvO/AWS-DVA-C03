# SAP-C02 試験前まとめ（マーク済52問の分析 - 演習1・2版）

> **試験日**: 1週間後。第一優先は合格。
> **本資料の範囲**: 演習1（24問）+ 演習2（28問）= 計52問。演習3・4は追加でまとめ予定。
> **ユーザー正誤内訳**: 正解 23問 / 不正解 29問（不正解率 56%）
> **構成**: ①苦手分野の俯瞰 → ②ドメイン別の頻出パターン → ③問題ごとのコア教訓 → ④試験当日の即決判断表

---

## ① 苦手分野の俯瞰

| 優先度 | ドメイン | 問題数 | 不正解率 | 状況 |
|---|---|---|---|---|
| 🔴 **最優先** | D2 新規ソリューション設計 | 18問 | **61%** (11/18) | **問題数最多かつ最も苦手**。設計系の"組合せ"が弱い |
| 🔴 高 | D3 既存ソリューションの継続的改善 | 15問 | **60%** (9/15) | コスト最適化・性能改善の選択で失敗しやすい |
| 🟠 中 | D1 複雑な組織向けの設計 | 8問 | 50% (4/8) | Organizations系の対策で回復可能 |
| 🟡 低 | D4 移行・モダナイゼーション | 11問 | 45% (5/11) | 移行系サービスの使い分けを整理すれば強化可能 |

**戦略**:
1. **D2の設計問題**を最優先で潰す。SAP-C02では複数のAWSサービスを組み合わせる問題が多く、キーワードから正しい組合せを即座に選べるようにする
2. **D3の"最も○○"問題**（最もコスト効率的/最も運用負荷が低い/最もパフォーマンスが良い）の"最適解"パターンを覚える
3. D1・D4はサービス選択の判断表（後述）で救える

---

## ② ドメイン別の頻出パターン・引っかけ

### D2 新規ソリューション設計（61%不正解・最優先）

**頻出の引っかけパターン**:

**〈ネットワーク・IP系〉**
- **サードパーティAPIホワイトリスト対応** → **BYOIP + Elastic IP + NAT Gateway**。カスタマー所有のパブリックIPブロックをAWSに登録することで、送信元IPを固定化できる。「NAT Gatewayに標準EIPを付ける」だけでは、そのEIPは事前登録できない
- **静的TCPポート + 高可用性 + 固定IPで許可リスト対応** → **NLB + AZ毎にEIP**。ALBはL7で不可。「ECS + パブリックIP + ALB」は誤り
- **CloudFront経由のみALBアクセスを許可** → ALBのSGに **CloudFrontのAWSマネージドプレフィックスリスト（`com.amazonaws.global.cloudfront.origin-facing`）** からのトラフィックのみ許可。CloudFrontとALBに共通ヘッダーを設定して検証する方法もあるが、SGレベルでの制限が最強
- **Lambda(VPC内) + 外部サービス統合** → **NAT Gateway + EIP**。Lambda VPC内のインターネットアクセスにはNAT必須
- **マルチリージョンで低レイテンシ + 固定IP要件** → **Global Accelerator**。Route 53 DNSベース(レイテンシルーティング)では固定IPにならない

**〈DR・高可用性〉**
- **単一EC2の重要アプリを最小コストでHA化** → **ELB + Auto Scaling(最小2/最大4)**。Multi-AZ ElastiCache Redisとの組合せ
- **アプリケーションのDR: EC2 + RDSを別リージョンへ** → **AWS Elastic Disaster Recovery (DRS) + RDSクロスリージョンリードレプリカ**。CloudEndure DRの後継。「AMIバックアップを別リージョンへコピー」よりも高速な復旧が可能
- **既存にマルチリージョンDR追加(CloudFront配下)** → 別リージョンに **ALB/ASG/EC2をプロビジョニング + CloudFrontに origin group** を設定してoriginフェイルオーバー
- **リアルタイム取引システム(RPO=0要求)** → **Aurora Global Database マルチリージョン + CloudTrail監査**。DynamoDB Globalではトランザクション整合性が保てない場合あり

**〈IoT・イベント駆動〉**
- **MQTT + 数千デバイス + リアルタイム** → **AWS IoT Core + thing + 証明書プロビジョニング**。IoT Coreがマネージドかつスケーラブル
- **異なるベンダーのIoT信号を変換して保管** → **IoT Core → Rule → Lambda → S3(.csv)**。Kinesis経由は過剰
- **マイクロサービス間のイベント連携** → **EventBridge カスタムイベントバス + 各サービスがルールをサブスクライブ**。SNSでも可能だが、フィルタリング能力・イベントスキーマ管理で EventBridge が優位

**〈サーバーレス・API〉**
- **サーバーレスAPI + DynamoDB直接返却** → **API Gateway HTTP API + Lambda + DynamoDB**。REST APIより低コスト・低レイテンシ
- **Lambda + 大きな共有ライブラリ** → **Lambdaのコンテナイメージ機能(ECR)**。Lambda Layerでは10GB未満のイメージ制限あり
- **CloudFront + Lambda@Edge**: グローバルなHTTP応答編集はLambda@Edge、リージョンイベント処理はLambda

**〈EBSスナップショット/DLM〉**
- **EBS スナップショットを別リージョンへ複製 + 3ヶ月保持** → **Amazon Data Lifecycle Manager (DLM) + クロスリージョンコピー**。EBS スナップショット管理の標準ソリューション

### D3 既存ソリューションの継続的改善（60%不正解・高優先）

**〈コスト最適化〉**
- **S3の階層自動判定** → **S3 Intelligent-Tiering**。アクセスパターンが読めない or 変動する場合の万能解。手動でライフサイクルルール設定は運用負荷高
- **DynamoDB のコスト最適化 + ピークあり** → **Application Auto Scaling(ピーク時) + Reserved Capacity(ベース)**。オンデマンドは高負荷向きだがコスト高
- **Compute Optimizer のリコメンド出力** → **Lambdaで ExportLambdaFunctionRecommendations を呼び S3 に出力 → QuickSight で可視化**
- **OpenSearch のログデータコスト削減** → **UltraWarm ノード**。書き込み頻度が低いデータをS3ベースのストレージへ

**〈パフォーマンス改善〉**
- **DynamoDB のレイテンシ改善** → **DAX(DynamoDB Accelerator)**。マイクロ秒応答。ElastiCacheでも可能だがDynamoDB専用のDAXの方が実装が簡単
- **APIスロットリング** → **API Gateway使用量プラン(429返却) + クライアント側で指数バックオフ**
- **Lambda@Edge によるリージョン振り分け** → 特定リージョンからのアクセスを別リージョンS3に切り替えなど、リクエストレベルでの最適化

**〈運用改善〉**
- **プライベートサブネット EC2 へアクセス** → **Session Manager**。踏み台不要、SSH鍵不要、監査可能。IAMポリシー`AmazonSSMManagedInstanceCore`をロールに付与
- **Auth情報保管** → **Secrets Manager + Lambda ローテーション(90日)**。Parameter Store SecureString はローテーション機能弱
- **CloudFormation リソースは `AWS::SecretsManager::RotationSchedule`**（重要：この実在するリソース名を覚える）
- **Lambda カナリアデプロイ** → **AWS SAM + AWS CodeDeploy 統合 + pre/post-traffic フック**でトラフィック段階シフト
- **EC2 メモリメトリクス収集** → **CloudWatch エージェント**(標準メトリクスにはメモリ・ディスク使用率は含まれない)
- **SQS + Auto Scaling + 30分処理** → **スケールイン保護**を処理中インスタンスに設定。処理途中でスケールインされないよう防止
- **IAM Policyでインスタンスタイプ制限** → 条件付きIAMポリシーで `ec2:InstanceType` を制限

### D1 複雑な組織向けの設計（50%不正解）

**〈マルチアカウント認証・アクセス〉**
- **オンプレAD + マルチアカウント条件付きアクセス** → **IAM Identity Center + SAML 2.0 + SCIM v2.0 + ABAC(属性ベースアクセス制御)**
- **クロスアカウントLambdaデプロイ** → 移行元でパッケージダウンロード → 移行先で新規Lambda作成。CloudFormation StackSets 経由が本命
- **IAMユーザー作成の承認フロー** → **CloudTrail の CreateUser イベント → EventBridge ルール → SNSでセキュリティチーム通知 → 承認後に権限付与**

**〈組織全体のガバナンス・コスト〉**
- **AWS WAFルールを複数アカウントで統一管理** → **AWS Firewall Manager + Systems Manager Parameter Store**でルールセット管理
- **組織のクラウド請求配分** → **管理アカウントでAWS Budgets + Cost Allocation Tags(アプリ/環境/所有者でグルーピング) + SNS通知**

**〈クロスアカウントネットワーク〉**
- **Direct Connect マルチリージョン化** → **Direct Connect Gateway + 既存プライベートVIF削除 + 新プライベートVIF作成**
- **クロスアカウントRoute 53プライベートホストゾーン** → **Account A のプライベートホストゾーンを Account B のVPCと関連付ける認証(create-vpc-association-authorization)**
- **オンプレ + マルチリージョンAWSを Transit Gateway 経由で統合** → **Transit VIF + Direct Connect Gateway** で両リージョンに接続

### D4 ワークロード移行・モダナイゼーション（45%不正解）

**〈VMware・オンプレ移行〉**
- **VMware → EC2 で設定完全保持** → **VM Import/Export**。vSphere Client で OVF エクスポート → S3 → `vmimport` IAMロール → AWS CLI で import
- **オンプレ → AWS DR** → **AWS Elastic Disaster Recovery (旧CloudEndure) の Replication Agent** をソースにインストール
- **オンプレ全体の移行評価** → **AWS Migration Evaluator(旧TSO Logic)**。CMDBからデータインポート → TCO分析

**〈データベース移行〉**
- **SQL Server → RDS MySQL** → **AWS Schema Conversion Tool (SCT) + AWS DMS**。異なるDBエンジン間の移行標準
- **MongoDB → AWS** → **DocumentDB(MongoDB互換)** + Multi-AZ

**〈ストレージ・ファイル移行〉**
- **オンプレ→S3 大量データ移行** → **DataSync**(ネットワーク経由の効率的転送) + S3イベント → Step Functions
- **オンプレでNFSアクセスしつつS3同期** → **S3 File Gateway**。NFS/SMBインターフェースでS3にアクセス
- **SFTPサーバーの管理を減らす** → **AWS Transfer for SFTP**。既存クライアントを変更せずマネージド化

**〈アプリケーションのモダン化〉**
- **オンプレのメールサーバー(SMTP)** → **Amazon SES**。テンプレート機能で顧客データ差込
- **ゲノム処理ワークフロー(Docker+ジョブ)** → **AWS DataSync + Step Functions + Batch(コンテナ実行)**

---

## ③ 問題ごとのコア教訓（52問）

### D2 新規ソリューション設計（18問）

**不正解問題**:
- **[✗] 演習1-問題7 | サードパーティAPIホワイトリスト対応** → **BYOIP + EIP + NAT Gateway**。カスタマー所有IPブロックを事前登録
- **[✗] 演習1-問題8 | 静的TCP + 高可用 + 固定IP** → **NLB + AZ毎EIP**。ALBはL7で不可
- **[✗] 演習1-問題18 | Lambda(VPC)から外部プロバイダAPI呼び出し** → **NAT Gateway + EIP**
- **[✗] 演習1-問題28 | 既存にマルチリージョンDR追加** → 別リージョンにALB/ASG/EC2 + CloudFront origin group
- **[✗] 演習1-問題37 | 単一EC2の重要アプリHA化** → ELB + Auto Scaling(最小2) + Multi-AZ ElastiCache
- **[✗] 演習1-問題62 | サーバーレスAPI + DynamoDB** → API Gateway **HTTP API** + Lambda + DynamoDB
- **[✗] 演習1-問題64 | MQTT数千デバイス** → IoT Core + thing + 証明書
- **[✗] 演習2-問題1 | ALBをCloudFront経由のみに制限** → **CloudFrontマネージドプレフィックスリスト**でSGルール
- **[✗] 演習2-問題19 | 工場のリアルタイム推論** → **IoT Greengrass**(ローカルデプロイ)にMLモデル
- **[✗] 演習2-問題29 | グローバルゲーム大量アクセス** → S3 CRR + CloudFront origin group(originフェイルオーバー)
- **[✗] 演習2-問題47 | マイクロサービス間ユーザー削除イベント** → **EventBridge カスタムバス** + 各サービスがルールをサブスクライブ
- **[✗] 演習2-問題49 | Lambda + 共有ライブラリ + カスタムクラス** → **Lambdaコンテナイメージ(ECR)** で対応

**理解浅めの正解問題**:
- **[○] 演習2-問題10 | ステートフルアプリ + RDS のDR** → Elastic DR + RDSクロスリージョンリードレプリカ
- **[○] 演習2-問題12 | リアルタイム取引 + マルチリージョン** → **Aurora Global Database** + CloudTrail監査
- **[○] 演習2-問題36 | ECS Fargate + Aurora のマルチリージョン** → Aurora クロスリージョンレプリカ
- **[○] 演習2-問題66 | 異なるベンダーIoT信号統合** → IoT Core Rule → Lambda → S3 (.csv)
- **[○] 演習2-問題67 | EBSスナップショット別リージョン2週保持** → **DLM(Data Lifecycle Manager)** ポリシー
- **[○] 演習2-問題69 | DynamoDB Global + Global Accelerator** → GA複数エンドポイントグループでリージョン振り分け

### D3 既存ソリューションの継続的改善（15問）

**不正解問題**:
- **[✗] 演習1-問題16 | APIの過剰リクエストからDynamoDB守る** → API Gateway使用量プラン(429) + クライアント指数バックオフ
- **[✗] 演習1-問題31 | Compute Optimizer リコメンド出力** → **Lambda で ExportLambdaFunctionRecommendations 呼び出し → S3 → QuickSight**
- **[✗] 演習1-問題40 | Lambda カナリアデプロイ** → **AWS SAM + CodeDeploy** + pre/post-traffic Lambda で新旧比較
- **[✗] 演習1-問題44 | DynamoDBのピーク+ベース最適化** → **Application Auto Scaling + Reserved Capacity**の組合せ
- **[✗] 演習1-問題49 | 200TB共有ファイル→クラウド最適化** → **S3 Intelligent-Tiering + Athena**でクエリ、または **FSx for Lustre + S3リンク**
- **[✗] 演習1-問題60 | OpenSearch 10ノードを最適化** → データノード減 + **UltraWarm ノード**追加(データを最適配置)
- **[✗] 演習2-問題7 | 開発者のインスタンスタイプ制限** → **IAMポリシーで ec2:InstanceType 条件**制限
- **[✗] 演習2-問題8 | eu-west-1のS3を北米ユーザーに近く** → **Lambda@Edge**で北米リクエストをus-east-1のS3に振り分け
- **[✗] 演習2-問題65 | EC2 CPU+メモリ+ネットワーク監視** → **CloudWatchエージェント**でメモリメトリクス収集
- **[✗] 演習2-問題72 | Webアプリのグローバル低レイテンシ** → **Global Accelerator + 固定IP**を顧客に提供
- **[✗] 演習2-問題75 | 30分処理中のスケールイン防止** → **スケールイン保護**を処理中インスタンスに設定

**理解浅めの正解問題**:
- **[○] 演習1-問題4 | RDS パスワード管理・自動ローテ** → Secrets Manager + Lambda + **AWS::SecretsManager::RotationSchedule** (90日)
- **[○] 演習1-問題11 | プライベートEC2にアクセス** → **Session Manager** + `AmazonSSMManagedInstanceCore` IAMロール
- **[○] 演習1-問題21 | 写真/動画のアクセス頻度不明** → **S3 Intelligent-Tiering**
- **[○] 演習2-問題2 | DynamoDBレイテンシ改善** → **DAX 3ノードクラスター**(マイクロ秒応答)

### D1 複雑な組織向けの設計（8問）

**不正解問題**:
- **[✗] 演習1-問題32 | クロスアカウントLambda移行** → デプロイパッケージダウンロード → 別アカウントで新規作成(または StackSets)
- **[✗] 演習1-問題45 | 事業部門別クラウド請求配分** → 管理アカウントでAWS Budgets + Cost Allocation Tags(アプリ/環境/所有者)
- **[✗] 演習2-問題33 | オンプレ+マルチリージョンAWS統合** → Transit VIF + Direct Connect Gateway で両リージョン
- **[✗] 演習2-問題71 | IAMユーザー作成の承認フロー** → CloudTrail + EventBridge + SNS通知

**理解浅めの正解問題**:
- **[○] 演習1-問題9 | オンプレAD + マルチアカウント** → **IAM Identity Center + SAML 2.0 + SCIM v2.0 + ABAC**
- **[○] 演習1-問題50 | DX単一→マルチリージョン化** → **Direct Connect Gateway** + 既存プライベートVIF削除 + 新VIF
- **[○] 演習1-問題74 | クロスアカウントプライベートホストゾーン** → プライベートホストゾーンとVPCの関連付け認証(create-vpc-association-authorization)
- **[○] 演習2-問題3 | マルチアカウントWAFルール** → **Firewall Manager + Systems Manager Parameter Store**

### D4 ワークロード移行・モダナイゼーション（11問）

**不正解問題**:
- **[✗] 演習1-問題10 | オンプレGit Webhook → AWS** → **API Gateway HTTP API + Lambda** で各Webhookロジック
- **[✗] 演習2-問題14 | オンプレPG(時系列)→AWS** → **Kinesis Firehose バッファ + Lambda変換**
- **[✗] 演習2-問題17 | ドキュメント処理をS3で** → **S3 File Gateway** + NFSでEC2からアクセス
- **[✗] 演習2-問題30 | オンプレ→AWSのDR** → **Elastic DR Replication Agent**をソースにインストール
- **[✗] 演習2-問題32 | MongoDB→AWS** → **DocumentDB Multi-AZ**

**理解浅めの正解問題**:
- **[○] 演習1-問題2 | VMware→EC2 設定完全保持** → **VM Import/Export**(vSphere→OVF→S3→`vmimport`ロール)
- **[○] 演習1-問題69 | オンプレSFTP→AWSマネージド** → **AWS Transfer for SFTP** + Route 53レコード
- **[○] 演習2-問題37 | SQL Server → RDS MySQL** → **AWS SCT + AWS DMS**
- **[○] 演習2-問題41 | オンプレSMTP → AWS** → **Amazon SES + テンプレート**
- **[○] 演習2-問題44 | オンプレ全体の移行評価** → **AWS Migration Evaluator**(CMDBインポート)
- **[○] 演習2-問題56 | ゲノム処理(Docker)** → **DataSync + Step Functions + Batch**

---

## ④ 試験当日の「迷ったら思い出す」即決判断表

### 移行・データ転送
| やりたいこと | 選ぶサービス | 引っかけ |
|---|---|---|
| VMwareのVM完全保持でEC2化 | **VM Import/Export** | Application Migration Service (MGN) は物理/仮想サーバー移行の後継 |
| オンプレ→AWSのDR | **Elastic Disaster Recovery (DRS)** | CloudEndureの後継 |
| DB移行(異エンジン) | **SCT + DMS** | SCTはスキーマ変換、DMSはデータ移行 |
| DB移行(同エンジン) | **DMS** のみでOK | |
| 大量ファイル→S3 | **DataSync** | Snow Family はネットワーク使えない場合 |
| SFTP マネージド化 | **Transfer Family (SFTP/FTPS/FTP/AS2)** | |
| オンプレNFS+S3同時アクセス | **S3 File Gateway** | Storage Gateway系 |
| オンプレ移行評価 | **Migration Evaluator** | |

### 認証・IDM
| やりたいこと | 選ぶサービス | 引っかけ |
|---|---|---|
| 社員のマルチアカウントAWS認証 | **IAM Identity Center + SAML 2.0 + ABAC** | SCIM 2.0で自動プロビジョニング |
| エンドユーザーのアプリ認証 | **Cognito User Pool** | |
| ADと連携 | **IAM Identity Center + AD Connector** or **Managed Microsoft AD** | |
| IAMユーザー作成の承認 | **CloudTrail + EventBridge + SNS** | |

### ネットワーク
| やりたいこと | 選ぶサービス | 引っかけ |
|---|---|---|
| 静的IP + TCP + 高可用 | **NLB + AZ毎EIP** | ALBはL7で不可 |
| BYOIPで固定IP | **BYOIP + EIP + NAT Gateway** | 標準EIPは事前登録できない |
| CloudFront経由のみALB許可 | **CloudFrontマネージドプレフィックスリスト** | ホストヘッダー検証も可 |
| マルチリージョン低レイテンシ | **Global Accelerator** | Route 53レイテンシルーティングは固定IPにできない |
| Direct Connect マルチリージョン | **DX Gateway** | Transit VIF/プライベートVIFの使い分け |
| クロスアカウントプライベートDNS | **プライベートホストゾーン + VPC関連付け認証** | |
| Lambda VPC→インターネット | **NAT Gateway** | Lambda自身はVPC外だが、VPC設定するとNAT必須 |

### コスト最適化
| やりたいこと | 選ぶサービス | 引っかけ |
|---|---|---|
| S3のアクセスパターン不明 | **S3 Intelligent-Tiering** | 手動ライフサイクル管理は運用負荷 |
| DynamoDB ピーク+ベース | **Auto Scaling + Reserved Capacity** | オンデマンドは変動大きい場合 |
| Compute Optimizer レポート | **ExportLambdaFunctionRecommendations Lambda呼び出し** | |
| 事業部門別請求 | **Cost Allocation Tags + Budgets** | Cost Explorerはレポート、Budgetsはアラート |
| OpenSearchログの長期保管 | **UltraWarm ノード** | S3ベースで安価 |

### DR・高可用性
| RPO/RTO要件 | 適切な戦略 | 主なサービス |
|---|---|---|
| RPO=数時間 / RTO=数時間 | **Backup & Restore** | AWS Backup |
| RPO=数分 / RTO=数十分 | **Pilot Light** | 最小限のリソースを別リージョンに |
| RPO=数分 / RTO=数分 | **Warm Standby** | 縮小版フル環境 |
| RPO≈0 / RTO≈0 | **Multi-Site Active-Active** | Aurora Global DB, Route 53 |

### DB選択
| 要件 | サービス |
|---|---|
| リレーショナル + マルチリージョン強整合 | **Aurora Global Database** |
| KVSでミリ秒応答+マルチリージョン | **DynamoDB Global Tables** |
| DynamoDBのマイクロ秒応答 | **DAX** |
| MongoDB互換 | **DocumentDB** |
| 時系列データ | **Timestream** |
| グラフ | **Neptune** |
| データウェアハウス | **Redshift** |
| インメモリ | **ElastiCache** (Redis/Memcached) |

### イベント・メッセージング
| ユースケース | サービス |
|---|---|
| マイクロサービス間の疎結合イベント | **EventBridge カスタムバス** |
| ファンアウト通知 | **SNS** |
| キューイング | **SQS** |
| ストリーミング大量データ | **Kinesis Data Streams / Firehose** |
| MQTT IoT | **IoT Core** |
| エッジML推論 | **IoT Greengrass** |

### CloudFront周辺
| やりたいこと | 手段 |
|---|---|
| リージョン振り分け(HTTP応答編集) | **Lambda@Edge** |
| フェイルオーバーorigin | **Origin Group** |
| オリジン間の共通処理 | **Lambda@Edge origin response** |
| ALB直接アクセス防止 | **マネージドプレフィックスリスト**でSG制限 |

---

## ⑤ 最終チェック（試験前日にこれだけ見る）

1. **D2が最重要**: ネットワーク(BYOIP/NLB/GA)、DR(Elastic DR/Aurora Global)、IoT(Core/Greengrass)、EventBridge の使い分けを再確認
2. **D3の"最適解"パターン**: Intelligent-Tiering / DAX / Session Manager / DLM / Auto Scaling+Reserved / スケールイン保護
3. **移行系サービスの守備範囲**: VM Import↔DRS↔MGN、SCT+DMS↔DMS単体、DataSync↔Storage Gateway↔Transfer Family
4. **IAM Identity Center + ABAC** は組織問題の万能解
5. **Firewall Manager** は WAF/SG/Shieldをマルチアカウントで統一管理
6. **CloudFront + マネージドプレフィックスリスト** で ALB直接アクセス防止

---

## ⑥ 追加すべき観点（演習3・4も届いた時点で更新予定）

現時点で見えている苦手傾向から、以下は追加でチェックしたい:
- **Storage Gateway 3種**(File/Volume/Tape)の使い分け → 出題頻度高
- **Systems Manager 各機能**(Session Manager/Patch Manager/Automation/Parameter Store)
- **AWS Backup** vs 個別バックアップの選択基準
- **Cost & Usage Report** vs Cost Explorer vs Budgets の役割分担
- **AWS Config** vs CloudTrail vs Security Hub のガバナンス役割

演習3・4を送ってもらったら、上記の観点も含めて追記します。

頑張ってください 🍀
