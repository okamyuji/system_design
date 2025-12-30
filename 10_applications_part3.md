# 応用問題 Part 3

システム設計の応用問題として、競技プログラミング、求人サービス、分析サービスの設計を行います。

## 競技プログラミングコンテストの設計

### 概要

競技プログラミングコンテストのプラットフォームを設計します。
コード提出、ジャッジシステム、リアルタイム順位表、同時アクセス対策を含みます。

### システム設計図

```mermaid
graph TB
    subgraph "参加者"
        User1[参加者1]
        User2[参加者2]
        UserN[参加者N]
    end
    
    subgraph "API層"
        LB[ロードバランサー]
        ContestAPI[コンテストAPI]
        SubmitAPI[提出API]
    end
    
    subgraph "ジャッジシステム"
        Queue[ジャッジキュー]
        Judge1[ジャッジサーバー1]
        Judge2[ジャッジサーバー2]
        JudgeN[ジャッジサーバーN]
        Sandbox[Dockerサンドボックス]
    end
    
    subgraph "データ層"
        RDS[(RDS マスタ)]
        Replica[(RDS レプリカ)]
        S3[S3 コード保存]
    end
    
    subgraph "順位表"
        Standings[順位表サービス]
        Redis[(Redis 順位キャッシュ)]
    end
    
    User1 --> LB
    User2 --> LB
    UserN --> LB
    
    LB --> ContestAPI
    LB --> SubmitAPI
    
    SubmitAPI --> Queue
    SubmitAPI --> S3
    SubmitAPI --> RDS
    
    Queue --> Judge1
    Queue --> Judge2
    Queue --> JudgeN
    
    Judge1 --> Sandbox
    Judge2 --> Sandbox
    JudgeN --> Sandbox
    
    Judge1 --> RDS
    Judge2 --> RDS
    JudgeN --> RDS
    
    ContestAPI --> Standings
    Standings --> Redis
    Standings --> Replica
```

