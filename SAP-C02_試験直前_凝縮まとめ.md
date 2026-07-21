# SAP-C02 試験直前 凝縮まとめ（苦手102問・分野別要点集約版）

> **試験日**: 1週間後 / 第一優先: 合格
> **対象**: 演習1-4(全300問)中、**苦手102問**（不正解45問 + マーク済のみ57問）
> **凡例**: ✗=不正解 / 🏷=マーク済(理解浅め)
> **本資料の狙い**: 前の完全版が長すぎたため、**類似問題を分野グループに集約**し、各グループの共通教訓+個別問題は1行要約に凝縮。

---

## 📊 苦手分野の俯瞰

| ドメイン | 苦手数 | 不正解 | マーク済のみ |
|---|---|---|---|
| **D1 複雑な組織向けの設計** | 27問 | 20 | 7 |
| **D2 新規ソリューションの設計** | 33問 | 20 | 13 |
| **D3 既存ソリューションの継続的改善** | 24問 | 11 | 13 |
| **D4 ワークロードの移行・モダナイゼーション** | 18問 | 10 | 8 |
| **合計** | **102問** | 45 | 41 |

**優先度**: 不正解率が高い **D2(61%) > D1(74%)** から潰す。D2は問題数最多かつ複合設計問題が中心。

---

## 🎯 全体キーワード早見表(頻出のみ)

### ネットワーク・接続
- **Direct Connect Gateway** = グローバル、マルチリージョンVPC接続
- **Transit Gateway** = リージョナル、VPC間統合、RAMで組織共有
- **PrivateLink (エンドポイントサービス)** = VPC間プライベート接続、CIDR重複解消、3rd party SaaS
- **BYOIP + EIP + NAT Gateway** = 送信元IP固定(ホワイトリスト対応)
- **NLB + AZ毎EIP** = 静的TCP + 固定IP + HA
- **Global Accelerator** = 固定IP + マルチリージョン低レイテンシ
- **CloudFrontマネージドプレフィックスリスト** = ALBのSGでCloudFront経由のみ許可

### 認証・アクセス制御
- **IAM Identity Center** = SAML/SCIM/ABAC対応、マルチアカウント認証の本命
- **クロスアカウント**: リソース側+アイデンティティ側 **両方の許可** が必須
- **SCP戦略** = `FullAWSAccess`を削除して制限的SCPを付与(削除ではなく置換)
- **`aws:PrincipalOrgID`** = 組織内リソース限定条件
- **`${aws:username}`変数** = ユーザー毎のS3プレフィックス制限
- **External ID** = クロスアカウントロールに追加保護(監査人など)
- **OrganizationAccountAccessRole** = メンバーアカウントへの管理アクセス標準ロール
- **AWS RAM** = プレフィックスリスト/TGW/VPCサブネット/CFn StackSet 共有

### DB・ストレージ
- **RDS Proxy** = 接続プール、フェイルオーバー高速化
- **Aurora Global Database** = マルチリージョン + RPO=0
- **DynamoDB Global Tables** = マルチリージョン + 結果整合性
- **DAX** = DynamoDB用マイクロ秒応答キャッシュ
- **S3 Intelligent-Tiering** = アクセスパターン不明時の万能解
- **S3 File Gateway** = オンプレNFS/SMB→S3
- **DocumentDB** = MongoDB互換
- **OpenSearch UltraWarm** = 低頻度アクセスログの安価保管

### 移行系
- **VM Import/Export** = VMware→EC2 (OVF, `vmimport`ロール)
- **AWS MGN (Application Migration Service)** = 物理/仮想サーバー大量移行
- **AWS DRS (Elastic Disaster Recovery)** = オンプレDR、Replication Agent
- **SCT + DMS** = スキーマ変換+データ移行(異エンジン間)
- **DataSync** = 大量ファイル→S3/EFS、NFS/SMB
- **Transfer Family** = マネージド SFTP/FTPS/FTP/AS2
- **Migration Evaluator** = 移行前TCO分析(CMDBインポート)
- **Migration Hub** = 移行状況一元管理、依存関係可視化
- **App2Container** = .NET/Java→コンテナ化

