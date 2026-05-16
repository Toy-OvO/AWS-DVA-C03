# SCS-C03 試験前まとめ（マーク済65問の分析）

> **試験日**: 明後日。第一優先は合格。
> **マーク済問題**: 全65問（うちユーザー不正解 40問、正解 25問）。
> **本資料の構成**: ①苦手分野の俯瞰 → ②ドメイン別の頻出ポイント → ③問題ごとのコア教訓。

---

## ① 苦手分野の俯瞰（ここから優先的に潰す）

| 優先度 | ドメイン | マーク済問題数 | 不正解率 | 状況 |
|---|---|---|---|---|
| 🔴 **最優先** | D6 マネジメント・ガバナンス | 6問 | **100%** (6/6) | **全滅。Organizations系を要復習** |
| 🔴 高 | D4 IAM | 11問 | **73%** (8/11) | フェデレーション/Cognitoが弱い |
| 🟠 高 | D2 ロギング・監視 | 13問 | **69%** (9/13) | "ログ集約"の選択肢を間違えやすい |
| 🟠 中 | D3 インフラセキュリティ | 11問 | **64%** (7/11) | PrivateLink/VPCエンドポイントが弱い |
| 🟡 低 | D1 脅威検知・対応 | 9問 | 44% (4/9) | GuardDuty周辺はOK、新サービスで取り違え |
| 🟢 低 | D5 データ保護 | 15問 | 40% (6/15) | KMS/Nitro Enclavesは強い |

**戦略**: D6→D4→D2の順で残り時間を使う。試験で迷ったら「Organizationsはガバナンス用、IAM Identity Centerは認証、Security Lakeはログ集約」を思い出す。

---

## ② ドメイン別の「狙われるポイント」

### D6 マネジメント・ガバナンス（全滅。最重要）

**頻出の引っかけパターン**:
- **SCP は "予防的"、AWS Config は "発見的"、Lambda/SSM Automation は "修正的"**。問題文に「**既存の**非準拠リソースを修正」とあれば SCP では不可（SCPは新規作成の防止のみ）。
- **CloudFormation**: 「一部メンバーだけデプロイ失敗」→ **サービスロール**を作成し、全員に `iam:PassRole` を付与（自分のIAM権限ではなくロールでデプロイ）。
- **root保護**: アクセスキー削除だけでは不十分。**SCP で root のサービスアクセスをブロック**するのが本命。
- **AWS Backup のライフサイクル**: S3 ライフサイクル設定は **AWS Backup 配下のスナップショットには効かない**。Backup プランの「保持期間（lifecycle）」で指定する。
- **SSM State Manager**: EC2 内部の **OS/アプリ設定のドリフト修復**。Config では OS 内部は監視できない（Configは "AWS リソース" の設定変更が対象）。
- **SCP で「組織外への共有を防ぐ」**: `aws:PrincipalOrgID` を `StringNotEquals` で条件指定して **Deny**（PrincipalArn の列挙では不十分）。

### D4 IAM（フェデレーション・Cognito で取りこぼし）

**頻出の引っかけパターン**:
- **IAM Identity Center のカスタマーマネージドポリシー**: ポリシーが**各メンバーアカウントに同じ名前で存在**していないと割り当て失敗。AWSマネージドポリシーは自動でOK。
- **Cognito の地域制限**: Hosted UI 自体には地理制限機能なし。**AWS WAF の Geo Match ルール**を Cognito の前で適用する。
- **マルチアカウント認証のスケーラブル解**: **IAM Identity Center + 外部IdP統合 + 権限セット**。「中央IAMロール＋各アカウントにIAMロール手動配布」は**運用負荷大で不適**。
- **サードパーティへの一時アクセス**: IAM Identity Center は使えない。**IAM Role + `sts:ExternalId` の信頼ポリシー**が正解（混乱代理問題の防止）。
- **CLI で MFA 強制下の操作**: `aws sts get-session-token --serial-number ... --token-code ...` で一時クレデンシャルを取得して使う。
- **Cognito + SAML IdP の組み合わせ**: SAML IdP を Cognito User Pool に追加し、属性マッピングする。API Gateway は Cognito Authorizer で保護、S3 静的サイトは CloudFront + OAC。
- **AWS と Cognito 両方でパスワードポリシー**: **IAM パスワードポリシー** で AWS 側、**Cognito ユーザープール設定**で Cognito 側、両方を個別に設定する必要がある。
- **AWS CLI で他のプロファイルと競合させない**: **`--profile` などコマンドラインオプションで都度指定**。環境変数や `~/.aws/config` の変更はリスク。

