# トランザクション・分散システム設計(Part 2)

## メッセージキューとデータベースの書き込みを同一のトランザクション内で実行する

### 概要

データベースへの書き込みとメッセージキューへの発行を同時に行う場合、両方が成功または両方が失敗することを保証する必要があります。
Outboxパターンを使用して、トランザクショナルメッセージングを実現します。

### システム設計図

```mermaid
graph TB
    subgraph "Outboxパターン"
        App[アプリケーション]
        DB[(データベース)]
        OutboxTable[(Outboxテーブル)]
        Relay[メッセージリレー<br/>CDC/ポーリング]
        MessageQueue[メッセージキュー]
        
        App -->|1. トランザクション開始| DB
        App -->|2. ビジネスデータ書き込み| DB
        App -->|3. Outboxレコード挿入| OutboxTable
        App -->|4. コミット| DB
        
        OutboxTable -->|5. 変更検知| Relay
        Relay -->|6. メッセージ発行| MessageQueue
        Relay -->|7. 発行済みフラグ更新| OutboxTable
    end
    
    subgraph "CDC(Change Data Capture)"
        DBLog[データベースログ<br/>Binlog/WAL]
        Debezium[Debezium<br/>CDC コネクター]
        Kafka[Kafkaトピック]
        
        DBLog --> Debezium
        Debezium --> Kafka
    end
    
    subgraph "ポーリング方式"
        Scheduler[スケジューラー<br/>定期実行]
        SelectOutbox[SELECT * FROM outbox<br/>WHERE published=false]
        Publish[メッセージ発行]
        UpdateOutbox[UPDATE outbox<br/>SET published=true]
        
        Scheduler --> SelectOutbox
        SelectOutbox --> Publish
        Publish --> UpdateOutbox
    end
```

```mermaid
sequenceDiagram
    participant App as アプリケーション
    participant DB as データベース
    participant Outbox as Outboxテーブル
    participant Relay as メッセージリレー
    participant Queue as メッセージキュー
    participant Consumer as コンシューマー

    Note over App,Consumer: Outboxパターンによるトランザクショナルメッセージング
    
    App->>DB: BEGIN TRANSACTION
    
    App->>DB: INSERT INTO orders<br/>VALUES(123, 'user_456', 10000)
    DB-->>App: 成功
    
    App->>Outbox: INSERT INTO outbox<br/>VALUES(uuid, 'OrderCreated', payload, false)
    Outbox-->>App: 成功
    
    App->>DB: COMMIT
    DB-->>App: トランザクション完了
    
    Note over Relay,Queue: バックグラウンド処理
    
    Relay->>Outbox: SELECT * FROM outbox<br/>WHERE published=false<br/>ORDER BY created_at
    Outbox-->>Relay: [event1, event2, ...]
    
    loop 未発行イベント
        Relay->>Queue: Publish(event)
        Queue-->>Relay: ACK
        
        Relay->>Outbox: UPDATE outbox<br/>SET published=true<br/>WHERE id=event.id
        Outbox-->>Relay: 更新成功
    end
    
    Queue->>Consumer: イベント配信(OrderCreated)
    Consumer->>Consumer: 処理実行
    Consumer-->>Queue: ACK
    
    Note over App,Consumer: 障害時の動作
    
    App->>DB: BEGIN TRANSACTION
    App->>DB: INSERT INTO orders
    App->>Outbox: INSERT INTO outbox
    App->>DB: COMMIT失敗(ネットワークエラー)
    
    Note over App,Outbox: ロールバック(両方とも書き込まれない)
    
    App->>App: リトライ
    App->>DB: BEGIN TRANSACTION
    App->>DB: INSERT INTO orders
    App->>Outbox: INSERT INTO outbox
    App->>DB: COMMIT成功
    
    Relay->>Outbox: SELECT未発行イベント
    Relay->>Queue: Publish
    Queue->>Consumer: イベント配信
```

### 設計のポイント

Outboxテーブルとビジネステーブルを同一トランザクションで更新することで、原子性を保証します。
メッセージリレーは、CDCまたはポーリングでOutboxテーブルを監視します。
CDCは低レイテンシですが、データベースの変更ログに依存します。
ポーリングはシンプルですが、レイテンシが高くなります。
メッセージの重複配信を考慮して、コンシューマー側で冪等性を保証します。
Outboxテーブルの古いレコードは定期的に削除します。