### サーバーレス・イベント
- **API Gateway HTTP API** = 低コスト・低レイテンシ(REST APIより)
- **API Gateway REST API** = AWS統合(DynamoDB直接)可能
- **Lambda コンテナイメージ** = 10GBまで、大きな共有ライブラリ対応
- **Lambda@Edge** = CloudFrontでリクエスト/レスポンス編集
- **EventBridge カスタムバス** = マイクロサービス間の疎結合イベント
- **Amazon SES + STARTTLS** = SMTP互換マネージドメール

### コスト・ガバナンス
- **Cost and Usage Report (CUR)** = 詳細請求データ、Athena+QuickSightで可視化
- **AWS Budgets** = 予算超過アラート
- **Cost Allocation Tags** = アプリ/環境/所有者で分類
- **Compute Optimizer** = EC2/Lambda/EBS推奨、`ExportLambdaFunctionRecommendations`
- **Control Tower** = ランディングゾーン + ガードレール
- **Service Catalog** = 承認済リソースのみ提供
- **AWS Config タグポリシー** = 組織全体でタグ標準を強制

---

## 🏢 D1: 複雑な組織向けの設計 (27問)

### G1-1. クロスアカウントアクセスと権限委譲 (6問)

**共通教訓**:
- クロスアカウントは **リソース側(バケット/KMS/Secrets) と アイデンティティ側(IAMロール) の両方の許可** が必須
- 監査人・パートナーには **`sts:ExternalId`** を信頼ポリシーに追加して混乱代理問題を防ぐ
- 全アカウント読み取りには **`OrganizationAccountAccessRole`** を利用してメンバーアカウントに読み取り専用ロールを作成
- Lambda/クロスリージョン共有: デプロイパッケージのダウンロード → 別アカウントで新規作成 (RAM/Resource共有)

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [🏷] 演習1-問題7 | S3バケットポリシー(Account A) + IAMポリシー(Account B User) の両側許可 |
| [✗] 演習1-問題36 | クロスアカウントLambda: パッケージダウンロード→新規作成 + RAM共有 |
| [🏷] 演習2-問題8 | クロスアカウントSecrets: 各アカウントにIAMロール + AssumeRoleチェーン |
| [🏷] 演習2-問題12 | 監査人: External ID付きIAMロール + 読み取り専用ポリシー |
| [✗🏷] 演習2-問題17 | マルチアカウントKMS: 別アカウントIAMロールにKMS復号権限を明示 |
| [✗] 演習3-問題33 | 全アカウント読み取り: `OrganizationAccountAccessRole`経由で各メンバーにロール作成 |

---

### G1-2. マルチアカウントネットワーク (8問) ⚠️ 高頻度・要注意

**共通教訓**:
- **Direct Connect Gateway = グローバル** vs **Transit Gateway = リージョナル**
- **マルチリージョン展開**: DX Gateway + プライベートVIF(各接続) が本命
- **クロスアカウントVPC接続**: TGW を共有ネットワークアカウントに作成 → **RAM で組織/OU共有**
- **CIDR重複してるVPC間の共有**: TGWは使えない → **PrivateLink エンドポイントサービス** が正解
- **各OUで独自ネットワークが必要**: OU別TGW + RAM で分離
- **プレフィックスリスト**: セキュリティチームで一元管理 → RAM共有 → 各SGで参照

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [✗🏷] 演習1-問題9 | DX Gateway + 2本目DX + 各接続にプライベートVIFで冗長化 |
| [✗] 演習2-問題24 | プレフィックスリスト(セキュリティチーム集中管理) + RAM共有 |
| [🏷] 演習2-問題30 | 共有ネットワークアカウントにTGW + 各アカウントVPCアタッチ |
| [✗🏷] 演習2-問題32 | TGWを他アカウントにRAM共有 + プライベートサブネットのみ |
| [✗🏷] 演習2-問題72 | 各DX接続からDX Gatewayへtransit VIF (HA) |
| [✗] 演習3-問題37 | 共有VPC(CFn) + RAMでサブネットを組織共有 |
| [✗] 演習3-問題47 | **CIDR重複の共有 = PrivateLink エンドポイントサービス** |
| [✗] 演習3-問題53 | 各OU専用TGW + RAM でOU内共有 |

