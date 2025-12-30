# スケーラビリティ設計

## 垂直スケーリングと水平スケーリング

### 概要

垂直スケーリング(Scale Up)は、サーバーのスペックを向上させる方法です。
水平スケーリング(Scale Out)は、サーバーの台数を増やす方法です。
それぞれの特性を理解し、システムの要件に応じて選択します。

### システム設計図

```mermaid
graph TB
    subgraph "垂直スケーリング"
        VClient[クライアント]
        VLB[ロードバランサー]
        VServer1[サーバー<br/>CPU: 2 core, RAM: 4GB]
        VServer2[サーバー<br/>CPU: 8 core, RAM: 32GB]
        
        VClient -->|リクエスト| VLB
        VLB -->|スペック増強| VServer1
        VServer1 -.->|アップグレード| VServer2
    end
    
    subgraph "水平スケーリング"
        HClient[クライアント]
        HLB[ロードバランサー]
        HServer1[サーバー1<br/>CPU: 2 core]
        HServer2[サーバー2<br/>CPU: 2 core]
        HServer3[サーバー3<br/>CPU: 2 core]
        HServer4[サーバー4<br/>CPU: 2 core]
        
        HClient -->|リクエスト| HLB
        HLB -->|負荷分散| HServer1
        HLB -->|負荷分散| HServer2
        HLB -.->|台数増加| HServer3
        HLB -.->|台数増加| HServer4
    end
```

### 設計のポイント

垂直スケーリングは、実装がシンプルですが、物理的な上限があります。
水平スケーリングは、理論上無限に拡張できますが、ステート管理が複雑になります。
本番環境では、水平スケーリングを基本とし、必要に応じて垂直スケーリングを組み合わせます。
データベースは垂直スケーリングから始め、必要に応じてシャーディングで水平スケーリングします。

## オートスケール

### 概要

負荷に応じて自動的にサーバーの台数を増減させる仕組みです。
コスト効率を向上させながら、ピーク時のパフォーマンスを維持します。

### システム設計図

```mermaid
graph TB
    subgraph "監視と制御"
        Monitor[メトリクス監視<br/>CloudWatch/Datadog]
        ASG[オートスケールグループ]
        
        Monitor -->|CPU > 70%| ASG
        Monitor -->|CPU < 30%| ASG
    end
    
    subgraph "オートスケール動作"
        LB[ロードバランサー]
        Server1[サーバー1]
        Server2[サーバー2]
        Server3[サーバー3<br/>スケールアウト]
        Server4[サーバー4<br/>スケールイン]
        
        ASG -->|インスタンス追加| Server3
        ASG -->|インスタンス削除| Server4
        
        LB --> Server1
        LB --> Server2
        LB -.->|追加| Server3
        Server4 -.->|削除| LB
    end
    
    subgraph "スケーリングポリシー"
        Policy[ポリシー設定]
        TargetTracking[ターゲット追跡<br/>CPU使用率50%維持]
        StepScaling[ステップスケーリング<br/>負荷に応じて段階的]
        Scheduled[スケジュールスケーリング<br/>時間帯で予測]
        
        Policy --> TargetTracking
        Policy --> StepScaling
        Policy --> Scheduled
    end
```

```mermaid
sequenceDiagram
    participant Monitor as メトリクス監視
    participant ASG as オートスケールグループ
    participant LB as ロードバランサー
    participant New as 新サーバー

    Monitor->>Monitor: CPU使用率75%を検知
    Monitor->>ASG: スケールアウトトリガー
    ASG->>New: 新しいインスタンス起動
    New->>New: アプリケーション初期化
    New->>LB: ヘルスチェック準備完了
    LB->>New: ヘルスチェック実行
    New-->>LB: 正常応答
    LB->>LB: トラフィック振り分け開始
    
    Note over Monitor,New: クールダウン期間(5分)
    
    Monitor->>Monitor: CPU使用率25%を検知
    Monitor->>ASG: スケールイントリガー
    ASG->>LB: 削除対象インスタンス通知
    LB->>LB: 新規接続停止
    LB->>New: 既存接続の完了待機
    ASG->>New: インスタンス終了
```

