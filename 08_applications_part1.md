# 応用問題 Part 1

システム設計の応用問題として、実際のサービスを題材にした設計を行います。

## 画像アップロードサービスの設計

### 概要

画像のアップロード、保存、配信を効率的に行うサービスを設計します。
画像の変換処理、CDNによる配信、メタデータ管理を含む完全なシステムを構築します。

### システム設計図

```mermaid
graph TB
    subgraph "クライアント"
        Web[Webブラウザ]
        Mobile[モバイルアプリ]
    end
    
    subgraph "API層"
        LB[ロードバランサー]
        API[APIサーバー]
    end
    
    subgraph "ストレージ層"
        S3Original[S3 オリジナル画像]
        S3Processed[S3 処理済み画像]
        CDN[CloudFront CDN]
    end
    
    subgraph "処理層"
        Queue[SQSキュー]
        Worker[画像処理ワーカー]
    end
    
    subgraph "データ層"
        RDS[(RDS メタデータ)]
        Redis[(Redis キャッシュ)]
    end
    
    Web --> LB
    Mobile --> LB
    LB --> API
    API --> S3Original
    API --> Queue
    API --> RDS
    API --> Redis
    
    Queue --> Worker
    Worker --> S3Original
    Worker --> S3Processed
    Worker --> RDS
    
    S3Processed --> CDN
    CDN --> Web
    CDN --> Mobile
```

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant API as APIサーバー
    participant S3 as S3 Original
    participant Queue as SQSキュー
    participant Worker as 画像処理ワーカー
    participant S3Proc as S3 Processed
    participant RDS as データベース
    participant CDN as CloudFront

    Note over User,CDN: 画像アップロードフロー
    
    User->>API: POST /upload/presigned-url<br/>filename: photo.jpg, size: 5MB
    API->>API: ファイル名を生成<br/>uuid-original.jpg
    API->>S3: PutObjectのPresigned URL生成<br/>有効期限: 15分
    API-->>User: presigned_url, upload_id
    
    User->>S3: PUT presigned_url<br/>Content-Type: image/jpeg
    S3-->>User: 200 OK
    
    User->>API: POST /upload/complete<br/>upload_id
    API->>RDS: INSERT INTO images<br/>status: processing
    API->>Queue: Enqueue<br/>job_type: resize, image_id
    API-->>User: image_id, status: processing
    
    Worker->>Queue: Dequeue
    Worker->>S3: GET original image
    S3-->>Worker: 画像データ
    
    Worker->>Worker: サムネイル生成 100x100
    Worker->>Worker: 中サイズ生成 800x800
    Worker->>Worker: 大サイズ生成 1920x1920
    Worker->>Worker: WebP変換
    
    par 複数サイズをアップロード
        Worker->>S3Proc: PUT thumbnail.webp
        Worker->>S3Proc: PUT medium.webp
        Worker->>S3Proc: PUT large.webp
    end
    
    Worker->>RDS: UPDATE images<br/>status: completed, sizes: [...]
    Worker->>Queue: Delete message
    
    Note over User,CDN: 画像取得フロー
    
    User->>CDN: GET /images/uuid/large.webp
    
    alt CDNキャッシュヒット
        CDN-->>User: 画像データ Cache-Control: max-age=31536000
    else CDNキャッシュミス
        CDN->>S3Proc: GET large.webp
        S3Proc-->>CDN: 画像データ
        CDN->>CDN: エッジにキャッシュ
        CDN-->>User: 画像データ
    end
