# AWS Application Migration Service (MGN) と Refactor Spaces

作成日: 2026-01-06

## 概要

2024年以降のSAP試験では、単なる**Re-host（リフト&シフト）**ではなく、**Re-platform（最適化移行）**とモダナイゼーションが重視されます。**AWS Application Migration Service (MGN)**は旧SMS（Server Migration Service）に代わる新しい移行サービス、**AWS Migration Hub Refactor Spaces**はモノリスからマイクロサービスへの段階的移行を支援するサービスです。

## 7つのR（移行戦略）

| 戦略 | 名称 | 説明 | 関連サービス |
|---|---|---|---|
| **Re-host** | Lift & Shift | そのままクラウドに移行 | **MGN**、CloudEndure |
| **Re-platform** | Lift, Tinker & Shift | 軽微な最適化を加えて移行 | **MGN** + 最適化 |
| **Re-purchase** | Drop & Shop | SaaSに置き換え | Marketplace |
| **Refactor/Re-architect** | Modernize | アーキテクチャを再設計 | **Refactor Spaces**、App2Container |
| **Retire** | - | 使わないものを廃止 | - |
| **Retain** | - | 現状維持 | - |
| **Relocate** | - | VMware移行 | VMware Cloud on AWS |

**試験での重要ポイント**: Re-hostとRe-platformの違い、MGNの最適化機能、Refactor Spacesのストラングラーフィグパターン

## AWS Application Migration Service (MGN)

### Server Migration Service (SMS) vs MGN

| 項目 | SMS（旧、非推奨） | MGN（新、推奨） |
|---|---|---|
| **リリース** | 2016年 | 2021年 |
| **ステータス** | 2022年3月31日でサポート終了 | 現行サービス |
| **レプリケーション** | スナップショットベース（定期的） | 継続的レプリケーション（CDC） |
| **ダウンタイム** | 長い（数時間） | 短い（数分） |
| **最適化** | 手動 | **自動最適化（インスタンスタイプ、EBS最適化）** |
| **ロールバック** | 困難 | 容易（ソース環境保持） |
| **料金** | 無料 | 90日間無料、以降レプリケーションサーバー料金 |

**試験での重要ポイント**: SMSは古い選択肢、MGNが正解

### MGNのアーキテクチャ

#### MGN Agentのインストールが必要

**重要**: MGNは**エージェントベース**のサービスです。ソースサーバーに**MGN Replication Agent**をインストールする必要があります。

```mermaid
graph TB
    Source[オンプレミス<br/>物理/仮想サーバー]
    Agent[MGN Replication Agent<br/>ソースにインストール<br/>継続的レプリケーション]
    Staging[ステージングエリア<br/>レプリケーションサーバー<br/>軽量EC2]
    EBS[EBS<br/>レプリケートデータ]
    Launch[Launch Template<br/>最適化設定]
    Target[ターゲットEC2<br/>本番環境]

    Source -->|1. Agentインストール| Agent
    Agent -->|2. 継続的データ転送| Staging
    Staging --> EBS
    EBS -->|カットオーバー時| Launch
    Launch --> Target
```

| コンポーネント | 役割 | 詳細 |
|---|---|---|
| **MGN Replication Agent** | ソースサーバーにインストール、ブロックレベルでデータを継続的にレプリケート | Windows/Linux対応、root権限必要 |
| **レプリケーションサーバー** | AWS側の軽量EC2インスタンス（t3.small等）、データ受信 | MGNが自動作成 |
| **ステージングエリア** | レプリケートされたデータを保持するEBS | MGNが自動管理 |
| **Launch Template** | カットオーバー時のEC2設定（インスタンスタイプ、セキュリティグループ等） | 最適化推奨を含む |
| **ターゲットEC2** | カットオーバー後の本番EC2インスタンス | カットオーバー時に作成 |

#### MGN Agentのインストール方法

**Linux**:

```bash
wget -O ./aws-replication-installer-init.py https://aws-application-migration-service-us-east-1.s3.amazonaws.com/latest/linux/aws-replication-installer-init.py
sudo python3 aws-replication-installer-init.py --region us-east-1 --aws-access-key-id AKIAIOSFODNN7EXAMPLE --aws-secret-access-key wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

**Windows**:

```powershell
Invoke-WebRequest -Uri https://aws-application-migration-service-us-east-1.s3.amazonaws.com/latest/windows/AwsReplicationWindowsInstaller.exe -OutFile .\AwsReplicationWindowsInstaller.exe
.\AwsReplicationWindowsInstaller.exe --region us-east-1 --aws-access-key-id AKIAIOSFODNN7EXAMPLE --aws-secret-access-key wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

| 項目 | 内容 |
|---|---|
| **インストール権限** | Linux: root、Windows: Administrator |
| **ネットワーク要件** | TCP 443（HTTPS）でAWS MGNエンドポイントへの接続 |
| **対応OS** | Windows Server 2008 R2以降、RHEL、Ubuntu、SUSE等 |
| **Agent機能** | ブロックレベルレプリケーション、圧縮転送、暗号化 |

### MGNの移行フロー

```mermaid
sequenceDiagram
    participant Admin as 管理者
    participant Source as ソースサーバー
    participant Agent as MGN Agent
    participant Staging as ステージングエリア
    participant MGN as MGN Console
    participant Target as ターゲットEC2

    Admin->>Source: 1. MGN Agentインストール
    Agent->>Staging: 2. 初期同期（フルデータ転送）
    Agent->>Staging: 3. 継続的レプリケーション（CDC）
    Admin->>MGN: 4. Launch Template設定<br/>（インスタンスタイプ最適化等）
    Admin->>MGN: 5. テストカットオーバー
    MGN->>Target: 6. テスト環境起動
    Admin->>Admin: 7. テスト検証
    Admin->>MGN: 8. 本番カットオーバー
    MGN->>Target: 9. 本番環境起動
    Admin->>Source: 10. ソースサーバー停止
```