### 設計のポイント

スケーリングメトリクスは、CPU使用率、メモリ使用率、リクエスト数などを組み合わせます。
クールダウン期間を設定して、頻繁なスケーリングを防ぎます。
最小インスタンス数と最大インスタンス数を適切に設定します。
ヘルスチェックを実装して、正常なインスタンスのみをトラフィック対象にします。
グレースフルシャットダウンを実装して、処理中のリクエストを完了させてから終了します。

## ロードバランサーでリクエストを負荷分散

### 概要

ロードバランサーは、複数のサーバーにリクエストを分散し、システム全体の可用性とパフォーマンスを向上させます。
アルゴリズムの選択により、異なる負荷分散戦略を実現します。

### システム設計図

```mermaid
graph TB
    subgraph "ロードバランサー階層"
        Client[クライアント]
        DNS[DNSラウンドロビン]
        GLB[グローバルロードバランサー]
        
        Client --> DNS
        DNS -->|地理的分散| GLB
    end
    
    subgraph "リージョンA"
        LB_A[ロードバランサーA<br/>Layer 7: ALB]
        Server_A1[Webサーバー1]
        Server_A2[Webサーバー2]
        Server_A3[Webサーバー3]
        
        GLB -->|ヘルスチェック正常| LB_A
        LB_A -->|ラウンドロビン| Server_A1
        LB_A -->|ラウンドロビン| Server_A2
        LB_A -->|ラウンドロビン| Server_A3
    end
    
    subgraph "リージョンB"
        LB_B[ロードバランサーB<br/>Layer 4: NLB]
        Server_B1[Webサーバー4]
        Server_B2[Webサーバー5]
        
        GLB -->|フェイルオーバー| LB_B
        LB_B -->|最小接続数| Server_B1
        LB_B -->|最小接続数| Server_B2
    end
    
    subgraph "負荷分散アルゴリズム"
        Algo[アルゴリズム選択]
        RoundRobin[ラウンドロビン<br/>均等分散]
        LeastConn[最小接続数<br/>負荷が少ないサーバー優先]
        IPHash[IPハッシュ<br/>同一クライアントは同一サーバー]
        WeightedRR[重み付きラウンドロビン<br/>サーバー性能に応じて分散]
        
        Algo --> RoundRobin
        Algo --> LeastConn
        Algo --> IPHash
        Algo --> WeightedRR
    end
```

### 設計のポイント

Layer 7ロードバランサー(ALB)は、HTTPヘッダーやパスベースのルーティングが可能です。
Layer 4ロードバランサー(NLB)は、低レイテンシで高スループットが必要な場合に使用します。
ヘルスチェックを定期的に実行して、異常なサーバーへのルーティングを停止します。
セッション維持が必要な場合は、Sticky Sessionまたは外部セッションストアを使用します。
SSL/TLS終端をロードバランサーで行い、バックエンドサーバーの負荷を軽減します。

## データベースレプリケーションによるクエリの分散

### 概要

データベースのレプリケーションにより、読み取り負荷を複数のレプリカに分散します。
マスタースレーブ構成により、書き込みはマスターに、読み取りはスレーブに振り分けます。

### システム設計図

```mermaid
graph TB
    subgraph "アプリケーション層"
        App1[アプリケーションサーバー1]
        App2[アプリケーションサーバー2]
        App3[アプリケーションサーバー3]
    end
    
    subgraph "データベース層"
        Master[(マスターDB<br/>書き込み専用)]
        Replica1[(レプリカDB1<br/>読み取り専用)]
        Replica2[(レプリカDB2<br/>読み取り専用)]
        Replica3[(レプリカDB3<br/>読み取り専用)]
        
        Master -->|非同期レプリケーション| Replica1
        Master -->|非同期レプリケーション| Replica2
        Master -->|非同期レプリケーション| Replica3
    end
    
    App1 -->|INSERT/UPDATE/DELETE| Master
    App2 -->|INSERT/UPDATE/DELETE| Master
    App3 -->|INSERT/UPDATE/DELETE| Master
    
    App1 -->|SELECT| Replica1
    App2 -->|SELECT| Replica2
    App3 -->|SELECT| Replica3
```

