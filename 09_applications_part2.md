# 応用問題 Part 2

システム設計の応用問題として、SNS、ストレージサービスの設計を行います。

## SNSプラットフォームの設計

### 概要

数億ユーザーをサポートするSNSプラットフォームを設計します。
ツイート投稿、タイムライン生成、フォロー関係、リツイート、いいねを含みます。

### システム設計図

```mermaid
graph TB
    subgraph "クライアント"
        Web[Webブラウザ]
        Mobile[モバイルアプリ]
    end
    
    subgraph "API層"
        LB[ロードバランサー]
        TweetAPI[ツイートAPI]
        TimelineAPI[タイムラインAPI]
        UserAPI[ユーザーAPI]
    end
    
    subgraph "タイムライン生成"
        FanoutService[Fanoutサービス]
        TimelineCache[(Redis タイムライン)]
    end
    
    subgraph "データ層"
        TweetDB[(Cassandra ツイート)]
        UserDB[(RDS ユーザー)]
        GraphDB[(Neo4j フォローグラフ)]
        Redis[(Redis キャッシュ)]
    end
    
    subgraph "非同期処理"
        Kafka[Kafka]
        Worker[イベント処理ワーカー]
    end
    
    subgraph "検索"
        SearchIndex[Elasticsearch]
    end
    
    Web --> LB
    Mobile --> LB
    LB --> TweetAPI
    LB --> TimelineAPI
    LB --> UserAPI
    
    TweetAPI --> TweetDB
    TweetAPI --> Kafka
    
    Kafka --> FanoutService
    Kafka --> Worker
    
    FanoutService --> GraphDB
    FanoutService --> TimelineCache
    
    TimelineAPI --> TimelineCache
    TimelineAPI --> TweetDB
    
    UserAPI --> UserDB
    UserAPI --> GraphDB
    UserAPI --> Redis
    
    Worker --> SearchIndex
```

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant TweetAPI as ツイートAPI
    participant TweetDB as Cassandra
    participant Kafka as Kafka
    participant Fanout as Fanoutサービス
    participant GraphDB as Neo4j
    participant Timeline as Redis タイムライン
    participant TimelineAPI as タイムラインAPI
    participant Followers as フォロワー

    Note over User,Followers: ツイート投稿フロー
    
    User->>TweetAPI: POST /tweets<br/>text: "Hello World", media_ids: [...]
    
    TweetAPI->>TweetAPI: ツイートID生成<br/>Snowflake ID
    TweetAPI->>TweetDB: INSERT INTO tweets<br/>tweet_id, user_id, text, created_at
    TweetDB-->>TweetAPI: OK
    
    TweetAPI->>Kafka: Publish<br/>topic: new_tweets, tweet_id, user_id
    TweetAPI-->>User: tweet_id, created_at
    
    Fanout->>Kafka: Subscribe new_tweets
    Kafka-->>Fanout: tweet_id, user_id
    
    Fanout->>GraphDB: MATCH user:User WHERE user.id = user_id<br/>MATCH user<-[:FOLLOWS]-follower<br/>RETURN follower.id
    GraphDB-->>Fanout: フォロワーID一覧 10,000件
    
    alt フォロワー数 < 10,000 通常ユーザー
        loop 各フォロワー
            Fanout->>Timeline: LPUSH timeline:follower_id tweet_id
            Fanout->>Timeline: LTRIM timeline:follower_id 0 799
        end
    else フォロワー数 >= 10,000 セレブリティ
        Fanout->>Fanout: Fanout-on-readに切り替え
        Fanout->>Timeline: SADD celebrity_users user_id
    end
    
    Note over User,Followers: タイムライン取得フロー
    
    User->>TimelineAPI: GET /timeline?cursor=0&limit=20
    
    TimelineAPI->>TimelineAPI: ユーザー情報取得<br/>フォロー中のセレブリティ確認
    
    par Fanout-on-writeタイムライン
        TimelineAPI->>Timeline: LRANGE timeline:user_id 0 19
        Timeline-->>TimelineAPI: ツイートID一覧 20件
    and Fanout-on-readタイムライン
        TimelineAPI->>GraphDB: セレブリティのフォロー確認
        GraphDB-->>TimelineAPI: celebrity_ids: [123, 456]
        TimelineAPI->>TweetDB: SELECT * FROM tweets<br/>WHERE user_id IN (123, 456)<br/>AND created_at > last_24h<br/>LIMIT 20
        TweetDB-->>TimelineAPI: 最新ツイート
    end
    
    TimelineAPI->>TimelineAPI: マージソート<br/>created_at降順
    
    TimelineAPI->>TweetDB: SELECT * FROM tweets<br/>WHERE tweet_id IN (...)
    TweetDB-->>TimelineAPI: ツイート詳細
    
    TimelineAPI->>Timeline: HMGET tweet:tweet_id:stats<br/>likes, retweets, replies
    Timeline-->>TimelineAPI: 統計情報
    
    TimelineAPI-->>User: タイムライン20件, next_cursor: 20
    
    Note over User,Followers: いいねフロー
    
    User->>TweetAPI: POST /tweets/tweet_id/like
    
    TweetAPI->>TweetDB: INSERT INTO likes<br/>user_id, tweet_id, created_at
    TweetAPI->>Timeline: HINCRBY tweet:tweet_id:stats likes 1
    TweetAPI->>Kafka: Publish topic: likes
    
    TweetAPI-->>User: 200 OK
    
    Note over User,Followers: リツイートフロー
    
    User->>TweetAPI: POST /tweets/tweet_id/retweet
    
    TweetAPI->>TweetAPI: 新しいツイートID生成
    TweetAPI->>TweetDB: INSERT INTO tweets<br/>tweet_id, user_id, retweet_of: tweet_id
    TweetAPI->>Timeline: HINCRBY tweet:tweet_id:stats retweets 1
    TweetAPI->>Kafka: Publish topic: new_tweets
    
    TweetAPI-->>User: retweet_id
