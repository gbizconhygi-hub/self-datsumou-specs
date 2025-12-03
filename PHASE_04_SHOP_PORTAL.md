# PHASE_04_SHOP_PORTAL - 仕様設計書 (v1.1 Draft)

## 1. 目的・範囲

### 1.1 フェーズ概要

本フェーズ（Phase 4：SHOP_PORTAL）は、

`shop.self-datsumou.net`（オーナーポータル／加盟店向けサポート UI）の仕様を定める。

- 主体は **owners（オーナー）** であり、
1オーナーに紐づく複数店舗（shops）がぶら下がる構造とする。
- SHOP_PORTAL は、以下を 1 つの UI 群として提供する。
    - オーナー単位の状況把握（お知らせ／請求／売上サマリ／店舗ごとの問い合わせサマリ）
    - 店舗単位の運営・サポート UI（顧客問い合わせ対応／備品発注／各種申請／簡易分析）
    - 本部とのやり取り（AI相談／MTG予約／キャンペーン申請／デザイン依頼）

### 1.2 本フェーズで実現する主な機能

**オーナーレベル（オーナーダッシュボード）**

- 本部からのお知らせ（アナウンス）を受け取る
- オーナー情報（契約名義・請求先等）は「初期入力時のみ」オーナー編集可
→ 完了後の編集権限は HQ のみ
- ロイヤリティ等の **当月請求書発行状況・過去請求書閲覧**
- 売上サマリー（全店舗合算）
- 店舗ごとの問い合わせ状況サマリー（open/pending/closed 件数など）
- オーナー向け AI 相談（owner_ai）
- 本部 MTG 予約（Google Meet 参加URL 付き）

**店舗レベル（店舗ダッシュボード）**

- 顧客からの問い合わせ（help./line. 経由）の「要対応分」の受付・返信
- 電話代行（OP_MAIL）経由の緊急連絡チケットの確認・返信
（`tickets.source='operator_mail'` ＋ `severity='urgent'` を特別扱い）
- 店舗ごとの基本契約・オプション契約情報の閲覧（編集不可）
- 本部が販売する備品発注
→ 発注分は当月のロイヤリティ請求に自動加算される前提
- 本部への各種申請（キャンペーン作成依頼、看板変更依頼 など）
→ 金額がかかるものは HQ が請求書にプラスオン
- 売上データからの簡易分析
（お試し体験獲得数、メニューごとの購入数など）

### 1.3 対応デバイスとレイアウト方針

- **メインターゲット：PC／タブレット（横幅 1024px 前後）**
    - 左サイドバーにオーナーレベルのメニュー
    - メインペインに店舗ダッシュボードや一覧・詳細を表示
- **スマホ（縦長）もレスポンシブ対応**
    - 下部タブバーやドロップダウンでオーナーメニューを切り替え
    - 「オーナーダッシュボード → 店舗一覧 → 店舗詳細」の階層構造は PC と共通

### 1.4 UI デザイン・カラー方針（ブランド準拠）