### D2 ロギング・監視（"集約" の選択肢を間違えやすい）

**頻出の引っかけパターン**:
- **マルチアカウント・マルチソースのログ集約 → Amazon Security Lake**（OCSF 形式、Parquet、サブスクライバーで Splunk/Datadog 連携）。CloudWatch Logs + サブスクリプションフィルターでは要件を満たさない。
- **過去90日超の CloudTrail 分析 → CloudTrail Lake**（最大7年保持、SQLクエリ可）。S3+Athena は安価だが、CloudTrail に特化していない。
- **Security Lake セットアップ順序**: ①delegated admin指定 → ②ログソース/リージョン選択 → ③ストレージ設定 → ④作成 → ⑤検証 → ⑥サブスクライバー追加。
- **Cognito のログイン履歴をクエリ**: **CloudTrail で `InitiateAuth` イベントを S3 に → Athena**。CloudWatch メトリクスは集計で詳細クエリ不可。
- **Secrets Manager の異常 API 検出**: EventBridge → **CloudWatch Logs** → **メトリクスフィルター** + **異常検出** → SNS 通知。
- **マルウェアの C2 通信先 EC2 を特定**: NACLで既にブロック済なら、**VPC Flow Logs の REJECT レコード**を見る。Network Firewall を追加するのは過剰。
- **NTP ポリシー違反検出**: VPC Flow Logs で **Amazon Time Sync Service (`169.254.169.123`) "以外"** への通信を検出（"正しい使用"の検出ではない）。
- **Lambda が CloudWatch Logs に出ない**: ①**実行ロールに logs:CreateLogGroup/Stream/PutLogEvents** ②**ログ出力するコード**自体があるか確認。
- **CloudWatch エージェント＋VPCエンドポイントでログが出ない**: ①EC2 のロールに **logs:CreateLogStream / PutLogEvents** がある ②**VPC エンドポイントに DNS 名前解決有効** ③CloudWatch エージェント設定ファイル。
- **Config 非準拠をリアルタイム通知**: AWS Config の compliance change → **EventBridge** → SNS（CloudWatch Logs + メトリクスフィルター経由は遠回り）。

### D3 インフラセキュリティ（PrivateLink/VPCエンドポイントが弱い）

**頻出の引っかけパターン**:
- **TLS 終端 + リダイレクト + 性能考慮 → ALB**（NLBはL4でHTTPリダイレクトルール不可）。証明書は ACM、リスナー443にアタッチ。
- **DDoS L3/L4 防御の3点セット**: **AWS Shield + Route 53 + NLB**（ACM/S3/GuardDuty は無関係）。
- **多数(1500個)の CIDR を許可**: SG は60件、NACLは20件で破綻。**AWS PrivateLink + エンドポイントサービス**で許可リスト不要に。
- **クロスアカウント DB アクセス**: アプリVPCに**インバウンドルールなしSG**、DB VPCに「**アプリVPCのSGを信頼**するSG」（IP範囲やNACLは不要）。
- **VPC内 Lambda が Secrets Manager に届かない**: **Secrets Manager の Interface VPC エンドポイント**を Lambda のサブネットに作る（NAT GW追加はインターネット禁止ポリシー違反）。
- **EC2 がプライベートサブネットから S3/SQS だけ許可**: **VPC エンドポイントのエンドポイントポリシー**でリソース制限。Firewall Manager は組織全体のセキュリティポリシー管理用で別物。
- **Verified Access**: VPN なしでデバイスポスチャ・IDPでアプリへのアクセス制御。**ALBのエンドポイント**を Verified Access に登録。WAFやVerified Permissionsとは混同しない。
- **IMDSv1 利用検出**: CloudWatch メトリクス `MetadataNoToken` が 0 を維持しているか監視（SGではIMDSはブロック不可、リンクローカル169.254.169.254）。
- **EC2 マルウェア即時封じ込め**: **NACL で送信トラフィック全拒否**（SGは既存接続継続するので不十分）→ 別のSG/インスタンスから調査。WAFは EC2に直接適用不可。
- **CloudFront 全オリジン共通ヘッダー追加**: **Lambda@Edge** を **origin response** イベントで実行（オリジンカスタムヘッダーは送信用なので別物）。
- **クロスアカウントS3+KMS**: ①S3バケットポリシーで cross-account 許可 ②KMSキーポリシーでクロスアカウント許可 ③**KMS用のInterface VPCエンドポイント**で kms:Decrypt をインターネット経由させない。