| フェーズ | 内容 | 所要時間 |
|---|---|---|
| **1. Agent インストール** | ソースサーバーにMGN Agentをインストール | 数分 |
| **2. 初期同期** | 全データをAWSにレプリケート | 数時間〜数日（データサイズ依存） |
| **3. 継続的レプリケーション** | 変更データのみ継続的にレプリケート | カットオーバーまで継続 |
| **4. Launch Template設定** | **インスタンスタイプ最適化、EBS最適化** | 数分 |
| **5. テストカットオーバー** | テスト環境でEC2起動、動作確認 | 数分〜数時間 |
| **6. 本番カットオーバー** | 本番環境でEC2起動、ソース停止 | 数分（ダウンタイム） |

### MGNの自動最適化機能（試験頻出）

#### 従来の問題点（SMS）

```
オンプレミス:
  - CPU: 8コア
  - メモリ: 32GB
  - ディスク: HDD 500GB

SMS移行:
  → そのまま同等スペックのEC2を選定（手動）
  → m5.2xlarge（8vCPU、32GB）を選択
  → 過剰スペック、コスト高
```

#### MGNの解決策

**MGNは移行中に自動でインスタンスタイプを最適化**します:

```mermaid
graph TB
    Source[ソースサーバー<br/>8CPU、32GB、500GB HDD]
    MGN[MGN<br/>パフォーマンス分析]
    Recommend[推奨インスタンス<br/>m5.xlarge<br/>4vCPU、16GB]
    Launch[Launch Template<br/>最適化設定適用]
    Target[ターゲットEC2<br/>m5.xlarge<br/>コスト50%削減]

    Source --> MGN
    MGN --> Recommend
    Recommend --> Launch
    Launch --> Target
```

| 最適化項目 | 内容 |
|---|---|
| **インスタンスタイプ** | CPU使用率、メモリ使用率を分析し、適切なインスタンスタイプを推奨 |
| **EBS最適化** | HDDをgp3に変更、IOPS/スループット最適化 |
| **EBS暗号化** | 自動でEBS暗号化を有効化 |
| **ネットワーク** | ENAドライバー有効化、拡張ネットワーキング |
| **ライセンス** | BYOLまたはAWSライセンス（BYOL: Bring Your Own License） |

**例: インスタンスタイプ最適化**

```
オンプレミス:
  - CPU使用率: 平均20%、ピーク40%
  - メモリ使用率: 平均50%、ピーク70%

MGN推奨:
  - m5.2xlarge（8vCPU、32GB）→ m5.xlarge（4vCPU、16GB）
  - コスト削減: 50%
```

**例: EBS最適化**

```
オンプレミス:
  - HDD 500GB、IOPS 100

MGN推奨:
  - gp3 300GB（使用量ベース）、IOPS 3000、スループット 125MB/s
  - パフォーマンス向上、コスト削減
```

### MGNのテストカットオーバー（重要）

本番カットオーバー前に、**テスト環境で動作確認**できます:

```mermaid
graph LR
    Source[ソースサーバー<br/>稼働中]
    Staging[ステージングエリア]
    TestEC2[テストEC2<br/>検証用]
    ProdEC2[本番EC2<br/>カットオーバー後]

    Source -->|継続的レプリケーション| Staging
    Staging -->|テストカットオーバー| TestEC2
    Staging -->|本番カットオーバー| ProdEC2
```

| メリット | 説明 |
|---|---|
| **ノーリスク** | ソースサーバーは稼働継続、影響なし |
| **検証** | アプリケーション動作、パフォーマンスを検証 |
| **繰り返し可能** | 何度でもテストカットオーバー可能 |
| **ロールバック** | テスト失敗時、ソースに戻れる |

### MGNの継続的レプリケーション（CDC）

```mermaid
sequenceDiagram
    participant Source as ソースサーバー
    participant Agent as MGN Agent
    participant Staging as ステージングエリア

    loop 継続的レプリケーション
        Source->>Source: データ変更
        Agent->>Agent: 変更ブロック検出
        Agent->>Staging: 変更データ転送
        Staging->>Staging: EBS更新
    end
```

| 項目 | 内容 |
|---|---|
| **方式** | ブロックレベルCDC（Change Data Capture） |
| **レイテンシ** | 数秒〜数分 |
| **帯域使用量** | 変更データのみ（圧縮転送） |
| **ダウンタイム** | カットオーバー時のみ（数分） |

### MGNのWave管理（複数サーバーの一括移行）

**Wave**は、複数のサーバーをグループ化して一括で移行するための機能です。

```mermaid
graph TB
    Wave1[Wave 1: Webサーバー層]
    Wave2[Wave 2: アプリサーバー層]
    Wave3[Wave 3: DBサーバー層]

    Web1[Webサーバー1]
    Web2[Webサーバー2]
    App1[アプリサーバー1]
    App2[アプリサーバー2]
    DB1[DBサーバー1]

    Wave1 --> Web1
    Wave1 --> Web2
    Wave2 --> App1
    Wave2 --> App2
    Wave3 --> DB1
```

| 項目 | 内容 |
|---|---|
| **Wave** | 同時にカットオーバーするサーバーのグループ |
| **用途** | 依存関係のあるサーバーを同時移行（Web→App→DBの順など） |
| **管理** | Migration HubでWaveごとに進捗管理 |
| **利点** | 段階的移行、依存関係管理、ロールバック容易 |

