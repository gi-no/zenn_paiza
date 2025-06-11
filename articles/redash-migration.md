---
title: "Redash運用基盤移行プロジェクト：EC2からマネージドサービスへ"
emoji: "😄"
type: "tech"
topics: ["redash","aws","dms","ecs","redis"]
published: false
published_at: 2025-06-16 09:00
publication_name: "paiza"
---

## はじめに：実施の背景

こんにちは。  
paizaでSREをしている長谷川です。  
今回の記事では、最近取り組んだRedash運用基盤の移行プロジェクトについて事例を共有したいと思います。

paizaでは、データの可視化と分析にRedashを活用していましたが、EC2ベースでの運用における課題が顕在化してきたため、AWSマネージドサービスを活用した構成への移行プロジェクトに取り組むこととなりました。

## 現状の課題

paizaでは、[Redashの公式セットアップ](https://github.com/getredash/setup/tree/master)に準拠したDocker Compose構成でEC2上にRedashを構築し運用していました。この構成では、Redash本体、PostgreSQL、Redis、Nginxなどのサービスが単一のEC2インスタンス上で稼働していました。

### サービス可用性の問題

このEC2上でのRedash運用において、最も深刻だったのはWorkerプロセスによるメモリ枯渇問題でした。大量データの集計処理や複雑なクエリを実行した際に、Workerプロセスがメモリを消費し尽くし、サーバー全体がダウンするという事象が定期的に発生していました。

この問題により、日中の業務時間帯に障害対応が必要となり、分析業務の中断やチーム全体の運用負荷が増大していました。

### スケーラビリティの制約

データ量の増加に伴い、クエリ実行時間が長期化する傾向にありました。EC2ベースの構成では、スケールアップに以下のような制約がありました。

- 事前メンテナンス計画の策定が必要
- サービス停止を伴うインスタンスタイプ変更
- 手動での起動確認とテスト実行
- 利用者への影響を最小化するための調整

これらの制約により、ビジネス要件に対する迅速な対応が困難な状況でした。

### 運用負荷の増大

EC2ベースの運用では、以下のような継続的な運用タスクが発生していました。

- **OS・ミドルウェアのパッチ適用**：セキュリティアップデートの定期実行
- **監視設定の保守**：CloudWatchメトリクスの細粒度な設定・調整
- **バックアップ管理**：AMI世代管理とリストア手順の検証
- **セキュリティ設定**：ファイアウォール設定やアクセス制御の継続的見直し

これらの運用タスクにより、本来の分析業務やサービス改善に割く時間が制限されていました。

## 目指した解決策

### 技術選定と設計

上記課題を解決するため、以下の方針でアーキテクチャを検討しました。

- **高可用性の実現**：障害時の自動復旧機能
- **スケーラビリティの向上**：必要に応じたリソース調整の簡易化
- **運用負荷の軽減**：マネージドサービスによるインフラ管理の委託

### アーキテクチャ設計

これらの要件を満たすため、以下の構成への移行を決定しました。

**移行前（EC2ベース）**
![移行前のアーキテクチャ](/images/redash-migration/redash-before.png)

**移行後（マネージドサービス）** 
![移行後のアーキテクチャ](/images/redash-migration/redash-after.png)

主要な変更点は以下の通りです。
- **PostgreSQL** → **Amazon RDS**：データベース運用の完全マネージド化
- **Redis** → **Amazon ElastiCache（Valkey）**：キュー管理のマネージド化
- **EC2** → **Amazon ECS Fargate**：コンテナベース実行環境への移行
- **Worker分離**：リソース問題の影響範囲限定化

## 実装：EC2からマネージドサービスへの移行

### 全体のデータフロー

移行プロジェクトでは、既存のダッシュボードやクエリ資産を保持するため、AWS DMSを活用したデータ移行戦略を採用しました。

1. **事前準備**：移行先リソース（RDS、ElastiCache、ECS）の構築
2. **データ同期**：DMSによる継続レプリケーションの開始
3. **最終切り替え**：サービス停止を伴う最終データ移行と環境切り替え

### 考慮した点

#### ElastiCache（Valkey）の選定と設定課題

キューイング機能の移行先として、Amazon ElastiCache for Valkeyを選定しました。選定理由は以下の通りです。

- **コスト効率**：Redis OSSと比較した運用コストの削減
- **運用負荷軽減**：セキュリティパッチの自動適用
- **AWS統合**：他AWSサービスとのシームレスな連携

**設定時の注意点**

移行作業で最初に躓いたのが、ElastiCacheの設定でした。クラスターモードを有効にして高性能化を図ろうとしたところ、Redashが起動時にエラーを出力して正常に動作しませんでした。

調査の結果、Redashはクラスターモードに対応していないことが判明し、以下の設定で構築し直しました。

- マルチAZ構成：高可用性の確保
- 暗号化有効：セキュリティ要件への対応
- **クラスターモード無効**：Redash互換性のため（重要）

#### RDSパラメータ設定とDMS対応

DMS移行を成功させるため、移行先RDSでは以下のパラメータ設定が必要でした。

```sql
-- レプリケーション機能の有効化
session_replication_role = replica

-- 論理レプリケーションライブラリの読み込み  
shared_preload_libraries = pglogical

-- WALレベルの設定（EC2側にも必要）
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
```

特に、EC2側のPostgreSQLにも同様の設定追加が必要で、本番環境での設定変更には慎重な検証を行いました。

#### DMSでのデータ移行課題

DMS設定で最も困難だったのが、`queries`テーブルの移行でした。このテーブルには大容量のJSONデータが格納されており、DMSの制限に抵触してエラーが発生しました。

そこで、以下の戦略で対応しました。

1. DMSのテーブルマッピングで`queries`テーブルを除外
2. 他のテーブルは自動レプリケーションで移行
3. `queries`テーブルのみ手動でダンプ・リストアを実行

```json
{
  "rules": [
    {
      "rule-type": "selection",
      "rule-action": "exclude", 
      "object-locator": {
        "table-name": "queries"
      }
    }
  ]
}
```

### 移行実行と結果

データ移行作業は、事前検証を重ねた上で実行しました。

#### 事前準備作業

**1. EC2環境の設定変更**

既存のEC2上PostgreSQLにレプリケーション設定を追加する作業が最も緊張する部分でした。本番環境での設定変更のため、以下の手順で慎重に実行しました。

```bash
# PostgreSQL設定ファイルのバックアップ
cp postgres-data/postgresql.conf postgres-data/postgresql.conf.backup
cp postgres-data/pg_hba.conf postgres-data/pg_hba.conf.backup

# pglogicalライブラリ追加のためのDockerfile作成
cat << EOF > Dockerfile
FROM postgres:13
RUN apt-get update && apt-get install -y postgresql-13-pglogical
EOF

# Docker Composeでの再ビルド
docker-compose build --no-cache
docker-compose down
docker-compose up -d
```

**2. ECS環境構築**

RedashをコンテナベースでFargate上に展開しました。Worker暴走問題を解決するため、server、worker、schedulerを個別のタスクとして分離しました。

```json
{
  "family": "redash-server",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "containerDefinitions": [
    {
      "name": "redash-server",
      "image": "redash/redash:latest",
      "command": ["server"],
      "environment": [
        {
          "name": "REDASH_DATABASE_URL",
          "value": "postgresql://user:pass@rds-endpoint:5432/redash"
        },
        {
          "name": "REDASH_REDIS_URL", 
          "value": "redis://valkey-endpoint:6379/0"
        }
      ]
    }
  ]
}
```

#### 本番切り替え作業

移行当日は以下の手順で実行しました。

**1. サービス停止（5分）**
```bash
# Redashサービスの停止（PostgreSQLは継続）
docker-compose stop server scheduler scheduled_worker adhoc_worker redis nginx worker
```

**2. 残データ移行（20分）**
```bash
# queriesテーブルの手動移行
docker exec postgres pg_dump -U postgres -d postgres -t queries --data-only -Fc > queries.dump
pg_restore -h rds-endpoint -U postgres -d postgres queries.dump
```

**3. シーケンス調整（5分）**

DMSはデータのコピーはしますが、シーケンスの現在値は調整してくれません。この点は事前の移行テストで判明していたため、本番移行時には以下の対応を実施しました。

```sql
-- 全シーケンスの現在値を適切に調整
DO $$
DECLARE
  r RECORD;
BEGIN
  FOR r IN
    SELECT c.relname as seqname, t.relname as tablename, a.attname as colname
    FROM pg_class c
    JOIN pg_depend d ON d.objid = c.oid
    JOIN pg_class t ON d.refobjid = t.oid
    JOIN pg_attribute a ON a.attrelid = t.oid AND a.attnum = d.refobjsubid
    WHERE c.relkind = 'S'
  LOOP
    EXECUTE format('SELECT setval(%L, (SELECT COALESCE(MAX(%I), 0) + 1 FROM %I), false);',
                   r.seqname, r.colname, r.tablename);
  END LOOP;
END $$;
```

**4. ALB切り替え（5分）**

最後にALBのターゲットグループをEC2からECSサービスに変更し、動作確認を実施しました。

実際のダウンタイムは約35分で、予定していた30分を若干上回りましたが、シーケンス調整以外は想定通りに進行し、大きな問題なく移行を完了することができました。

:::details 参考：DMS設定のポイント
```hcl
# エンドポイント設定（ソース：EC2、ターゲット：RDS）
resource "aws_dms_endpoint" "postgres_source" {
  endpoint_type = "source"
  engine_name   = "postgres"
  server_name   = "EC2のプライベートIP"
  ssl_mode      = "none"
}

resource "aws_dms_endpoint" "postgres_target" {
  endpoint_type = "target"
  engine_name   = "postgres"
  server_name   = "RDSエンドポイント"
  ssl_mode      = "require"
}

# レプリケーションタスク
resource "aws_dms_replication_task" "postgres" {
  migration_type = "full-load-and-cdc"  # フルロード+継続レプリケーション
  # table_mappingsでqueriesテーブルを除外
}
```
:::

## 移行で学んだ教訓

### トラブルシューティングで得た知見

移行プロジェクトを通じて、いくつかの重要な学びがありました。

**1. Redashクラスターモード非対応**
ElastiCacheの設定でクラスターモードを有効にすると、Redashが正常に動作しないことが事前の移行テストで判明しました。この制約事項を踏まえ、本番環境では適切な設定で構築することができました。

**2. DMSの制限事項**
大容量JSONカラムを含むテーブルはDMSで移行できない場合があります。事前の移行テストで`queries`テーブルの問題を発見し、本番移行では手動移行戦略で対応することができました。

**3. シーケンス調整の重要性**
DMSはテーブルデータは移行しますが、シーケンスの現在値は調整されません。この点は事前の移行テストで判明し、本番移行時には適切に対応することができました。

### 事前検証の重要性

本番移行前に、ステージング環境での完全なリハーサルを3回実施しました。これにより以下のメリットを得ることができました。

- 移行時間の正確な見積もりが可能
- 手順書の詳細化と最適化
- 想定外の問題への対処法の事前準備

特に、本番データのコピーを使用した検証により、実際の移行で発生する問題を事前に発見・対処することができました。

## 結果と今後の展望

### 得られた成果

このマネージドサービス移行により、以下の成果を得ることができました。

**定量的効果**
- **可用性の向上**：Worker暴走によるサービス停止の完全解消（月2-3回 → 0回）
- **運用工数の削減**：インフラ管理工数を約60%削減
- **復旧時間の短縮**：障害時の復旧時間を15分から3分に短縮

**定性的効果**
- **運用負荷の大幅削減**：OSパッチ適用やデータベース管理作業の自動化
- **スケーラビリティの改善**：ECSによるリソース調整の簡易化（将来的な自動スケーリング実装基盤）
- **チーム生産性の向上**：業務時間中の障害対応による分析業務中断が解消

特に、分析チームが本来の業務に集中できる安定した環境を実現できました。

### 今後の取り組み

移行完了後も、以下の観点で継続的な改善を進めていく予定です。

- **パフォーマンス監視**：新環境でのレスポンス時間やリソース使用状況の継続監視
- **自動スケーリング実装**：負荷に応じたECSタスクの自動増減機能の追加
- **コスト最適化**：実際の使用パターンに基づいたリソースサイジングの調整
- **自動化推進**：デプロイメントや運用タスクのさらなる自動化

現在のECS環境では手動でのスケーリングが可能ですが、今後は使用状況を分析しながら自動スケーリング機能の実装を検討していきます。

この移行プロジェクトの知見を活用し、他のシステムについても同様のマネージドサービス活用を検討していきます。

## 参考
https://qiita.com/t_odash/items/8d28afd2ffd1e5929b99