---

### G1-3. SCP・権限制限 (4問)

**共通教訓**:
- **SCPは削除ではなく置換**: `FullAWSAccess` を削除 → 制限的SCPを付与 (デフォルトDenyになる)
- **CloudFormationサービスロール**: エンジニアに `iam:PassRole` 権限 + サービスロールがリソース作成権限を持つ
- **S3個人フォルダ**: IAMポリシーで **`${aws:username}` 変数** をPathプレフィックスに条件付与
- **承認済リソースのみ**: **Service Catalog** ポートフォリオを開発者と共有

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [✗🏷] 演習1-問題31 | 開発者OUから`FullAWSAccess` SCP削除 → EC2/S3/DynamoDBのみ許可SCP付与 |
| [✗] 演習1-問題34 | エンジニアIAMは CloudFormation実行のみ、サービスロールで承認済リソース作成 |
| [✗] 演習4-問題44 | S3個人フォルダ: `${aws:username}` プレフィックス条件 |
| [✗] 演習4-問題54 | Service Catalog ポートフォリオ + IAMポリシーで承認製品のみ許可 |

---

### G1-4. マルチアカウントガバナンス (4問)

**共通教訓**:
- **AD連携**: IAM Identity Center + **SAML 2.0 + SCIM v2.0 + ABAC**
- **タグ標準化**: **SCP + タグポリシー** の組合せ(SCPで作成拒否、タグポリシーで値定義)
- **新規AWS環境の統制**: **Control Tower + Configガードレール**
- **IAMユーザー作成の承認**: CloudTrail → EventBridge → SNS/Lambda

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [🏷] 演習1-問題49 | IAM Identity Center + SAML 2.0 + SCIM v2.0 + ABAC |
| [✗] 演習3-問題24 | SCP(タグ必須で作成拒否) + タグポリシー(OU別値定義) |
| [✗] 演習4-問題63 | Control Tower有効化 + Configガードレールレビュー |
| [✗] 演習2-問題74 | EventBridgeルール: detail-type=`AWS API Call via CloudTrail`, eventName=`CreateUser` |

---

### G1-5. コスト管理・可視化 (4問)

**共通教訓**:
- **詳細分析**: **CUR + Athena + QuickSight** で自由なグルーピング
- **予算アラート**: **Budgets + Cost Allocation Tags** で事業部別
- **RI共有制御**: Billing Console → 特定アカウントのRI共有OFF
- **請求配分**: Cost Explorer は分析用、Budgetsはアラート用、CURは生データ

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [✗] 演習1-問題46 | Organization CUR + タグ/コストカテゴリ + Athena + QuickSight |
| [🏷] 演習1-問題60 | 管理アカウントCUR + QuickSightで各チームに可視化 |
| [✗] 演習1-問題70 | 管理アカウントBudgets + Cost Allocation Tags (app/env/owner) + SNS |
| [🏷] 演習2-問題29 | Billing Consoleで特定アカウントのRI共有OFF |

---

### G1-6. S3クロスアカウント公開 (1問)

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [✗] 演習3-問題35 | S3データレイクのマルチアカウント公開: **S3アクセスポイント + ゲートウェイエンドポイント + エンドポイントポリシー** |

---

## 🏗️ D2: 新規ソリューションの設計 (33問)

### G2-1. 高可用性・DR設計 (14問) ⚠️ 最頻出

**共通教訓 - RPO/RTOで戦略を選ぶ**:
- **RPO=0 / RTO=数分**: Multi-Site Active-Active (**Aurora Global**, **DynamoDB Global Tables**)
- **RPO=数分 / RTO=数分**: Warm Standby (縮小版フル環境)
- **RPO=数分 / RTO=数十分**: Pilot Light (最小限リソース、リードレプリカ)
- **RPO=数時間**: Backup & Restore (AWS Backup)