- SHOP_PORTAL の UI デザイン（カラー／雰囲気）は、**ハイジ公式ページ（[https://self-datsumou.com](https://self-datsumou.com/)）とブランド整合を取ることを前提とする**。
- 方針レベルでの決定事項：
    - ベースカラー：公式サイトと同系統の「明るく清潔感のある白／淡色系」を採用
    - アクセントカラー：公式サイトのブランドカラー（ロゴ・ボタン色）を踏襲し、
    SHOP_PORTAL でも「ハイジらしさ」が一目でわかる配色とする
    - 禁止事項：
        - SHOP_PORTAL 独自で、ブランドコンセプトと乖離するダークテーマ／原色多用を導入しない
        - 本番環境では公式ブランドガイドラインに反する独自色の上書きを行わない
- 具体的な色コード・フォント指定は、別途「ブランドガイドライン／UIデザインガイド」にて管理し、
本仕様書では **「公式ページのブランドに準拠する」という原則レベルの宣言** にとどめる。

### 1.5 関連ファイル（レイヤ分割）

SHOP_PORTAL に関する仕様は、以下の 4 ファイルで構成する。

1. **PHASE_04_SHOP_PORTAL.md**（本書）
    - 構造・責務・I/O・フローなど CORE 仕様
2. **SHOP_PORTAL_POLICY.md**
    - オーナー向け AI／UI の運用ポリシー・NG ライン
3. **SHOP_PORTAL_DEFAULT_TEXTS.md**
    - 画面メッセージ／AI システムメッセージ等の初期文面テンプレート集
4. **SHOP_PORTAL_DEFAULT_PARAMS.md**
    - ダッシュボード期間／一覧件数／AI温度／締め日など、SHOP_PORTAL 向けパラメータのデフォルト値

help./line. の HELP_UI_POLICY／LINE_UI_POLICY 等と同じレイヤ分割を踏襲する。

---

## 2. 前フェーズからの継承ポイント（抜粋）

### 2.1 ドメイン構成と役割

- help.：顧客向け Web チュートリアル（AI＋メール）
- line.：顧客向け LINE チュートリアル（AI＋LIFF／リッチメニュー）
- **shop.：加盟店ポータル（オーナー向けステータス／請求／AI相談／申請）**
- ai.：本部ポータル（AI-INBOX／AIチューニング／KPI）

### 2.2 コア層との関係

- SYS_CORE：sys_params／api_call_logs／sys_events／tickets／通知コア 等の共通基盤
- API_COMM：STORES予約 API などを業務 API としてラップ
- AI_CORE：help./line./shop./ai. 共通の AI 頭脳
    - SHOP_PORTAL では `channel="shop"`, `user_role="owner"` を標準とする
- OP_MAIL（電話代行メール解析／エスカレーション）は Phase7 で実装され、`tickets.source='operator_mail'` のチケットを生成する。
Phase4 では、**そのチケットを SHOP_PORTAL 上でどう見せるかのみ定義する**。

### 2.3 DB BOX（BOX_INTENT）

SHOP_PORTAL が主に利用する BOX：

- owners／owner_contacts（オーナー単位情報）
- shops／shop_secrets／shop_links／shop_internal_info
- options_master／shop_options
- sys_params（scope=global/shop）
- sales_records（売上）
- invoices／invoice_items（請求）
- tickets／ticket_messages（問い合わせ）
- ai_sessions／ai_turns（AI 会話ログ）
- hq_meeting_slots／hq_meetings（本部 MTG枠／予約。Google Meet 参加URL（meeting_join_url）を保持）
- supply_items／shop_supplies（備品マスタ／発注）
- hq_requests（各種申請）
- campaign_requests／campaign_banners／banner_templates（キャンペーン・バナー）
- design_requests（デザイン依頼 DR）

---

## 3. 新規決定事項（SHOP_PORTAL 固有仕様）

### 3.1 情報構造・ロール

- 主軸は **owner → shops** の階層構造
    - ログイン単位：portal_users
        - owner ログイン … portal_users.role='owner'（[owners.id](http://owners.id/) と紐づく）
        - staff ログイン … portal_users.role='staff'（shop_staff_users.id と紐づく）
        - HQ ログイン … portal_users.role='admin'（HQ 側UI：ai.）
    - 1 オーナーに 1..N 店舗（shops）が紐づく
    - 1 スタッフに 1..N 店舗（shop_staff_assignments）が紐づく
- ロールと権限（SHOP 側）
    - owner ロール：
        - 自身の owner_id に紐づく **全店舗** の情報にアクセス可能
        - スタッフアカウント（shop_staff_users）の追加・編集・無効化が可能
        - スタッフごとのマスク項目（shop_staff_mask_rules）を編集可能
    - staff_basic ロール：
        - `shop_staff_assignments` に紐づく店舗のみ閲覧・操作可
        - 売上金額・請求金額・契約条件など、一部の項目はデフォルトでマスク表示される
        - マスク範囲は owner が shop_staff_mask_rules で制御する
    - staff_multi_shop ロール：
        - オーナー配下の複数店舗を横断して問い合わせ状況を閲覧可能
        - ただし金額やKPI詳細などの表示可否は staff_basic と同様に mask_rules に従う
    - HQ ロール（本部）：
        - ai. 側 UI で全 owner／shop に対して横断的に閲覧・編集可
        - SHOP_PORTAL 画面は「オーナー／スタッフが見る画面」という前提で設計し、HQ は原則参照のみ（例外はデバッグ用途等）とする

### 3.2 機能モジュール（骨）

**オーナーレベル**

1. オーナーダッシュボード
    - 本部お知らせ（hq_announcements）
    - オーナー情報（初期入力ウィザード＋完了後は閲覧のみ）
    - 当月／過去の請求サマリ
    - 店舗別問い合わせサマリ（help./line./operator_mail 由来の tickets を統合）
2. 請求・売上一覧
    - invoices／invoice_items（オーナー全体の請求）
    - sales_records（全店舗合算＋フィルタ）
3. オーナー向け AI 相談
    - `AI_CORE_REQUEST.channel="shop", user_role="owner"`
4. 本部 MTG 予約
    - hq_meeting_slots／hq_meetings を利用した枠選択・予約。
    - `hq_meetings.meeting_tool` は v1.4 時点では `google_meet` 固定とし、`meeting_join_url` に本部が作成した Google Meet 参加URLを保持する。
    - SHOP_PORTAL の MTG 詳細画面では「Google Meet に参加」ボタンとして `meeting_join_url` を開き、「このMTGは Google Meet で実施されます」という説明文を表示する。

**店舗レベル**

1. 店舗ダッシュボード
    - 店舗別の問い合わせサマリ／簡易売上サマリ
2. 顧客問い合わせ（要対応分）
    - `tickets.shop_id = 店舗` かつ `status in (open, pending)` の一覧＆返信
    - `source='help_web'/'line'/'operator_mail'/...` を含む
    - `source='operator_mail' AND severity='urgent'` は「電話代行（緊急）」ラベルとして強調表示
3. 店舗契約・オプション情報（閲覧のみ）
    - shops／options_master＋shop_options
4. 備品発注
    - supply_items → shop_supplies への発注
5. 各種申請（hq_requests）
    - キャンペーン申請・運営系申請・その他依頼
6. デザイン依頼（design_requests）
    - テンプレバナー（デザイナーA）／新規クリエイティブ（デザイナーB）を含む
7. 簡易売上分析
    - お試し体験数／メニュー別売上など

### 3.3 オーナー情報編集ポリシー

- `owners.is_profile_complete=false` の間のみ SHOP_PORTAL から編集可能
- `is_profile_complete=true` に遷移した後は、SHOP_PORTAL 上は閲覧のみ
- 再編集が必要な場合：
    - SHOP_PORTAL からは「修正申請」（hq_requests）を出すのみ
    - 実際の更新操作は HQ 側 UI から行う

### 3.4 顧客問い合わせの扱い（OP_MAIL 含む）

- help./line. で AI が一次対応し、エスカレーション条件を満たした問い合わせは tickets に登録される。
- Phase7（OP_MAIL）において、電話代行からの報告メールは
    - `tickets.source='operator_mail'`
    - `tickets.severity`（例：'urgent'/'normal'）
    として tickets テーブルに生成される前提とする
    （生成タイミングは Phase7 で設計。cron ではなく「受信都度リアルタイム処理」を想定）。
- SHOP_PORTAL 側での扱い：
    - `shop_id` が一致する tickets は、チャネル種別に関わらず店舗ダッシュボードに表示する。
    - 特に `source='operator_mail' AND severity='urgent'` のチケットは、
        - 店舗ダッシュボードの「要対応」リストの最上段に表示する
        - ラベル「電話代行（緊急）」を付与し、視覚的に目立たせる
    - 店舗は、通常チケットと同様に返信・ステータス変更が可能。
- クレーム／賠償／FC契約などの危険カテゴリは AI_CORE 側 intent で識別し、
HQ 直行とする（SHOP_PORTAL には「参照のみ」または非表示）。

### 3.5 備品発注フロー（骨）

- 店舗ダッシュボードから supply_items の一覧を表示し、数量入力→発注。
- 発注は `shop_supplies` に記録：
    - shop_id, supply_item_id, quantity, unit_price, status='ordered', ordered_at
- 月次請求生成処理で、`status='ordered'` かつ締め日以前のものを集計し、
invoices／invoice_items に加算する。

### 3.6 各種申請（hq_requests）

- SHOP_PORTAL の「申請」メニューから起票：
    - request_type（campaign_plan / signage_change / design_request / other 等）
    - shop_id／owner_id／内容
- `hq_requests` テーブルで状態管理：
    - status：requested / in_progress / need_confirm / completed / cancelled
- 金額が発生する申請は HQ 側で actual_cost／offer_price を設定し、
invoices／invoice_items に反映。

### 3.7 デザイン依頼フロー（キャンペーン＋DR）

- キャンペーン申請（campaign_requests）：
    - オーナーが SHOP_PORTAL から起票
    - HQ が内容確認・修正依頼・承認（approved）
    - 承認後、テンプレ適用 or デザイナーAに依頼
- デザイナーA（テンプレバナー）：
    - campaign_requests に紐づく design_requests（type='campaign_banner_A', designer_type='A'）を作成
    - Chatwork「本部／デザイナーAルーム」でのみ制作・納品
    - オーナーへの追加請求は基本発生させず、本部負担とする
- デザイナーB（新規クリエイティブ）：
    - デザイン依頼フォーム（design_requests origin_type='new_creative'）→ DR-ID 発行
    - HQ が internal_cost／offer_price を設定 → SHOP_PORTAL 上でオーナーに見積提示・承認
    - Chatwork「オーナー／デザイナーB（＋本部）」ルームで仕様詰め〜校正
    - 納品後、offer_price を invoices／invoice_items に反映

### 3.8 🔧 パラメータ候補（SHOP_PORTAL 全体）

詳細は `SHOP_PORTAL_DEFAULT_PARAMS.md` に定義し、ここでは代表例のみ示す。

- ダッシュボード／一覧：
    - `shop_portal.dashboard_default_range_days`
    - `shop_portal.announcement_max_items`
    - `shop_portal.tickets_page_size`
    - `shop_portal.analytics_default_range_days`
- 備品・申請：
    - `shop_portal.supply_order_cutoff_day`
    - `shop_portal.enable_supply_order`
- オーナーAI：
    - `shop_portal.owner_ai_temperature`
    - `shop_portal.owner_ai_max_turns_before_escalation_suggest`
    - `shop_portal.owner_ai_max_turns_before_force_escalation`
    - `shop_portal.owner_ai_quiet_hours_start` / `end`
- MTG 予約：
    - `shop_portal.meeting_slots_visible_days_ahead`
    - `shop_portal.meeting_request_notify_chatwork`
    - `shop_portal.meeting_request_notify_email`
- デザイン：
    - `shop_portal.design.dr_code_prefix`
    - `shop_portal.design.cr_code_prefix`
    - `shop_portal.design_designerA_offer_price_zero`
    - `shop_portal.design_default_estimate_expire_days`

### 3.9 スタッフアカウント管理（owner 操作）

- 対象：owner ロールのみ（portal_users.role='owner'）。
- 目的：
    - オーナー自身が、店舗運営を任せるスタッフアカウントを SHOP_PORTAL 上から発行・管理できるようにする。
    - スタッフごとに「どの店舗を担当するか」「どこまで見せるか（マスク）」を制御する。

### 画面概要

- メニュー：「アカウント管理」タブをオーナーダッシュボードに追加。
- 一覧：
    - shop_staff_users の一覧（name / email / default_role / 担当店舗数 / 有効・無効）。
    - 行クリックでスタッフ編集モーダルを開く。

### 主な操作

1. 新規スタッフ追加
    - 入力項目：
        - 氏名
        - メールアドレス（ログインID）
        - ChatWork ID（必須）
        - ロール（staff_basic / staff_multi_shop）
        - 担当店舗（1..N）
    - 保存時の動作：
        - shop_staff_users／shop_staff_assignments にレコード作成
        - portal_users（role='staff'） に初期パスワード付きアカウント作成
        - オプションで案内メール送信（初期パスワードとログインURL）
2. スタッフ編集
    - 変更可能な項目：
        - default_role（staff_basic / staff_multi_shop）
        - 担当店舗（追加・削除）
        - 有効／無効フラグ（is_active）
    - メールアドレスは原則変更不可（変更が必要な場合は HQ 経由で portal_users を再発行）。
    - 無効化したスタッフはログイン不可とし、過去ログとの紐づきは維持する。
3. マスク項目設定
    - スタッフ行から「権限詳細」モーダルを開き、マスク項目をトグル形式で設定。
    - 背後では shop_staff_mask_rules にレコードを登録／更新する。
    - デフォルト運用：
        - `staff_user_id IS NULL` のレコード … オーナー配下全スタッフ共通のマスク（デフォルト）。
        - 特定スタッフにだけ異なる権限を与えたい場合、
        同じ `field_key` で `staff_user_id=該当スタッフ` の行を作成し、デフォルトを上書きする。

### マスク対象の例（UI 表現）

- 売上関連：
    - 総売上金額 → 「***」表示（件数や構成比のみ表示）
    - メニュー別売上金額 → 「***」表示（件数のみ表示）
- 請求関連：
    - ロイヤリティ金額 → 「***」表示
    - 請求明細単価 → 「***」表示
- KPI 詳細：
    - LTV／CPA など感度の高い指標 → マスク前提（owner が必要に応じて解禁）

---

## 4. データ構造 / I/O 定義（抜粋）

### 4.1 既存テーブルの利用（概要）

- owners／owner_contacts／shops／shop_*／options_master／shop_options／sys_params
→ オーナー／店舗／設定の SoR
- sales_records／invoices／invoice_items
→ 売上・請求
- tickets／ticket_messages
→ 顧客・オーナー・HQ すべての問い合わせ器
→ `source='help_web'/'line'/'operator_mail'/...`
- ai_sessions／ai_turns
→ AI 会話ログ（owner AI 相談もここに統合）
- hq_meeting_slots／hq_meetings
→ 本部 MTG 枠・予約。`hq_meetings.meeting_tool`（v1.4 では `google_meet` 固定）と `meeting_join_url`（Google Meet 参加URL）を保持し、SHOP_PORTAL から「Google Meet に参加」ボタンとして利用する。
- supply_items／shop_supplies
→ 備品マスタ／発注

### 4.2 新規テーブル（Phase4 提案）

（hq_announcements／hq_requests／campaign_requests／banner_templates／campaign_banners／design_requests／各 comments テーブルの DDL は前回案をそのまま採用。ここでは割愛し、実ファイル側に完全形を記述する。）

---

## 5. フロー / 画面遷移

### 5.1 ログイン〜店舗選択

1. owner が shop. にアクセスし、ログイン
2. JWT に `owner_id`, `role='SHOP'`, `shop_ids` を付与
3. shop_ids が 1 件なら、その店舗前提でオーナーダッシュボードへ
4. 複数店舗の場合は「店舗一覧」からメイン店舗を選択（後から切替可）

### 5.2 オーナーダッシュボード

1. hq_announcements から未読お知らせを取得し、上部カードで表示
2. 当月請求／売上サマリを集計表示
3. 店舗ごとの問い合わせサマリ：
    - `tickets.shop_id = shop_id` をチャネル種別問わず集計
    - `source='operator_mail' AND severity='urgent'` を含む件数も可視化（「電話代行（緊急）」件数）
4. 店舗名クリックで店舗ダッシュボードへ遷移

### 5.3 店舗ダッシュボード（shop 単位）

1. 店舗別の問い合わせサマリ：
    - open/pending/closed
    - 緊急（電話代行）件数（source='operator_mail' AND severity='urgent'）
2. 「顧客問い合わせ」タブ：
    - `tickets.shop_id = shop_id AND status IN (open,pending)` の一覧
    - `source='operator_mail' AND severity='urgent'` は最上段に固定表示し、「電話代行（緊急）」バッジを表示
    - チケット詳細では、顧客メッセージ・AI 要約・店舗／HQ 返信をタイムライン表示
3. 「備品発注」タブ：
    - supply_items 一覧→数量入力→発注→shop_supplies 登録
4. 「申請・デザイン」タブ：
    - hq_requests／campaign_requests／design_requests を一覧表示
    - 新規キャンペーン申請／デザイン依頼フォーム起票
5. 「分析」タブ：
    - 簡易売上分析指標を期間選択で表示

---

## 6. 通知 / 外部連携

### 6.1 Chatwork 連携（概要）

- デザイナーA：
    - design_requests（designer_type='A'）作成時：本部／デザイナーAルームに DR-ID＋キャンペーン情報を自動投稿
- デザイナーB：
    - design_requests が approved になったタイミングで、オーナー／デザイナーBルームに
    「【DR-XXXX】新規案件が承認されました」と投稿
- OP_MAIL 由来チケット：
    - Chatwork/SMS 連携の詳細（どの room に投げるか／テンプレ文言など）は Phase7(OP_MAIL) 側で定義。
    Phase4 では「チケットが SHOP_PORTAL に表示される」こと、および「緊急ラベル表示」のみを規定する。

### 6.2 AI チャット連携（オーナー向け）

- 以下のイベント時に、オーナー向け AI チャットへシステムメッセージを送信：
    - 新しいお知らせ（重要度が高いもの）
    - キャンペーンバナー完成
    - デザイン依頼の見積提示・納品完了
    - HQ から案件コメント（campaign_requests／design_requests コメント）追加

### 6.3 メール／その他通知

- 請求書発行／MTG 確定／重要な申請ステータス変更などは、
billing_email／owner_contacts.email を宛先にメール通知
（通知ルール詳細は SYS_CORE 通知仕様に従う）

---

## 7. セキュリティ・権限・ログ

### 7.1 権限

- SHOP ロール：
    - JWT に含まれる owner_id／shop_ids の範囲のみ操作可能
    - 他オーナー／他店舗のデータにはアクセス不可
- HQ ロール：
    - ai. 側 UI で全体を横断閲覧可能（SHOP_PORTAL とは別 UI）

### 7.2 PII／ログポリシー

- 顧客の氏名・来店履歴は、店舗が元々持っている情報の範囲で表示してよい。
- メールアドレス／電話番号は、SHOP_PORTAL 上ではマスク済み表現を基本とする（末尾4桁など）。
- AI ログ（ai_turns.raw_text）は SHOP_PORTAL では直接閲覧不可とし、
    - 要約／intent だけを表示対象とする。
    - 詳細ログは [ai.self-datsumou.net](http://ai.self-datsumou.net/)（HQ 専用）で管理。
- 重要イベント（発注／申請／価格承認／請求生成／OP_MAIL由来緊急チケットの作成など）は sys_events にイベント記録。

---

## 8. 未決課題 / 次フェーズへの引き継ぎ

1. クレーム系問い合わせを「どこまで店舗に直接見せるか」の最終ライン
2. 簡易分析の指標範囲（最小セットは本仕様の通りだが、追加指標は KPI_ANALYTICS フェーズと調整）
3. hq_requests と design_requests／campaign_requests の責務境界（将来の統合・整理の余地）
4. OP_MAIL（Phase7）側での詳細設計：
    - 電話代行メールの受信方式（Webhook／IMAP IDLE／短周期ポーリング）
    - 緊急判定ロジック（intent／キーワード／電話代行フォーマット）
    - Chatwork／SMS の通知テンプレート
    - tickets.severity／source の詳細定義
5. ブランドガイドライン（カラーコード／フォント／コンポーネント）の別途明文化
→ Phase4 では「公式サイト準拠」という原則だけ定義し、具体ガイドは UI デザイン側で管理
