# AWS SAP試験 勉強メモ

このリポジトリは、AWS Certified Solutions Architect - Professional (SAP) 試験の合格を目指すための勉強メモです。

## ドキュメント作成ルール

- **[ドキュメント作成ルール](rule.md)** - メモの書き方、形式、スタイルガイドライン

## 目次

### 2026-01-06
- **[AWS Application Migration Service (MGN) と Refactor Spaces](20260106_mgn-refactor-spaces.md)**
  - MGN（旧SMS代替、継続的レプリケーション、自動最適化）とRefactor Spaces（ストラングラーフィグパターン、モノリス→マイクロサービス段階的移行）。Re-host/Re-platform/Refactor違い、MGNのインスタンスタイプ・EBS自動最適化、テストカットオーバー、Refactor SpacesのURLパスベースルーティング、Transit Gateway自動管理をMermaid図と表で整理。
- **[Security Hub & Config 自動修復](20260106_security-hub-config-auto-remediation.md)**
  - 2024年以降の自動修復（Auto-remediation）設計。EventBridge + Lambda（カスタム修復）とEventBridge + SSM Automation（標準修復）、Security HubとConfigの統合（ASFF形式、Finding集約）、Severity別修復戦略（CRITICAL/HIGH即座修復、MEDIUM通知のみ）、Step Functionsによる複数リソースタイプ統合修復をMermaid図と表で整理。
- **[AWS Control Tower Landing Zone 3.0以降](20260106_control-tower-landing-zone-3.md)**
  - Landing Zone 3.0の新機能。既存アカウントインポート（Organizations配下アカウント管理下追加）、カスタムガードレール（SCP、Config Rules、CloudFormation Hooks）、Account Factory for Terraform (AFT、Git+CI/CD自動化)、Control Tower API（EnrollAccount等）、統制の自動展開、自動修復設計（EventBridge + Lambda/SSM）をMermaid図と表で整理。
- **[AWSセキュリティサービスの通知機能](20260106_security-notifications.md)**
  - Inspector、Security Hub、GuardDuty、Config、Macieの通知方法比較。直接SNS通知可能なサービス（Config）とEventBridge経由必須のサービス（GuardDuty、Security Hub、Inspector）の違い、SNSトピックポリシーのPrincipal設定（`events.amazonaws.com` vs `config.amazonaws.com`）、Severity別通知パターン、よくある設定ミスをMermaid図と表で整理。
- **[データベース移行（SCT・DMS）](20260106_database-migration.md)**
  - AWS SCT（スキーマ変換、ローカルPC、無料）とDMS（データ移行、レプリケーションインスタンス）の比較。SCTエージェント（DWH移行・Cassandra→DynamoDB移行、並列処理）、Cassandra→DynamoDB移行の2つの方法（SCTエージェント推奨、DMS代替）、EC2クローン経由、CDC（継続的レプリケーション）、移行パターンをMermaid図と表で整理。

### 2026-01-03
- **[Amazon EMR](20260103_emr.md)**
  - EMRのノードタイプ（マスター、コア、タスク）、スポットインスタンス戦略（タスクノード推奨）、HDFS vs EMRFS（S3）、EMR Serverless、自動スケーリング、コスト最適化をMermaid図と表で整理。
- **[CloudFront 署名付きURL・キーグループ](20260103_cloudfront-signed-url.md)**
  - CloudFront署名付きURL/Cookie、キーグループ（推奨、RSA公開鍵、最大5個）vs 信頼された署名者（レガシー）、Custom Policy（IP制限・時限公開）、キーローテーション、S3署名付きURLとの違い、OAC組み合わせをMermaid図と表で整理。
- **[CloudHSM・KMS比較](20260103_cloudhsm-kms.md)**
  - AWS CloudHSM（FIPS 140-2 Level 3、シングルテナント、$1,045/月）とKMS（Level 2、マルチテナント、$1/月）の比較。RDS暗号化はKMSのみ対応、CloudHSMは完全ユーザー制御、Custom Key Store、ユースケース別推奨をMermaid図と表で整理。
- **[クロスVPC DNS・RDS接続](20260103_cross-vpc-dns.md)**
  - 異なるVPCのプライベートサブネットRDS接続時のDNS挙動。Route 53 Private Hosted Zone（VPC関連付け、Aレコード）、Transit Gateway + PHZ、VPC Peering + PHZ、Route 53 Resolver Endpoint（DNS転送）、AWS RAM PHZ共有をMermaid図と表で整理。
- **[VPC IPv6対応](20260103_vpc-ipv6.md)**
  - VPCでのIPv6設定（/56 VPC、/64サブネット）、デュアルスタック構成、Egress-only Internet Gateway（IPv6アウトバウンド専用、無料）、NAT Gateway比較、セキュリティグループ（::/0）、IPv6移行戦略をMermaid図と表で整理。

