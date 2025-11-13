SYS_CORE（共通基盤） - 仕様設計書 (PHASE_01_SYS_CORE.md Draft)

フェーズ番号: Phase 1

フェーズ名: SYS_CORE（共通基盤）

対応要素: SYS_CORE（help. / line. / shop. / ai. 共通）

1. 目的・範囲

本フェーズでは、加盟店サポート自動化プロジェクト v1.4 の 共通基盤（SYS_CORE） を定義する。

全サブドメイン（help. / line. / shop. / ai. / assets.）で共通利用する システム基盤 の仕様を確定

「パラメータ化ポリシー」に基づき、固定値を極力 DB＋管理画面から変更可能なパラメータとして扱う仕組み を用意

通知（ChatWork / メール / SMS）、AIコア、外部API（STORES / 将来RemoteLock / OpenAI / LINE）に共通する
抽象インターフェース・ログ・エラー処理 を定義

マルチテナント（複数店舗）前提の shop単位の設定・権限・ログ構造 を定義

後続フェーズ（HELP_UI / LINE_UI / SHOP_PORTAL / AI_INBOX 等）が
SYS_COREにぶら下がる形 で設計できるよう、I/Oインターフェースを固定する

2. 前フェーズからの継承ポイント
2.1 ドメイン・チャネル構成（継承）

サブドメイン構成

help.self-datsumou.net（顧客AIチュートリアル / Web）

line.self-datsumou.net（LINEチュートリアル）

shop.self-datsumou.net（加盟店ポータル）

ai.self-datsumou.net（本部AI-INBOX）

assets.self-datsumou.net（静的アセット / CDN想定）

ルート: self-datsumou.net（メール送信ドメイン）

チャネル別構造

Web（help.）: ブラウザAIチャット → メール返信

LINE（line.）: LINEトーク → LINEメッセージ返信

共通AIコア: shared/ai_core.php から全チャネルで利用

2.2 外部サービス連携（継承）

STORES予約API（顧客照合／予約確認）

RemoteLock API（解錠PIN取得）
※本フェーズでは 「将来接続前提の拡張ポイント」としてのみ考慮。実装は別エンジニアによる外部アプリ。

OpenAI API（分類・回答生成）

ChatWork API（本部／運用通知）

Twilio（SMS）

LINE Messaging API + LIFF（LINE本人認証）

2.3 既存のDB設計要素（継承）

ai_core_params, ai_core_templates, ai_core_patterns

shop_secrets（各種APIキー・Webhookシークレット等）

customer_links（LINE ID ↔ STORES顧客IDのマッピング）

audit_logs（操作履歴）

operator_mail_logs, tutorial_feedbacks など

※既存テーブルおよびカラム詳細は DB_FIELDS.md を正とする。
本フェーズでは「新規テーブル候補」と「利用方針」を定義し、
カラム定義が重複する場合は DB_FIELDS.md を優先して整合させる。

3. 新規決定事項（確定済み）

このセクションは、経営判断が完了した「確定仕様」 をまとめる。

3.1 パラメータ管理モデル（A: 決裁済）

方針: 全店舗共通＆店舗別の“設定値”は DB で管理する

固定値になりがちな値（通知タイミング・AI温度・閾値・リトライ回数・トーン・特例フラグ等）は、

グローバル（全体共通）

shop単位
のいずれかのスコープで パラメータ化 する。

パラメータは以下の順で解決する：

shop_params（店舗別上書き）

sys_params（全体共通）

.env（最後のデフォルト）

テーブル構造（案）

sys_params（システム全体共通パラメータ）

id, key, value, type, description, scope='global', updated_by, updated_at

shop_params（店舗別パラメータ）

id, shop_id, key, value, type, description, updated_by, updated_at

🔧 パラメータ候補（パラメータ管理モデル）

パラメータ保存先デフォルト:

全体共通 → sys_params

店舗別 → shop_params

パラメータ解決順: shop_params → sys_params → .env

パラメータ編集権限: role = 'hq_admin' のみ（shop側は読取専用）

パラメータ種別: string / int / float / bool / json

前提宣言:
特に指定がなければ、これらは DB管理＋本部管理画面から編集可能 として設計する。

3.2 通知基盤（Notification Core）（B: キュー方式で決裁済）

方針: 通知（Mail・ChatWork・SMS）は“キュー方式”を標準とする

ChatWork / メール / SMS / 将来のPush通知などは、すべて共通インターフェース SYS_NOTIFY で扱う。

アプリ側は「誰に」「どのテンプレ」「変数」を渡すだけで通知要求を発行し、
実際の送信は notification_queue ＋ cron による非同期処理で行う。

必要なケース（例: パスワードリセットなど）は、パラメータにより同期送信も例外的に許容可能とする。

テーブル構造（案）

notification_queue

id

channel（'chatwork' | 'mail' | 'sms' | 'system' 等）

shop_id（null可・全体通知の場合null）

to（宛先ID/アドレス）

payload_json（本文・テンプレID・変数一式）

priority（'low' | 'normal' | 'high'）

status（'pending' | 'sent' | 'error' | 'canceled'）

error_message