**共通教訓 - 実装パターン**:
- **単一EC2の重要アプリHA化** → ELB + Auto Scaling(最小2) + Multi-AZ ElastiCache/RDS
- **CloudFront配下の複数リージョン化** → 別リージョンにALB/ASG/EC2 + **CloudFront origin group** でフェイルオーバー
- **S3のクロスリージョン**: **CRR** + 2バケットを **CloudFront origin group** に
- **オンプレ→AWS DR**: **AWS Elastic Disaster Recovery (DRS)** + RDSクロスリージョンリードレプリカ
- **SNS+マルチリージョン**: 各リージョンのSQSがSNSトピックにサブスクライブ
- **API マルチリージョン**: Route 53 マルチバリューアンサー or レイテンシベース + カスタムドメイン
- **IoT マルチリージョン**: **IoT Core ドメイン設定** + Route 53 ヘルスチェック

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [✗🏷] 演習1-問題2 | 3層アプリHA化: RDS Multi-AZ + ElastiCache レプリケーション(Multi-AZ) + ELB + ASG(最小2) |
| [✗🏷] 演習1-問題62 | 別リージョンにALB/ASG/EC2 + CloudFront origin group (originフェイルオーバー) |
| [✗] 演習1-問題63 | DR自動化: バックアップリージョンのLambda(リードレプリカ昇格+ASG拡張) + Route 53ヘルスチェック |
| [🏷] 演習1-問題64 | S3 CRR + 2バケットをCloudFront origin group + Route53 ヘルスチェック |
| [🏷] 演習2-問題7 | リアルタイム取引: Aurora Global (RPO=0) + CloudTrail監査 |
| [✗🏷] 演習2-問題14 | SNS + 各リージョンSQSがサブスクライブ + Lambda各リージョン配置 |
| [✗] 演習2-問題15 | DynamoDB Global Tables + Aurora Global Database 両方 |
| [✗] 演習2-問題55 | S3 CRR + CloudFront 2オリジン + Route 53 |
| [🏷] 演習2-問題66 | Elastic DR(EC2) + RDSクロスリージョンリードレプリカ + Global Accelerator |
| [✗] 演習3-問題56 | ホットスタンバイ: Multi-AZ ASG + ゾーンRI + DR用リソース事前確保 |
| [✗] 演習3-問題57 | 単一AZ→Multi-AZ ASG + ALB (+ セキュリティ強化) |
| [✗] 演習3-問題74 | IoT Coreドメイン設定 + Route 53 ヘルスチェック + DynamoDB Global |
| [✗] 演習4-問題18 | マルチリージョンAPI Gateway + Route 53マルチバリューアンサー |
| [✗] 演習4-問題53 | Route 53 レイテンシベース + CloudFront + 各リージョンASG |

---

### G2-2. ネットワーク設計 (6問)

**共通教訓**:
- **静的TCP + 固定IP + HA** → **NLB + AZ毎EIP** (ALB不可: L7)
- **BYOIP登録** → EIP作成 → **NAT Gatewayに割当** で送信元IP固定
- **プライベートサブネットから S3/SQS/DynamoDB** → **VPCゲートウェイエンドポイント** + エンドポイントポリシー
- **3rd party SaaS接続** → **PrivateLink Interface VPCエンドポイント** + エンドポイントサービス
- **RDSアクセス制御** → EC2 SG を DB SG のインバウンドソースに指定(**SG参照**)
- **内部ALBを別のNLBに繋げる** → NLB の ALBタイプターゲットグループ

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [✗🏷] 演習1-問題8 | NLB + AZ毎EIP + Route 53エイリアスレコード |
| [🏷] 演習1-問題10 | S3ゲートウェイVPCエンドポイント + エンドポイントポリシー |
| [🏷] 演習1-問題18 | RDS SG のインバウンドで EC2 SG を参照 (IPではなくSG参照) |
| [✗] 演習1-問題47 | BYOIP登録 + EIP + NAT Gateway (送信元IP固定) |
| [✗] 演習1-問題75 | PrivateLink Interface VPCエンドポイント + SaaSのエンドポイントサービスに接続 |
| [🏷] 演習2-問題10 | NLB作成 + ALBタイプターゲットグループで既存内部ALBを追加 |