## API Gatewayでマイクロサービスの入口を作る

### 概要

API Gatewayは、クライアントとマイクロサービス群の間に配置され、ルーティング、認証、レート制限、ロギングなどを一元管理します。
バックエンドサービスの複雑さをクライアントから隠蔽します。

### システム設計図

```mermaid
graph TB
    subgraph "クライアント"
        Web[Webアプリ]
        Mobile[モバイルアプリ]
        Partner[パートナーAPI]
    end
    
    subgraph "API Gateway"
        Gateway[API Gateway<br/>Kong/AWS API Gateway]
        
        Auth[認証・認可<br/>JWT検証]
        RateLimit[レート制限<br/>リクエスト制限]
        Transform[リクエスト変換<br/>レスポンス変換]
        Cache[レスポンスキャッシュ<br/>Redis]
        Logging[ロギング・監視<br/>CloudWatch]
        Circuit[サーキットブレーカー<br/>障害対策]
        
        Gateway --> Auth
        Gateway --> RateLimit
        Gateway --> Transform
        Gateway --> Cache
        Gateway --> Logging
        Gateway --> Circuit
    end
    
    subgraph "マイクロサービス"
        UserService[ユーザーサービス<br/>/api/users]
        OrderService[注文サービス<br/>/api/orders]
        ProductService[商品サービス<br/>/api/products]
        PaymentService[決済サービス<br/>/api/payments]
        NotificationService[通知サービス<br/>/api/notifications]
    end
    
    Web --> Gateway
    Mobile --> Gateway
    Partner --> Gateway
    
    Gateway -->|ルーティング| UserService
    Gateway -->|ルーティング| OrderService
    Gateway -->|ルーティング| ProductService
    Gateway -->|ルーティング| PaymentService
    Gateway -->|ルーティング| NotificationService
```

```mermaid
sequenceDiagram
    participant Client as クライアント
    participant Gateway as API Gateway
    participant Auth as 認証サービス
    participant Service as マイクロサービス
    participant Cache as キャッシュ

    Client->>Gateway: GET /api/products/123<br/>Authorization: Bearer token
    
    Gateway->>Gateway: リクエストログ記録
    
    Gateway->>Gateway: レート制限チェック<br/>(100 req/min)
    alt レート制限超過
        Gateway-->>Client: 429 Too Many Requests
    end
    
    Gateway->>Auth: トークン検証
    Auth-->>Gateway: ユーザー情報(user_id: 456)
    
    Gateway->>Cache: GET cache:products:123
    alt キャッシュヒット
        Cache-->>Gateway: 商品データ
        Gateway-->>Client: 200 OK(X-Cache: HIT)
    else キャッシュミス
        Gateway->>Service: GET /products/123<br/>X-User-ID: 456
        Service-->>Gateway: 商品データ
        Gateway->>Cache: SET cache:products:123(TTL: 300s)
        Gateway-->>Client: 200 OK(X-Cache: MISS)
    end
    
    Gateway->>Gateway: レスポンスログ記録
    
    Note over Client,Cache: サービス障害時の動作
    
    Client->>Gateway: GET /api/orders/789
    Gateway->>Service: GET /orders/789
    Service--xGateway: タイムアウト
    
    Gateway->>Gateway: サーキットブレーカー検知
    Gateway->>Gateway: フォールバックレスポンス
    Gateway-->>Client: 503 Service Unavailable<br/>{"error": "Service temporarily unavailable"}
```

### 設計のポイント

API Gatewayは水平スケールして、単一障害点にならないようにします。
認証トークンの検証をGatewayで行い、バックエンドサービスの負荷を軽減します。
GraphQL Gatewayを使用することで、複数のマイクロサービスからデータを集約できます。
BFF(Backend for Frontend)パターンを適用して、クライアントごとに最適化されたAPIを提供します。
API GatewayとService Meshを組み合わせることで、包括的な制御を実現します。

## コーディネーションサービスを活用した分散システムの管理

### 概要

ZooKeeperやetcdなどのコーディネーションサービスを使用して、分散システムの設定管理、サービスディスカバリ、リーダー選出を行います。
強整合性を保ちながら、複数のノード間で情報を共有します。

### システム設計図

