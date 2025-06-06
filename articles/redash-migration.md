---
title: "Redashの移行作業レポート：EC2からECS/RDSへの移行"
emoji: "😄"
type: "tech"
topics: ["redash","aws","dms","ecs","redis"]
published: false
publication_name: "paiza"
---

## はじめに
本記事では、Redashの運用環境をEC2からECS/RDSへ移行した際の手順について共有します。

## 移行の背景

paizaではRedashを活用してデータ分析を行っていましたが
以下の課題を解決するため、Redash運用環境をEC2から各種AWSマネージド・サービスへ移行することにしました。

- **可用性とスケーラビリティの向上**  
  EC2での運用では、インスタンス障害時の復旧やスケールアップに手動介入が必要でした。  
  マネージドサービスへの移行により、運用の自動化と安定性の向上を図ります。

- **Workerプロセスの分離**  
  Redashのworkerプロセスがメモリを消費し尽くし、サービス全体が停止する事象が発生していました。  
  コンテナ化によりworkerを分離し、リソース管理の改善を目指します。

- **インフラ管理の効率化**  
  インスタンスのパッチ適用や死活監視など、インフラ管理の工数を削減し、  
  より本質的な業務に注力できる環境を構築します。

リソースはterraformで管理する想定です。
  
## 移行元・移行先の環境

移行前イメージ
![移行前のアーキテクチャ](/images/redash-migration/redash-before.png)

移行後イメージ
![移行後のアーキテクチャ](/images/redash-migration/redash-after.png)

### EC2上のRedash環境

- OS: Amazon Linux 2
- データベース: PostgreSQL（EC2内に構築）
- ストレージ: EBS
- バックアップ: AMI自動バックアップ

### 移行目標

- 運用負荷の軽減
- 障害時の復旧時間短縮
- コスト削減

### 使用したAWSサービス

- RDS（PostgreSQL）
- ECS（コンテナ管理）
- Redis(キュー管理)

## 移行作業内容

既存DB内容に設定されている内容(ダッシュボードやクエリ等)を引き続き利用するため、DMSを利用したフルロード・レプリケーション移行を実施しました。

実際の移行作業内容を記載します。
(ネットワーク設定については割愛します。環境に応じてご準備ください)

### 1. 移行先Redis(valkey)作成

Redashクエリ実行時のキューイングに利用されているRedisを作成します。
サービス停止を挟む移行のため、中身の移行は不要と判断しました。

主な設定ポイント：
- マルチAZ構成で可用性を確保
- 暗号化を有効化
- メンテナンス時間帯の設定
- クラスターモードは無効

Valkeyを選定した理由：
- AWSのマネージドサービスとして提供されており、運用負荷の軽減が可能
- Redis OSSと比較して、セキュリティアップデートやパッチ適用が自動化されている
- コスト効率が良く、特に小規模な利用ではRedis OSSよりも経済的
- AWSの他のサービスとの統合が容易で、監視やバックアップの設定が簡単

注意点として、Redisのクラスターモードを無効に設定しないとRedash側でエラーになります。

![valkey](/images/redash-migration/elasticache-valkey.png)

:::details elasticache.tf
```hcl
resource "aws_elasticache_replication_group" "redash" {
  replication_group_id       = "${aws_elasticache_replication_group_redash}"
  description                = ""
  engine                     = "valkey"
  engine_version             = "8.0"
  automatic_failover_enabled = true
  multi_az_enabled           = true
  num_cache_clusters         = 2
  node_type                  = "cache.t4g.micro"
  port                       = "6379"
  parameter_group_name       = aws_elasticache_parameter_group.redash.name
  at_rest_encryption_enabled = true
  transit_encryption_enabled = false
  maintenance_window         = "sun:16:00-sun:17:00"
  apply_immediately          = true

  subnet_group_name = aws_elasticache_subnet_group.protect.name
  security_group_ids = [
    "${redash_security_group_ids}"
  ]
}

resource "aws_elasticache_subnet_group" "protect" {
  name = "${aws_elasticache_subnet_group_protect_id}"
  subnet_ids = [
    "${redash_subnet_id}"
  ]
}

resource "aws_elasticache_parameter_group" "redash" {
  name   = "${aws_elasticache_parameter_group_redash_id}"
  family = "valkey8"
}
```
:::