---

### G2-3. IoT・イベント駆動 (5問)

**共通教訓**:
- **MQTT + 大量デバイス + X.509認証** → **IoT Core + thing + 証明書プロビジョニング**
- **IoTデータ処理** → IoT Core Rule → Lambda → S3(.csv) → Athena/Glue
- **マイクロサービス間の疎結合** → **EventBridge カスタムバス** (SNSより高機能)
- **アップロード動画処理** → S3 → **S3イベント通知** → SQS → Rekognition/Lambda
- **スキーマレスストリーミング** → Kinesis Data Streams → DynamoDB

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [🏷] 演習1-問題21 | IoT Core + thing per デバイス + X.509 証明書 |
| [✗] 演習2-問題5 | IoT Core Rule → Lambda → S3(.csv) → Glueカタログ → Athena |
| [🏷] 演習1-問題28 | S3ホスト + S3イベント → SQS → Rekognition解析 |
| [✗] 演習2-問題50 | EventBridge カスタムバス + マイクロサービスがルールでサブスクライブ |
| [✗] 演習4-問題64 | Kinesis Data Streams → DynamoDB (スキーマレス) |

---

### G2-4. サーバーレスAPI/Lambda拡張 (3問)

**共通教訓**:
- **DynamoDB を直接公開するAPI** → **API Gateway REST API + AWS統合タイプ**でDynamoDB直接
- **大きな共有ライブラリ + Lambda** → **Lambda コンテナイメージ + ECR** (Layerは足りない)
- **Lambda → 制限DB** → **SQS 経由でスロットリング**、Lambda同時実行数制限

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [🏷] 演習1-問題69 | API Gateway REST API + AWS統合でDynamoDB直接統合 |
| [🏷] 演習2-問題47 | Lambda コンテナイメージ + ECR (共有ライブラリ対応) |
| [✗] 演習3-問題23 | Lambda → SQS → 別Lambda → DB (キュー経由で流量制御) |

---

### G2-5. セキュリティ設計 (4問)

**共通教訓**:
- **クライアントサイド暗号化** → **KMS `GenerateDataKey`** でデータキー生成 → CMKで暗号化
- **DDoS + Web攻撃対策** → **AWS WAF + Shield Advanced** をALB/CloudFrontに関連付け
- **CloudFrontフィールドレベル暗号化** → RSA鍵ペア + プロファイル設定 (エンドツーエンド)
- **ECS + WAF** → **WAF web ACL + CloudFrontに関連付け** (ALB直接ではなく)

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [🏷] 演習2-問題37 | クライアントサイド暗号化: `kms:GenerateDataKey` でデータキー生成 |
| [🏷] 演習2-問題49 | AWS Shield Advanced + WAFをALBに追加 |
| [✗] 演習3-問題19 | WAF Web ACL + CloudFront関連付け + ALBオリジン |
| [✗] 演習4-問題73 | CloudFront フィールドレベル暗号化(RSA公開鍵) |

---

### G2-6. コンピュート最適化 (1問)

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [🏷] 演習2-問題44 | ASG起動テンプレート: **属性ベースインスタンスタイプ選択** (CPU/メモリ要件指定) |

---

## 📈 D3: 既存ソリューションの継続的改善 (24問)

### G3-1. コスト最適化(S3/OpenSearch/Compute Optimizer) (3問)

**共通教訓**:
- **アクセスパターン不明のS3** → **S3 Intelligent-Tiering** 一択(手動ライフサイクル不要)
- **OpenSearchのログ長期保管** → **UltraWarmノード**(S3ベース)でデータノード削減
- **Lambda最適化推奨のエクスポート** → **`ExportLambdaFunctionRecommendations`** をLambdaで呼出 → S3

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [🏷] 演習1-問題4 | S3 Intelligent-Tiering (ミリ秒取得 + 自動階層化) |
| [🏷] 演習1-問題50 | OpenSearch: データノード削減 + UltraWarm追加 + インデックス自動移行 |
| [🏷] 演習1-問題74 | Compute Optimizer + `ExportLambdaFunctionRecommendations` Lambda → S3 |

