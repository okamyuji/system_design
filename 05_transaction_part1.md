# トランザクション・分散システム設計(Part 1)

## データベーストランザクション

### 概要

トランザクションは、ACID特性(原子性、一貫性、独立性、永続性)を保証する処理の単位です。
複数のデータ操作を1つのまとまりとして扱い、整合性を維持します。

### システム設計図

```mermaid
graph TB
    subgraph "ACID特性"
        Atomicity[Atomicity原子性<br/>全て成功or全て失敗]
        Consistency[Consistency一貫性<br/>制約を維持]
        Isolation[Isolation独立性<br/>他のトランザクションと分離]
        Durability[Durability永続性<br/>コミット後は永続化]
    end
    
    subgraph "分離レベル"
        ReadUncommitted[Read Uncommitted<br/>ダーティリード発生]
        ReadCommitted[Read Committed<br/>コミット済みのみ読取]
        RepeatableRead[Repeatable Read<br/>同一結果保証]
        Serializable[Serializable<br/>完全に直列化]
    end
    
    subgraph "トランザクション制御"
        Begin[BEGIN TRANSACTION]
        Operations[データ操作<br/>INSERT/UPDATE/DELETE]
        Validation[制約チェック]
        Commit[COMMIT]
        Rollback[ROLLBACK]
        
        Begin --> Operations
        Operations --> Validation
        Validation -->|成功| Commit
        Validation -->|失敗| Rollback
    end
```

```mermaid
sequenceDiagram
    participant App as アプリケーション
    participant DB as データベース
    participant Account1 as 口座A
    participant Account2 as 口座B

    Note over App,Account2: 銀行振込トランザクション(1000円)
    
    App->>DB: BEGIN TRANSACTION
    App->>DB: SELECT balance FROM accounts<br/>WHERE id=1 FOR UPDATE
    DB->>Account1: ロック取得
    Account1-->>DB: balance: 5000
    DB-->>App: 5000
    
    App->>App: 残高チェック(5000 >= 1000)
    
    App->>DB: UPDATE accounts<br/>SET balance=balance-1000<br/>WHERE id=1
    DB->>Account1: balance: 4000
    
    App->>DB: UPDATE accounts<br/>SET balance=balance+1000<br/>WHERE id=2
    DB->>Account2: balance: 3000
    
    App->>DB: COMMIT
    DB->>DB: WAL書き込み
    DB->>Account1: ロック解放
    DB-->>App: トランザクション成功
    
    Note over App,Account2: エラーケース(残高不足)
    
    App->>DB: BEGIN TRANSACTION
    App->>DB: SELECT balance FROM accounts<br/>WHERE id=1 FOR UPDATE
    Account1-->>App: balance: 500
    
    App->>App: 残高チェック(500 < 1000)
    App->>DB: ROLLBACK
    DB->>Account1: ロック解放
    DB-->>App: トランザクション中止
```

### 設計のポイント

トランザクションは必要最小限の範囲で使用し、長時間ロックを保持しないようにします。
適切な分離レベルを選択します(多くの場合、Read Committedで十分です)。
デッドロックを避けるため、常に同じ順序でロックを取得します。
トランザクション内でネットワークI/Oや外部APIコールを行わないようにします。
楽観的ロックと悲観的ロックを使い分けます。

## データベースロックを使って排他制御を行う

### 概要

データベースロックにより、複数のトランザクションが同時に同じデータを変更することを防ぎます。
行ロック、テーブルロック、楽観的ロック、悲観的ロックを適切に使い分けます。

### システム設計図

```mermaid
graph TB
    subgraph "悲観的ロック"
        PessBegin[トランザクション開始]
        PessLock[SELECT FOR UPDATE<br/>ロック取得]
        PessUpdate[データ更新]
        PessCommit[コミット&ロック解放]
        
        PessBegin --> PessLock
        PessLock --> PessUpdate
        PessUpdate --> PessCommit
    end
    
    subgraph "楽観的ロック"
        OptRead[データ読み取り<br/>version取得]
        OptProcess[処理実行]
        OptUpdate[UPDATE WHERE version=old_version<br/>SET version=new_version]
        OptCheck{更新成功?}
        OptCommit[コミット]
        OptRetry[リトライ]
        
        OptRead --> OptProcess
        OptProcess --> OptUpdate
        OptUpdate --> OptCheck
        OptCheck -->|成功| OptCommit
        OptCheck -->|失敗| OptRetry
        OptRetry --> OptRead
    end
    
    subgraph "ロックの種類"
        RowLock[行ロック<br/>特定の行のみ]
        TableLock[テーブルロック<br/>テーブル全体]
        IntentionLock[意図ロック<br/>階層的ロック]
    end
```