```mermaid
sequenceDiagram
    participant App as アプリケーション
    participant Master as マスターDB
    participant Replica as レプリカDB
    participant Cache as キャッシュ

    App->>Master: UPDATE users SET name='太郎'
    Master->>Master: データ更新
    Master-->>App: 更新成功
    
    Master->>Replica: Binlog転送
    Note over Master,Replica: レプリケーション遅延(数ms~数秒)
    Replica->>Replica: データ反映
    
    App->>Cache: キャッシュ削除
    App->>Replica: SELECT * FROM users
    Replica-->>App: データ返却
    App->>Cache: キャッシュ保存
```

### 設計のポイント

レプリケーション遅延を考慮して、強整合性が必要な読み取りはマスターから行います。
レプリカの台数は、読み取り負荷に応じて調整します。
レプリカの障害時は、他のレプリカに自動的にフェイルオーバーします。
非同期レプリケーションによる遅延を許容できない場合は、準同期レプリケーションを検討します。
読み取り後に即座に更新が必要な場合は、Read-after-Write整合性を保証します。

## パーティショニングによるスケーラビリティの向上

### 概要

データベースのパーティショニング(シャーディング)により、データを複数のサーバーに分散します。
水平分割により、大量のデータと高い書き込み負荷に対応します。

### システム設計図

```mermaid
graph TB
    subgraph "シャーディング戦略"
        App[アプリケーション]
        Router[シャードルーター]
        
        App --> Router
    end
    
    subgraph "レンジベースシャーディング"
        Shard1[(シャード1<br/>user_id: 1-1000000)]
        Shard2[(シャード2<br/>user_id: 1000001-2000000)]
        Shard3[(シャード3<br/>user_id: 2000001-3000000)]
        
        Router -->|user_id <= 1000000| Shard1
        Router -->|1000000 < user_id <= 2000000| Shard2
        Router -->|user_id > 2000000| Shard3
    end
    
    subgraph "ハッシュベースシャーディング"
        HShard1[(シャード1<br/>hash % 3 = 0)]
        HShard2[(シャード2<br/>hash % 3 = 1)]
        HShard3[(シャード3<br/>hash % 3 = 2)]
        
        Router -->|hash user_id % 3| HShard1
        Router -->|hash user_id % 3| HShard2
        Router -->|hash user_id % 3| HShard3
    end
    
    subgraph "ディレクトリベースシャーディング"
        Lookup[(ルックアップテーブル<br/>user_id -> shard_id)]
        DShard1[(シャード1)]
        DShard2[(シャード2)]
        DShard3[(シャード3)]
        
        Router --> Lookup
        Lookup --> DShard1
        Lookup --> DShard2
        Lookup --> DShard3
    end
```

### 設計のポイント

シャードキーは、データの分散が均等になるように選択します。
レンジベースシャーディングは、範囲検索に有利ですが、ホットスポットが発生しやすいです。
ハッシュベースシャーディングは、データが均等に分散されますが、範囲検索ができません。
ディレクトリベースシャーディングは、柔軟性が高いですが、ルックアップテーブルがボトルネックになる可能性があります。
クロスシャードクエリを最小化するようなデータモデルを設計します。

## Consistent Hashing(コンシステントハッシュ法)

### 概要

Consistent Hashingは、ノードの追加や削除時に、再配置が必要なデータ量を最小限に抑える手法です。
分散キャッシュやデータベースのシャーディングで使用されます。

### システム設計図

```mermaid
graph TB
    subgraph "コンシステントハッシュリング"
        Ring[ハッシュリング<br/>0 to 2^32-1]
        Node1[ノード1<br/>hash: 100]
        Node2[ノード2<br/>hash: 500]
        Node3[ノード3<br/>hash: 900]
        VNode1_1[仮想ノード1-1<br/>hash: 150]
        VNode1_2[仮想ノード1-2<br/>hash: 650]
        VNode2_1[仮想ノード2-1<br/>hash: 300]
        VNode2_2[仮想ノード2-2<br/>hash: 800]
        
        Ring --> Node1
        Ring --> Node2
        Ring --> Node3
        Ring --> VNode1_1
        Ring --> VNode1_2
        Ring --> VNode2_1
        Ring --> VNode2_2
    end
    
    subgraph "データ配置"
        Data1[データA<br/>hash: 250]
        Data2[データB<br/>hash: 450]
        Data3[データC<br/>hash: 750]
        
        Data1 -->|時計回りで最初のノード| VNode2_1
        Data2 -->|時計回りで最初のノード| Node2
        Data3 -->|時計回りで最初のノード| VNode2_2
    end
```