### D1 脅威検知・インシデント対応

**頻出の引っかけパターン**:
- **Fargate コンテナ調査**: **ECS Exec** + Systems Manager 権限をタスクロールに（EC2に移行は運用負荷大）。
- **GuardDuty High検出を通知＋分析**: **GuardDuty → EventBridge** → ①SNS(通知) ②**Kinesis Data Firehose → OpenSearch Service → OpenSearch Dashboards**（QuickSightは静的可視化向け、Kinesis Data Streams は重い）。
- **EKS/RDS 監視を最小運用負荷で**: GuardDuty + **EKS Protection** + **RDS Protection** + 管理者アカウントの委任。
- **Lambda の継続的脆弱性監視（タグで除外も）**: **Amazon Inspector** の組織委任管理者 + **タグで除外（exclusion）**。
- **Bedrock の LLM セキュリティ**: ①**Bedrock Guardrails** で機密情報/有害コンテンツをブロック ②入力サニタイズ + プロンプトテンプレートでユーザー入力を分離。
- **DDoS 再発防止の2点**: **AWS Shield Advanced**（DDoSコスト保護込み）+ **Route 53 ヘルスチェック**でフェイルオーバー（または WAFカスタムルール）。
- **Amazon Q Developer**: ①**IDEセキュリティスキャン**を有効化（リアルタイム脆弱性検出） ②管理者が**コードリファレンス（OSS追跡）**を有効化。

### D5 データ保護（強み分野だが KMS の細部に注意）

**頻出の引っかけパターン**:
- **S3 改ざん防止90日（root/admin含めて削除不可）**: S3 Object Lock **コンプライアンスモード**（ガバナンスモードは `BypassGovernanceRetention` 権限で削除可能）。
- **KMS マルチリージョンキー作成順**: ①プライマリ作成（マルチリージョン有効化）→ ②エイリアス/説明設定 → ③レプリカ作成 → ④アプリ設定 → ⑤検証。「キーの用途を後から指定」は不可。
- **退職者の Lambda コード署名を無効化**: **AWS Signer の署名プロファイルを Revoke**（IAM権限剥奪では既存署名済みコードは動く）。
- **トークン化を他コンポーネントから隔離**: **AWS Nitro Enclaves**（Dedicated/Partition Placement Group はセキュリティ境界ではない）。
- **PFS で秘密鍵漏洩時の被害最小化**: HTTPS リスナー + **PFS をサポートする ALB セキュリティポリシー**（TCP リスナーは TLS終端をALBで行えない）。
- **KMS スロットリング回避（クライアントサイド暗号化）**: **AWS Encryption SDK のデータキーキャッシング** + キャッシング暗号マテリアルマネージャー。
- **KMS Grant 直後の AccessDeniedException**: 伝播遅延あり。**CreateGrant のレスポンスの GrantToken** をそのまま使うと即時利用可。
- **クロスアカウント数百ベンダーのキーアクセス（毎週変動）**: **KMS Grant をプログラム管理**（信頼ポリシー更新は各ベンダー調整必要で運用負荷大）。
- **SSE-KMS で職務分離**: 運用チーム=**バケットポリシーで SSE-KMS強制**、セキュリティチーム=**KMSキーポリシー**でアクセス制御。
- **S3 アクセス Access Denied (SSE-KMS暗号化)**: IAMで `kms:Decrypt` 許可済でも、**KMSキーポリシー側でアカウント/プリンシパル拒否**されていればNG（両方の許可が必要）。
- **Secrets Manager のローテーション失敗 "Unable to log into database"**: ①Lambda が DB に到達できるか（**SG/サブネット**）②**rotation 関数の DB プロビジョニング部** のロジック。
- **SNS の機密情報マスキング**: **Inbound Message Data Protection Policy** + **De-identify** オペレーション（MacieはS3用、リアルタイム不可）。
- **EC2 Image Builder で S3 アクセス失敗**: ①**インスタンスプロファイルに s3:PutObject 許可** ②**バケットポリシー で Image Builder ロール許可**（最小権限のため両方）。