scheduled_at, sent_at, created_at, updated_at

🔧 パラメータ候補（通知基盤）

通知キュー処理間隔（cron間隔）: 1〜5分（デフォルト案: 1分）

再送リトライ回数: 3回（チャネル別に上書き可）

チャネル別タイムアウト: 3〜10秒（API種別による）

デフォルトChatWorkルームID: sys_params で管理、shop別に shop_params で上書き可

デフォルト送信元メールアドレス: noreply@self-datsumou.net 等

未送信/失敗ログの保管期間: 180日（通知ログとして扱う）

3.3 外部APIクライアント共通基盤（C: 共通APIレイヤーで決裁済）

方針: STORES / 将来RemoteLock / OpenAI / LINE / Twilio などは、すべて共通関数経由で呼び出す

すべての外部APIは、共通インターフェース
call_api($service, $method, $endpoint, $params, $options)
を経由させる。

サービス名例:

'stores' … STORES予約API

'remotelock' … RemoteLock API（将来追加）

'openai' … OpenAI API

'line' … LINE Messaging API

'twilio' … SMS送信（Twilio）

APIキーの管理:

STORES予約APIキーは店舗ごとに保持（店舗マスタの定義に従う。DB_FIELDS.mdに準拠）

shops テーブル等から shop_id 経由で取得

その他のシークレット（LINEチャネルシークレット等）は shop_secrets または .env を使用

RemoteLockについて:

今回の開発では RemoteLock連携の実装は行わない。

別エンジニアが作成した「RemoteLock参照用APIアプリ」を、
将来 call_api('remotelock', ...) の実装として差し込めるようにしておく。

つまり、SYS_CORE側には

'remotelock' サービス名

エンドポイントURL／APIキー／タイムアウトなどをパラメータで設定できる拡張ポイント
を用意しておく。（実装は空のスタブ or 未実装ハンドリング）

APIコールログ:

api_call_logs テーブルに、
マスキング済みのリクエスト／レスポンス とステータス・所要時間を記録する。

テーブル構造（案）

api_call_logs

id

service（'stores' / 'remotelock' / 'openai' / 'twilio' / 'line' 等）

shop_id（null可）

endpoint

request_json_masked

response_json_masked

http_status

duration_ms

result（'ok' | 'error'）

error_message

created_at

🔧 パラメータ候補（外部API基盤）

サービス別ベースURL: sys_params（全体） + shop_params（必要な場合の上書き）

サービス別タイムアウト: デフォルト 5秒（OpenAIのみ 10〜15秒など調整）

サービス別リトライ回数: デフォルト 1回

APIログ保管期間: 90日（容量とトラブルシュートのバランス）

OpenAI利用モデル・温度・top_p: ai_core_params で管理

マスキングルール: メールアドレス・電話番号・氏名の一部を伏字

3.4 例外・エラー・監査ログポリシー（保存期間Eと連動）

方針: 各種ログの保管期間を明示し、容量とセキュリティのバランスを取る

PHP致命的エラー → ローカルログ（例: /logs/dev.log）に出力しつつ、重要なものは system_alerts に登録

APIエラー／cronエラー → system_alerts に登録し、必要に応じて ChatWork・メールで通知

テーブル構造（案）

system_alerts

id

level（'info' | 'warning' | 'error' | 'critical'）

category（'api' | 'cron' | 'security' | 'performance' 等）

message

context_json

is_notified（通知済フラグ）

created_at

🔧 パラメータ候補（例外・アラート / 保存期間）

APIログ（api_call_logs）保管期間: 90日

通知ログ（notification_queue + 必要に応じて別テーブル）: 180日

システムアラート（system_alerts）: 365日

アラート通知対象レベル: 'error' 以上 もしくは 'critical' のみ（sys_paramsで制御）

3.5 タイムゾーン・日付時刻ポリシー（D: 決裁済）

方針: DB保存はすべて UTC、表示はJapan Time（JST）で行う

DBの日時カラムは原則すべて UTC で保存する

PHPアプリケーション・画面表示・ログの出力は Asia/Tokyo を使用

外部APIがローカルタイム（JST）を返す場合、保存前に UTC に変換

🔧 パラメータ候補（時刻関連）

システムデフォルトタイムゾーン: 'Asia/Tokyo'

DB保存タイムゾーン: 'UTC'

KPI集計の丸め単位: 分 / 時 / 日（集計基盤設計時に別途指定）

3.6 LINE ID ↔ STORES顧客ID マッピングの扱い（Eとの関係）

方針: 「ログ」ではなく「永続的な顧客リンク情報」として扱う

customer_links（または同等テーブル）で管理する LINE ID ↔ STORES顧客ID の紐づけは、

単なる履歴ログではなく、運用上、継続的に利用する基幹データ とみなす。

保存期間は「期限付き」ではなく、原則永久保管 とする。

例外として、ユーザー本人からの削除要請（個人情報削除）や
一定の退会済み条件を満たした場合に限り、削除または匿名化する運用を設計する。

🔧 パラメータ候補（マッピング保持）

「自動削除を行うまでの放置期間」: 現時点では 無期限 とし、将来必要なら sys_params で制御可能な形を想定