### 2. 移行先DB作成

Redashのメタデータを保存するPostgreSQLを作成します。
PostgreSQLに独自設定を追加している場合は、移行先DBにも忘れずに設定しておきます。

主な設定ポイント：
- パラメータグループの設定
- パブリックアクセス無効

![postgreSQL](/images/redash-migration/redash-rds.png)

パラメータグループにレプリケーション用の設定を追加しています

![paramgp](/images/redash-migration/rds-paramgp.png)

:::details rds.tf
```hcl
resource "aws_db_instance" "redash" {
  identifier               = "${aws_db_instance_redash_identifier}"
  allocated_storage        = 100
  max_allocated_storage    = 300
  storage_type             = "gp3"
  storage_encrypted        = true
  engine                   = "postgres"
  engine_version           = "13.20"
  instance_class           = "db.m7g.large"
  ca_cert_identifier       = "rds-ca-rsa2048-g1"
  engine_lifecycle_support = "open-source-rds-extended-support-disabled"

  username            = "${postgres_username}"
  password            = "${postgres_password}"
  port                = "5432"
  publicly_accessible = false
  availability_zone   = "ap-northeast-1c"
  apply_immediately   = true
  vpc_security_group_ids = [
    "{redash_security_group_ids}"
  ]
  db_subnet_group_name    = aws_db_subnet_group.protect.name
  parameter_group_name    = aws_db_parameter_group.redash_parameter_group.name
  option_group_name       = aws_db_option_group.redash_option_group.name
  multi_az                = true
  backup_retention_period = "7"
  backup_window           = "16:46-17:16"
  maintenance_window      = "sat:13:57-sat:14:27"
  skip_final_snapshot     = true
  performance_insights_enabled = true
  deletion_protection          = true

  enabled_cloudwatch_logs_exports = [
    "postgresql",
    "upgrade"
  ]
}

resource "aws_db_subnet_group" "protect" {
  name        = "${aws_db_subnet_group_protect_name}"
  description = "${aws_db_subnet_group_protect_description}"
  subnet_ids = [
    "${redash_subnet_id}"
  ]
}

resource "aws_db_parameter_group" "redash_parameter_group" {
  name   = "${aws_db_parameter_group_redash_parameter_group}"
  family = "postgres13"

  # 処理に1000ms以上かかるSQLをログに出力
  parameter {
    name  = "log_min_duration_statement"
    value = "1000"
  }

  parameter {
    name  = "session_replication_role"
    value = "replica"
  }

  parameter {
    apply_method = "pending-reboot"
    name         = "shared_preload_libraries"
    value        = "pglogical"
  }
}

resource "aws_db_option_group" "redash_option_group" {
  name                     = "${aws_db_option_group_redash_option_group_name}"
  engine_name              = local.rds_setting.engine
  major_engine_version     = local.rds_setting.engine_version_for_family_name
  option_group_description = "${aws_db_option_group_redash_option_group_option_group_description}"
}

```
:::

### 3. DMS動作用の設定を既存DBに反映

DMSのレプリケーションインスタンスからの通信を受け付ける設定を既存EC2で動作中のPostgreSQLに設定します

#### EC2インスタンスの設定変更

redashを起動しているEC2インスタンスへログインして、下記の設定を行います(各自の環境で必要な作業を実施してください)