```mermaid
sequenceDiagram
    participant User1 as ユーザー1
    participant User2 as ユーザー2
    participant DB as データベース

    Note over User1,DB: 悲観的ロック(在庫更新)
    
    User1->>DB: BEGIN TRANSACTION
    User1->>DB: SELECT stock FROM products<br/>WHERE id=1 FOR UPDATE
    DB->>DB: 行ロック取得
    DB-->>User1: stock: 10
    
    User2->>DB: BEGIN TRANSACTION
    User2->>DB: SELECT stock FROM products<br/>WHERE id=1 FOR UPDATE
    Note over User2,DB: ロック待機
    
    User1->>DB: UPDATE products<br/>SET stock=stock-1<br/>WHERE id=1
    User1->>DB: COMMIT
    DB->>DB: ロック解放
    
    DB-->>User2: stock: 9(ロック取得)
    User2->>DB: UPDATE products<br/>SET stock=stock-1<br/>WHERE id=1
    User2->>DB: COMMIT
    
    Note over User1,DB: 楽観的ロック(記事編集)
    
    User1->>DB: SELECT content, version<br/>FROM articles WHERE id=1
    DB-->>User1: content: "text", version: 5
    
    User2->>DB: SELECT content, version<br/>FROM articles WHERE id=1
    DB-->>User2: content: "text", version: 5
    
    User1->>User1: 編集作業
    User1->>DB: UPDATE articles<br/>SET content="new text", version=6<br/>WHERE id=1 AND version=5
    DB-->>User1: 1 row updated
    User1->>DB: COMMIT
    
    User2->>User2: 編集作業
    User2->>DB: UPDATE articles<br/>SET content="other text", version=6<br/>WHERE id=1 AND version=5
    DB-->>User2: 0 rows updated(競合検知)
    User2->>DB: ROLLBACK
    User2-->>User1: 編集競合エラー、再読み込み必要
```

### 設計のポイント

悲観的ロックは、競合が頻繁に発生する場合に使用します(在庫管理等)。
楽観的ロックは、競合が稀な場合に使用します(記事編集等)。
SELECT FOR UPDATEは、更新対象の行のみに使用し、範囲を最小限にします。
デッドロックを検知して自動的にリトライする仕組みを実装します。
ロックのタイムアウト値を設定して、無期限の待機を防ぎます。

## 長時間のトランザクションへ分散共有ロックを活用する

### 概要

長時間にわたるビジネストランザクションでは、データベーストランザクションを長時間保持できません。
分散ロック(Redis、ZooKeeper等)を使用して、アプリケーションレベルでの排他制御を実現します。

### システム設計図

```mermaid
graph TB
    subgraph "分散ロック管理"
        App1[アプリケーション1]
        App2[アプリケーション2]
        App3[アプリケーション3]
        
        RedisCluster[Redisクラスタ<br/>Redlock実装]
        
        App1 -->|ロック取得試行| RedisCluster
        App2 -->|ロック取得試行| RedisCluster
        App3 -->|ロック取得試行| RedisCluster
    end
    
    subgraph "Redlockアルゴリズム"
        Redis1[Redis1]
        Redis2[Redis2]
        Redis3[Redis3]
        Redis4[Redis4]
        Redis5[Redis5]
        
        Client[クライアント]
        
        Client -->|SET lock NX EX| Redis1
        Client -->|SET lock NX EX| Redis2
        Client -->|SET lock NX EX| Redis3
        Client -->|SET lock NX EX| Redis4
        Client -->|SET lock NX EX| Redis5
    end
    
    subgraph "ロックのライフサイクル"
        Acquire[ロック取得]
        Execute[処理実行]
        Extend[ロック延長<br/>必要に応じて]
        Release[ロック解放]
        
        Acquire --> Execute
        Execute --> Extend
        Extend --> Execute
        Execute --> Release
    end
```

```mermaid
sequenceDiagram
    participant App as アプリケーション
    participant Redis1 as Redis1
    participant Redis2 as Redis2
    participant Redis3 as Redis3
    participant DB as データベース

    Note over App,DB: Redlockによる分散ロック取得
    
    App->>App: ロックキー生成<br/>(resource_id, unique_id, ttl: 30s)
    
    par 複数のRedisインスタンスにロック試行
        App->>Redis1: SET lock:resource NX EX 30
        Redis1-->>App: OK(成功)
    and
        App->>Redis2: SET lock:resource NX EX 30
        Redis2-->>App: OK(成功)
    and
        App->>Redis3: SET lock:resource NX EX 30
        Redis3-->>App: OK(成功)
    end
    
    App->>App: 過半数(3/3)取得成功を確認
    App->>App: ロック取得完了
    
    App->>DB: ビジネス処理開始
    
    Note over App,DB: 処理が長引く場合
    
    App->>App: 処理完了まで20秒かかる見込み
    App->>App: ロック延長判定(残り10秒)
    
    par ロック延長
        App->>Redis1: EXPIRE lock:resource 30
        Redis1-->>App: OK
    and
        App->>Redis2: EXPIRE lock:resource 30
        Redis2-->>App: OK
    and
        App->>Redis3: EXPIRE lock:resource 30
        Redis3-->>App: OK
    end
    
    App->>DB: ビジネス処理完了
    
    par ロック解放
        App->>Redis1: DEL lock:resource
    and
        App->>Redis2: DEL lock:resource
    and
        App->>Redis3: DEL lock:resource
    end
    
    Note over App,DB: 別のアプリケーションがロック取得可能に
```