```

### 設計のポイント

Presigned URLを使用して、クライアントから直接S3にアップロードすることで、APIサーバーの負荷を軽減します。
画像処理を非同期で行い、複数のサイズとフォーマットを生成することで、デバイスに最適な画像を配信します。
CloudFront CDNを使用して、世界中のユーザーに低レイテンシで画像を配信します。
WebPフォーマットを使用することで、画像サイズを30-50%削減します。
メタデータをRDSで管理し、画像の検索や分類を可能にします。
オリジナル画像は別バケットで保存し、削除や再処理に備えます。

## リアルタイムチャットの設計

### 概要

数億ユーザーをサポートするリアルタイムチャットシステムを設計します。
WebSocket接続管理、メッセージ配信、既読管理、オフラインメッセージ配信を含みます。

### システム設計図

```mermaid
graph TB
    subgraph "クライアント"
        UserA[ユーザーA]
        UserB[ユーザーB]
    end
    
    subgraph "接続層"
        WSGateway[WebSocketゲートウェイ]
        ConnMgr[接続管理サービス]
        Redis1[(Redis 接続情報)]
    end
    
    subgraph "メッセージ層"
        MsgAPI[メッセージAPIサーバー]
        Kafka[Kafka メッセージストリーム]
        MsgWorker[メッセージ配信ワーカー]
    end
    
    subgraph "データ層"
        Cassandra[(Cassandra メッセージDB)]
        RDS[(RDS ユーザー/チャット)]
        Redis2[(Redis 未読カウント)]
    end
    
    subgraph "通知層"
        PushQueue[プッシュ通知キュー]
        PushWorker[プッシュ通知ワーカー]
        FCM[Firebase Cloud Messaging]
    end
    
    UserA --> WSGateway
    UserB --> WSGateway
    WSGateway --> ConnMgr
    ConnMgr --> Redis1
    
    WSGateway --> MsgAPI
    MsgAPI --> Kafka
    MsgAPI --> RDS
    
    Kafka --> MsgWorker
    MsgWorker --> Cassandra
    MsgWorker --> Redis2
    MsgWorker --> WSGateway
    MsgWorker --> PushQueue
    
    PushQueue --> PushWorker
    PushWorker --> FCM
    FCM --> UserB
```

```mermaid
sequenceDiagram
    participant UserA as ユーザーA
    participant WSA as WebSocketゲートウェイA
    participant ConnMgr as 接続管理
    participant MsgAPI as メッセージAPI
    participant Kafka as Kafka
    participant Worker as メッセージ配信ワーカー
    participant Cassandra as Cassandra
    participant WSB as WebSocketゲートウェイB
    participant UserB as ユーザーB
    participant Push as プッシュ通知

    Note over UserA,Push: 接続確立
    
    UserA->>WSA: WebSocket接続<br/>Authorization: Bearer jwt
    WSA->>WSA: JWT検証
    WSA->>ConnMgr: RegisterConnection<br/>user_id: A, ws_gateway: ws-server-1
    ConnMgr->>ConnMgr: Redis SET user:A:connection<br/>ws-server-1, heartbeat: now
    WSA-->>UserA: Connected
    
    UserB->>WSB: WebSocket接続
    WSB->>ConnMgr: RegisterConnection<br/>user_id: B, ws_gateway: ws-server-2
    ConnMgr->>ConnMgr: Redis SET user:B:connection
    WSB-->>UserB: Connected
    
    Note over UserA,Push: メッセージ送信
    
    UserA->>WSA: SendMessage<br/>to: B, text: "こんにちは", chat_id: 123
    WSA->>MsgAPI: POST /messages<br/>from: A, to: B, text: "こんにちは"
    
    MsgAPI->>MsgAPI: メッセージID生成<br/>msg_id: timestamp-uuid
    MsgAPI->>Kafka: Publish<br/>topic: messages, msg_id, from: A, to: B
    MsgAPI-->>WSA: msg_id, created_at
    WSA-->>UserA: MessageSent msg_id
    
    Worker->>Kafka: Subscribe topic: messages
    Kafka-->>Worker: メッセージデータ
    
    par データ永続化
        Worker->>Cassandra: INSERT INTO messages<br/>chat_id, msg_id, from_user, to_user, text, created_at
    and 未読カウント更新
        Worker->>Worker: Redis INCR chat:123:unread:B
    end
    
    Worker->>ConnMgr: GetUserConnection user_id: B
    ConnMgr->>ConnMgr: Redis GET user:B:connection
    
    alt ユーザーBがオンライン
        ConnMgr-->>Worker: ws-server-2
        Worker->>WSB: SendToUser user_id: B
        WSB->>UserB: NewMessage<br/>from: A, text: "こんにちは", msg_id
        
        UserB->>WSB: MessageReceived msg_id
        WSB->>MsgAPI: POST /messages/msg_id/delivered
        MsgAPI->>Cassandra: UPDATE messages SET delivered_at
        MsgAPI->>WSA: MessageDelivered msg_id
        WSA->>UserA: MessageDelivered msg_id
        
        UserB->>WSB: MessageRead msg_id
        WSB->>MsgAPI: POST /messages/msg_id/read
        MsgAPI->>Cassandra: UPDATE messages SET read_at
        MsgAPI->>Worker: Redis DECR chat:123:unread:B
        MsgAPI->>WSA: MessageRead msg_id
        WSA->>UserA: MessageRead msg_id, read_at
        
    else ユーザーBがオフライン
        ConnMgr-->>Worker: offline
        Worker->>Push: EnqueuePushNotification<br/>user_id: B, title: "ユーザーA", body: "こんにちは"
        Push->>UserB: プッシュ通知
    end