:::details EC2インスタンスでの作業内容
```json
cd ${redashの設定場所}

# 必要なツールをインストール
sudo apt install postgresql-client-common postgresql-client

# postgreSQLのスキーマ情報をエクスポートする
docker exec -it ${docker_postgres_id} pg_dump -U postgres -d postgres --schema-only > redash_schema.sql

# エクスポートしたスキーマを作成したRDSに投入する
# EC2インスタンス上から実行していますが、疎通できない場合は各自対応をお願いします
psql -h ${rds_postgresql_host} -p 5432 -U ${postgres_username} -d ${postgres_password} < redash_schema.sql

# 既存postgresを編集
cp postgres-data/pg_hba.conf postgres-data/pg_hba.conf-xxxxxxxx
sudo vi postgres-data/pg_hba.conf

# 以下の設定を最終行に追加
host replication dms all md5

sudo cp postgres-data/postgresql.conf postgres-data/postgresql.conf-xxxxxxxx
sudo vi postgres-data/postgresql.conf

# 以下の設定を最終行に追加
shared_preload_libraries = 'pglogical'
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
wal_sender_timeout = 240s

# postgreSQL用のdockerfileを作成(レプリケーションに必要なライブラリインストールするため)
vi dockerfile
# 以下の内容を設定
FROM postgres:13
RUN apt-get update && apt-get install -y postgresql-13-pglogical

# docker-composeファイルを修正
cp docker-compose.yml docker-compose.yml-xxxxxxxx

vi docker-compose.yml

# postgresの設定に以下を追加する
postgres:
    build: .
    ports:
      - "5432:5432"

vi .dockerignore
# 以下の内容を設定
postgres-data

# 設定変更を反映する
docker-compose build --no-cache
docker-compose down
docker-compose up -d
```
:::

上記変更を反映後、redashの全機能が問題なく実行できることを確認する
- クエリ実行・保存
- ダッシュボード作成・閲覧
- スケジューラ実行
- ユーザ作成・変更

### 4. DMSリソース作成

データ移行用のDMSリソースを作成します。
今回はレプリケーションインスタンスを利用したデータ移行を実行するため下記4つのリソースを作成します

- ソースエンドポイント
- ターゲットエンドポイント
- レプリケーションインスタンス
- データ移行タスク

注意点としてデータ移行タスクからqueriesテーブルを除外しています。
queriesテーブルにはDMSデータ移行タスクにおいて、移行不可カラムが設定されていたため
後工程にて手動で移行しています。

![dms_source_endpoint](/images/redash-migration/dms_source_endpoint.png)

![dms_target_endpoint](/images/redash-migration/dms_target_endpoint.png)

![dms_migrate_instance](/images/redash-migration/dms_migration_instance.png)

![dms_db_task](/images/redash-migration/dms_db_task.png)

:::details dms_endpoint.tf
```hcl
resource "aws_dms_endpoint" "postgres_source" {
  endpoint_id   = "${aws_dms_endpoint_postgres_source_endpoint_id}"
  endpoint_type = "source"
  engine_name   = "postgres"
  username      = "${postgres_username}"
  password      = "${postgres_password}"
  port          = 5432
  server_name   = "${redashが動作しているEC2サーバのプライベートIPアドレス}"
  ssl_mode      = "none"
  database_name = "postgres"
}

resource "aws_dms_endpoint" "postgres_target" {
  endpoint_id   = format("%s-%s-dms-endpoint-%s-postgres-target", local.resource_group, var.environment, local.resource_sub_group)
  endpoint_type = "target"
  engine_name   = "postgres"
  username      = "${postgres_username}"
  password      = "${postgres_password}"
  port          = 5432
  server_name   = "${移行先postgreSQLのendpoint}"
  ssl_mode      = "require"
  database_name = "postgres"
}
```
:::