---

### G3-2. データ層改善(RDS/Secrets/認証) (4問)

**共通教訓**:
- **DBパスワード管理・自動ローテーション** → **Secrets Manager + Lambda + `AWS::SecretsManager::RotationSchedule`** (90日)
- **DB接続プール + フェイルオーバー高速化** → **RDS Proxy**
- **パスワードレスDB接続** → **IAM DB認証** (Auroraで有効化 + Lambda ロールに権限)
- **Aurora + フェイルオーバー最適化** → RDS Proxy + Aurora MySQL + リーダーエンドポイント

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [🏷] 演習1-問題11 | Secrets Manager + Lambda + `AWS::SecretsManager::RotationSchedule` (90日) |
| [🏷] 演習1-問題45 | RDS Proxyでリーダーエンドポイント接続プール |
| [✗] 演習2-問題62 | Aurora IAM DB認証 + Lambda ロール変更 + VPCエンドポイント |
| [🏷] 演習2-問題75 | RDS Proxy + Aurora MySQL移行 |

---

### G3-3. 運用改善(Session Manager/Instance Connect) (3問)

**共通教訓**:
- **SSHキー不要のEC2アクセス** → **Session Manager**(踏み台不要、監査可能) or **EC2 Instance Connect**(一時公開鍵)
- **SSMアクセス**: **`AmazonSSMManagedInstanceCore`** ポリシーをIAMロールに付与
- **ASGインスタンスのシャットダウン時ログ収集** → **ライフサイクルフック + EventBridge + SSMドキュメント**

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [🏷] 演習1-問題44 | Session Manager: `AmazonSSMManagedInstanceCore` ポリシー + SG は 22ポート閉じてOK |
| [✗] 演習4-問題41 | EC2 Instance Connect: SSHキーなし + IAMで `ec2-instance-connect:SendSSHPublicKey` |
| [🏷] 演習1-問題40 | ASG ライフサイクルフック + EventBridge + SSMドキュメント(S3にログコピー) |

---

### G3-4. コンピュート改善(ASG/インスタンス制限) (3問)

**共通教訓**:
- **開発者のインスタンスタイプ制限** → **IAMポリシー条件 `ec2:InstanceType`**
- **EC2 のCPU/メモリ最適化** → **CloudWatchエージェント**でメモリメトリクス取得(標準はCPU/ネットワークのみ) + Compute Optimizer
- **既存ASGのインスタンスタイプ最適化** → 起動テンプレートを **属性ベース** に更新

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [✗] 演習2-問題27 | IAMポリシーで `ec2:InstanceType` 条件付与 |
| [🏷] 演習2-問題65 | CloudWatchエージェント(メモリ) + Compute Optimizer |
| [✗] 演習4-問題14 | 起動テンプレートを属性ベース(CPU/メモリ要件)に更新 |

---

### G3-5. CloudFront改善(Lambda@Edge) (3問)

**共通教訓**:
- **CloudFront + S3で動画配信** → EFSの動画をS3に移行 + CloudFront配信
- **リージョン別振り分け** → **Lambda@Edge** をorigin request/viewer requestで実行
- **キャッシュキー正規化(効率化)** → **Lambda@Edge viewer request** でクエリ文字列ソート・小文字化

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [🏷] 演習1-問題32 | 動画をEFSからS3に移行 + CloudFront配信 |
| [✗] 演習2-問題58 | Lambda@Edge で北米→us-east-1 S3、それ以外→eu-west-1 S3 |
| [✗] 演習4-問題22 | Lambda@Edge viewer request でクエリ文字列ソート・小文字化 |

---

### G3-6. デプロイ改善(WAF段階/ALB重み付き) (2問)

**共通教訓**:
- **WAFルールの段階展開** → **Countモードで様子見** → ロギングで偽陽性確認 → Blockモードへ
- **カナリアリリース** → **ALB 重み付きターゲットグループ**で新旧トラフィック分配

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [🏷] 演習2-問題41 | WAF Count → ロギング → 偽陽性修正 → Block |
| [✗] 演習3-問題36 | ALB 2ターゲットグループ + 重み付きリスナールール(カナリア) |