```mermaid
sequenceDiagram
    participant User as 参加者
    participant SubmitAPI as 提出API
    participant S3 as S3
    participant Queue as ジャッジキュー
    participant Judge as ジャッジサーバー
    participant Sandbox as Dockerサンドボックス
    participant RDS as データベース
    participant Standings as 順位表サービス
    participant Redis as Redis

    Note over User,Redis: コード提出フロー
    
    User->>SubmitAPI: POST /contests/contest_id/submit<br/>problem_id: A, language: cpp, code: "..."
    
    SubmitAPI->>SubmitAPI: 提出ID生成<br/>submission_id
    SubmitAPI->>SubmitAPI: 提出制限チェック<br/>30秒間隔制限
    
    alt 制限内
        SubmitAPI->>S3: PUT submissions/submission_id.cpp<br/>コード保存
        SubmitAPI->>RDS: INSERT INTO submissions<br/>submission_id, user_id, problem_id, language, status: WJ
        SubmitAPI->>Queue: Enqueue<br/>submission_id, priority: contest開催中は高
        SubmitAPI-->>User: submission_id, status: WJ
        
        Note over User,Redis: ジャッジ実行
        
        Judge->>Queue: Dequeue
        Queue-->>Judge: submission_id
        
        Judge->>S3: GET submissions/submission_id.cpp
        S3-->>Judge: コード
        
        Judge->>RDS: SELECT * FROM problems WHERE problem_id = A
        RDS-->>Judge: テストケース情報<br/>test_cases: 20件, time_limit: 2秒, memory_limit: 1024MB
        
        Judge->>Sandbox: Docker run --cpus=1 --memory=1024m<br/>--network=none --read-only
        Sandbox->>Sandbox: コンパイル g++ -O2 code.cpp
        
        alt コンパイルエラー
            Sandbox-->>Judge: CE Compilation Error
            Judge->>RDS: UPDATE submissions SET status = CE, message = "..."
            Judge-->>User: WebSocket: status: CE
        else コンパイル成功
            loop 各テストケース
                Judge->>Sandbox: 標準入力: test01.txt
                Sandbox->>Sandbox: プログラム実行<br/>タイムアウト: 2秒
                
                alt 時間制限超過
                    Sandbox-->>Judge: TLE
                    Judge->>Judge: テストケース結果: TLE
                else メモリ制限超過
                    Sandbox-->>Judge: MLE
                    Judge->>Judge: テストケース結果: MLE
                else ランタイムエラー
                    Sandbox-->>Judge: RE
                    Judge->>Judge: テストケース結果: RE
                else 正常終了
                    Sandbox-->>Judge: 標準出力, 実行時間: 0.5秒
                    Judge->>Judge: 出力比較<br/>expected vs actual
                    
                    alt 出力一致
                        Judge->>Judge: テストケース結果: AC
                    else 出力不一致
                        Judge->>Judge: テストケース結果: WA
                    end
                end
            end
            
            Judge->>Judge: 全体結果判定<br/>すべてACならAC、それ以外は最初のエラー
            
            alt すべてAC
                Judge->>RDS: UPDATE submissions<br/>SET status = AC, exec_time = max_time, score = 100
                Judge->>Standings: UpdateStandings<br/>user_id, problem_id, AC
            else 一部エラー
                Judge->>RDS: UPDATE submissions<br/>SET status = WA, failed_case = 5
            end
        end
        
        Judge->>Judge: Sandbox削除
        Judge-->>User: WebSocket: 最終結果
        
    else 制限超過
        SubmitAPI-->>User: 429 Too Many Requests
    end
    
    Note over User,Redis: 順位表取得フロー
    
    User->>SubmitAPI: GET /contests/contest_id/standings
    
    SubmitAPI->>Standings: GetStandings
    
    Standings->>Redis: GET standings:contest_id
    
    alt キャッシュヒット 10秒以内
        Redis-->>Standings: 順位表データ
        Standings-->>User: 順位表
    else キャッシュミス
        Standings->>RDS: SELECT user_id, problem_id, status, submitted_at<br/>FROM submissions<br/>WHERE contest_id = ...<br/>ORDER BY submitted_at
        
        RDS-->>Standings: 全提出データ
        
        Standings->>Standings: 順位計算<br/>解いた問題数、ペナルティ時間
        Standings->>Standings: ソート<br/>問題数降順、ペナルティ昇順
        
        Standings->>Redis: SETEX standings:contest_id 10 順位表
        Standings-->>User: 順位表
    end
    
    Note over User,Redis: リアルタイム更新
    
    loop コンテスト中
        Judge->>Standings: AC通知
        Standings->>Redis: 順位表無効化<br/>DEL standings:contest_id
        Standings->>User: WebSocket: 順位表更新通知
    end
```

### 設計のポイント

提出をキューに入れて非同期にジャッジすることで、同時大量提出に対応します。
Dockerコンテナでサンドボックス環境を構築し、セキュリティとリソース制限を実現します。
ネットワークを無効化し、読み取り専用ファイルシステムで悪意あるコードを防ぎます。
CPU、メモリ、実行時間の制限をDockerで設定します。
テストケースごとに実行し、最初のエラーで判定を終了することで効率化します。
順位表を10秒キャッシュし、頻繁なアクセスによるDB負荷を軽減します。
提出に30秒の間隔制限を設け、スパム提出を防ぎます。
WebSocketでリアルタイムにジャッジ結果と順位表更新を通知します。
コンテスト開催中の提出は優先度を上げてキューに入れます。

## ビジネスSNS求人アラートサービスの設計

### 概要

ユーザーの希望条件に合った求人を自動的に通知するシステムを設計します。
求人マッチング、バッチ処理、メール配信、レコメンデーションを含みます。

### システム設計図

```mermaid
graph TB
    subgraph "ユーザー"
        JobSeeker[求職者]
        Company[企業]
    end
    
    subgraph "求人投稿"
        JobAPI[求人API]
        JobDB[(RDS 求人DB)]
    end
    
    subgraph "アラート設定"
        AlertAPI[アラートAPI]
        AlertDB[(RDS アラート設定)]
    end
    
    subgraph "マッチング処理"
        Scheduler[スケジューラー]
        MatchWorker[マッチングワーカー]
        ES[Elasticsearch]
    end
    
    subgraph "通知配信"
        NotificationQueue[通知キュー]
        EmailWorker[メール送信ワーカー]
        SES[Amazon SES]
    end
    
    subgraph "レコメンデーション"
        MLModel[MLモデル]
        FeatureStore[(特徴量ストア)]
    end
    
    Company --> JobAPI
    JobAPI --> JobDB
    JobAPI --> ES
    
    JobSeeker --> AlertAPI
    AlertAPI --> AlertDB
    
    Scheduler --> MatchWorker
    MatchWorker --> ES
    MatchWorker --> AlertDB
    MatchWorker --> MLModel
    
    MLModel --> FeatureStore
    
    MatchWorker --> NotificationQueue
    NotificationQueue --> EmailWorker
    EmailWorker --> SES
    SES --> JobSeeker
```