:::details dms_replication.tf
```hcl
resource "aws_dms_replication_instance" "migration_instance" {
  replication_instance_id     = "migration-instance"
  replication_instance_class  = "dms.r5.2xlarge"
  replication_subnet_group_id = aws_dms_replication_subnet_group.private.id
  vpc_security_group_ids      = [${vpc_security_group_ids}]
  allocated_storage           = 300
  publicly_accessible         = false
}

resource "aws_dms_replication_subnet_group" "private" {
  replication_subnet_group_description = "${aws_dms_replication_subnet_group_private_description}"
  replication_subnet_group_id          = "${aws_dms_replication_subnet_group_private_id}"
  subnet_ids = [
    ${private_subnet_ids}
  ]
}

resource "aws_dms_replication_task" "postgres" {
  migration_type            = "full-load-and-cdc"
  replication_instance_arn  = aws_dms_replication_instance.migration_instance.replication_instance_arn
  replication_task_id       = "postgres-migration-task"
  replication_task_settings = templatefile("${path.module}/replication_settings.json", {})
  source_endpoint_arn       = aws_dms_endpoint.postgres_source.endpoint_arn
  target_endpoint_arn       = aws_dms_endpoint.postgres_target.endpoint_arn
  table_mappings            = templatefile("${path.module}/table_mappings.json", {})
  lifecycle {
    ignore_changes = [
      replication_task_settings
    ]
  }
}
```
:::