---

### G3-7. ネットワーク・データ転送改善 (2問)

**共通教訓**:
- **大容量ファイル遠隔アップロード** → **マルチパートアップロード + S3 Transfer Acceleration**
- **既存アプリのグローバル低レイテンシ** → **Global Accelerator を ALB前に追加**

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [✗] 演習1-問題14 | マルチパートアップロード + S3 Transfer Acceleration |
| [✗] 演習3-問題62 | Global Accelerator + ALBをオリジン |

---

### G3-8. セキュリティ監視改善 (1問)

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [✗] 演習3-問題50 | IAM Access Analyzer + EventBridgeで `isPublic: true` フィルタ → SNS通知 |

---

### G3-9. アーキテクチャ全般改善 (3問)

**共通教訓**:
- **Cronスクリプト → イベント駆動** → S3イベント通知 + Lambda(ポーリング不要でコスト削減)
- **Webhook処理 → サーバーレス化** → **API Gateway HTTP API + Lambda**
- **FSx容量拡張の自動化** → CloudWatchメトリクス + EventBridge + Lambda(`update-file-system`)

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [✗] 演習1-問題33 | Python cron → Lambda + S3イベント通知(オブジェクト作成時) |
| [✗] 演習1-問題73 | Git Webhook → API Gateway HTTP API + Lambda (ALB/ASG不要) |
| [🏷] 演習2-問題56 | FSx: CloudWatch空き容量 + EventBridge + Lambda(`update-file-system`) |

---

## 🔄 D4: ワークロードの移行・モダナイゼーション (18問)

### G4-1. DB移行 (3問)

**共通教訓**:
- **異エンジン間** (SQL Server → MySQL/PostgreSQL) → **SCT でスキーマ変換 + DMS でデータ移行**
- **MongoDB互換** → **DocumentDB** Multi-AZ
- **IoTメタデータのNoSQL** → DocumentDB(可変スキーマ)

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [🏷] 演習2-問題3 | SQL Server → RDS MySQL: SCT(スキーマ) + DMS(データ) |
| [✗🏷] 演習2-問題19 | MongoDB → DocumentDB Multi-AZ |
| [✗] 演習3-問題22 | IoTメタデータ → DocumentDB (MongoDB互換) |

---

### G4-2. ファイル・ストレージ移行 (5問) ⚠️ 頻出

**共通教訓 - 使い分け**:
- **オンプレNFS/SMB + AWS両方から使いたい** → **S3 File Gateway**
- **オンプレのファイルをAWSに大量転送** → **DataSync** (エージェント + Direct Connect + PrivateLink)
- **SFTP/FTPSマネージド** → **AWS Transfer Family**
- **SQL Serverバックアップ(定期)を S3へ** → **File Gateway(SMB)** で見せかけローカルストレージ
- **移行後のワークフロー** → S3イベント → Step Functions

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [🏷] 演習2-問題1 | S3 File Gateway + NFSファイル共有(継続アクセス) |
| [✗] 演習2-問題57 | Transfer Family FTPサーバー → S3 + SNS通知 |
| [🏷] 演習2-問題67 | DataSync + S3イベント + Step Functions + Batch |
| [✗] 演習3-問題46 | オンプレSQLバックアップ → File Gateway SMB共有 |
| [✗] 演習4-問題26 | オンプレNFS → EFS: DataSyncエージェント + DX + PrivateLink |

---

### G4-3. サーバー移行(MGN/App2Container) (2問)

**共通教訓**:
- **物理/仮想サーバー大量移行(120VM等)** → **AWS Application Migration Service (MGN)** (旧CloudEndure Migration)
- **.NET/Javaレガシー → コンテナ化** → **App2Container** → **ECS/Fargate/App Runner**

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [🏷] 演習2-問題36 | App2Container → ECS Fargate + ECR |
| [✗] 演習4-問題31 | MGN + VMwareクラスター接続 + レプリケーションジョブ |