```mermaid
sequenceDiagram
    participant User as 求職者
    participant AlertAPI as アラートAPI
    participant AlertDB as アラート設定DB
    participant Scheduler as スケジューラー
    participant Matcher as マッチングワーカー
    participant ES as Elasticsearch
    participant ML as MLモデル
    participant Queue as 通知キュー
    participant Email as メール送信ワーカー
    participant SES as Amazon SES

    Note over User,SES: アラート設定フロー
    
    User->>AlertAPI: POST /alerts<br/>{keywords: ["Python", "機械学習"],<br/> location: "東京", experience: "3-5年",<br/> frequency: "daily"}
    
    AlertAPI->>AlertDB: INSERT INTO job_alerts<br/>user_id, keywords, location, experience, frequency, active: true
    AlertAPI-->>User: alert_id
    
    Note over User,SES: 求人投稿とインデックス化
    
    Note over Scheduler,SES: 企業が求人を投稿
    
    AlertAPI->>ES: POST /jobs/_doc/job_id<br/>{title, description, keywords, location, experience, salary, posted_at}
    
    Note over User,SES: バッチマッチング処理 毎日6:00
    
    Scheduler->>Scheduler: Cron: 0 6 * * *
    Scheduler->>Matcher: StartDailyMatching
    
    Matcher->>AlertDB: SELECT * FROM job_alerts<br/>WHERE frequency = 'daily' AND active = true
    AlertDB-->>Matcher: アラート設定一覧 10万件
    
    loop 各アラート設定
        Matcher->>ES: GET /jobs/_search<br/>{<br/>  query: {<br/>    bool: {<br/>      must: [<br/>        {match: {keywords: ["Python", "機械学習"]}},<br/>        {term: {location: "東京"}},<br/>        {range: {experience_min: {lte: 5}}},<br/>        {range: {posted_at: {gte: "now-1d"}}}<br/>      ]<br/>    }<br/>  },<br/>  size: 20<br/>}
        
        ES-->>Matcher: マッチング求人 15件
        
        alt 求人あり
            Matcher->>Matcher: ユーザープロフィール取得<br/>スキル、経験、過去の応募履歴
            
            Matcher->>ML: ScoreJobs<br/>user_profile, matched_jobs
            ML->>ML: 特徴量抽出<br/>スキルマッチ度、経験適合度、給与適合度
            ML->>ML: ランキングモデル適用<br/>LightGBM
            ML-->>Matcher: スコア付き求人<br/>[{job_id, score: 0.92}, ...]
            
            Matcher->>Matcher: トップ10求人を選択<br/>score降順
            
            Matcher->>Queue: Enqueue<br/>{user_id, alert_id, jobs: top10}
        else 求人なし
            Matcher->>Matcher: スキップ
        end
    end
    
    Note over User,SES: メール配信フロー
    
    Email->>Queue: Dequeue
    Queue-->>Email: user_id, jobs
    
    Email->>Email: メール本文生成<br/>HTMLテンプレート
    Email->>Email: 求人詳細埋め込み<br/>タイトル、企業名、給与
    Email->>Email: パーソナライズ<br/>ユーザー名、おすすめ理由
    
    Email->>SES: SendEmail<br/>to: user@example.com,<br/>subject: "10件の新しい求人があります",<br/>body: HTML
    
    SES-->>Email: message_id
    Email->>AlertDB: INSERT INTO alert_notifications<br/>alert_id, sent_at, job_count: 10
    
    SES->>User: メール配信
    
    Note over User,SES: クリック追跡
    
    User->>User: メール開封
    User->>AlertAPI: GET /jobs/job_id?utm_source=alert&alert_id=...
    
    AlertAPI->>AlertDB: INSERT INTO alert_clicks<br/>alert_id, job_id, clicked_at
    AlertAPI->>ML: フィードバック記録<br/>implicit feedback
    AlertAPI-->>User: 求人詳細ページ
    
    Note over User,SES: 週次アラート集計
    
    Scheduler->>Scheduler: Cron: 0 6 * * 1 月曜日
    Scheduler->>Matcher: StartWeeklyMatching
    
    Matcher->>ES: GET /jobs/_search<br/>posted_at: {gte: "now-7d"}
    Matcher->>Matcher: 週次アラートユーザーに配信<br/>同様の処理
```