:::details replication_setting.json
```json
{
    "Logging": {
      "EnableLogging": true,
      "EnableLogContext": true,
      "LogComponents": [
        {
          "Severity": "LOGGER_SEVERITY_DEFAULT",
          "Id": "TRANSFORMATION"
        },
        {
          "Severity": "LOGGER_SEVERITY_ERROR",
          "Id": "SOURCE_UNLOAD"
        },
        {
          "Severity": "LOGGER_SEVERITY_DEFAULT",
          "Id": "IO"
        },
        {
          "Severity": "LOGGER_SEVERITY_ERROR",
          "Id": "TARGET_LOAD"
        },
        {
          "Severity": "LOGGER_SEVERITY_DEFAULT",
          "Id": "PERFORMANCE"
        },
        {
          "Severity": "LOGGER_SEVERITY_ERROR",
          "Id": "SOURCE_CAPTURE"
        },
        {
          "Severity": "LOGGER_SEVERITY_DEFAULT",
          "Id": "SORTER"
        },
        {
          "Severity": "LOGGER_SEVERITY_DEFAULT",
          "Id": "REST_SERVER"
        },
        {
          "Severity": "LOGGER_SEVERITY_DEFAULT",
          "Id": "VALIDATOR_EXT"
        },
        {
          "Severity": "LOGGER_SEVERITY_ERROR",
          "Id": "TARGET_APPLY"
        },
        {
          "Severity": "LOGGER_SEVERITY_ERROR",
          "Id": "TASK_MANAGER"
        },
        {
          "Severity": "LOGGER_SEVERITY_DEFAULT",
          "Id": "TABLES_MANAGER"
        },
        {
          "Severity": "LOGGER_SEVERITY_DEFAULT",
          "Id": "METADATA_MANAGER"
        },
        {
          "Severity": "LOGGER_SEVERITY_DEFAULT",
          "Id": "FILE_FACTORY"
        },
        {
          "Severity": "LOGGER_SEVERITY_DEFAULT",
          "Id": "COMMON"
        },
        {
          "Severity": "LOGGER_SEVERITY_DEFAULT",
          "Id": "ADDONS"
        },
        {
          "Severity": "LOGGER_SEVERITY_DEFAULT",
          "Id": "DATA_STRUCTURE"
        },
        {
          "Severity": "LOGGER_SEVERITY_DEFAULT",
          "Id": "COMMUNICATION"
        },
        {
          "Severity": "LOGGER_SEVERITY_DEFAULT",
          "Id": "FILE_TRANSFER"
        }
      ]
    },
    "StreamBufferSettings": {
      "StreamBufferCount": 3,
      "CtrlStreamBufferSizeInMB": 5,
      "StreamBufferSizeInMB": 8
    },
    "ErrorBehavior": {
      "FailOnNoTablesCaptured": true,
      "ApplyErrorUpdatePolicy": "LOG_ERROR",
      "FailOnTransactionConsistencyBreached": false,
      "RecoverableErrorThrottlingMax": 1800,
      "DataErrorEscalationPolicy": "SUSPEND_TABLE",
      "ApplyErrorEscalationCount": 0,
      "RecoverableErrorStopRetryAfterThrottlingMax": true,
      "RecoverableErrorThrottling": true,
      "ApplyErrorFailOnTruncationDdl": false,
      "DataMaskingErrorPolicy": "STOP_TASK",
      "DataTruncationErrorPolicy": "LOG_ERROR",
      "ApplyErrorInsertPolicy": "LOG_ERROR",
      "EventErrorPolicy": "IGNORE",
      "ApplyErrorEscalationPolicy": "LOG_ERROR",
      "RecoverableErrorCount": -1,
      "DataErrorEscalationCount": 0,
      "TableErrorEscalationPolicy": "STOP_TASK",
      "RecoverableErrorInterval": 5,
      "ApplyErrorDeletePolicy": "IGNORE_RECORD",
      "TableErrorEscalationCount": 0,
      "FullLoadIgnoreConflicts": true,
      "DataErrorPolicy": "LOG_ERROR",
      "TableErrorPolicy": "SUSPEND_TABLE"
    },
    "ValidationSettings": {
      "ValidationPartialLobSize": 0,
      "PartitionSize": 10000,
      "RecordFailureDelayLimitInMinutes": 0,
      "SkipLobColumns": false,
      "FailureMaxCount": 10000,
      "HandleCollationDiff": false,
      "ValidationQueryCdcDelaySeconds": 0,
      "ValidationMode": "ROW_LEVEL",
      "TableFailureMaxCount": 1000,
      "RecordFailureDelayInMinutes": 5,
      "MaxKeyColumnSize": 8096,
      "EnableValidation": true,
      "ThreadCount": 5,
      "RecordSuspendDelayInMinutes": 30,
      "ValidationOnly": false
    },
    "TTSettings": {
      "TTS3Settings": null,
      "TTRecordSettings": null,
      "EnableTT": false
    },
    "FullLoadSettings": {
      "CommitRate": 10000,
      "StopTaskCachedChangesApplied": false,
      "StopTaskCachedChangesNotApplied": false,
      "MaxFullLoadSubTasks": 1,
      "TransactionConsistencyTimeout": 600,
      "CreatePkAfterFullLoad": false,
      "TargetTablePrepMode": "DO_NOTHING"
    },
    "TargetMetadata": {
      "ParallelApplyBufferSize": 0,
      "ParallelApplyQueuesPerThread": 0,
      "ParallelApplyThreads": 0,
      "TargetSchema": "",
      "InlineLobMaxSize": 0,
      "ParallelLoadQueuesPerThread": 0,
      "SupportLobs": true,
      "LobChunkSize": 128,
      "TaskRecoveryTableEnabled": false,
      "ParallelLoadThreads": 0,
      "LobMaxSize": 0,
      "BatchApplyEnabled": false,
      "FullLobMode": true,
      "LimitedSizeLobMode": false,
      "LoadMaxFileSize": 0,
      "ParallelLoadBufferSize": 0
    },
    "BeforeImageSettings": null,
    "ControlTablesSettings": {
      "historyTimeslotInMinutes": 5,
      "HistoryTimeslotInMinutes": 5,
      "StatusTableEnabled": false,
      "SuspendedTablesTableEnabled": false,
      "HistoryTableEnabled": false,
      "ControlSchema": "",
      "FullLoadExceptionTableEnabled": false
    },
    "LoopbackPreventionSettings": null,
    "CharacterSetSettings": null,
    "FailTaskWhenCleanTaskResourceFailed": false,
    "ChangeProcessingTuning": {
      "StatementCacheSize": 50,
      "CommitTimeout": 1,
      "RecoveryTimeout": -1,
      "BatchApplyPreserveTransaction": true,
      "BatchApplyTimeoutMin": 1,
      "BatchSplitSize": 0,
      "BatchApplyTimeoutMax": 30,
      "MinTransactionSize": 1000,
      "MemoryKeepTime": 60,
      "BatchApplyMemoryLimit": 500,
      "MemoryLimitTotal": 1024
    },
    "ChangeProcessingDdlHandlingPolicy": {
      "HandleSourceTableDropped": true,
      "HandleSourceTableTruncated": true,
      "HandleSourceTableAltered": true
    },
    "PostProcessingRules": null
  }
```
:::