**Waveの設計例（3層アーキテクチャ）**:

| Wave | サーバー | カットオーバー予定 | 依存関係 |
|---|---|---|---|
| **Wave 1** | Web層（2台） | 第1週末 | なし |
| **Wave 2** | App層（3台） | 第2週末 | Wave 1完了後 |
| **Wave 3** | DB層（1台） | 第3週末 | Wave 2完了後 |

### MGNのネットワーク要件

MGN Agentは**TCP 443（HTTPS）**でAWS MGNエンドポイントに接続する必要があります。

```mermaid
graph LR
    Source[オンプレミス<br/>ソースサーバー]
    FW[ファイアウォール]
    Internet[インターネット]
    DX[Direct Connect<br/>専用線]
    VPN[VPN接続]
    MGN[AWS MGN<br/>エンドポイント]

    Source --> FW
    FW -->|TCP 443| Internet
    FW -->|TCP 443| DX
    FW -->|TCP 443| VPN
    Internet --> MGN
    DX --> MGN
    VPN --> MGN
```

| 接続方法 | 帯域 | コスト | レイテンシ | 推奨用途 |
|---|---|---|---|---|
| **インターネット** | 変動 | 低（データ転送無料） | 高 | 小規模移行、テスト |
| **Direct Connect** | 専用帯域 | 高（専用線料金） | 低 | 大規模移行、データ転送量が多い |
| **VPN** | 変動 | 中 | 中 | 中規模移行、セキュリティ重視 |

**重要**: データ転送量が多い場合（TBクラス）、**Direct Connect**の使用を推奨（インターネット経由より安定、レイテンシ低）。

#### ネットワーク設定のベストプラクティス

| 項目 | 推奨設定 |
|---|---|
| **ファイアウォール** | ソース→AWSへのTCP 443アウトバウンドを許可 |
| **プロキシ** | プロキシ経由の接続も可能（MGN Agent設定で指定） |
| **帯域制御** | MGN Agent設定でスロットリング可能（本番業務への影響軽減） |
| **VPCエンドポイント** | PrivateLinkを使用してインターネット経由を回避可能 |

### MGNのライセンス管理（BYOL vs AWS License Included）

MGNでは、Windowsサーバー移行時に**BYOL（Bring Your Own License）**または**AWS License Included**を選択できます。

```mermaid
graph TB
    Source[オンプレミス<br/>Windows Server]
    MGN[MGN]
    BYOL[BYOL<br/>既存ライセンス使用]
    LI[License Included<br/>AWSライセンス使用]

    Source --> MGN
    MGN --> BYOL
    MGN --> LI
```

| 項目 | BYOL | License Included |
|---|---|---|
| **ライセンス** | 既存のWindows Serverライセンスを使用 | AWSのライセンスを使用 |
| **要件** | ボリュームライセンス契約（EA、MPSA等）が必要 | 不要 |
| **インスタンスタイプ** | Dedicated Host/Dedicated Instance | 通常のEC2インスタンス |
| **コスト** | ライセンス料金不要（Windows料金含まない） | インスタンス料金にライセンス込み |
| **推奨** | 既存ライセンスが多数ある場合 | 少数のサーバー、ライセンス管理を簡素化したい場合 |

**Launch Template設定**:

```json
{
  "LaunchTemplateName": "windows-server-byol",
  "LaunchTemplateData": {
    "LicenseSpecifications": [
      {
        "LicenseConfigurationArn": "arn:aws:license-manager:us-east-1:123456789012:license-configuration:lic-EXAMPLE"
      }
    ]
  }
}
```

**試験での重要ポイント**:
- BYOLは**Dedicated Host/Dedicated Instance**が必要（コスト高）
- License IncludedはシンプルだがWindows料金が含まれる
- 大規模移行（100台以上）ではBYOLの方がコスト削減になる場合が多い

### MGN料金

| 項目 | 料金 |
|---|---|
| **MGN利用料** | 90日間無料 |
| **90日以降** | $0.0152/時間/サーバー（約$11/月/サーバー） |
| **レプリケーションサーバー** | EC2料金（t3.small等） |
| **EBS** | ステージングエリアのEBS料金 |
| **データ転送** | インターネット → AWSは無料、Direct Connect経由は有料 |
| **BYOL（Dedicated Host）** | 追加料金発生 |

## AWS Migration Hub Refactor Spaces

### 概要

Refactor Spacesは、**モノリスアプリケーションをマイクロサービスに段階的に移行**するためのサービスです。**ストラングラーフィグパターン**を簡単に実装できます。

**重要な誤解を避けるために**:

Refactor Spacesは**コードのリファクタリング（書き換え）を行うサービスではありません**。開発者が別途マイクロサービスを開発し、Refactor Spacesは**ルーティングのみを管理**します。

```mermaid
graph TB
    Dev[開発者]
    Mono[既存モノリス<br/>そのまま稼働]
    NewService[新マイクロサービス<br/>開発者が新規開発]
    RS[Refactor Spaces<br/>ルーティング管理のみ]
    User[ユーザー]

    Dev -->|1. マイクロサービス開発| NewService
    Dev -->|2. Refactor Spaces設定| RS
    User -->|リクエスト| RS
    RS -->|/api/old| Mono
    RS -->|/api/new| NewService
```