```mermaid
graph TB
    subgraph "コーディネーションサービス"
        ZK1[ZooKeeper1<br/>Leader]
        ZK2[ZooKeeper2<br/>Follower]
        ZK3[ZooKeeper3<br/>Follower]
        
        ZK1 <-.->|Quorum| ZK2
        ZK2 <-.->|Quorum| ZK3
        ZK3 <-.->|Quorum| ZK1
    end
    
    subgraph "ユースケース1: サービスディスカバリ"
        Service1[サービス1]
        Service2[サービス2]
        Service3[サービス3]
        Client[クライアント]
        
        Service1 -.->|登録| ZK1
        Service2 -.->|登録| ZK1
        Service3 -.->|登録| ZK1
        Client -.->|検索| ZK1
    end
    
    subgraph "ユースケース2: リーダー選出"
        Node1[ノード1<br/>候補]
        Node2[ノード2<br/>候補]
        Node3[ノード3<br/>候補]
        Leader[リーダー<br/>選出されたノード]
        
        Node1 -.->|選出参加| ZK1
        Node2 -.->|選出参加| ZK1
        Node3 -.->|選出参加| ZK1
        ZK1 -.->|リーダー通知| Leader
    end
    
    subgraph "ユースケース3: 分散ロック"
        App1[アプリ1]
        App2[アプリ2]
        App3[アプリ3]
        
        App1 -.->|ロック取得| ZK1
        App2 -.->|ロック待機| ZK1
        App3 -.->|ロック待機| ZK1
    end
```

```mermaid
sequenceDiagram
    participant Node1 as ノード1
    participant Node2 as ノード2
    participant Node3 as ノード3
    participant ZK as ZooKeeper
    participant Watch as Watch通知

    Note over Node1,Watch: リーダー選出プロセス
    
    par 全ノードが選出に参加
        Node1->>ZK: create /election/node1<br/>EPHEMERAL_SEQUENTIAL
        ZK-->>Node1: /election/0000000001
    and
        Node2->>ZK: create /election/node2<br/>EPHEMERAL_SEQUENTIAL
        ZK-->>Node2: /election/0000000002
    and
        Node3->>ZK: create /election/node3<br/>EPHEMERAL_SEQUENTIAL
        ZK-->>Node3: /election/0000000003
    end
    
    par 全ノードが子ノードリスト取得
        Node1->>ZK: getChildren /election
        ZK-->>Node1: [0000000001, 0000000002, 0000000003]
        Node1->>Node1: 自分が最小番号<br/>リーダーとして動作開始
    and
        Node2->>ZK: getChildren /election
        ZK-->>Node2: [0000000001, 0000000002, 0000000003]
        Node2->>Node2: 0000000001をWatch
        Node2->>ZK: exists /election/0000000001 WATCH
    and
        Node3->>ZK: getChildren /election
        ZK-->>Node3: [0000000001, 0000000002, 0000000003]
        Node3->>Node3: 0000000001をWatch
        Node3->>ZK: exists /election/0000000001 WATCH
    end
    
    Note over Node1,Watch: リーダー障害発生
    
    Node1->>Node1: クラッシュ
    ZK->>ZK: /election/0000000001削除<br/>(EPHEMERAL)
    
    ZK->>Watch: Watch通知
    Watch->>Node2: NodeDeleted通知
    Watch->>Node3: NodeDeleted通知
    
    Node2->>ZK: getChildren /election
    ZK-->>Node2: [0000000002, 0000000003]
    Node2->>Node2: 自分が最小番号<br/>新リーダーとして動作開始
    
    Node3->>ZK: getChildren /election
    ZK-->>Node3: [0000000002, 0000000003]
    Node3->>Node3: 0000000002をWatch
    Node3->>ZK: exists /election/0000000002 WATCH
```

### 設計のポイント

ZooKeeperは、ZAB(ZooKeeper Atomic Broadcast)プロトコルにより、強整合性を保証します。
Ephemeralノードを使用することで、クライアントのセッション切断時に自動的にノードが削除されます。
Watchメカニズムにより、データ変更を即座に検知できます。
Quorumベースの動作により、過半数のノードが生存していれば動作し続けます。
リーダー選出は、シーケンシャルノードの番号で判定し、公平性を保ちます。

## 分散システムでユニークIDを設計する