:::details table_mapping.json
```json
{
    "rules": [
        {
            "rule-id": 1,
            "rule-name": "queries_selection",
            "rule-type": "selection",
            "rule-action": "exclude",
            "object-locator": {
                "schema-name": "public",
                "table-name": "queries%"
            },
            "filters": []
        },
        {
            "rule-id": 2,
            "rule-name": "other_tables_selection",
            "rule-type": "selection",
            "rule-action": "include",
            "object-locator": {
                "schema-name": "public",
                "table-name": "%"
            },
            "filters": []
        }
    ]
}


```
:::

### 5. ECR/ECSサービス作成

EC2の代わりにredash機能を提供するweb/workerを動作するためにECR/ECSで作成します。<br>
一例となりますが、以下のように各種ECR/ECSサービスを準備します。

![redash_ecs_task](/images/redash-migration/redash-ecr.png)

環境変数に値を設定し、起動コマンドを役割ごとに変更しています。
(redisやpostgreSQLの接続情報等はパラメータストアから値を参照するようにしています)

:::details ECSタスクの設定例
```json
{
  "family": "redash-server",
  "taskRoleArn": "arn:aws:iam::account:role/ecsTaskRole",
  "executionRoleArn": "arn:aws:iam::account:role/ecsTaskExecutionRole",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "containerDefinitions": [
    {
      "name": "redash-server",
      "image": "redash/redash:latest",
      "command": ["server"],
      "secrets": [
        {
          "name": "REDASH_DATABASE_URL",
          "valueFrom": "/redash/database_url"
        }
      ]
    }
  ]
}
```
:::

![redash_ecs_env](/images/redash-migration/redash-ecs-env.png)

### 6. データの移行前同期

作成したDMSリソースを利用して既存DBデータを新DBへ移行します。
エンドポイントの接続設定を確認後、データベース移行タスクを起動します。

DBサイズによっては数時間以上かかる場合もあるので、本移行前に余裕を持って実行してください。
フルロード・CDC連携設定なので、データ移行後のレプリケーションは自動で行われます。
本移行前までデータ移行タスクを停止しないように注意してください。

![dms_db_task](/images/redash-migration/dms_db_task_start.png)

### 7. サービスを停止して、マネージドサービスへの移行作業 

DMSタスクで各テーブルのフルロード完了・検証で失敗がないことを確認後、既存サービスを停止して移行作業を開始します。

:::details EC2インスタンスでの作業内容
```bash
cd ${redashの設定場所}

# postgres以外のdockerサービスを停止する
docker-compose stop server scheduler scheduled_worker adhoc_worker redis nginx worker

# DMSで移行できないquerisテーブルのデータをダンプする
docker exec -it ${docker_postgres_id} pg_dump -U postgres -d postgres -t public.queries --data-only -Fc -f /tmp/queries.dump
# ローカルにコピー
docker cp ${docker_postgres_id}:/tmp/queries.dump ./queries.dump

# queriesテーブルのデータをRDSに投入する
# EC2インスタンス上から実行していますが、疎通できない場合は各自対応をお願いします
pg_restore -h ${rds_postgresql_host} -p 5432 -U postgres -d postgres -Fc queries.dump
```
:::

上記設定が完了し、DMSのレプリケーションに失敗していないことを確認します(失敗している場合は、再検証を試す)