| Refactor Spacesの役割 | 開発者の役割 |
|---|---|
| ルーティング管理（URLパスで振り分け） | モノリスから機能を切り出してマイクロサービスを開発 |
| ALB/API Gateway自動作成 | 新しいマイクロサービスのコード作成 |
| Transit Gateway管理 | アプリケーションロジックの実装 |
| ネットワーク設定自動化 | テスト、デプロイ |

**エージェント不要の理由**: Refactor Spacesは既存のアプリケーション（モノリス、マイクロサービス）をそのまま使用し、ALB/API Gatewayで自動ルーティングするだけなので、サーバーにエージェントをインストールする必要がありません。

### モノリス vs マイクロサービス

| 項目 | モノリス | マイクロサービス |
|---|---|---|
| **構成** | 単一の大きなアプリケーション | 小さな独立したサービス群 |
| **デプロイ** | 全体を一度にデプロイ | サービスごとに独立デプロイ |
| **スケーリング** | 全体をスケール | サービスごとにスケール |
| **開発** | 単一チーム | チームごとにサービス担当 |
| **技術スタック** | 統一 | サービスごとに選択可能 |
| **障害影響** | 全体停止 | 該当サービスのみ |

### ストラングラーフィグパターン

**徐々にモノリスから機能を切り出し、マイクロサービスに移行**する戦略:

```mermaid
graph TB
    subgraph Phase1[フェーズ1: 初期状態]
        User1[ユーザー]
        Mono1[モノリス<br/>すべての機能]
        User1 --> Mono1
    end

    subgraph Phase2[フェーズ2: 一部移行]
        User2[ユーザー]
        Router2[ルーター<br/>Refactor Spaces]
        Mono2[モノリス<br/>機能A、B]
        Micro2[マイクロサービス<br/>機能C]
        User2 --> Router2
        Router2 -->|/api/a、/api/b| Mono2
        Router2 -->|/api/c| Micro2
    end

    subgraph Phase3[フェーズ3: 完全移行]
        User3[ユーザー]
        Router3[ルーター]
        MicroA[サービスA]
        MicroB[サービスB]
        MicroC[サービスC]
        User3 --> Router3
        Router3 -->|/api/a| MicroA
        Router3 -->|/api/b| MicroB
        Router3 -->|/api/c| MicroC
    end

    Phase1 --> Phase2
    Phase2 --> Phase3
```

| フェーズ | 状態 | Refactor Spacesの役割 |
|---|---|---|
| **フェーズ1** | 全機能がモノリス | なし |
| **フェーズ2** | 一部機能をマイクロサービス化 | ルーティング管理（パスベース） |
| **フェーズ3** | 全機能がマイクロサービス | 完全なマイクロサービスアーキテクチャ |

### Refactor Spacesのアーキテクチャ

```mermaid
graph TB
    User[ユーザー]
    RS[Refactor Spaces<br/>Environment]
    ALB[ALB<br/>自動作成]
    TG1[Target Group<br/>モノリス]
    TG2[Target Group<br/>マイクロサービスA]
    TG3[Target Group<br/>マイクロサービスB]
    Mono[モノリスEC2/ECS]
    MicroA[ECS/Lambda<br/>サービスA]
    MicroB[ECS/Lambda<br/>サービスB]

    User --> RS
    RS --> ALB
    ALB -->|/api/legacy| TG1
    ALB -->|/api/users| TG2
    ALB -->|/api/orders| TG3
    TG1 --> Mono
    TG2 --> MicroA
    TG3 --> MicroB
```

| コンポーネント | 役割 |
|---|---|
| **Environment** | Refactor Spacesの管理単位、VPC、サブネット等を指定 |
| **Application** | アプリケーションの論理グループ |
| **Service** | 個別のサービス（モノリスまたはマイクロサービス） |
| **Route** | URLパスベースのルーティング設定 |
| **自動ALB** | Refactor Spacesが自動でALBを作成・管理 |

### Refactor Spacesの移行フロー

#### ステップ1: Environment作成

```bash
aws migration-hub-refactor-spaces create-environment \
  --name my-app-environment \
  --network-fabric-type TRANSIT_GATEWAY \
  --description "Refactor Spaces Environment for MyApp"
```

| 項目 | 内容 |
|---|---|
| **Network Fabric** | TRANSIT_GATEWAY または NONE |
| **Transit Gateway** | モノリスとマイクロサービスが異なるVPCの場合に使用 |

#### ステップ2: Application作成

```bash
aws migration-hub-refactor-spaces create-application \
  --environment-identifier env-12345 \
  --name my-application \
  --vpc-id vpc-12345 \
  --proxy-type API_GATEWAY
```

| Proxy Type | 用途 |
|---|---|
| **API_GATEWAY** | REST API（推奨） |
| **NONE** | プロキシなし |

#### ステップ3: Service作成

```bash
# モノリスサービス
aws migration-hub-refactor-spaces create-service \
  --application-identifier app-12345 \
  --name monolith-service \
  --endpoint-type URL \
  --url-endpoint "http://monolith-alb.example.com"

# マイクロサービスA
aws migration-hub-refactor-spaces create-service \
  --application-identifier app-12345 \
  --name user-service \
  --endpoint-type LAMBDA \
  --lambda-endpoint arn:aws:lambda:us-east-1:123456789012:function:user-service
```

#### ステップ4: Route作成

```bash
# /api/users → マイクロサービスA
aws migration-hub-refactor-spaces create-route \
  --application-identifier app-12345 \
  --service-identifier svc-67890 \
  --route-type URI_PATH \
  --uri-path-route "/api/users"

# デフォルト（/api/*） → モノリス
aws migration-hub-refactor-spaces create-route \
  --application-identifier app-12345 \
  --service-identifier svc-54321 \
  --route-type DEFAULT
```

