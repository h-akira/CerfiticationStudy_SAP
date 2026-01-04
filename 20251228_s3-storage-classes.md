# S3ストレージクラス

作成日: 2025-12-28

## 概要

Amazon S3には、アクセス頻度やデータの重要度に応じて選択できる複数のストレージクラスがあります。

## ストレージクラス比較表

| ストレージクラス | 取り出し時間 | 可用性 | 最小保存期間 | 最小サイズ | 取り出し料金 | 用途 |
|---|---|---|---|---|---|---|
| **Standard** | ミリ秒 | 99.99% | なし | なし | なし | 頻繁アクセス |
| **Intelligent-Tiering** | ミリ秒※ | 99.9% | なし | 128KB | なし※ | アクセスパターン不明 |
| **Standard-IA** | ミリ秒 | 99.9% | 30日 | 128KB | あり | 月1回程度 |
| **One Zone-IA** | ミリ秒 | 99.5% | 30日 | 128KB | あり | 再作成可能データ |
| **Glacier Instant Retrieval** | ミリ秒 | 99.9% | 90日 | 128KB | あり | 四半期1回程度 |
| **Glacier Flexible Retrieval** | 分〜時間 | 99.99% | 90日 | 40KB | あり | 年数回 |
| **Glacier Deep Archive** | 12〜48時間 | 99.99% | 180日 | 40KB | あり | 年1回以下 |

※Intelligent-TieringのArchive階層は復元が必要。監視料金が別途発生。

### Glacier Flexible Retrievalの取り出しオプション

| オプション | 取り出し時間 | コスト |
|---|---|---|
| Expedited | 1〜5分 | 高 |
| Standard | 3〜5時間 | 中 |
| Bulk | 5〜12時間 | 低 |

### Glacier Deep Archiveの取り出しオプション

| オプション | 取り出し時間 |
|---|---|
| Standard | 12時間以内 |
| Bulk | 48時間以内 |

## Intelligent-Tiering の階層移行

| 階層 | 移行条件 | 取り出し時間 | 設定 |
|---|---|---|---|
| **Frequent Access** | デフォルト | ミリ秒 | 自動 |
| **Infrequent Access** | 30日間アクセスなし | ミリ秒 | 自動 |
| **Archive Instant Access** | 90日間アクセスなし | ミリ秒 | 自動 |
| **Archive Access** | 90〜730日間アクセスなし | 3〜5時間 | 要有効化 |
| **Deep Archive Access** | 180〜730日間アクセスなし | 12時間 | 要有効化 |

**重要**: アクセスがあれば即座にFrequent Accessに復帰

## コスト比較（概算）

| ストレージクラス | 相対コスト |
|---|---|
| Standard | 100%（基準） |
| Intelligent-Tiering | 100% + 監視料金 |
| Standard-IA | 50% |
| One Zone-IA | 40% |
| Glacier Instant Retrieval | 25% |
| Glacier Flexible Retrieval | 10% |
| Glacier Deep Archive | 4%（最安） |

## SAP試験の重要ポイント

### 取り出し時間
- **即座（ミリ秒）**: Standard、IA系、Glacier Instant Retrieval
- **復元必要**: Glacier Flexible Retrieval（分〜時間）、Deep Archive（時間〜日）

### 最小保存期間
- **なし**: Standard、Intelligent-Tiering
- **30日**: Standard-IA、One Zone-IA
- **90日**: Glacier Instant Retrieval、Flexible Retrieval
- **180日**: Glacier Deep Archive

### 耐久性とAZ
- **耐久性**: すべて 99.999999999% (11 9's)
- **Single AZ**: One Zone-IAのみ（AZ破壊でデータ損失リスク）
- **Multi-AZ**: その他すべて（3つ以上のAZ）

### その他
- **自動階層化**: Intelligent-Tieringのみ
- **取り出し料金**: Standard以外はすべて発生
- **最小オブジェクトサイズ**: IA系=128KB、Glacier系=40KB