### 設計のポイント

Redlockアルゴリズムは、過半数のRedisインスタンスでロック取得に成功した場合に、ロックが取得できたとみなします。
ロックのTTL(有効期限)は、処理時間より十分長く設定します。
処理中にロックが失効しないよう、必要に応じてロックを延長します。
ロック解放時は、自分が取得したロックのみを解放します(unique_idで確認)。
ロック取得に失敗した場合は、適切なリトライ戦略を実装します。

## CAP定理と整合性モデル

### 概要

CAP定理は、分散システムにおいて、一貫性(Consistency)、可用性(Availability)、分断耐性(Partition tolerance)の3つのうち、2つまでしか同時に満たせないという定理です。
システムの要件に応じて、適切なトレードオフを選択します。

### システム設計図

```mermaid
graph TB
    subgraph "CAP定理"
        CAP["CAP定理<br/>3つのうち2つを選択"]
        
        CP["CP: 一貫性 + 分断耐性<br/>例: HBase, MongoDB, Redis"]
        AP["AP: 可用性 + 分断耐性<br/>例: Cassandra, DynamoDB, Riak"]
        CA["CA: 一貫性 + 可用性<br/>例: RDBMS 単一ノード"]
        
        CAP --> CP
        CAP --> AP
        CAP --> CA
    end
    
    subgraph "整合性モデル"
        Strong["強整合性<br/>Strong Consistency"]
        Eventual["結果整合性<br/>Eventual Consistency"]
        Causal["因果整合性<br/>Causal Consistency"]
        
        Strong -->|読み取り時点で最新| StrongRead["最新データ保証"]
        Eventual -->|最終的に一致| EventualRead["遅延あり"]
        Causal -->|因果関係保持| CausalRead["順序保証"]
    end
    
    subgraph "実装パターン"
        ReadQuorum["Read Quorum<br/>R + W > N"]
        WriteQuorum["Write Quorum<br/>過半数書き込み"]
        VectorClock["Vector Clock<br/>バージョン管理"]
        CRDT["CRDT<br/>競合解決"]
    end
```

```mermaid
sequenceDiagram
    participant Client as クライアント
    participant Node1 as ノード1
    participant Node2 as ノード2
    participant Node3 as ノード3

    Note over Client,Node3: 強整合性(Quorum: W=3, R=2)
    
    Client->>Node1: WRITE(key: "user_123", value: "太郎")
    
    par 全ノードに書き込み
        Node1->>Node1: データ保存
        Node1-->>Client: ACK
    and
        Node1->>Node2: レプリケーション
        Node2->>Node2: データ保存
        Node2-->>Client: ACK
    and
        Node1->>Node3: レプリケーション
        Node3->>Node3: データ保存
        Node3-->>Client: ACK
    end
    
    Client->>Client: 3つのACK確認(W=3)
    Client->>Client: 書き込み成功
    
    Client->>Node1: READ(key: "user_123")
    
    par 複数ノードから読み取り
        Node1-->>Client: version: 5, value: "太郎"
        Node2-->>Client: version: 5, value: "太郎"
    end
    
    Client->>Client: 2つのレスポンス確認(R=2)
    Client->>Client: 最新バージョン選択
    Client->>Client: データ返却: "太郎"
    
    Note over Client,Node3: 結果整合性(Eventual Consistency)
    
    Client->>Node1: WRITE(key: "user_456", value: "花子")
    Node1->>Node1: データ保存
    Node1-->>Client: ACK(即座に返却)
    
    Note over Node1,Node3: バックグラウンドレプリケーション
    
    par 非同期レプリケーション
        Node1->>Node2: レプリケーション(遅延)
        Node1->>Node3: レプリケーション(遅延)
    end
    
    Client->>Node2: READ(key: "user_456")
    Note over Client,Node2: レプリケーション未完了
    Node2-->>Client: nil(古いデータ)
    
    Note over Node1,Node3: レプリケーション完了後
    
    Client->>Node2: READ(key: "user_456")
    Node2-->>Client: value: "花子"(最新データ)
```