```

### 設計のポイント

Snowflake IDを使用してツイートIDを生成し、時系列ソートと分散システムでの一意性を両立します。
Fanout-on-writeとFanout-on-readのハイブリッドアプローチを使用します。通常ユーザーはFanout-on-writeで高速なタイムライン取得を実現し、フォロワー数が多いセレブリティはFanout-on-readでストレージコストを削減します。
Cassandraを使用してツイートを時系列で保存し、高速な書き込みと読み取りを実現します。
Neo4jを使用してフォローグラフを管理し、フォロワー取得を高速化します。
Redisでタイムラインをキャッシュし、各ユーザーの最新800ツイートを保持します。
Kafkaで非同期処理を行い、タイムライン生成、検索インデックス更新、通知配信を分離します。
いいね数、リツイート数はRedisで管理し、リアルタイムに更新します。

## SNSリアルタイム検索の設計

### 概要

ツイートをリアルタイムに検索できるシステムを設計します。
全文検索、トレンド検出、ハッシュタグ検索を含みます。

### システム設計図

```mermaid
graph TB
    subgraph "クライアント"
        User[ユーザー]
    end
    
    subgraph "API層"
        SearchAPI[検索API]
        TrendAPI[トレンドAPI]
    end
    
    subgraph "検索エンジン"
        ES1[Elasticsearch<br/>シャード1]
        ES2[Elasticsearch<br/>シャード2]
        ES3[Elasticsearch<br/>シャード3]
    end
    
    subgraph "トレンド分析"
        StreamProcessor[Flink ストリーム処理]
        TrendDB[(Redis トレンドスコア)]
    end
    
    subgraph "データ取り込み"
        Kafka[Kafka ツイートストリーム]
        Indexer[インデクサー]
    end
    
    subgraph "キャッシュ"
        QueryCache[(Redis クエリキャッシュ)]
        PopularCache[(Redis 人気ツイート)]
    end
    
    User --> SearchAPI
    User --> TrendAPI
    
    SearchAPI --> QueryCache
    SearchAPI --> ES1
    SearchAPI --> ES2
    SearchAPI --> ES3
    
    Kafka --> Indexer
    Kafka --> StreamProcessor
    
    Indexer --> ES1
    Indexer --> ES2
    Indexer --> ES3
    
    StreamProcessor --> TrendDB
    TrendAPI --> TrendDB
    TrendAPI --> PopularCache