```

### 設計のポイント

WebSocketゲートウェイを水平スケールさせ、数百万の同時接続をサポートします。
接続情報をRedisで管理し、どのゲートウェイサーバーにユーザーが接続しているか追跡します。
Kafkaを使用してメッセージを非同期に処理し、高スループットを実現します。
Cassandraを使用してメッセージを時系列で効率的に保存し、チャット履歴を高速に取得します。
未読カウントをRedisで管理し、リアルタイムに更新します。
オフラインユーザーにはプッシュ通知を送信し、メッセージの見逃しを防ぎます。
メッセージの配信確認と既読確認を実装し、主要チャットアプリと同様のUXを提供します。

## 動画配信ライブコメントサービスの設計

### 概要

リアルタイムで大量のコメントを処理し、視聴者に配信するシステムを設計します。
スパム対策、モデレーション、コメントのランキングを含みます。

### システム設計図

```mermaid
graph TB
    subgraph "視聴者"
        Viewer1[視聴者1]
        Viewer2[視聴者2]
        ViewerN[視聴者N]
    end
    
    subgraph "配信者"
        Streamer[配信者]
        ModPanel[モデレーションパネル]
    end
    
    subgraph "コメント投稿"
        CommentAPI[コメントAPIサーバー]
        SpamFilter[スパム検出サービス]
        ModerationQueue[モデレーションキュー]
    end
    
    subgraph "コメント配信"
        WSGateway[WebSocketゲートウェイ]
        RoomMgr[ルーム管理サービス]
        Redis1[(Redis ルーム情報)]
    end
    
    subgraph "データ層"
        Kafka[Kafka コメントストリーム]
        CommentWorker[コメント処理ワーカー]
        Cassandra[(Cassandra コメントDB)]
        Redis2[(Redis トップコメント)]
    end
    
    Viewer1 --> CommentAPI
    Viewer2 --> CommentAPI
    ViewerN --> CommentAPI
    
    CommentAPI --> SpamFilter
    CommentAPI --> Kafka
    
    Kafka --> CommentWorker
    CommentWorker --> Cassandra
    CommentWorker --> Redis2
    CommentWorker --> WSGateway
    
    SpamFilter --> ModerationQueue
    ModerationQueue --> ModPanel
    ModPanel --> Streamer
    
    WSGateway --> RoomMgr
    RoomMgr --> Redis1
    WSGateway --> Viewer1
    WSGateway --> Viewer2
    WSGateway --> ViewerN