### Refactor Spacesの段階的ルーティング（Incremental Routing）

Refactor Spacesは**カナリアデプロイ的な段階的移行**をサポートします。トラフィックの一部だけを新マイクロサービスに流し、徐々に増やすことができます。

```mermaid
graph TB
    subgraph Phase1[フェーズ1: 10%トラフィック]
        User1[ユーザー 100%]
        Router1[Refactor Spaces]
        Mono1[モノリス 90%]
        Micro1[マイクロサービス 10%]

        User1 --> Router1
        Router1 -->|90%| Mono1
        Router1 -->|10%| Micro1
    end

    subgraph Phase2[フェーズ2: 50%トラフィック]
        User2[ユーザー 100%]
        Router2[Refactor Spaces]
        Mono2[モノリス 50%]
        Micro2[マイクロサービス 50%]

        User2 --> Router2
        Router2 -->|50%| Mono2
        Router2 -->|50%| Micro2
    end

    subgraph Phase3[フェーズ3: 100%トラフィック]
        User3[ユーザー 100%]
        Router3[Refactor Spaces]
        Micro3[マイクロサービス 100%]

        User3 --> Router3
        Router3 -->|100%| Micro3
    end

    Phase1 --> Phase2
    Phase2 --> Phase3
```

| フェーズ | トラフィック配分 | 目的 | 期間 |
|---|---|---|---|
| **フェーズ1** | モノリス90%、マイクロ10% | 新サービスの動作確認、パフォーマンステスト | 1-2週間 |
| **フェーズ2** | モノリス50%、マイクロ50% | 負荷テスト、エラー率監視 | 1-2週間 |
| **フェーズ3** | マイクロ100% | 完全移行 | - |

**Refactor Spacesの設定例**:

```bash
# Phase 1: 10%のトラフィックを新サービスに
aws migration-hub-refactor-spaces update-route \
  --route-identifier route-12345 \
  --route-type URI_PATH \
  --uri-path-route "/api/users" \
  --default-route '{"activationState": "ACTIVE"}' \
  --append-source-path-header false \
  --include-child-paths true \
  --methods GET,POST \
  --source-path "/api/users" \
  --target-weight-distribution '[{"Weight": 90, "TargetId": "mono-svc-123"}, {"Weight": 10, "TargetId": "micro-svc-456"}]'
```

**CloudWatch監視メトリクス**:

| メトリクス | 説明 | 閾値例 |
|---|---|---|
| **TargetResponseTime** | レスポンスタイム | < 500ms |
| **HTTPCode_Target_5XX_Count** | 5xxエラー数 | < 1% |
| **RequestCount** | リクエスト数 | トラフィック配分通り |
| **UnHealthyHostCount** | 異常ホスト数 | 0 |

**試験での重要ポイント**: Refactor Spacesは段階的にトラフィックを移行できるため、**リスク最小化**と**ロールバック容易性**が特徴。

### Refactor Spacesの利点

| 利点 | 説明 |
|---|---|
| **段階的移行** | 一度に全てを移行せず、1つずつマイクロサービス化 |
| **トラフィック制御** | 10% → 50% → 100%と段階的にトラフィックを移行 |
| **ダウンタイムなし** | ルーティングを変更するだけ、ユーザーへの影響なし |
| **ロールバック容易** | 問題があれば即座にモノリスに戻せる |
| **ネットワーク管理簡素化** | Transit Gateway、ALBを自動管理 |
| **独立開発** | チームごとに独立してマイクロサービスを開発・デプロイ |
| **リスク低減** | カナリアデプロイ的に少量トラフィックで検証 |

### Refactor Spacesのユースケース

#### ユースケース1: ECサイトのモダナイゼーション

**現状**: モノリスのECサイト（商品、注文、ユーザー管理が一体）

**移行計画**:

| フェーズ | 作業 | 期間 |
|---|---|---|
| **フェーズ1** | ユーザー管理をマイクロサービス化（Lambda） | 2ヶ月 |
| **フェーズ2** | 注文処理をマイクロサービス化（ECS） | 2ヶ月 |
| **フェーズ3** | 商品管理をマイクロサービス化（ECS） | 2ヶ月 |
| **フェーズ4** | モノリス廃止 | 1ヶ月 |

**Refactor Spacesでのルーティング**:

```
/api/users/*     → Lambda (User Service)
/api/orders/*    → ECS (Order Service)
/api/products/*  → ECS (Product Service)
/api/*           → モノリス（フェーズ4で廃止）
```

#### ユースケース2: マルチVPC環境での移行

**シナリオ**: モノリス（VPC A）、マイクロサービス（VPC B）が異なるVPCにある

```mermaid
graph TB
    User[ユーザー]
    RS[Refactor Spaces<br/>Transit Gateway]
    VPCA[VPC A<br/>モノリス]
    VPCB[VPC B<br/>マイクロサービス]

    User --> RS
    RS -->|Transit Gateway| VPCA
    RS -->|Transit Gateway| VPCB
```

**Refactor Spacesの自動管理**:
- Transit Gatewayアタッチメント自動作成
- ルートテーブル自動更新
- セキュリティグループ自動設定

## その他の移行・モダナイゼーションサービス

### コード変換を行うサービス（Refactor Spacesとの違い）

**重要**: Refactor Spacesはルーティング管理のみで、コード変換は行いません。実際のコード変換・モダナイゼーションは以下のサービスを使用します。

#### AWS App2Container

**概要**: Java/.NETアプリケーションをコンテナ化（Docker）するサービス。