---

### G4-4. 移行評価・分析 (2問)

**共通教訓**:
- **移行前のTCO分析** → **Migration Evaluator**(CMDBインポート)
- **アプリケーション依存関係の可視化** → **Migration Hub + Application Discovery Service**

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [✗] 演習2-問題68 | Migration Evaluator + データインポートテンプレート(CMDB) |
| [✗] 演習4-問題8 | Migration Hub + ネットワーク依存関係グラフ + Athena |

---

### G4-5. アプリのモダナイゼーション (4問)

**共通教訓**:
- **オンプレWebアプリ + PGDB → AWS** → EC2 Auto Scaling + Aurora Auto Scaling
- **ドキュメント処理サーバーレス化** → **Textract + Comprehend + Step Functions**
- **PGDB スケール問題** → OpenSearch Service + Kibana + Kinesis (時系列に強い)
- **RabbitMQ → AWS** → **Amazon MQ** (プロトコル互換)

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [🏷] 演習1-問題16 | ALB + ASG(スティッキーセッション) + Aurora Auto Scaling |
| [🏷] 演習2-問題2 | Step Functions + Lambda + Textract + Comprehend (フォーム処理) |
| [✗🏷] 演習2-問題69 | OpenSearch + Kibana + 時系列イベント処理 |
| [🏷] 演習2-問題71 | AMI + ASG + Amazon MQ(RabbitMQ互換) + Fargate |

---

### G4-6. メール送信 (2問)

**共通教訓**:
- **レガシーSMTP → SES 移行** → **STARTTLS + SES SMTP認証情報**
- **顧客データ差し込みメール** → **SES + テンプレート(SendTemplatedEmail API)**

**問題ごとの1行要約**:
| 問題 | 教訓 |
|---|---|
| [🏷] 演習1-問題37 | STARTTLS + SES SMTP認証情報でアプリを再設定 |
| [🏷] 演習2-問題33 | SES + SendTemplatedEmail API + 顧客データパラメータ |

---

## 🎯 試験前日にこれだけ見る(超要点)

### D1で最頻出の判断ポイント
1. **クロスアカウント = リソース側+アイデンティティ側の両方の許可**
2. **DX Gateway=グローバル / TGW=リージョナル**
3. **CIDR重複VPC = PrivateLinkエンドポイントサービス**
4. **SCPは削除ではなく置換** (`FullAWSAccess`削除+制限SCP付与)
5. **プレフィックスリスト/TGW/サブネット は RAM で共有**
6. **AD連携 = IAM Identity Center + SAML/SCIM/ABAC**

### D2で最頻出の判断ポイント
1. **RPO=0要求 = Aurora Global Database**
2. **CloudFront配下マルチリージョン = origin group でフェイルオーバー**
3. **静的TCP+固定IP+HA = NLB + AZ毎EIP** (ALBは不可)
4. **BYOIP + NAT Gateway で送信元IP固定**
5. **IoT大量デバイス = IoT Core + thing + X.509**
6. **マイクロサービス間 = EventBridge カスタムバス**
7. **DDoS対策 = Shield Advanced + WAF**

### D3で最頻出の判断ポイント
1. **アクセスパターン不明S3 = Intelligent-Tiering**
2. **RDS = Proxy + Secrets Manager + IAM DB認証**
3. **EC2 = Session Manager / Instance Connect (SSHキー不要)**
4. **既存メモリ監視 = CloudWatchエージェント**
5. **Lambda@Edge = リージョン振り分け/キャッシュキー正規化**
6. **WAFの安全な展開 = Count → Block**

### D4で最頻出の判断ポイント
1. **VMware→EC2 = VM Import/Export or MGN**
2. **オンプレ→S3ハイブリッド = S3 File Gateway**
3. **大量ファイル転送 = DataSync**
4. **異エンジンDB移行 = SCT + DMS**
5. **MongoDB → DocumentDB**
6. **SMTP → SES + STARTTLS**
7. **移行分析: Migration Evaluator(TCO) / Migration Hub(依存)**

---

**頑張ってください 🍀**