```

```mermaid
sequenceDiagram
    participant Viewer as 視聴者
    participant CommentAPI as コメントAPI
    participant Spam as スパム検出
    participant Kafka as Kafka
    participant Worker as コメント処理ワーカー
    participant Cassandra as Cassandra
    participant Redis as Redis
    participant WSGateway as WebSocketゲートウェイ
    participant Viewers as 視聴者全員

    Note over Viewer,Viewers: コメント投稿フロー
    
    Viewer->>CommentAPI: POST /live/live_id/comments<br/>text: "面白い!", user_id
    
    CommentAPI->>CommentAPI: レート制限チェック<br/>user_id: 10コメント/分
    
    alt レート制限超過
        CommentAPI-->>Viewer: 429 Too Many Requests
    else 制限内
        CommentAPI->>Spam: CheckSpam<br/>text, user_id
        Spam->>Spam: NGワードチェック
        Spam->>Spam: 連投パターン検出
        Spam->>Spam: ML モデルでスパム判定
        
        alt スパム検出
            Spam-->>CommentAPI: is_spam: true
            CommentAPI-->>Viewer: 400 Bad Request
        else クリーン
            Spam-->>CommentAPI: is_spam: false
            
            CommentAPI->>CommentAPI: コメントID生成<br/>comment_id: timestamp-uuid
            CommentAPI->>Kafka: Publish<br/>topic: comments, live_id, comment_id, text, user_id
            CommentAPI-->>Viewer: comment_id, created_at
            
            Worker->>Kafka: Subscribe
            Kafka-->>Worker: コメントデータ
            
            par データ永続化
                Worker->>Cassandra: INSERT INTO comments<br/>live_id, comment_id, text, user_id, created_at
            and いいね数初期化
                Worker->>Redis: SET comment:comment_id:likes 0
            and トップコメント更新
                Worker->>Redis: ZADD live:live_id:top_comments<br/>score: 0, member: comment_id
            end
            
            Worker->>WSGateway: BroadcastToRoom<br/>room: live_id, comment
            WSGateway->>Viewers: NewComment<br/>comment_id, text, user, created_at
        end
    end
    
    Note over Viewer,Viewers: いいねフロー
    
    Viewer->>CommentAPI: POST /comments/comment_id/like
    CommentAPI->>Redis: INCR comment:comment_id:likes
    Redis-->>CommentAPI: 新しいいいね数: 42
    
    CommentAPI->>Redis: ZINCRBY live:live_id:top_comments 1 comment_id
    CommentAPI->>WSGateway: UpdateCommentLikes<br/>comment_id, likes: 42
    WSGateway->>Viewers: CommentLikesUpdated<br/>comment_id, likes: 42
    
    Note over Viewer,Viewers: トップコメント取得
    
    Viewer->>CommentAPI: GET /live/live_id/top-comments
    CommentAPI->>Redis: ZREVRANGE live:live_id:top_comments 0 9
    Redis-->>CommentAPI: トップ10のcomment_id
    CommentAPI->>Cassandra: SELECT * FROM comments<br/>WHERE comment_id IN (...)
    Cassandra-->>CommentAPI: コメント詳細
    CommentAPI-->>Viewer: トップコメント一覧
```

### 設計のポイント

Kafkaを使用してコメントを非同期に処理し、秒間数万件のコメントをサポートします。
WebSocketで視聴者にリアルタイムにコメントを配信します。
スパム検出サービスでNGワード、連投パターン、MLモデルによる判定を行います。
レート制限により、ユーザーごとのコメント投稿頻度を制限します。
Redisのソート済みセットを使用して、いいね数によるトップコメントをリアルタイムに更新します。
Cassandraを使用してコメントを時系列で保存し、配信終了後のアーカイブ視聴でも取得できるようにします。
配信者用のモデレーションパネルで、不適切なコメントを削除できるようにします。

## チケット予約サービスの設計

### 概要

コンサートやイベントのチケット予約システムを設計します。
在庫管理、同時予約の競合解決、決済処理、公平性の担保が重要です。

### システム設計図

```mermaid
graph TB
    subgraph "ユーザー"
        User1[ユーザー1]
        User2[ユーザー2]
        UserN[ユーザーN]
    end
    
    subgraph "待機列"
        Queue[待機列システム]
        Redis1[(Redis 待機列)]
    end
    
    subgraph "予約システム"
        BookingAPI[予約APIサーバー]
        InventoryService[在庫管理サービス]
        Redis2[(Redis 在庫ロック)]
    end
    
    subgraph "決済"
        PaymentAPI[決済APIサーバー]
        PaymentGateway[決済ゲートウェイ]
    end
    
    subgraph "データ層"
        RDS[(RDS マスタ)]
        RDSReplica[(RDS レプリカ)]
        TicketQueue[チケット失効キュー]
    end
    
    User1 --> Queue
    User2 --> Queue
    UserN --> Queue
    
    Queue --> Redis1
    Queue --> BookingAPI
    
    BookingAPI --> InventoryService
    InventoryService --> Redis2
    InventoryService --> RDS
    
    BookingAPI --> PaymentAPI
    PaymentAPI --> PaymentGateway
    PaymentAPI --> RDS
    
    RDS --> RDSReplica
    BookingAPI --> TicketQueue
    TicketQueue --> RDS