```mermaid
graph LR
    App[既存アプリ<br/>Java/.NET]
    A2C[App2Container<br/>AI分析]
    Docker[Dockerイメージ]
    ECS[ECS/EKS]

    App -->|分析・変換| A2C
    A2C -->|自動生成| Docker
    Docker --> ECS
```

| 項目 | 内容 |
|---|---|
| **対象** | Java（Tomcat等）、.NET Framework |
| **機能** | アプリケーション分析、Dockerfile自動生成、ECS/EKSデプロイ設定 |
| **エージェント** | ○必要（ソースサーバーにインストール） |
| **用途** | モノリスのコンテナ化（マイクロサービス化の第一歩） |

#### AWS Microservice Extractor for .NET

**概要**: .NETモノリスからマイクロサービスを自動抽出（AI/ML使用）。

| 項目 | 内容 |
|---|---|
| **対象** | .NET Framework、ASP.NET |
| **機能** | AIでモノリスを分析、マイクロサービス候補を特定、コード自動分割 |
| **用途** | モノリスから特定機能を切り出してマイクロサービス化 |

#### AWS Blu Age（Mainframe Modernization用）

**概要**: メインフレーム（COBOL）をJavaに自動変換（AI/ML）。

| 項目 | 内容 |
|---|---|
| **対象** | COBOL、PL/I、JCL |
| **機能** | AIでコードを解析、Javaに自動変換 |
| **用途** | メインフレームモダナイゼーション |

### コード変換サービス vs Refactor Spaces

| 項目 | App2Container/Microservice Extractor | Refactor Spaces |
|---|---|---|
| **役割** | コード変換・分割 | ルーティング管理のみ |
| **AI使用** | ○（コード分析・変換） | ×（ルーティングのみ） |
| **エージェント** | ○必要 | ×不要 |
| **開発者の作業** | 変換後のコードレビュー・調整 | マイクロサービスを手動開発 |
| **用途** | モノリスの自動分割 | 段階的移行のルーティング管理 |

**試験での重要ポイント**:
- 「AIでモノリスを自動分割」→ Microservice Extractor（.NET）またはBlu Age（COBOL）
- 「段階的にマイクロサービスへ移行、ルーティング管理」→ Refactor Spaces
- 「Javaアプリをコンテナ化」→ App2Container

### AWS Mainframe Modernization

**概要**: メインフレームアプリケーション（COBOL、PL/I等）をAWSにモダナイズするサービス（2023年GA）。

```mermaid
graph LR
    Mainframe[メインフレーム<br/>COBOL/PL/I]
    Transform[AWS Mainframe<br/>Modernization]
    Refactor[Refactor<br/>Java等に変換]
    Replatform[Replatform<br/>マネージドランタイム]
    Target[AWS環境<br/>EC2/ECS/Lambda]

    Mainframe --> Transform
    Transform --> Refactor
    Transform --> Replatform
    Refactor --> Target
    Replatform --> Target
```

| アプローチ | 説明 | 用途 |
|---|---|---|
| **Automated refactoring** | COBOLをJavaに自動変換（AI/ML使用） | 完全モダナイゼーション、長期的 |
| **Automated replatform** | COBOLをそのまま実行（Micro Focus、BluAge等） | 短期移行、既存資産活用 |

**試験での重要ポイント**: メインフレームモダナイゼーションはMainframe Modernization、一般的なサーバー移行はMGN

### AWS Transformation（AWS移行・モダナイゼーションの総称）

**重要**: 「AWS Transformation」は特定のサービス名ではなく、AWSの移行・モダナイゼーション関連サービス群の総称です。

#### AWS Transformationに含まれる主なサービス

```mermaid
graph TB
    Transform[AWS Transformation<br/>移行・モダナイゼーション]
    Migration[移行サービス]
    Modernization[モダナイゼーション]

    MGN[MGN<br/>サーバー移行]
    DMS[DMS<br/>データベース移行]
    DataSync[DataSync<br/>ストレージ移行]

    App2C[App2Container<br/>コンテナ化]
    MicroExt[Microservice Extractor<br/>マイクロサービス分割]
    Mainframe[Mainframe Modernization<br/>メインフレーム]
    RefactorSpace[Refactor Spaces<br/>ルーティング管理]

    Transform --> Migration
    Transform --> Modernization
    Migration --> MGN
    Migration --> DMS
    Migration --> DataSync
    Modernization --> App2C
    Modernization --> MicroExt
    Modernization --> Mainframe
    Modernization --> RefactorSpace
```

| カテゴリ | サービス | 用途 |
|---|---|---|
| **サーバー移行** | MGN | 物理/仮想サーバー → EC2 |
| **データベース移行** | DMS | データベース間のデータ移行・レプリケーション |
| **データ移行** | DataSync | オンプレミス → S3/EFS/FSx |
| **ストレージ移行** | Storage Gateway | オンプレミスとAWSストレージの統合 |
| **コンテナ化** | App2Container | Java/.NET → Docker/ECS |
| **マイクロサービス化** | Microservice Extractor | .NETモノリス → マイクロサービス |
| **メインフレーム** | Mainframe Modernization | COBOL → Java/マネージドランタイム |
| **ルーティング管理** | Refactor Spaces | モノリス/マイクロサービスのルーティング |

### AWS Application Transformation（旧CloudEndure）

**重要**: CloudEndure Migrationは**MGNに統合**されました（2021年）。試験で「CloudEndure」が出た場合、MGNが正解です。