---

## ③ 問題ごとのコア教訓（マーク済65問）

> 各行: `[マーク] 問題ID | シナリオ要約 → 正解 / 引っかけポイント`
> `✗` = ユーザー不正解、`○` = 正解だがマーク済（理解浅め）

### D6 マネジメント・ガバナンス（最優先・全滅）

- **[✗] 演習2-問題48 | rootユーザー漏洩リスクに備える** → **SCPでrootのサービスアクセスをブロック**。アクセスキー削除や EventBridge 監視だけでは不十分。rootパスワードは削除不可。
- **[✗] 演習2-問題49 | CloudFormation で一部メンバーだけデプロイ失敗** → **サービスロール作成（cloudformation.amazonaws.com信頼）+ ユーザーに iam:PassRole 付与 + CFn 実行時にロール指定**。3つ全部必要。
- **[✗] 演習4-問題28 | 既存+将来のS3非準拠を修正** → **Config アグリゲーター + Lambda 自動修復**。SCP は新規作成防止のみで "既存の修正" 不可。
- **[✗] 演習4-問題32 | EC2 のOS/ソフトウェアの設定維持＆ドリフト修復** → **SSM State Manager + Configでコンプライアンス監視**。Config 自身は OS 内部のドリフト修復不可。StackSets はデプロイ用。
- **[✗] 演習4-問題53 | OrganizationS3共有を組織外へ防ぐ SCP** → `s3:*` を **Deny** し、`StringNotEquals` で `aws:PrincipalOrgID` を `${aws:PrincipalOrgID}` と比較。PrincipalArn列挙は網羅不可。
- **[✗] 演習4-問題58 | RDSスナップショット5年保持** → **AWS Backup のバックアッププランの保持期間** で5年指定。AWS Backup配下のスナップショットは S3ライフサイクル不適用。

### D4 IAM（高優先）

**不正解だった問題**:
- **[✗] 演習1-問題40 | Cognito Hosted UIでフランス国外サインアップ防止** → ①**Lambda Pre Sign-up Trigger** でカスタム検証（メールドメイン等） ②**WAF Geo Matchルール** を Cognito の前で適用。
- **[✗] 演習1-問題59 | 100以上のアカウント+外部IdPでロールベースアクセス** → **IAM Identity Center + 外部IdP統合 + 権限セット**。各アカウントに手動でIAMロール配布は運用大。
- **[✗] 演習2-問題51 | マルチアカウント認証をAWSネイティブのみで** → IAM Identity Center を有効化済の次は、**IAM Identity Center のデフォルトディレクトリでユーザー/グループ作成、権限セットを割り当て**。AD Connector/Microsoft AD は不要。
- **[✗] 演習3-問題8 | サードパーティアカウントへの一時アクセス（同一認証情報の使い回し防止）** → **各外部アカウントに固有のIAMロール + `sts:ExternalId`条件**。IAM Identity Center は社内向けで非対応。
- **[✗] 演習3-問題46 | SAML IdPで S3静的+API GW+DynamoDB のWebアプリ認証** → ①**Cognito User Pool に SAML IdP 追加**＋属性マッピング ②**API Gateway を Cognito オーソライザーで保護** ③**CloudFront + OAC で S3 静的サイト**。
- **[✗] 演習4-問題22 | SAMLフェデレーション失敗のトラブルシュート** → **SAML IdPログを確認＋CloudTrailでユーザーAPI呼び出し確認**。「別アカウントで再現」「ポリシーシミュレーター」は最終手段で非効率。
- **[✗] 演習4-問題45 | 複数アカウントロール使用環境で認証情報競合** → **AWS CLI のコマンドラインオプション(`--profile`等)で都度指定**。環境変数や設定ファイルは永続化されて競合の元。
- **[✗] 演習4-問題51 | IAMとCognito両方でパスワード最小文字数** → ①**IAM パスワードポリシー更新** ②**Cognito User Pool のパスワード設定更新**。IdPごとに個別設定が必要。