### 設計のポイント

Elasticsearchで求人を全文検索し、キーワード、場所、経験年数で効率的にマッチングします。
バッチ処理で毎日決まった時間にマッチングを実行し、システム負荷を平準化します。
MLモデルでユーザープロフィールと求人の適合度をスコアリングし、パーソナライズされたレコメンデーションを提供します。
過去の応募履歴、クリック履歴を暗黙的フィードバックとしてモデルに学習させます。
メール配信を非同期キューで処理し、数十万通のメール送信を効率化します。
Amazon SESでメール配信し、開封率、クリック率を追跡します。
アラート頻度daily、weekly、monthlyを選択可能にし、ユーザーの好みに対応します。
求人が投稿されてから24時間以内のものだけを通知し、古い求人を除外します。

## Webアクセス解析サービスの設計

### 概要

Webサイトのアクセス解析プラットフォームを設計します。
イベント収集、リアルタイム分析、集計レポート、ダッシュボードを含みます。

### システム設計図

```mermaid
graph TB
    subgraph "Webサイト"
        Website[Webサイト]
        JSTracker[JavaScript SDK]
    end
    
    subgraph "データ収集"
        Collector[イベントコレクター]
        Kafka[Kafka ストリーム]
    end
    
    subgraph "リアルタイム処理"
        Flink[Flink リアルタイム集計]
        Redis[(Redis リアルタイムメトリクス)]
    end
    
    subgraph "バッチ処理"
        Spark[Spark バッチ集計]
        S3[S3 Raw Data]
    end
    
    subgraph "データウェアハウス"
        BigQuery[(BigQuery)]
    end
    
    subgraph "分析API"
        AnalyticsAPI[分析API]
        Cache[(Redis クエリキャッシュ)]
    end
    
    subgraph "ダッシュボード"
        Dashboard[ダッシュボード]
    end
    
    Website --> JSTracker
    JSTracker --> Collector
    Collector --> Kafka
    
    Kafka --> Flink
    Kafka --> S3
    
    Flink --> Redis
    
    S3 --> Spark
    Spark --> BigQuery
    
    Dashboard --> AnalyticsAPI
    AnalyticsAPI --> Redis
    AnalyticsAPI --> BigQuery
    AnalyticsAPI --> Cache
```