| 項目 | CloudEndure Migration（旧） | MGN（現在） |
|---|---|---|
| **ステータス** | 2021年にMGNに統合 | 現行サービス |
| **機能** | 継続的レプリケーション | 同じ + 最適化機能 |
| **推奨** | ×（既に統合済み） | ○ |

**CloudEndureの歴史**:
- 2015年: CloudEndure社設立、災害復旧サービス提供
- 2019年: AWSがCloudEndure社を買収
- 2020年: CloudEndure Migration、CloudEndure Disaster Recoveryを提供
- 2021年: MGNに統合、CloudEndureブランドは実質終了

### AWS Transformation Hub（Migration Hub）

**概要**: AWSの移行・モダナイゼーションプロジェクトを一元管理するコンソール。

```mermaid
graph LR
    MH[Migration Hub<br/>中央管理コンソール]
    MGN[MGN]
    DMS[DMS]
    DataSync[DataSync]
    App2C[App2Container]
    RefactorSpace[Refactor Spaces]

    MGN -->|進捗レポート| MH
    DMS -->|進捗レポート| MH
    DataSync -->|進捗レポート| MH
    App2C -->|進捗レポート| MH
    RefactorSpace -->|進捗レポート| MH
```

| 機能 | 説明 |
|---|---|
| **統合ダッシュボード** | 複数の移行サービスの進捗を一元管理 |
| **アプリケーショングルーピング** | 関連するサーバー・DBをグループ化 |
| **移行追跡** | リソースごとの移行ステータス追跡 |
| **コスト推定** | 移行後のAWSコスト推定 |
| **Refactor Spaces統合** | モダナイゼーションプロジェクト管理 |
| **6Rフレームワーク** | 各リソースに移行戦略（6R）をタグ付け |

#### Migration Hubと6Rフレームワークの統合

Migration Hubでは、各サーバー・アプリケーションに**6R（7R）の移行戦略**をタグ付けして管理できます。

```mermaid
graph TB
    MH[Migration Hub]
    Server1[Server 1<br/>Tag: Re-host]
    Server2[Server 2<br/>Tag: Re-platform]
    App1[App A<br/>Tag: Refactor]
    App2[App B<br/>Tag: Retire]
    DB1[DB<br/>Tag: Re-purchase]

    MH --> Server1
    MH --> Server2
    MH --> App1
    MH --> App2
    MH --> DB1

    Server1 -->|MGN| EC2_1[EC2]
    Server2 -->|MGN + 最適化| EC2_2[EC2最適化]
    App1 -->|Refactor Spaces| Micro[マイクロサービス]
    App2 -->|廃止| Delete[削除]
    DB1 -->|RDS| RDS[Aurora]
```

**Migration Hubでの管理例**:

| リソース | 6R戦略 | 使用サービス | 優先度 | Wave |
|---|---|---|---|---|
| Webサーバー（3台） | Re-host | MGN | 高 | Wave 1 |
| Appサーバー（5台） | Re-platform | MGN + 最適化 | 高 | Wave 2 |
| DBサーバー（2台） | Re-purchase | DMS → Aurora | 中 | Wave 3 |
| モノリスアプリ | Refactor | Refactor Spaces | 低 | Wave 4-6 |
| レガシーツール | Retire | - | - | - |

**Migration Hubの6R管理機能**:

| 機能 | 説明 |
|---|---|
| **戦略タグ付け** | リソースごとに6R戦略をタグ付け |
| **進捗可視化** | 6R戦略別の進捗をダッシュボード表示 |
| **コスト試算** | 6R戦略別のコスト比較（Re-host vs Re-purchase等） |
| **依存関係管理** | アプリケーション内のサーバー依存関係を可視化 |
| **Wave管理** | 6R戦略に基づいたWave設計 |

**試験での重要ポイント**:
- Migration Hubは管理コンソールであり、実際の移行作業はMGN、DMS等の各サービスが行う
- 6R戦略を事前にタグ付けすることで、適切なサービス選定と進捗管理が可能
- **Re-host → MGN**、**Refactor → Refactor Spaces**、**Re-purchase → Marketplace/RDS**の組み合わせ

## MGN vs Refactor Spaces vs Mainframe Modernization

| 項目 | MGN | Refactor Spaces | Mainframe Modernization |
|---|---|---|---|
| **用途** | サーバー移行（Re-host、Re-platform） | モダナイゼーション（Refactor） | メインフレームモダナイゼーション |
| **対象** | 物理/仮想サーバー → EC2 | モノリス → マイクロサービス | メインフレーム → AWS |
| **エージェント** | ○必要（MGN Agent） | ×不要 | ○必要（評価用） |
| **移行方式** | リフト&シフト + 最適化 | 段階的リファクタリング | Refactor/Replatform |
| **ダウンタイム** | 数分（カットオーバー時） | なし（ルーティング変更） | アプローチによる |
| **料金** | 90日間無料、以降$11/月/サーバー | 使用リソース（ALB、Transit Gateway等）のみ | ランタイム料金 |

## SAP試験の重要ポイント

### MGN（Application Migration Service）
- **SMSは古い選択肢**: 試験ではMGNが正解
- **CloudEndureは統合済み**: CloudEndure MigrationはMGNに統合（2021年）、試験ではMGNを選択
- **エージェント必須**: ソースサーバーにMGN Replication Agentをインストール必要
- **継続的レプリケーション**: CDCで変更データのみ転送、ダウンタイム最小化
- **自動最適化**: インスタンスタイプ、EBS最適化を自動推奨（試験頻出）
- **テストカットオーバー**: 本番前にテスト環境で検証可能
- **料金**: 90日間無料、以降$11/月/サーバー
- **Wave管理**: 複数サーバーをグループ化して段階的移行（依存関係管理）
- **ネットワーク**: TCP 443でMGNエンドポイントに接続、Direct Connect/VPN対応
- **ライセンス**: BYOL（Dedicated Host必要）vs License Included（簡単）