**理解浅めの正解問題**:
- **[○] 演習1-問題5 | IAM Identity Center 権限セット割り当て失敗** → **各メンバーアカウントに同名のカスタマーマネージドポリシーを作成**。AWSマネージドポリシーは全アカウント共有だがカスタマー管理は各アカウントに必要。
- **[○] 演習2-問題16 | MFA強制IAMポリシー後 CLI が失敗** → **`aws sts get-session-token --serial-number ... --token-code ...`** で一時credentialsを取得して使う。
- **[○] 演習2-問題30 | クロスアカウント AssumeRole で AccessDenied** → ①ターゲット側ロールの**信頼ポリシー**で sts:AssumeRole 許可 ②呼び出し側ロール(Lambda)の**権限ポリシー**で sts:AssumeRole 許可。両方必要。

### D2 ロギング・監視（高優先）

**不正解だった問題**:
- **[✗] 演習1-問題46 | VPCエンドポイント設定済みでCloudWatch Logsが届かない（3つ）** → ①**EC2インスタンスロールに `logs:CreateLogStream` 等の権限** ②**VPCで DNSホスト名/解決有効化**（VPCエンドポイントは PrivateDNS で動くため） ③**CloudWatchエージェントの設定ファイルでログパス確認**。
- **[✗] 演習2-問題55 | Security Lake セットアップ順序** → ①delegated admin → ②**ログソース/リージョン/アカウント指定** → ③ストレージ＆作成 → ④検証 → ⑤サブスクライバー。順序逆転は要注意。
- **[✗] 演習2-問題59 | 60日間のIAMアクセスキー使用調査（PII含む）** → ①**Macie**でS3バケット内のPIIスキャン ②**Athena**でCloudTrailログから当該アクセスキーの GetObject 呼び出しをクエリ ③**S3 サーバーアクセスログ**を有効化して詳細追跡。
- **[✗] 演習4-問題11 | Secrets Manager の異常 GetSecretValue 通知** → EventBridge → **CloudWatch Logs** をターゲットに → **メトリクスフィルター + 異常検出（anomaly detection）** → CloudWatch Alarm → SNS。CloudTrail を EventBridge のターゲットには直接できない。
- **[✗] 演習4-問題12 | Cognito ログイン試行（成功/失敗）の履歴クエリ** → **CloudTrail有効化 → S3保存 → Athena で `InitiateAuth` イベントをクエリ**。CloudWatchメトリクスは集計のみ、詳細分析不可。
- **[✗] 演習4-問題31 | NACL でブロック済の TCP 2905 元 EC2 を特定** → **VPC Flow Logs を有効化、REJECT レコードをクエリ**（既にブロック済なので追加 Firewall は過剰）。
- **[✗] 演習4-問題40 | Amazon Time Sync 以外を使うEC2を検出** → **VPC Flow Logs で `169.254.169.123` 以外 への NTP トラフィックを検出**。CloudTrail は API 呼び出し用で NTP は対象外。
- **[✗] 演習4-問題60 | Lambda の CloudWatch Logs が出ない（2つ）** → ①**実行ロールに CloudWatch Logs 書き込み権限** ②**コードに console.log 等のログ出力がある**か確認。
- **[✗] 演習4-問題62 | SG 0.0.0.0/0 SSH違反のリアルタイム通知** → **AWS Config (restricted-ssh) + EventBridgeイベントルール (compliance change) → SNS**。CloudWatch Logs経由は遠回り。

**理解浅めの正解問題**:
- **[○] 演習1-問題18 | Organizations+Marketplace+オンプレのログ一元化** → **Security Lake + 委任管理者 + Athena**。OCSF正規化対応。
- **[○] 演習2-問題2 | ELBログ集約と暗号スイートメトリクス** → **S3にELBログ + Athenaクエリ + 結果をCloudWatchにpublish**。
- **[○] 演習2-問題32 | CloudTrail 7年保持+IAMユーザー単位検索+異常検出** → **CloudTrail Lake** の7年保持料金プラン（最大2,557日）+ SQLクエリ。
- **[○] 演習2-問題40 | マルチアカウント+OCSF+サードパーティ統合** → **Security Lake** + Athena + サブスクライバー。