### 2026-01-02
- **[DynamoDB と DocumentDB](20260102_dynamodb-documentdb.md)**
  - DynamoDB（キー・バリュー/ドキュメント）とDocumentDB（MongoDB互換）の比較。データモデル、スケーラビリティ、クエリ機能、料金モデル、MongoDB互換性、ユースケース別推奨を表形式で整理。
- **[ALB と NLB](20260102_alb-nlb.md)**
  - Application Load Balancer（レイヤー7、HTTP/HTTPS）とNetwork Load Balancer（レイヤー4、TCP/UDP）の比較。プロトコル、ルーティング機能、パフォーマンス（NLB:100万req/秒）、静的IP、送信元IP保持、セキュリティ機能、ユースケース別推奨を解説。
- **[ゲートウェイ比較（VGW・DX Gateway・Transit Gateway）](20260102_gateway-comparison.md)**
  - Virtual Private Gateway（単一VPC）、Direct Connect Gateway（最大10 VPC、マルチリージョン）、Transit Gateway（最大5,000 VPC、ハブ&スポーク）の比較。Private VIF/Transit VIF、ルーティング、TGW Peering、組み合わせ構成をMermaid図と表で整理。
- **[ネットワーク管理サービス（Manager系）](20260102_network-managers.md)**
  - Transit Gateway Network Manager（グローバル監視）、VPC IPAM（CIDR自動割り当て）、Cloud WAN（SD-WAN）、VPC Reachability Analyzer（到達性分析）、Network Access Analyzer（意図しないアクセス検出）の比較。Route Analyzer、セグメント分離をMermaid図と表で整理。
- **[バックアップ・ライフサイクル管理サービス](20260102_backup-lifecycle.md)**
  - AWS Backup（15種類以上対応、Vault Lock）、Data Lifecycle Manager（EBS/AMI、無料）、S3 Lifecycle（ストレージクラス移行）、EFS Lifecycle（IA/Archive）、RDS自動バックアップ（PITR）、Aurora Backtrack（数分巻き戻し）の比較。保持期間、コンプライアンス、料金をMermaid図と表で整理。

### 2026-01-01
- **[EC2 料金・リザーブドインスタンス](20260101_ec2-pricing-ri.md)**
  - EC2の料金モデル比較、リザーブドインスタンスの正規化係数（small=1、倍々で増加）、サイズ柔軟性、Savings Plansとの違い、購入戦略を表形式で解説。
- **[AWSコスト管理サービス](20260101_cost-management.md)**
  - Cost Explorer（3ヶ月予測）、Budgets（予測アラート）、CUR、Cost Anomaly Detection、Compute Optimizer、Amazon Forecastの違いと使い分けをMermaid図と表で整理。
- **[AWS Snow Family](20260101_snow-family.md)**
  - Snowcone、Snowball Edge（Storage/Compute）、Snowmobileの容量・用途比較。廃止された初代Snowballと現行製品の違い、DataSync連携、エッジコンピューティング機能を表形式で整理。
- **[ネットワーク帯域・Direct Connect](20260101_network-bandwidth.md)**
  - Direct Connect（1/10/100Gbps）、Site-to-Site VPN（最大1.25Gbps/トンネル）の帯域比較。データ転送時間計算表（1TB@1Gbps=2.8時間等）、接続方法選択基準、DataSync・Transfer Accelerationによる最適化を解説。

### 2025-12-31
- **[ElastiCache](20251231_elasticache.md)**
  - Amazon ElastiCacheのRedis/Memcached比較、クラスタモード、マルチAZ、セッション管理（スティッキーセッション含む）、スケーリング戦略を表形式で整理。
- **[RDS・Aurora エンドポイント](20251231_rds-aurora-endpoints.md)**
  - RDSとAuroraの各種エンドポイント（クラスター、リーダー、カスタム、インスタンス）の違い、フェイルオーバー動作、負荷分散、接続戦略のベストプラクティスをMermaid図と表で解説。

### 2025-12-30
- **[セキュリティ・コンプライアンスサービス](20251230_security-compliance-services.md)**
  - Security Hub、Config、Audit Manager、Detective、Macie、GuardDuty、Inspector、IAM Access Analyzer、Firewall Managerの特徴と依存関係をMermaid図と表で整理。各サービスの用途と連携方法を解説。

### 2025-12-28
- **[S3ストレージクラス](20251228_s3-storage-classes.md)**
  - Amazon S3の各ストレージクラスの特徴、用途、データ取り出し時間、コスト比較を表形式でまとめたドキュメント。Intelligent-Tieringの階層移行条件も解説。
