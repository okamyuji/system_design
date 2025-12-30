# ビッグデータ処理設計

## 大量のイベントデータをバッチ処理する

### 概要

大量のイベントデータを定期的にバッチ処理することで、集計やレポート生成を効率的に実行します。
Lambda Architecture または Kappa Architecture を採用し、バッチレイヤーとスピードレイヤーを組み合わせます。

### システム設計図

```mermaid
graph TB
    subgraph "データ収集"
        Events[イベントソース<br/>Web/Mobile/IoT]
        Collector[データコレクター<br/>Fluentd/Logstash]
        RawStorage[生データストレージ<br/>S3/HDFS]
        
        Events --> Collector
        Collector --> RawStorage
    end
    
    subgraph "バッチ処理レイヤー"
        Scheduler[スケジューラー<br/>Airflow/Cron]
        Spark[Apache Spark<br/>分散処理エンジン]
        MapReduce[MapReduce<br/>大規模集計]
        
        Scheduler -->|日次バッチ起動| Spark
        RawStorage --> Spark
        Spark --> MapReduce
    end
    
    subgraph "データ変換"
        ETL[ETLパイプライン]
        Cleansing[データクレンジング<br/>欠損値処理]
        Aggregation[集計処理<br/>GROUP BY, SUM]
        Enrichment[データ濃縮<br/>外部データ結合]
        
        MapReduce --> ETL
        ETL --> Cleansing
        Cleansing --> Aggregation
        Aggregation --> Enrichment
    end
    
    subgraph "データストレージ"
        DWH[データウェアハウス<br/>BigQuery/Redshift]
        OLAP[OLAPキューブ<br/>多次元分析]
        
        Enrichment --> DWH
        DWH --> OLAP
    end
    
    subgraph "可視化"
        BI[BIツール<br/>Tableau/Looker]
        Dashboard[ダッシュボード]
        
        OLAP --> BI
        BI --> Dashboard
    end
```

```mermaid
sequenceDiagram
    participant Cron as スケジューラー
    participant Spark as Spark クラスター
    participant S3 as S3(生データ)
    participant DWH as データウェアハウス
    participant BI as BIツール

    Note over Cron,BI: 日次バッチ処理(午前2時実行)
    
    Cron->>Cron: 定期実行トリガー(2:00 AM)
    Cron->>Spark: バッチジョブ起動<br/>process_daily_events.py
    
    Spark->>Spark: クラスター初期化<br/>(Master + 10 Workers)
    
    Spark->>S3: データ読み込み<br/>s3://events/2025/01/01/*.json<br/>(100GB)
    S3-->>Spark: データ返却
    
    Note over Spark: Map フェーズ
    
    par 並列処理(10 Workers)
        Spark->>Spark: Worker1: パーティション1処理<br/>フィルタリング、変換
        Spark->>Spark: Worker2: パーティション2処理
        Spark->>Spark: Worker3: パーティション3処理
    end
    
    Note over Spark: Shuffle & Reduce フェーズ
    
    Spark->>Spark: データシャッフル<br/>(user_id でグループ化)
    
    par 集計処理
        Spark->>Spark: Worker1: ユーザー別PV集計
        Spark->>Spark: Worker2: ユーザー別CV集計
        Spark->>Spark: Worker3: ユーザー別滞在時間集計
    end
    
    Spark->>Spark: 集計結果マージ
    
    Spark->>DWH: 集計データ書き込み<br/>INSERT INTO daily_user_stats<br/>VALUES (...)
    DWH-->>Spark: 書き込み完了(1000万レコード)
    
    Spark->>Cron: ジョブ完了通知<br/>(処理時間: 45分)
    
    Note over Cron,BI: BI ツールでデータ可視化
    
    BI->>DWH: SELECT * FROM daily_user_stats<br/>WHERE date = '2025-01-01'
    DWH-->>BI: 集計データ
    BI->>BI: ダッシュボード更新
```

### 設計のポイント

データパーティショニング(日付、ユーザーID等)により、処理対象データを絞り込みます。
Sparkのメモリ内処理により、MapReduceより高速に処理できます。
増分処理(Incremental Processing)を実装して、新しいデータのみを処理します。
バッチ処理の失敗時は、自動リトライと通知機能を実装します。
データスキーマの進化に対応するため、スキーマレジストリを使用します。
処理時間を監視して、SLA(Service Level Agreement)を満たすように最適化します。

## 大量のイベントデータをストリーム処理する

### 概要

リアルタイムでイベントデータを処理することで、即座に分析やアラートを実行します。
Apache Kafka、Apache Flink、Kinesis等のストリーム処理基盤を使用します。

### システム設計図