### D3 インフラセキュリティ（中優先）

**不正解だった問題**:
- **[✗] 演習1-問題14 | L3/L4 DDoS 防御（3つ）** → **AWS Shield + Route 53 + NLB**。ACM/S3/GuardDuty は無関係。
- **[✗] 演習3-問題5 | SSH管理サブネットからのみ、HTTPSは全公開（2つ）** → SG で `port 22` は `192.168.100.0/24` のみ、`port 443` は `0.0.0.0/0` から許可。
- **[✗] 演習3-問題21 | クロスアカウントS3+KMSをインターネット経由なし（2つ）** → ①**S3バケットポリシーで cross-accountアクセス許可確認** ②**KMSのInterface VPC エンドポイント追加 + KMSキーポリシーでクロスアカウント許可**。
- **[✗] 演習3-問題35 | EC2 から S3/SQS のみプライベート接続（3つ）** → ①**S3ゲートウェイ VPC エンドポイント** ②**SQS インターフェース VPC エンドポイント** ③**VPC エンドポイントのエンドポイントポリシーでリソース制限**（最小権限）。
- **[✗] 演習3-問題37 | プライベート VPC の Lambda が Secrets Manager 到達不可** → **Secrets Manager の Interface VPC エンドポイントを Lambda のサブネットに追加**。NAT GW追加は「インターネット禁止」ポリシー違反。
- **[✗] 演習4-問題10 | 1,500子会社のみアクセス（NLB 後ろ）** → **PrivateLink エンドポイントサービス**を親で作成、子会社が Interface エンドポイントを作成。SG 60制限・NACL 20制限を超える数に対応。
- **[✗] 演習4-問題30 | VPN なし＋デバイスポスチャでアプリアクセス** → **AWS Verified Access** に ALB エンドポイント登録。WAF はデバイスポスチャ非対応、Verified Permissions は app内認可用。

**理解浅めの正解問題**:
- **[○] 演習1-問題11 | TLS終端+HTTP→HTTPSリダイレクト+性能考慮** → **ALB**（NLBはL4でHTTPリダイレクト不可）+ ACM証明書を443にアタッチ。
- **[○] 演習2-問題43 | クロスアカウントVPCピアリング先DBへ条件付きアクセス** → アプリVPC側に**インバウンド無しSG**、DB側で「**アプリVPCのSG**からのTCP1521許可」。NACL は不要。
- **[○] 演習3-問題64 | CloudFront 2オリジン共通ヘッダー追加** → **Lambda@Edge** を **origin response** イベントで実行。両オリジンに統一適用。
- **[○] 演習4-問題18 | IMDSv1 利用検出** → **CloudWatch メトリクス `MetadataNoToken` が 0**を維持しているか監視。SG では IMDS のリンクローカルアドレスをブロック不可。

### D1 脅威検知・インシデント対応

**不正解だった問題**:
- **[✗] 演習1-問題43 | EC2 不審通信の即時遮断＋安全な調査** → **NACL でサブネットの送信トラフィック全拒否** → 別のSG/新EC2 + 診断ツールで安全に調査。WAF は EC2 直接適用不可、SG ではブロック既存接続継続するので不十分。
- **[✗] 演習3-問題55 | GuardDuty High検出を通知＋可視化** → **GuardDuty → EventBridge (イベントパターンで High のみ) → ①SNS(通知) ②Kinesis Data Firehose → OpenSearch Service → OpenSearch Dashboards**。QuickSightは静的、Data Streams は重い。
- **[✗] 演習4-問題36 | DDoS 再発防止（2つ）** → **AWS Shield Advanced**（DDoSコスト保護込み）+ **Route 53 ヘルスチェック でフェイルオーバー** または **AWS WAF カスタムルール**。
- **[✗] 演習4-問題41 | 数百アカウントの Lambda 脆弱性監視+テスト除外（2つ）** → ①**Inspector の Organizations 委任管理者** で全アカウント有効化 ②**Lambda のタグで除外（exclusion）**設定し、ダッシュボードに表示しない。