### 設計のポイント

金融システムなど、強整合性が必要な場合はCP型を選択します。
SNSやコメントシステムなど、可用性が重要な場合はAP型を選択します。
Quorumベースのシステムでは、W + R > Nとすることで強整合性を保証します。
結果整合性を採用する場合、アプリケーションレベルで整合性の遅延を考慮した設計にします。
競合解決戦略(Last Write Wins、Vector Clock等)を明確に定義します。

## マイクロサービス間のトランザクションの整合性を維持する

### 概要

マイクロサービスアーキテクチャでは、サービス間にまたがるトランザクションを分散トランザクションとして扱います。
Sagaパターンを使用して、結果整合性を保ちながらビジネストランザクションを実現します。

### システム設計図

```mermaid
graph LR
    subgraph "Sagaパターン(Orchestration)"
        Orchestrator[オーケストレーター<br/>トランザクション管理]
        
        Service1[注文サービス]
        Service2[決済サービス]
        Service3[在庫サービス]
        Service4[配送サービス]
        
        Orchestrator -->|1. 注文作成| Service1
        Orchestrator -->|2. 決済処理| Service2
        Orchestrator -->|3. 在庫確保| Service3
        Orchestrator -->|4. 配送手配| Service4
        
        Service1 -.->|補償トランザクション| Cancel1[注文キャンセル]
        Service2 -.->|補償トランザクション| Cancel2[決済取消]
        Service3 -.->|補償トランザクション| Cancel3[在庫解放]
    end
    
    subgraph "Sagaパターン(Choreography)"
        OrderSvc[注文サービス]
        EventBus[イベントバス]
        PaymentSvc[決済サービス]
        InventorySvc[在庫サービス]
        ShippingSvc[配送サービス]
        
        OrderSvc -->|OrderCreated| EventBus
        EventBus --> PaymentSvc
        PaymentSvc -->|PaymentCompleted| EventBus
        EventBus --> InventorySvc
        InventorySvc -->|InventoryReserved| EventBus
        EventBus --> ShippingSvc
    end
```

```mermaid
sequenceDiagram
    participant Client as クライアント
    participant Orch as オーケストレーター
    participant Order as 注文サービス
    participant Payment as 決済サービス
    participant Inventory as 在庫サービス
    participant Shipping as 配送サービス

    Note over Client,Shipping: 成功パターン
    
    Client->>Orch: 注文リクエスト
    Orch->>Orch: Sagaトランザクション開始
    
    Orch->>Order: 注文作成(order_id: 123)
    Order->>Order: 注文データ保存(status: pending)
    Order-->>Orch: 成功
    
    Orch->>Payment: 決済処理(order_id: 123, amount: 10000)
    Payment->>Payment: 決済実行(status: completed)
    Payment-->>Orch: 成功
    
    Orch->>Inventory: 在庫確保(order_id: 123, product_id: 456)
    Inventory->>Inventory: 在庫引当(status: reserved)
    Inventory-->>Orch: 成功
    
    Orch->>Shipping: 配送手配(order_id: 123)
    Shipping->>Shipping: 配送予約(status: scheduled)
    Shipping-->>Orch: 成功
    
    Orch->>Order: 注文確定(status: confirmed)
    Orch-->>Client: 注文完了
    
    Note over Client,Shipping: 失敗パターン(補償トランザクション)
    
    Client->>Orch: 注文リクエスト
    Orch->>Orch: Sagaトランザクション開始
    
    Orch->>Order: 注文作成(order_id: 124)
    Order-->>Orch: 成功
    
    Orch->>Payment: 決済処理(order_id: 124)
    Payment-->>Orch: 成功
    
    Orch->>Inventory: 在庫確保(order_id: 124)
    Inventory-->>Orch: 失敗(在庫不足)
    
    Note over Orch,Shipping: 補償トランザクション開始
    
    Orch->>Payment: 決済取消(order_id: 124)
    Payment->>Payment: 返金処理
    Payment-->>Orch: 取消完了
    
    Orch->>Order: 注文キャンセル(order_id: 124)
    Order->>Order: status: cancelled
    Order-->>Orch: キャンセル完了
    
    Orch-->>Client: 注文失敗(在庫不足)
```

### 設計のポイント

各サービスのローカルトランザクションで処理を完了し、結果整合性を保ちます。
補償トランザクション(undo操作)を必ず実装して、ロールバックを可能にします。
Orchestrationパターンは、中央制御で管理しやすいですが、単一障害点になる可能性があります。
Choreographyパターンは、サービス間の疎結合を保ちますが、フロー全体の把握が難しくなります。
イベントの順序保証が必要な場合は、メッセージキューの順序保証機能を使用します。