```

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant SearchAPI as 検索API
    participant Cache as Redis キャッシュ
    participant ES as Elasticsearch
    participant Kafka as Kafka
    participant Indexer as インデクサー
    participant Flink as Flink
    participant TrendDB as Redis トレンド

    Note over User,TrendDB: ツイートインデックス化
    
    Kafka->>Indexer: 新しいツイート
    Indexer->>Indexer: テキスト正規化<br/>トークン化、ストップワード除去
    Indexer->>Indexer: ハッシュタグ抽出<br/>メンション抽出
    
    Indexer->>ES: POST /tweets/_doc/tweet_id<br/>{text, hashtags, user, created_at, ...}
    ES->>ES: 転置インデックス更新
    ES-->>Indexer: indexed
    
    par トレンド分析
        Kafka->>Flink: ツイートストリーム
        Flink->>Flink: 5分間のタンブリングウィンドウ
        Flink->>Flink: ハッシュタグカウント<br/>GROUP BY hashtag
        Flink->>TrendDB: ZINCRBY trends:5min hashtag count
        Flink->>TrendDB: ZINCRBY trends:1hour hashtag count
        Flink->>TrendDB: EXPIRE trends:5min 300
    end
    
    Note over User,TrendDB: リアルタイム検索フロー
    
    User->>SearchAPI: GET /search?q="地震"&limit=20
    
    SearchAPI->>Cache: GET query:"地震"
    
    alt キャッシュヒット 30秒以内
        Cache-->>SearchAPI: キャッシュ結果
        SearchAPI-->>User: ツイート一覧
    else キャッシュミス
        SearchAPI->>ES: GET /tweets/_search<br/>{<br/>  query: {match: {text: "地震"}},<br/>  sort: [{created_at: desc}],<br/>  size: 20<br/>}
        
        ES->>ES: クエリ実行<br/>転置インデックス検索
        ES->>ES: スコアリング<br/>BM25アルゴリズム
        ES->>ES: created_at降順ソート
        ES-->>SearchAPI: 検索結果20件
        
        SearchAPI->>Cache: SETEX query:"地震" 30 結果
        SearchAPI-->>User: ツイート一覧
    end
    
    Note over User,TrendDB: トレンド取得フロー
    
    User->>SearchAPI: GET /trends?location=japan
    
    SearchAPI->>TrendDB: ZREVRANGE trends:1hour 0 9<br/>WITHSCORES
    TrendDB-->>SearchAPI: トップ10ハッシュタグとスコア
    
    SearchAPI->>ES: 各ハッシュタグの代表ツイート取得
    ES-->>SearchAPI: サンプルツイート
    
    SearchAPI-->>User: トレンド一覧<br/>[{hashtag, tweet_count, sample_tweets}]
    
    Note over User,TrendDB: ハッシュタグ検索
    
    User->>SearchAPI: GET /search?q="#紅白歌合戦"
    
    SearchAPI->>ES: GET /tweets/_search<br/>{<br/>  query: {term: {hashtags: "紅白歌合戦"}},<br/>  sort: [{created_at: desc}],<br/>  size: 20<br/>}
    
    ES-->>SearchAPI: ハッシュタグ一致ツイート
    SearchAPI-->>User: ツイート一覧
```

### 設計のポイント

Elasticsearchを使用して全文検索を実装し、秒間数万件のツイートをリアルタイムにインデックス化します。
転置インデックスにより、キーワード検索を高速に実行します。
ハッシュタグは別フィールドとして保存し、完全一致検索を高速化します。
Flinkを使用してストリーム処理を行い、5分/1時間のウィンドウでハッシュタグをカウントしてトレンドを検出します。
Redisのソート済みセットでトレンドスコアを管理し、リアルタイムにランキングを更新します。
検索結果を30秒キャッシュし、同じクエリへの負荷を軽減します。
created_atフィールドで時系列ソートを行い、最新のツイートから表示します。
位置情報を使用して、地域別のトレンドを提供します。

## ファイル同期サービスの設計

### 概要

ファイル同期・共有サービスを設計します。
複数デバイス間のファイル同期、バージョン管理、共有機能を含みます。

### システム設計図

```mermaid
graph TB
    subgraph "クライアント"
        Desktop[デスクトップアプリ]
        Mobile[モバイルアプリ]
        Web[Webブラウザ]
    end
    
    subgraph "API層"
        LB[ロードバランサー]
        SyncAPI[同期API]
        MetadataAPI[メタデータAPI]
    end
    
    subgraph "同期サービス"
        SyncService[同期サービス]
        DiffService[差分検出サービス]
        Queue[同期キュー]
    end
    
    subgraph "ストレージ"
        S3[S3 ブロックストレージ]
        Metadata[(RDS メタデータ)]
    end
    
    subgraph "キャッシュ"
        Redis[(Redis ファイル状態)]
    end
    
    subgraph "通知"
        NotificationService[通知サービス]
        WebSocket[WebSocketサーバー]
    end
    
    Desktop --> LB
    Mobile --> LB
    Web --> LB
    
    LB --> SyncAPI
    LB --> MetadataAPI
    
    SyncAPI --> SyncService
    SyncService --> DiffService
    SyncService --> Queue
    SyncService --> S3
    SyncService --> Metadata
    SyncService --> Redis
    
    MetadataAPI --> Metadata
    MetadataAPI --> Redis
    
    Queue --> NotificationService
    NotificationService --> WebSocket
    WebSocket --> Desktop
    WebSocket --> Mobile
    WebSocket --> Web
```