**理解浅めの正解問題**:
- **[○] 演習1-問題31 | Fargate コンテナ内のメモリダンプ取得** → **ECS Exec** + タスクロールに SSM 権限。EC2 移行は運用負荷大。
- **[○] 演習2-問題35 | Amazon Q Developer のセキュア化（2つ）** → ①**IDE セキュリティスキャン**有効化 ②**コードリファレンス（OSSライセンス追跡）有効化**。
- **[○] 演習4-問題13 | Bedrock LLM のプロンプトインジェクション対策（2つ）** → ①**Bedrock Guardrails**（機密情報/有害ブロック） ②**入力サニタイズ+プロンプトテンプレートでユーザー入力分離**。
- **[○] 演習4-問題21 | EKS + Aurora を最小運用負荷で監視** → **GuardDuty + EKS Protection + RDS Protection + 委任管理者**。エージェントレス。
- **[○] 演習4-問題38 | EKS コンテナイメージ脆弱性+ノード間暗号化（2つ）** → **Inspector でコンテナイメージスキャン + EKS のノード間暗号化を有効化**。

### D5 データ保護（強み分野）

**不正解だった問題**:
- **[✗] 演習1-問題42 | KMS マルチリージョンキー作成順序** → ①**プライマリ作成（マルチリージョン有効）+ エイリアス/説明** → ②**用途指定（マルチリージョン）** → ③**レプリカ作成** → ④アプリ設定 → ⑤検証。順序を覚える。
- **[✗] 演習2-問題3 | EC2→S3+SSE-KMS（ゲートウェイ VPCE）でアクセス失敗（3つ）** → ①**IAMインスタンスプロファイル**に s3:GetObject 不足 ②**KMSキーポリシー**で IAMロール不許可 ③**KMSはゲートウェイエンドポイント非対応**なのでInterface VPCエンドポイントが必要。
- **[✗] 演習2-問題9 | Secrets Manager ローテーション "Unable to log into database"（2つ）** → ①Lambda の SG が EC2 への送信を許可しているか + EC2 の SG が Lambda の SG からの受信を許可しているか ②**Lambda 関数の `setSecret` ステップのコード**で DB プロビジョニング処理が正しいか。
- **[✗] 演習3-問題9 | SSE-KMS の S3 に IAM ユーザーから Access Denied** → **KMSキーポリシー側でアカウント/プリンシパルが拒否されている**可能性。IAMポリシーで kms:Decrypt 許可済でも、キーポリシーで拒否されればNG。
- **[✗] 演習3-問題57 | EC2 Image Builder で S3 ログに 403（2つ）** → ①**インスタンスプロファイルに `s3:PutObject` 許可** ②**S3 バケットポリシーで Image Builder ロールを許可**。最小権限のため両方明示。
- **[✗] 演習4-問題24 | 数百ベンダーへのKMSキーアクセス（毎週変動）** → **KMS Grant をプログラム的に作成・取り消し**。信頼ポリシー更新は各ベンダー側調整が必要で運用大。

**理解浅めの正解問題**:
- **[○] 演習1-問題25 | 退職開発者のコード署名無効化** → **AWS Signer の署名プロファイル Revoke**（IAM権限剥奪では既存署名は引き続き有効）。
- **[○] 演習1-問題58 | SNS メッセージの機密データを誤公開防止** → **Inbound Message Data Protection Policy + De-identify**。Macie はS3用でリアルタイム不可。
- **[○] 演習2-問題8 / 演習3-問題2 | クレジットカードトークン化を完全分離** → **AWS Nitro Enclaves**。Dedicated/Partition Placement Group はセキュリティ境界ではない。
- **[○] 演習2-問題21 | TLS秘密鍵漏洩でも過去通信を守る** → **ALB + PFS対応 HTTPSセキュリティポリシー**。TCP リスナーでは ALB が TLS 終端できない。
- **[○] 演習3-問題61 | KMS スロットリング回避（クライアントサイド暗号化）** → **AWS Encryption SDK のデータキーキャッシング**。
- **[○] 演習4-問題7 | S3 90日 root含めて削除不可** → **S3 Object Lock コンプライアンスモード**。ガバナンスモードは BypassGovernanceRetention で削除可。
- **[○] 演習4-問題16 | KMS Grant 直後の AccessDeniedException** → **CreateGrantレスポンスの GrantToken を即時利用**。伝播遅延に GrantToken で対応。
- **[○] 演習4-問題42 | S3 SSE で職務分離（運用 vs セキュリティ）** → 運用=**バケットポリシーで SSE-KMS 強制**、セキュリティ=**KMSキーポリシーで使用制御**。SSE-S3 や SSE-C は分離不能。