```mermaid
sequenceDiagram
    participant Client as クライアント
    participant Router as ルーター
    participant Node1 as ノード1
    participant Node2 as ノード2
    participant Node3 as ノード3(新規)

    Client->>Router: データAを保存(key: "user_123")
    Router->>Router: hash("user_123") = 250
    Router->>Router: リング上で時計回りに検索
    Router->>Node2: データ保存
    
    Note over Node1,Node3: ノード3を追加
    Router->>Router: ノード3をリングに追加
    Router->>Router: 仮想ノードを配置
    
    Client->>Router: データAを取得(key: "user_123")
    Router->>Router: hash("user_123") = 250
    Router->>Router: リング上で時計回りに検索
    alt データ再配置済み
        Router->>Node3: データ取得
    else データ未移行
        Router->>Node2: データ取得
        Router->>Node3: データ移行
    end
```

### 設計のポイント

仮想ノードを使用することで、データの分散をより均等にします。
ノードの追加時は、影響を受けるデータのみを再配置します。
レプリケーション係数を設定して、時計回りに複数のノードにデータを保存します。
ノードの削除時は、そのノードのデータを次のノードに移行します。
ハッシュ関数は、均一な分散を保証する関数(MD5、SHA-1など)を使用します。

## メッセージキューによるスケーラビリティの向上

### 概要

メッセージキューを使用して、同期処理を非同期化し、システムのスケーラビリティを向上させます。
プロデューサーとコンシューマーを分離することで、独立したスケーリングが可能になります。

### システム設計図

```mermaid
graph LR
    subgraph "プロデューサー"
        API1[APIサーバー1]
        API2[APIサーバー2]
        API3[APIサーバー3]
    end
    
    subgraph "メッセージキュー"
        Queue1[キュー: 画像処理]
        Queue2[キュー: メール送信]
        Queue3[キュー: レポート生成]
        DLQ[Dead Letter Queue]
    end
    
    subgraph "コンシューマー"
        Worker1_1[画像処理ワーカー1]
        Worker1_2[画像処理ワーカー2]
        Worker2_1[メール送信ワーカー1]
        Worker3_1[レポート生成ワーカー1]
    end
    
    API1 -->|Enqueue| Queue1
    API2 -->|Enqueue| Queue2
    API3 -->|Enqueue| Queue3
    
    Queue1 -->|Dequeue| Worker1_1
    Queue1 -->|Dequeue| Worker1_2
    Queue2 -->|Dequeue| Worker2_1
    Queue3 -->|Dequeue| Worker3_1
    
    Worker1_1 -.->|処理失敗 3回| DLQ
    Worker2_1 -.->|処理失敗 3回| DLQ
```

```mermaid
sequenceDiagram
    participant API as APIサーバー
    participant Queue as メッセージキュー
    participant Worker1 as ワーカー1
    participant Worker2 as ワーカー2
    participant S3 as S3ストレージ

    API->>Queue: メッセージ送信(画像処理タスク)
    API-->>Client: 即座にレスポンス(タスクID)
    
    par 並列処理
        Queue->>Worker1: メッセージ取得(Visibility Timeout: 30s)
        Worker1->>S3: 画像ダウンロード
        Worker1->>Worker1: 画像処理(リサイズ)
        Worker1->>S3: 処理済み画像アップロード
        Worker1->>Queue: メッセージ削除(ACK)
    and
        Queue->>Worker2: 別のメッセージ取得
        Worker2->>Worker2: 処理実行
        alt 処理成功
            Worker2->>Queue: メッセージ削除(ACK)
        else 処理失敗
            Note over Worker2,Queue: Visibility Timeout経過後に再試行
        end
    end
```