```mermaid
sequenceDiagram
    participant User as サイト訪問者
    participant Website as Webサイト
    participant Tracker as JavaScript SDK
    participant Collector as イベントコレクター
    participant Kafka as Kafka
    participant Flink as Flink
    participant Redis as Redis
    participant S3 as S3
    participant Spark as Spark
    participant BQ as BigQuery
    participant API as 分析API
    participant Dashboard as ダッシュボード

    Note over User,Dashboard: イベント収集フロー
    
    User->>Website: ページアクセス<br/>/products/item-123
    Website->>Tracker: SDK初期化<br/>tracking_id: UA-12345
    
    Tracker->>Tracker: イベント生成<br/>{<br/>  event: "page_view",<br/>  page: "/products/item-123",<br/>  user_id: cookie,<br/>  session_id: uuid,<br/>  timestamp: now,<br/>  user_agent, ip, referrer<br/>}
    
    Tracker->>Collector: POST /collect<br/>Content-Type: image/gif<br/>Base64エンコードイベント
    
    Collector->>Collector: IPからGeoLocation取得<br/>国: JP, 都市: 東京
    Collector->>Collector: User-Agentパース<br/>デバイス: Mobile, ブラウザ: Chrome
    Collector->>Collector: イベント拡張<br/>geo, device情報追加
    
    Collector->>Kafka: Publish<br/>topic: events, partition: tracking_id hash
    Collector-->>Tracker: 200 OK 1x1 GIF
    
    User->>Website: ボタンクリック "カートに追加"
    Tracker->>Collector: POST /collect<br/>{event: "add_to_cart", product_id: "item-123"}
    Collector->>Kafka: Publish
    
    Note over User,Dashboard: リアルタイム集計
    
    Flink->>Kafka: Subscribe
    Kafka-->>Flink: イベントストリーム
    
    Flink->>Flink: 1分間のタンブリングウィンドウ<br/>GROUP BY tracking_id, event
    Flink->>Flink: カウント集計<br/>page_view: 1523件, add_to_cart: 42件
    
    Flink->>Redis: HINCRBY realtime:UA-12345:page_view 1
    Flink->>Redis: HINCRBY realtime:UA-12345:add_to_cart 1
    Flink->>Redis: EXPIRE realtime:UA-12345 3600
    
    Note over User,Dashboard: バッチ集計 毎日2:00
    
    par Rawデータ保存
        Kafka->>S3: Rawイベント保存<br/>s3://events/date=2025-12-31/hour=14/
    end
    
    Spark->>S3: READ events WHERE date = '2025-12-31'
    S3-->>Spark: Rawイベント 1億件
    
    Spark->>Spark: 日次集計<br/>GROUP BY tracking_id, page, event, device, geo
    Spark->>Spark: セッション集計<br/>平均セッション時間、直帰率
    Spark->>Spark: ユーザー集計<br/>新規ユーザー、リピーター
    Spark->>Spark: コンバージョン集計<br/>ファネル分析
    
    Spark->>BQ: INSERT INTO daily_metrics<br/>tracking_id, date, page_views, sessions, users, bounce_rate, ...
    
    Note over User,Dashboard: ダッシュボード表示
    
    Dashboard->>API: GET /analytics/realtime?tracking_id=UA-12345
    
    API->>Redis: HGETALL realtime:UA-12345
    Redis-->>API: 現在のアクティブユーザー: 523, page_view: 1523
    API-->>Dashboard: リアルタイムメトリクス
    
    Dashboard->>API: GET /analytics/report<br/>?tracking_id=UA-12345<br/>&start_date=2025-12-01<br/>&end_date=2025-12-31<br/>&metrics=page_views,sessions,users<br/>&dimensions=page,device
    
    API->>API: クエリキャッシュチェック<br/>TTL: 1時間
    
    alt キャッシュミス
        API->>BQ: SELECT page, device,<br/>  SUM(page_views) as page_views,<br/>  SUM(sessions) as sessions,<br/>  COUNT(DISTINCT user_id) as users<br/>FROM daily_metrics<br/>WHERE tracking_id = 'UA-12345'<br/>  AND date BETWEEN '2025-12-01' AND '2025-12-31'<br/>GROUP BY page, device<br/>ORDER BY page_views DESC<br/>LIMIT 100
        
        BQ-->>API: 集計結果
        API->>API: 結果をキャッシュ<br/>TTL: 3600秒
        API-->>Dashboard: レポートデータ
    else キャッシュヒット
        API-->>Dashboard: キャッシュデータ
    end
    
    Dashboard->>Dashboard: グラフ描画<br/>折れ線グラフ、円グラフ
    Dashboard->>User: アナリティクスダッシュボード表示
```

### 設計のポイント

1x1 GIF画像リクエストでイベントを収集し、ブラウザの制約を回避します。
Kafkaでイベントストリームを管理し、リアルタイム処理とバッチ処理の両方に対応します。
Flinkでリアルタイム集計を行い、現在のアクティブユーザー数を1分遅延で表示します。
Sparkで日次バッチ処理を行い、詳細な集計レポートをBigQueryに保存します。
BigQueryをデータウェアハウスとして使用し、大規模データの高速クエリを実現します。
レポートクエリを1時間キャッシュし、同じレポートへの重複クエリを削減します。
イベントをS3にRaw形式で保存し、後から再処理や監査が可能にします。
セッションIDでユーザー行動を追跡し、平均セッション時間、直帰率を計算します。
GeoLocationとUser-Agentから地域、デバイス、ブラウザを抽出し、ディメンションとして分析します。