問題ない場合、DMSのデータベース移行タスクを停止してRDSのpostgreSQLで以下コマンドを実行してください。
(DBデータ移行のみではシーケンス未設定状態なので、設定されている値を元にズレを修正)

:::details シーケンス調整
```sql
# シーケンスとその紐づくテーブルカラムを特定し、連番のズレを修正
DO $$
DECLARE
  r RECORD;
BEGIN
  FOR r IN
    SELECT
      c.relname as seqname,
      t.relname as tablename,
      a.attname as colname
    FROM
      pg_class c
      JOIN pg_depend d ON d.objid = c.oid
      JOIN pg_class t ON d.refobjid = t.oid
      JOIN pg_attribute a ON a.attrelid = t.oid AND a.attnum = d.refobjsubid
    WHERE
      c.relkind = 'S'
  LOOP
    EXECUTE format(
      'SELECT setval(%L, (SELECT COALESCE(MAX(%I), 0) + 1 FROM %I), false);',
      r.seqname, r.colname, r.tablename
    );
  END LOOP;
```
:::

設定が完了したら既存redashのサービスをすべて停止します

```
docker-compose down
```

以降はECSサービスへ通信を行うように、ALBの設定変更を行います

#### ALBターゲットグループの切り替え手順

既存のEC2ベースの構成からECSサービスへの切り替えを行うため、ALBの設定を段階的に変更します。
**注意**: この作業中はサービスにアクセスできなくなるため、事前に利用者への告知を行ってください。

**1. 既存ターゲットグループからEC2インスタンスの登録解除**
- EC2コンソールまたはELBコンソールから、現在のターゲットグループを確認
- RedashサービスのEC2インスタンスを「Draining」状態に変更し、既存接続の完了を待機
- 完全に切断が完了したら、ターゲットグループからEC2インスタンスを削除

**2. ECSサービス用の新しいターゲットグループ作成**
- プロトコル：HTTP（または環境に応じてHTTPS）
- ポート：5000（Redashのデフォルトポート）
- ターゲットタイプ：IP（ECS Fargateの場合）
- ヘルスチェック設定：
  - パス：`/ping`（Redashのヘルスチェックエンドポイント）
  - 間隔：30秒
  - タイムアウト：5秒
  - 正常しきい値：2回
  - 異常しきい値：3回

**3. ALBリスナールールの更新**
- 既存のリスナールール（EC2向け）を新しいターゲットグループ（ECS向け）に変更
- パスベースまたはホストベースのルーティングが設定されている場合は、ルールの優先度も確認
- 設定変更後、ALBの設定が正しく反映されていることを確認

**4. ECSサービスの起動とターゲットグループへの登録**
- ECSサービスを起動し、タスクが正常に起動することを確認
- タスクが自動的にターゲットグループに登録されることを確認
- ヘルスチェックが正常に完了し、ターゲットが「Healthy」状態になることを確認

**5. 旧リソースのクリーンアップ**
- 新しい構成でのサービス動作確認が完了した後、不要となった旧ターゲットグループを削除
- セキュリティグループの設定も必要に応じて更新（ECSタスクからRDS/Redisへのアクセス許可等）

デプロイ完了後、redashの全機能が問題なく実行できることを確認

- ダッシュボード作成
- スケジューラ起動確認
- アラート起動確認
- 全データソースに対してクエリを実行
- クエリ実行・保存
- ユーザ作成・変更

実際の移行作業のダウンタイムは概ね以下になります
- 既存サービス停止からECSサービス起動まで：約30分(動作確認含む)

以上で、redashの移行作業は完了になります。

## まとめ

今までは重いクエリを叩いたりEC2のリソース上限に達した場合サービスが停止することもありましたが、
移行後はそのようなことは発生せず、安定した運用ができています。
各種スケーリングについても簡単に増減設定ができるので運用の幅も広がりました。

## 参考
https://qiita.com/t_odash/items/8d28afd2ffd1e5929b99