### 設計のポイント

メッセージの可視性タイムアウトは、処理時間より長く設定します。
冪等性を保証して、メッセージの重複処理に対応します。
リトライ回数の上限を設定し、処理に失敗したメッセージはDead Letter Queueに移動します。
優先度付きキューを使用して、重要なタスクを優先的に処理します。
コンシューマーの数を動的に調整して、キューの長さに応じてスケールします。

## ステートフル接続に対してスケーラブルな設計を行う

### 概要

WebSocketやTCP接続などのステートフル接続を扱う場合、セッションの維持とスケーラビリティの両立が必要です。
Sticky SessionやRedisを使用したセッション共有により、水平スケーリングを実現します。

### システム設計図

```mermaid
graph TB
    subgraph "クライアント層"
        Client1[クライアント1]
        Client2[クライアント2]
        Client3[クライアント3]
    end
    
    subgraph "ロードバランサー"
        LB[ロードバランサー<br/>Sticky Session有効]
    end
    
    subgraph "アプリケーション層"
        WS1[WebSocketサーバー1<br/>接続数: 5000]
        WS2[WebSocketサーバー2<br/>接続数: 3000]
        WS3[WebSocketサーバー3<br/>接続数: 4000]
    end
    
    subgraph "セッション管理"
        Redis[(Redis Cluster<br/>セッション共有)]
        SessionDB[(セッションDB<br/>永続化)]
    end
    
    subgraph "メッセージング"
        PubSub[Redis Pub/Sub<br/>サーバー間通信]
    end
    
    Client1 -->|WebSocket接続| LB
    Client2 -->|WebSocket接続| LB
    Client3 -->|WebSocket接続| LB
    
    LB -->|Sticky Session| WS1
    LB -->|Sticky Session| WS2
    LB -->|Sticky Session| WS3
    
    WS1 <-->|セッション読み書き| Redis
    WS2 <-->|セッション読み書き| Redis
    WS3 <-->|セッション読み書き| Redis
    
    Redis -->|定期的に保存| SessionDB
    
    WS1 <-->|メッセージ配信| PubSub
    WS2 <-->|メッセージ配信| PubSub
    WS3 <-->|メッセージ配信| PubSub
```

```mermaid
sequenceDiagram
    participant Client as クライアント
    participant LB as ロードバランサー
    participant WS1 as WebSocketサーバー1
    participant WS2 as WebSocketサーバー2
    participant Redis as Redis
    participant PubSub as Redis Pub/Sub

    Client->>LB: WebSocket接続リクエスト
    LB->>WS1: 接続転送(Sticky Cookie設定)
    WS1->>Redis: セッション作成(user_id, connection_id)
    WS1-->>Client: 接続確立
    
    Note over Client,PubSub: 別のクライアントからメッセージ送信
    
    Client->>WS1: メッセージ送信
    WS1->>Redis: セッション確認
    WS1->>PubSub: Publish(room_id, message)
    
    PubSub->>WS1: Subscribe通知
    PubSub->>WS2: Subscribe通知
    
    WS1->>WS1: ローカル接続にメッセージ配信
    WS2->>Redis: セッション検索(room_id)
    WS2->>WS2: ローカル接続にメッセージ配信
    
    Note over Client,PubSub: サーバー障害時の再接続
    
    WS1->>WS1: サーバー障害
    Client->>LB: 再接続リクエスト
    LB->>WS2: 接続転送(新しいサーバー)
    WS2->>Redis: セッション復元(user_id)
    WS2-->>Client: 接続確立、未配信メッセージ送信
```

### 設計のポイント

Sticky Sessionを使用して、同じクライアントを同じサーバーに接続し続けます。
Redisにセッション情報を保存して、サーバー障害時に別のサーバーでセッションを復元できるようにします。
コネクション数の上限を監視して、新しいサーバーを自動的に追加します。
ハートビートを定期的に送信して、死活監視を行います。
グレースフルシャットダウンを実装して、既存の接続を適切にクローズしてから新しいサーバーに移行します。