### 概要

分散システムでは、複数のサーバーで同時にIDを生成する必要があります。
Snowflake、UUID、データベースシーケンス、Redisカウンターなど、様々な手法を理解し、要件に応じて選択します。

### システム設計図

```mermaid
graph TB
    subgraph "Snowflake ID(64bit)"
        Timestamp[41bit: タイムスタンプ<br/>ミリ秒単位]
        DataCenter[5bit: データセンターID<br/>最大32個]
        Worker[5bit: ワーカーID<br/>最大32個]
        Sequence[13bit: シーケンス番号<br/>ミリ秒あたり8192個]
        
        Timestamp --- DataCenter
        DataCenter --- Worker
        Worker --- Sequence
    end
    
    subgraph "UUID v4(128bit)"
        Random[122bit: ランダム]
        Version[4bit: バージョン]
        Variant[2bit: バリアント]
    end
    
    subgraph "データベースシーケンス"
        DB1[(DB1: 1,3,5,7...)]
        DB2[(DB2: 2,4,6,8...)]
        AutoIncrement[AUTO_INCREMENT<br/>オフセット設定]
        
        DB1 --- AutoIncrement
        DB2 --- AutoIncrement
    end
    
    subgraph "Redisカウンター"
        RedisIncr[INCR counter<br/>アトミック操作]
        Prefix[プレフィックス付与<br/>日付-カウンター]
    end
```

```mermaid
sequenceDiagram
    participant App1 as アプリ1(DC1-Worker1)
    participant App2 as アプリ2(DC1-Worker2)
    participant App3 as アプリ3(DC2-Worker1)
    participant Time as タイムスタンプ

    Note over App1,Time: Snowflake ID生成
    
    App1->>Time: 現在時刻取得
    Time-->>App1: 1704067200000(ms)
    
    App1->>App1: タイムスタンプ: 1704067200000<br/>データセンターID: 1<br/>ワーカーID: 1<br/>シーケンス: 0
    App1->>App1: ID生成<br/>0001100111...0001 0001 0000000000000
    App1->>App1: ID: 7123456789012345678
    
    par 同一ミリ秒内の複数生成
        App1->>App1: シーケンス: 1<br/>ID: 7123456789012345679
        App1->>App1: シーケンス: 2<br/>ID: 7123456789012345680
    end
    
    Note over App1,Time: 異なるワーカーでの生成
    
    App2->>Time: 現在時刻取得
    Time-->>App2: 1704067200000(ms)
    
    App2->>App2: タイムスタンプ: 1704067200000<br/>データセンターID: 1<br/>ワーカーID: 2<br/>シーケンス: 0
    App2->>App2: ID: 7123456789012345681
    
    App3->>Time: 現在時刻取得
    Time-->>App3: 1704067200000(ms)
    
    App3->>App3: タイムスタンプ: 1704067200000<br/>データセンターID: 2<br/>ワーカーID: 1<br/>シーケンス: 0
    App3->>App3: ID: 7123456789012345682
    
    Note over App1,Time: 時刻が戻った場合の対処
    
    App1->>Time: 現在時刻取得
    Time-->>App1: 1704067199999(過去の時刻)
    
    App1->>App1: 時刻後退を検知
    App1->>App1: エラー発生またはスリープ待機
```

```mermaid
graph LR
    subgraph "ID生成手法の比較"
        Snowflake[Snowflake<br/>時系列ソート可能<br/>コンパクト64bit]
        UUID[UUID v4<br/>完全分散<br/>128bit長い]
        DBSeq[DBシーケンス<br/>シンプル<br/>DB依存]
        Redis[Redisカウンター<br/>高速<br/>Redis依存]
        ULID[ULID<br/>時系列ソート可能<br/>文字列表現]
    end
```

### 設計のポイント

Snowflake IDは、時系列でソート可能で、64bitとコンパクトです。
UUIDは、完全に分散して生成できますが、128bitと長く、ランダムなためインデックス効率が悪いです。
データベースシーケンスは、シンプルですが、データベースがボトルネックになります。
Redisカウンターは、高速ですが、Redisへの依存とネットワークレイテンシが課題です。
時刻同期の問題を考慮し、NTPで時刻を正確に保ちます。
ワーカーIDの管理は、環境変数や設定ファイルで行います。