```mermaid
graph LR
    subgraph "データソース"
        WebApp[Webアプリ]
        MobileApp[モバイルアプリ]
        IoT[IoTデバイス]
        API[API サーバー]
    end
    
    subgraph "メッセージブローカー"
        Kafka[Apache Kafka<br/>トピック: events]
        Topic1[トピック: user_events<br/>パーティション: 10]
        Topic2[トピック: system_events<br/>パーティション: 5]
        
        Kafka --> Topic1
        Kafka --> Topic2
    end
    
    subgraph "ストリーム処理"
        Flink[Apache Flink<br/>ストリーム処理エンジン]
        Window[ウィンドウ処理<br/>5分間の集計]
        StateStore[状態ストア<br/>RocksDB]
        
        Topic1 --> Flink
        Topic2 --> Flink
        Flink --> Window
        Flink <--> StateStore
    end
    
    subgraph "データシンク"
        Elasticsearch[Elasticsearch<br/>全文検索]
        TimeSeries[時系列DB<br/>InfluxDB]
        OLAP[OLAPストア<br/>ClickHouse]
        Alert[アラートサービス]
    end
    
    WebApp --> Kafka
    MobileApp --> Kafka
    IoT --> Kafka
    API --> Kafka
    
    Window --> Elasticsearch
    Window --> TimeSeries
    Window --> OLAP
    Window --> Alert
```

```mermaid
sequenceDiagram
    participant Producer as イベント生成
    participant Kafka as Kafka
    participant Flink as Flink
    participant State as 状態ストア
    participant Sink as データシンク
    participant Alert as アラート

    Note over Producer,Alert: リアルタイムストリーム処理
    
    loop イベント生成(秒間1000件)
        Producer->>Kafka: イベント発行<br/>{user_id: 123, event: "page_view", ...}
        Kafka->>Kafka: パーティション割り当て<br/>(user_id % 10)
    end
    
    Flink->>Kafka: イベント消費<br/>(並列度: 10)
    Kafka-->>Flink: イベントストリーム
    
    Flink->>Flink: イベント処理<br/>フィルタリング、変換
    
    Note over Flink,State: ウィンドウ集計(5分タンブリングウィンドウ)
    
    Flink->>State: 状態更新<br/>user_123: {pv: 10, cv: 2, ...}
    
    loop 5分ごと
        Flink->>Flink: ウィンドウクローズ<br/>(00:00-00:05)
        Flink->>State: 状態取得<br/>全ユーザーの集計値
        State-->>Flink: {user_123: {...}, user_456: {...}}
        
        Flink->>Flink: 集計結果計算
        
        par データシンク
            Flink->>Sink: 集計データ書き込み<br/>Elasticsearch, TimeSeries
            Sink-->>Flink: ACK
        end
        
        Flink->>Flink: 異常検知<br/>(PV急増 > 閾値)
        alt 異常検知
            Flink->>Alert: アラート送信<br/>user_123: PV 500% 増加
            Alert->>Alert: Slack/Email通知
        end
        
        Flink->>State: 状態クリア<br/>(次のウィンドウ準備)
    end
    
    Note over Producer,Alert: Exactly-Onceセマンティクス
    
    Flink->>Flink: チェックポイント作成<br/>(30秒ごと)
    Flink->>State: 状態スナップショット保存
    
    Note over Flink: Flink障害発生
    
    Flink->>Flink: 再起動
    Flink->>State: 最後のチェックポイントから復旧
    Flink->>Kafka: オフセットリセット<br/>(チェックポイント時点)
    Flink->>Flink: 処理再開(重複なし)
```

### 設計のポイント

ウィンドウ処理(Tumbling、Sliding、Session)を使用して、時間範囲内のイベントを集計します。
Exactly-Onceセマンティクスにより、イベントの重複処理や欠落を防ぎます。
状態ストアを使用して、ストリーム処理中の中間状態を保持します。
バックプレッシャー機能により、処理速度を調整して、メモリオーバーフローを防ぎます。
チェックポイント機能により、障害発生時に一貫性のある状態から復旧します。
遅延到着データ(Late Data)に対応するため、Watermarkを設定します。

## アナリティクス用のデータベースを活用する

### 概要

OLTP(Online Transaction Processing)とOLAP(Online Analytical Processing)を分離し、分析専用のデータベースを活用します。
カラムナストレージ、MPP(Massively Parallel Processing)により、大規模データの高速クエリを実現します。

### システム設計図