---

## ④ 試験当日の「迷ったら思い出す」短縮チェック

### サービス選択の即決判断
| やりたいこと | 選ぶサービス | 引っかけ |
|---|---|---|
| マルチアカウントのログ集約・OCSF | **Security Lake** | CloudTrail Lake は監査用 |
| 90日超のCloudTrail分析 | **CloudTrail Lake** | S3+Athena より特化 |
| マルチアカウント認証 | **IAM Identity Center** | 外部一時アクセスは IAM Role+ExternalId |
| マルチアカウント脅威検出 | **GuardDuty + 委任管理者** | EKS/RDS Protectionも有効化 |
| Lambda/EC2/コンテナの脆弱性 | **Amazon Inspector** | コードの欠陥はCodeGuru |
| OS/ソフトのドリフト修復 | **SSM State Manager** | ConfigはAWSリソース設定のみ |
| AWSリソース設定の修復 | **Config + Lambda or SSM Automation** | SCPは新規防止のみ |
| 組織外への共有を防止 | **SCP + `aws:PrincipalOrgID`** | PrincipalArn列挙はNG |
| クレデンシャル安全保管+ローテーション | **Secrets Manager** | Parameter Storeは無料だがローテーション弱い |
| EC2上で機密処理を隔離 | **Nitro Enclaves** | Dedicated Instance は性能用 |
| プライベートにAPI呼び出し | **Interface VPCエンドポイント** | S3/DynamoDBは Gateway |
| 多数のVPCから集中アクセス公開 | **PrivateLink エンドポイントサービス** | NLB の前で |
| DDoS L3/L4 | **Shield + Route 53 + NLB** | ACM/S3/GuardDuty 違う |
| VPNなしの社内アプリ | **Verified Access** | Verified Permissions はapp内認可 |
| CloudFront 全オリジン共通ヘッダー | **Lambda@Edge (origin response)** | カスタムヘッダーは送信用で別物 |
| マルウェア即時遮断 | **NACL で送信全拒否** | SGは既存接続継続 |
| Cognito 国別制限 | **WAF Geo Match** | Hosted UI自体には地理機能なし |

### Object Lock のモード
- **コンプライアンスモード**: root/admin **でも** 削除不可。WORM要件向け。
- **ガバナンスモード**: `s3:BypassGovernanceRetention` 権限保有者は削除可。

### KMS で迷ったら
- **クロスアカウントで多数の動的相手** → **Grant** をプログラム管理
- **直後のAccessDeniedException** → CreateGrant の **GrantToken** を即時利用
- **クライアントサイド大量暗号化のスロットリング** → **データキーキャッシング**
- **マルチリージョン** → プライマリ作成 → レプリカ作成（既存通常キーはマルチリージョン化不可）
- **SSE-KMS で S3 が Access Denied** → **キーポリシー側** の許可を疑う

### Cognito vs IAM Identity Center
- **エンドユーザー（消費者）の認証** → **Cognito User Pool**（OAuth/OIDC/SAML IdP連携）
- **社員のマルチアカウント AWS アクセス** → **IAM Identity Center**
- **サードパーティの一時アクセス** → **IAM Role + sts:ExternalId**

### Config 自動修復のパターン
- 自動修復アクション = **SSM Automation ドキュメント** を関連付け
- Configイベント → **EventBridge** → SNS（通知）/Lambda（修復）

---

## ⑤ 最終チェック（試験前夜にこれだけ見る）

1. **D6が全滅** → SCP/Config/CloudFormation/Backup/SSM の役割境界を再確認
2. **D4で Cognito と Identity Center の役割を混同しない**
3. **D2で「集約」「長期保存」「リアルタイム通知」の3パターンの正解サービスを暗記**
4. **D3で「VPCエンドポイント=サービスごとに必要」「PrivateLinkは多数接続向け」**
5. **D1で「GuardDuty→EventBridge→SNS+Firehose+OpenSearch」のお決まり構成**
6. **D5で「Object Lockコンプライアンス」「Nitro Enclaves」「PFS」「KMS Grant」**

頑張ってください 🍀