```

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant Queue as 待機列
    participant Redis as Redis
    participant BookingAPI as 予約API
    participant Inventory as 在庫管理
    participant RDS as データベース
    participant Payment as 決済API
    participant Gateway as 決済ゲートウェイ
    participant Worker as 失効ワーカー

    Note over User,Worker: 発売開始前の待機列
    
    User->>Queue: GET /events/event_id/queue<br/>10:00:00.123
    Queue->>Redis: ZADD event:event_id:queue<br/>score: 1234567890123, member: user_id
    Queue-->>User: position: 1523, estimated_wait: 5分
    
    loop ポーリング
        User->>Queue: GET /queue/status
        Queue->>Redis: ZRANK event:event_id:queue user_id
        Queue-->>User: position: 823, estimated_wait: 2分
    end
    
    Note over User,Worker: 発売開始 10:00
    
    Queue->>Queue: ZPOPMIN event:event_id:queue 100
    Queue->>User: token: access_token, expires_in: 600秒
    
    Note over User,Worker: チケット予約フロー
    
    User->>BookingAPI: POST /tickets/reserve<br/>event_id, seat_id: A-12, token
    
    BookingAPI->>BookingAPI: トークン検証
    
    BookingAPI->>Inventory: ReserveSeat event_id, seat_id
    
    Inventory->>Redis: SET LOCK:seat:A-12 user_id EX 600 NX
    
    alt ロック取得成功
        Redis-->>Inventory: OK
        
        Inventory->>RDS: BEGIN TRANSACTION
        Inventory->>RDS: SELECT * FROM seats<br/>WHERE seat_id = 'A-12' FOR UPDATE
        RDS-->>Inventory: status: available
        
        Inventory->>RDS: UPDATE seats<br/>SET status = 'reserved', user_id = ..., reserved_at = now()
        Inventory->>RDS: INSERT INTO reservations<br/>reservation_id, user_id, seat_id, expires_at: now() + 10分
        Inventory->>RDS: COMMIT
        
        Inventory-->>BookingAPI: reservation_id, expires_at
        BookingAPI-->>User: reservation_id, expires_at: 10分後
        
        Note over User,Worker: 決済フロー
        
        User->>Payment: POST /reservations/reservation_id/payment<br/>card_token
        
        Payment->>RDS: SELECT * FROM reservations<br/>WHERE reservation_id = ...
        RDS-->>Payment: user_id, seat_id, price, expires_at
        
        alt 予約期限内
            Payment->>Gateway: ChargeCard<br/>amount, card_token
            Gateway-->>Payment: transaction_id, status: success
            
            Payment->>RDS: BEGIN TRANSACTION
            Payment->>RDS: UPDATE reservations<br/>SET status = 'paid', paid_at = now()
            Payment->>RDS: UPDATE seats<br/>SET status = 'sold'
            Payment->>RDS: INSERT INTO tickets<br/>ticket_id, user_id, seat_id, qr_code
            Payment->>RDS: COMMIT
            
            Payment->>Redis: DEL LOCK:seat:A-12
            Payment-->>User: ticket_id, qr_code
            
        else 予約期限切れ
            Payment-->>User: 400 Reservation Expired
        end
        
    else ロック取得失敗
        Redis-->>Inventory: nil
        Inventory-->>BookingAPI: 409 Seat Already Reserved
        BookingAPI-->>User: 座席は既に予約されています
    end
    
    Note over User,Worker: 予約失効処理
    
    Worker->>RDS: SELECT * FROM reservations<br/>WHERE status = 'reserved'<br/>AND expires_at < now()
    RDS-->>Worker: 期限切れ予約一覧
    
    loop 期限切れ予約
        Worker->>RDS: BEGIN TRANSACTION
        Worker->>RDS: UPDATE reservations<br/>SET status = 'expired'
        Worker->>RDS: UPDATE seats<br/>SET status = 'available', user_id = null
        Worker->>RDS: COMMIT
        Worker->>Redis: DEL LOCK:seat:...
    end
```

### 設計のポイント

待機列システムを実装し、アクセス集中時にサーバー負荷を制御します。Redisのソート済みセットで先着順を公平に管理します。
分散ロックRedisとデータベーストランザクションの二重チェックで、同時予約の競合を防ぎます。
予約に10分の期限を設け、決済されない場合は自動的に座席を開放します。
ワーカープロセスで定期的に期限切れ予約を検出し、座席を開放します。
決済処理は冪等性を担保し、重複課金を防ぎます。
チケットにQRコードを生成し、入場時の本人確認に使用します。
レプリカDBを使用して、在庫確認などの読み取り処理をスケールします。