```mermaid
sequenceDiagram
    participant Desktop as デスクトップ
    participant SyncAPI as 同期API
    participant Diff as 差分検出
    participant S3 as S3ストレージ
    participant Meta as メタデータDB
    participant Redis as Redis
    participant Notify as 通知サービス
    participant Mobile as モバイル

    Note over Desktop,Mobile: ファイルアップロードフロー
    
    Desktop->>Desktop: ファイル変更検知<br/>document.pdf 変更
    Desktop->>Desktop: チャンク分割<br/>4MB x 3チャンク
    Desktop->>Desktop: 各チャンクのハッシュ計算<br/>SHA256
    
    Desktop->>SyncAPI: POST /files/sync<br/>file_path, chunks: [{hash, size}, ...]
    
    SyncAPI->>Diff: CheckChunks<br/>chunk_hashes
    Diff->>S3: HEAD chunk_hash1, chunk_hash2, ...
    
    alt チャンク既存
        S3-->>Diff: chunk_hash1: exists
    else チャンク新規
        S3-->>Diff: chunk_hash2: not found
    end
    
    Diff-->>SyncAPI: missing_chunks: [hash2, hash3]
    SyncAPI-->>Desktop: upload_urls: [presigned_url2, presigned_url3]
    
    par チャンクアップロード
        Desktop->>S3: PUT presigned_url2<br/>chunk2データ
        Desktop->>S3: PUT presigned_url3<br/>chunk3データ
    end
    
    Desktop->>SyncAPI: POST /files/commit<br/>file_path, chunk_refs: [hash1, hash2, hash3]
    
    SyncAPI->>Meta: BEGIN TRANSACTION
    SyncAPI->>Meta: INSERT INTO file_versions<br/>user_id, file_path, version, chunk_refs, size, modified_at
    SyncAPI->>Meta: UPDATE files<br/>SET current_version = version, modified_at = now()
    SyncAPI->>Meta: COMMIT
    
    SyncAPI->>Redis: SET file:user_id:file_path<br/>version, chunk_refs
    SyncAPI->>Redis: PUBLISH sync:user_id<br/>{action: "modified", file_path, version}
    
    SyncAPI-->>Desktop: version, commit_id
    
    Notify->>Redis: SUBSCRIBE sync:user_id
    Redis-->>Notify: file_path modified
    Notify->>Mobile: PUSH /files/file_path modified, version
    
    Note over Desktop,Mobile: ファイル同期フロー モバイル
    
    Mobile->>SyncAPI: GET /files/file_path/metadata
    SyncAPI->>Redis: GET file:user_id:file_path
    Redis-->>SyncAPI: version, chunk_refs
    SyncAPI-->>Mobile: version, chunk_refs: [hash1, hash2, hash3], size
    
    Mobile->>Mobile: ローカルバージョン確認<br/>old_version < new_version
    
    loop 各チャンク
        Mobile->>SyncAPI: GET /chunks/hash1/download-url
        SyncAPI->>S3: GeneratePresignedURL chunk_hash1
        S3-->>SyncAPI: presigned_url
        SyncAPI-->>Mobile: download_url
        
        Mobile->>S3: GET download_url
        S3-->>Mobile: チャンクデータ
        Mobile->>Mobile: チャンク検証 SHA256
    end
    
    Mobile->>Mobile: チャンク結合<br/>document.pdf再構築
    Mobile->>Mobile: ローカル保存
    
    Note over Desktop,Mobile: 共有フォルダ
    
    Desktop->>SyncAPI: POST /folders/folder_id/share<br/>email: "user@example.com", permission: "read"
    
    SyncAPI->>Meta: INSERT INTO shared_folders<br/>folder_id, shared_with_user_id, permission
    SyncAPI->>Notify: NotifyUser user@example.com
    SyncAPI-->>Desktop: shared_folder_id
    
    Note over Desktop,Mobile: バージョン履歴
    
    Desktop->>SyncAPI: GET /files/file_path/versions
    SyncAPI->>Meta: SELECT * FROM file_versions<br/>WHERE user_id = ... AND file_path = ...<br/>ORDER BY version DESC LIMIT 30
    Meta-->>SyncAPI: バージョン一覧
    SyncAPI-->>Desktop: [{version, modified_at, size}, ...]
    
    Desktop->>SyncAPI: POST /files/file_path/restore<br/>version: 5
    SyncAPI->>Meta: UPDATE files SET current_version = 5
    SyncAPI->>Redis: PUBLISH sync:user_id restore
    SyncAPI-->>Desktop: restored
```

### 設計のポイント

ファイルをチャンク単位4MBに分割し、変更されたチャンクのみをアップロードすることで帯域幅を削減します。
チャンクのSHA256ハッシュを使用して重複排除を行い、ストレージコストを削減します。
S3でチャンクを保存し、メタデータDBでファイル構造とチャンクの参照を管理します。
Redisで最新のファイル状態をキャッシュし、メタデータ取得を高速化します。
WebSocketまたはロングポーリングで他デバイスに変更を通知し、リアルタイム同期を実現します。
バージョン履歴を保持し、過去30日間のファイルを復元できるようにします。
共有フォルダ機能で、複数ユーザーでのコラボレーションをサポートします。
Presigned URLを使用して、クライアントから直接S3にアップロード・ダウンロードし、APIサーバーの負荷を軽減します。