### MGNの最適化機能（試験頻出）
- **インスタンスタイプ**: CPU/メモリ使用率を分析して最適なインスタンス推奨
- **EBS最適化**: HDD → gp3、IOPS/スループット最適化
- **暗号化**: EBS自動暗号化
- **ネットワーク**: ENAドライバー、拡張ネットワーキング
- **ライセンス最適化**: BYOLまたはAWS License Includedを選択可能

### MGNのネットワーク要件（試験頻出）
- **接続方式**: TCP 443（HTTPS）でAWS MGNエンドポイントに接続
- **インターネット経由**: 小規模移行、データ転送無料
- **Direct Connect**: 大規模移行（TBクラス）、安定した帯域、低レイテンシ
- **VPN**: 中規模移行、セキュリティ重視
- **プロキシ対応**: MGN Agent設定でプロキシ経由の接続も可能
- **帯域制御**: MGN Agent設定でスロットリング可能（本番業務への影響軽減）

### Refactor Spaces
- **役割**: ルーティング管理のみ（**コード変換は行わない**）
- **ストラングラーフィグパターン**: モノリスから段階的にマイクロサービス化
- **エージェント不要**: 既存アプリをそのまま使用、ALB/API Gatewayで自動ルーティング
- **開発者の作業**: マイクロサービスは開発者が別途開発、Refactor Spacesはルーティングのみ管理
- **ルーティング管理**: URLパスベースでモノリス/マイクロサービスに振り分け
- **段階的ルーティング**: トラフィックを10% → 50% → 100%と段階的に移行（カナリアデプロイ的）
- **ネットワーク簡素化**: ALB、Transit Gatewayを自動管理
- **ダウンタイムなし**: ルーティング変更のみ、ロールバック容易
- **リスク低減**: 少量トラフィックで検証してから完全移行

### コード変換サービス
- **App2Container**: Java/.NETアプリをコンテナ化（Docker）、エージェント必要
- **Microservice Extractor for .NET**: .NETモノリスをAIで自動分割
- **Blu Age**: COBOL→Java自動変換（AI/ML、Mainframe Modernization用）
- **試験ポイント**: Refactor Spacesはコード変換せず、これらのサービスが変換を担当

### Mainframe Modernization
- **用途**: メインフレーム（COBOL、PL/I）のモダナイゼーション
- **アプローチ**: Automated refactoring（Java変換）、Automated replatform（マネージドランタイム）
- **試験ポイント**: メインフレーム専用、一般サーバー移行はMGN

### エージェント要否（試験頻出）
- **MGN**: ○エージェント必須（MGN Replication Agent）
- **Refactor Spaces**: ×エージェント不要
- **Mainframe Modernization**: ○評価用エージェント必要

### Migration Hubと6Rフレームワーク（試験頻出）
- **Migration Hub**: 移行プロジェクト一元管理、6R戦略タグ付け
- **Re-host**: MGN（そのまま移行）
- **Re-platform**: MGN + 最適化（軽微な最適化）
- **Refactor**: Refactor Spaces（アーキテクチャ再設計）
- **Re-purchase**: Marketplace、RDS Aurora等（SaaS/マネージドサービス）
- **Retire**: 廃止
- **Retain**: 現状維持
- **Wave管理**: 依存関係に基づいたサーバーグループ化、段階的移行

### よくある試験問題パターン
- 「オンプレミスサーバーをAWSに移行、最適なインスタンスタイプを自動選定」→ MGN
- 「SMSで移行を計画」→ 誤り、MGNを使用
- 「CloudEndureで移行」→ 誤り、CloudEndureはMGNに統合済み
- 「複数サーバーをグループ化して依存関係を管理しながら段階的移行」→ MGNのWave管理
- 「大量データ（TBクラス）を安定的に移行」→ MGN + Direct Connect
- 「Windowsサーバー100台以上を移行、既存ライセンスを活用」→ MGN + BYOL（Dedicated Host）
- 「モノリスアプリをマイクロサービスに段階的移行、ルーティング管理」→ Refactor Spaces
- 「新マイクロサービスに10%のトラフィックを流してテスト」→ Refactor Spacesの段階的ルーティング
- 「AIでモノリスを自動分割してマイクロサービス化」→ Microservice Extractor for .NET
- 「Javaアプリをコンテナ化」→ App2Container
- 「移行中にダウンタイムを最小化」→ MGN（継続的レプリケーション）
- 「マイクロサービス移行でネットワーク管理を簡素化」→ Refactor Spaces（Transit Gateway自動管理）
- 「メインフレーム（COBOL）をモダナイズ」→ Mainframe Modernization
- 「COBOLをJavaに自動変換」→ Blu Age（Mainframe Modernization）
- 「ソースサーバーにエージェントをインストールして移行」→ MGN
- 「移行プロジェクト全体を一元管理、6R戦略で分類」→ Migration Hub
- 「本番業務に影響を与えずにレプリケーション」→ MGN Agent帯域制御（スロットリング）

### Re-host vs Re-platform vs Refactor
- **Re-host**: そのまま移行（MGN、エージェント必須）
- **Re-platform**: 軽微な最適化を加えて移行（MGN + 最適化）
- **Refactor**: アーキテクチャ再設計（Refactor Spaces、エージェント不要）
- **Mainframe**: メインフレームモダナイゼーション（Mainframe Modernization）