```mermaid
graph TB
    subgraph "OLTP システム"
        App[アプリケーション]
        OLTP_DB[(OLTPデータベース<br/>MySQL/PostgreSQL)]
        
        App --> OLTP_DB
    end
    
    subgraph "ETL パイプライン"
        CDC[Change Data Capture<br/>Debezium]
        ETL_Job[ETLジョブ<br/>dbt/Airflow]
        DataLake[データレイク<br/>S3]
        
        OLTP_DB --> CDC
        CDC --> ETL_Job
        ETL_Job --> DataLake
    end
    
    subgraph "OLAP システム"
        DWH[データウェアハウス<br/>BigQuery/Redshift/Snowflake]
        ColumnStore[カラムナストレージ<br/>圧縮効率高]
        MPP[MPPアーキテクチャ<br/>並列クエリ実行]
        
        DataLake --> DWH
        DWH --> ColumnStore
        DWH --> MPP
    end
    
    subgraph "分析ツール"
        SQL[SQL分析<br/>アドホッククエリ]
        BI[BIツール<br/>Tableau/Looker]
        ML[機械学習<br/>予測モデル]
        
        DWH --> SQL
        DWH --> BI
        DWH --> ML
    end
```

```mermaid
graph LR
    subgraph "行指向ストレージ(OLTP)"
        Row1[Row1: id=1, name=Alice, age=30, city=Tokyo]
        Row2[Row2: id=2, name=Bob, age=25, city=Osaka]
        Row3[Row3: id=3, name=Carol, age=35, city=Tokyo]
    end
    
    subgraph "列指向ストレージ(OLAP)"
        Col_ID[id: 1, 2, 3, ...]
        Col_Name[name: Alice, Bob, Carol, ...]
        Col_Age[age: 30, 25, 35, ...]
        Col_City[city: Tokyo, Osaka, Tokyo, ...]
    end
```

```mermaid
sequenceDiagram
    participant Analyst as アナリスト
    participant BI as BIツール
    participant DWH as データウェアハウス
    participant Compute as コンピュートノード
    participant Storage as ストレージレイヤー

    Note over Analyst,Storage: OLAP クエリ実行
    
    Analyst->>BI: ダッシュボード表示<br/>(売上分析)
    
    BI->>DWH: SELECT<br/>  date,<br/>  SUM(amount) as total_sales,<br/>  COUNT(*) as order_count<br/>FROM orders<br/>WHERE date BETWEEN '2024-01-01' AND '2024-12-31'<br/>  AND category = 'Electronics'<br/>GROUP BY date<br/>ORDER BY date
    
    DWH->>DWH: クエリプランニング<br/>(実行計画作成)
    
    Note over DWH: MPP並列実行
    
    par 並列スキャン(10ノード)
        DWH->>Compute: Node1: パーティション1スキャン<br/>(2024-01-01 ~ 2024-02-01)
        DWH->>Compute: Node2: パーティション2スキャン<br/>(2024-02-01 ~ 2024-03-01)
        DWH->>Compute: Node3-10: 他パーティション
    end
    
    Compute->>Storage: カラムスキャン<br/>(date, amount, category のみ)
    Note over Storage: 必要な列のみ読み取り<br/>圧縮解凍(Snappy/Zstandard)
    Storage-->>Compute: 圧縮データ返却
    
    par 集計処理(各ノード)
        Compute->>Compute: Node1: ローカル集計<br/>GROUP BY date, SUM(amount)
        Compute->>Compute: Node2: ローカル集計
        Compute->>Compute: Node3-10: ローカル集計
    end
    
    Compute->>DWH: 部分集計結果返却
    DWH->>DWH: 最終集計(Merge)<br/>全ノードの結果統合
    DWH->>DWH: ソート処理<br/>ORDER BY date
    
    DWH-->>BI: クエリ結果<br/>(365行、処理時間: 3秒)
    
    BI->>BI: データ可視化<br/>(折れ線グラフ作成)
    BI-->>Analyst: ダッシュボード表示
    
    Note over Analyst,Storage: クエリ最適化の利点
    
    Note over Storage: 列指向ストレージの利点<br/>- 必要な列のみスキャン(I/O削減)<br/>- 高圧縮率(同じデータタイプ)<br/>- ベクトル化処理(SIMD)
    
    Note over Compute: MPPの利点<br/>- 並列スキャン(10倍高速)<br/>- 並列集計(水平スケール)<br/>- メモリ内処理(高速)
```

### 設計のポイント

OLTPとOLAPを分離して、トランザクション処理と分析処理の相互影響を防ぎます。
カラムナストレージにより、集計クエリで必要な列のみをスキャンします。
パーティショニング(日付、地域等)により、スキャン範囲を絞り込みます。
マテリアライズドビューを使用して、頻繁に実行されるクエリを事前計算します。
クエリキャッシュにより、同じクエリの再実行を高速化します。
データの圧縮(Snappy、Zstandard)により、ストレージコストとI/Oを削減します。
クエリコストを監視して、高コストなクエリを最適化します。