削除ポリシー: 本人からの削除要請時のみ削除 をデフォルトとする

4. データ構造 / I/O定義
4.1 新規テーブル候補一覧（SYS_CORE案）

sys_params … 全体パラメータ

shop_params … 店舗別パラメータ

notification_queue … 通知キュー

api_call_logs … 外部API呼び出しログ

system_alerts … システムアラート

実際の定義は DB_FIELDS.md に追記／整合させる。
既存定義がある場合はそちらを優先し、SYS_CORE案を寄せていく。

4.2 パラメータ解決I/O（概念）

入力

shop_id（null可）

key

処理

shop_params から shop_id + key を検索

見つからない場合 sys_params から key を検索

見つからない場合 .env から検索

出力

型変換済みの value

5. フロー / 画面遷移

SYS_CORE自身はUIをほとんど持たないため、主に内部フローのみ定義する。

5.1 通知フロー（高レベル）

各サブドメイン（help./line./shop./ai.）から
notify($channel, $to, $template_key, $vars, $options) を呼ぶ。

SYS_CORE が notification_queue にレコード挿入（status='pending'）。

cronジョブ（例: cron_notifications.php）が pending を取得し、チャネル別送信処理を実行。

成功時: status='sent', sent_at 更新
失敗時: status='error', error_message 更新

連続エラー・致命的エラーは system_alerts にも登録し、必要に応じて ChatWork / メールへ通知。

6. 通知 / 外部連携
6.1 通知インターフェース（抽象）

概念的な統一API

notify($channel, $to, $template_key, $vars = [], $options = [])

チャネル

'chatwork', 'mail', 'sms', 'system_log'（将来 'push' など追加可）

🔧 パラメータ候補（通知インターフェース）

チャネル別デフォルトテンプレ: sys_params 管理

テンプレキャッシュ有効期限: 10分（5〜60分の範囲で調整可能）

緊急度に応じた送信チャネル組み合わせ:

例: 'critical' → mail + ChatWork + SMS

6.2 外部APIインターフェース（抽象）

概念的な統一API

call_api('stores', 'GET', '/reservations', [...])

call_api('openai', 'POST', 'chat.completions', [...])

call_api('remotelock', 'GET', '/guests', [...])（将来実装）

認証情報の取得方針

call_api('stores', ...) の場合:

shop_id から店舗マスタ（shops）を参照し、STORES予約APIキーを取得

その他サービス:

shop_secrets または .env から参照

解決ルール自体は SYS_CORE 内の1箇所に集約

🔧 パラメータ候補（外部API）

サービス別ベースURL・APIバージョン: sys_params で管理

接続タイムアウト / リードタイムアウト: デフォルト5秒（OpenAIのみ10〜15秒）

リトライ戦略:

HTTP 5xx のみ再試行

リトライ間隔: 1〜5秒（デフォルト2秒）

サーキットブレーカー閾値:

連続エラーN回で一定時間リクエスト抑制（採用する場合は sys_params で制御）

7. セキュリティ・権限
7.1 権限モデル（SYS_CORE観点）

役割（例）

hq_admin … 全体パラメータ編集・全ログ閲覧

hq_staff … ログ閲覧（範囲限定）、パラメータ閲覧のみ

shop_owner … 自店舗に紐づくログ閲覧のみ（パラメータ編集不可）

SYS_CORE レイヤーで role と shop_id を参照し、
パラメータ編集・ログ閲覧・アラート閲覧のアクセス制御を行う。

🔧 パラメータ候補（権限）

パラメータ編集可能ロール一覧: ['hq_admin'] をデフォルトとし、sys_paramsで上書き可

ログ閲覧可能ロール一覧: 種別（API／通知／アラート）ごとに設定可能

7.2 シークレット管理

APIキー等のシークレットは

店舗マスタ（STORES予約APIキー）

shop_secrets（LINEチャネルシークレットなど）

.env（環境共通値）
で管理する。

api_call_logs / system_alerts には 生シークレットを絶対に出力しない。

マスキング処理は共通ヘルパー関数として実装し、全レイヤーから利用する。

8. 未決課題 / 次フェーズへの引き継ぎ
8.1 未決・確認項目（現時点）

ブランド／エリア別など、sys_params と shop_params の間に
中間スコープ（brand / region）のレイヤーが必要かどうか
→ 現時点では不要として設計を進め、必要になった時点で拡張できる構造にしておく。

アラート通知の「どのレベルから ChatWork / メールを鳴らすか」の初期値詳細
→ デフォルト 'error' 以上 とし、運用開始後のノイズを見ながら調整。

8.2 次フェーズへの引き継ぎポイント

HELP_UI / LINE_UI フェーズでは：

固定値（通知タイミング、文言種別、AI温度など）は、すべてSYS_COREのパラメータ解決関数経由 で取得する。

通知送信は 必ず notify() → notification_queue 経由 とし、直接ChatWork/メールAPIを叩かない。

AIコアフェーズでは：

モデル設定・自信度閾値などの「AI専用パラメータ」は ai_core_params

タイムアウト・リトライなどの「基盤パラメータ」は sys_params / shop_params
を使い分ける。