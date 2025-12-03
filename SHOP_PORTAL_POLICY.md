# SHOP_PORTAL_POLICY (v1.1 Draft)

## 1. 目的・位置づけ

本ドキュメントは、`shop.self-datsumou.net`（オーナーポータル／SHOP_PORTAL）における

- 加盟店向け AI 応答ポリシー
- オーナー／店舗向け UI の運用ルール
- 本部（HQ）との責任分界・エスカレーション方針
- ロール（owner / staff_basic / staff_multi_shop）ごとの見せ方・権限・マスク方針

を定める。

SHOP_PORTAL は、help./line. と同様に AI_CORE／SYS_CORE／API_COMM を利用しつつ、

- 「オーナー（super_category=owner）」と、その配下スタッフ向けのチャネル
- 契約・請求・運営などビジネス寄りの情報を扱うチャネル

として運用する。

## 2. 適用範囲

本ポリシーは、以下の機能に適用する。

- オーナーダッシュボード（お知らせ／請求／売上サマリ／店舗サマリ）
- 店舗ダッシュボード（顧客問い合わせ／備品発注／申請／簡易分析）
- オーナー向け AI 相談（owner_ai）
- スタッフ向け画面（staff_basic / staff_multi_shop）
- キャンペーン申請／デザイン依頼（campaign_requests／design_requests）
- 本部 MTG 予約（hq_meetings：Google Meet 参加URL 前提）

help./line. 向けポリシーは HELP_UI_POLICY／LINE_UI_POLICY に従う。

## 3. AI 応答ポリシー（オーナー向け owner_ai）

### 3.1 super_category / intent

- SHOP_PORTAL 上のオーナー向け AI 相談は、AI_CORE に対して
    - `super_category = owner`
    - `channel = "shop"`
    を指定する。
- スタッフが同じ画面から AI に質問する場合も、
    
    「オーナー組織としての相談」とみなし、原則として同じ super_category / channel を用いる。
    
    （actor_type は `shop_owner` / `shop_staff` で区別する）
    
- 代表的な intent（例）：
    - `owner/operation_store/...`（店舗運営一般）
    - `owner/campaign_idea/...`（キャンペーン・集客相談）
    - `owner/system_usage/...`（システムの使い方）
    - `owner/design_flow/...`（デザイン依頼フロー関連）
    - `owner/contract_fc/...`（FC 契約・ロイヤリティ関連）
    - `owner/trouble_customer/...`（顧客トラブル／クレーム）

### 3.2 AI が完結してよい領域（auto_ok）

AI がそのまま返答してよい（auto_ok）領域の例：

- マニュアルに基づく一般的な運営 FAQ
    - 開店前チェックリスト
    - タオル／ジェルなど備品の目安
    - help./line.／SHOP_PORTAL の使い方
- 公式に定義されたキャンペーンの説明
- デザイン依頼フローの説明（DR-ID の仕組み含む）
- AI／システムの機能説明
- KPI の「見方」や一般的な解釈（ただし具体的な契約金額・ロイヤリティ計算結果には踏み込まない）

### 3.3 必ず HQ へエスカレーションする領域（human_required）

以下の intent／内容は、AI の自動回答を禁止し、必ず HQ にエスカレーションする。

- FC 契約の個別条件（違約金・契約解除条件・更新条件）
- ロイヤリティ計算に関する「具体的な金額・率・これで合っているか？」系の確認
- 法的トラブル／賠償責任に関わる相談
- 顧客との深刻なトラブル（警察沙汰・訴訟など）
- デザイナーとのやり取りにおける報酬・原価・利ザヤの話題
- Google Meet MTG の録画・情報開示等、法務・労務に関わる判断

AI はこれらの質問に遭遇した場合：

- 一般論レベルの注意喚起や考え方を短く示す **まで** とし、
- 最終判断は HQ に委ねる旨を明示した上で、HQへの相談を案内する。

### 3.4 DR-ID／正式案件の扱い（デザイン系）

- デザイン新規制作・修正は、必ず `design_requests`（DR）を起点とする。
- Chatwork 上で DR-ID が付いていない相談が来た場合、
    - オーナー向け AI／デザイナーBは
    「この内容は新規案件になるため、まず SHOP_PORTAL からデザイン依頼（DR）を起票してください。」
    と案内する。
- 「DR-ID のない仕事は正式案件として扱わない／請求にも載せない」を一貫したポリシーとする。

## 4. エスカレーション & 責任分界

### 4.1 エスカレーション先

- SHOP_PORTAL 上の owner_ai は、以下の 2 層でエスカレーションを行う。
    1. HQ_CHAT／AI-INBOX（本部担当者）
    2. 場合によっては店舗側（顧客対応チケット）へ再度戻す

### 4.2 責任分界の基本

- AI はあくまで「補助的な説明役」であり、
    - 契約・請求・法務の最終責任は HQ にある
    - 現場判断が必要な顧客対応の最終責任は店舗にある
- AI は「本部が承認したルール・FAQ」に基づいて回答する
    
    → FAQ／Q&A の SoR は HQ が管理する ai_qa_entries／ドキュメント群
    
- スタッフが AI を利用して回答する場合も、
    
    「オーナー組織としての回答」であり、最終責任はオーナー／HQ 側にあることを明確にする。
    

## 5. PII・ログポリシー（SHOP_PORTAL）

- 顧客の氏名・来店履歴は、店舗が元々持っている情報の範囲で表示してよい。
- メールアドレス・電話番号は、SHOP_PORTAL 上ではマスク済み表現を基本とする（末尾4桁など）。
- AI ログ（ai_turns.raw_text）は SHOP_PORTAL では直接閲覧不可とし、
    - 要約／意図（intent）だけを表示対象とする。
    - 詳細ログは [ai.self-datsumou.net](http://ai.self-datsumou.net/)（HQ 専用）で管理。
- 金額や粗利などセンシティブな数値情報は、
    - owner ロールにはフル表示を許可する一方で、
    - staff ロールでは **マスクされた表示をデフォルト** とし、
    mask_rules（shop_staff_mask_rules）と SHOP_PORTAL_DEFAULT_PARAMS の設定に従って制御する。

## 6. デザイン案件ポリシー（A/B 共通）

- デザイナーと HQ の間では **internal_cost（原価）のみ共有** し、
利ザヤや最終価格（offer_price）は HQ 内部情報とする。
- デザイナーはオーナーに対して一切金額を伝えない。
- キャンペーンバナー（デザイナーA）：
    - オーナー請求は基本的に発生させず、HQ 負担（社内サービス）とする。
- 新規クリエイティブ（デザイナーB）：
    - 見積提示・承認・請求はすべて SHOP_PORTAL 上で行う。
    - Chatwork 上では「DR-ID」と制作内容のみ扱う。

## 7. 通知・お知らせポリシー

- 「運用ルール／注意喚起／重要告知」は hq_announcements で管理し、
    - target_scope（全オーナー／特定オーナー／特定店舗）を指定する。
- 通知経路：
    - SHOP_PORTAL：オーナーダッシュボードの「未読お知らせ」
    - 必要に応じてオーナー向け AI チャットへのシステムメッセージ
    - メール通知（billing_email／owner_contacts.email）との併用
- 緊急案件（OP_MAIL 由来の「電話代行（緊急）」など）は、
escalation_rules に基づき ChatWork／SMS と連携する。

## 8. トーン & スタイル（オーナー向け）

- 基本は丁寧な敬語・ビジネスライクだが、
オーナーの不安・怒りを受け止めるトーンを優先する。
- NG：
    - ぶっきらぼうな一言返信
    - 「規約に書いてあります」のみで終わる投げやりな回答
- 推奨：
    - 「まず結論 → 理由 → 具体的な次の一手」の構成
    - AI が答えづらい領域では、早めに「本部担当が確認します」と人へつなぐ

---

## 9. 見直し

本ポリシーは、運用開始後の状況に応じて定期的に見直す。

特に以下の場合は改定を検討する。

- クレーム・トラブルの傾向が変化した場合
- オーナー向け AI の誤解・誤案内が問題になった場合
- デザイン／キャンペーンの運用ルールを変更する場合
- ロールごとのマスク設定（staff_basic / staff_multi_shop）が現場ニーズと乖離してきた場合
- MTG ツール（Google Meet 以外）を採用するなど、MTG運用方針に変更が生じた場合

---

## 10. ロール別ポリシー（owner / staff_basic / staff_multi_shop）

### 10.1 owner ロール

- 表示範囲：
    - 自身の owner_id に紐づく **全店舗** の売上・請求・KPI・契約条件をフル表示。
    - 顧客問い合わせ／OP_MAIL 由来チケットを含む、全チケットの内容を閲覧可能。
- 編集権限：
    - スタッフアカウント（shop_staff_users）の追加・編集・無効化。
    - スタッフごとのマスク項目（shop_staff_mask_rules）の編集。
    - 備品発注・各種申請・デザイン依頼の起票。
- 責任：
    - スタッフにどこまで情報を見せるか（マスク設定）は owner の責任で決定する。
    - 「DR-ID のない案件を走らせない」「金額に関する回答は原則 HQ 経由にする」など、
    本ポリシーをスタッフと共有・徹底する責任を負う。

### 10.2 staff_basic / staff_multi_shop ロール

- 共通方針：
    - スタッフは「オーナーの代理として問い合わせ対応や運営処理を行う」立場であり、
    契約金額や粗利などセンシティブな情報は、デフォルトでマスクされる。
    - 実際にどこまで見せるかは、owner ごとに shop_staff_mask_rules で制御する。
- staff_basic：
    - `shop_staff_assignments` に紐づく **担当店舗のみ** の情報を閲覧・操作可。
    - 売上合計・ロイヤリティ金額・請求明細単価・詳細KPI はマスク表示がデフォルト。
    - 顧客問い合わせ／OP_MAIL 緊急チケットの一次対応を担当する想定。
- staff_multi_shop：
    - 複数店舗を横断した問い合わせ一覧・簡易サマリを閲覧可能。
    - ただし、マスク対象は staff_basic と同様に mask_rules に従う。
    - owner に代わってマルチ店舗運営を支える「オーナー代行」的な立ち位置を想定しつつも、
    契約条件・請求額など、最終的な意思決定領域は owner／HQ の責務とする。

### 10.3 ChatWork ID に関するポリシー

- owners.chatwork_room_id、および owner_contacts.chatwork_id／shop_staff_users.chatwork_id は、
    - エスカレーション／緊急連絡／MTG 確定通知／請求通知 などの主な連絡経路となるため、
    - **運用上必須項目** とする（DB 上は NULL 許容でも、入力フロー上は必須扱い）。
- 通知の基本方針：
    - オーナー単位の通知：owners.chatwork_room_id を起点とし、必要に応じて担当者 chatwork_id をメンション。
    - スタッフ単位の通知：shop_staff_users.chatwork_id を用いて、担当者をピンポイントでメンション。
- ChatWork ID が未登録の場合：
    - 登録を促すシステムメッセージを SHOP_PORTAL 上に表示する。
    - 重大通知（請求／緊急チケット）については、暫定的にメールのみでフォローするが、
    「ChatWork 登録が完了するまでは運用リスクがある」旨を明示する。

---

## 11. MTG / Google Meet 利用ポリシー

### 11.1 MTG ツールの標準

- v1.4 時点では、オンライン MTG は **Google Meet を標準ツール** とする。
- システム側は、MTG 情報として以下のみを保持する。
    - 開催日時（hq_meetings.meeting_start_at 等）
    - 参加URL（Google Meet：hq_meetings.meeting_join_url）
    - MTG 種別・メモなどの最小限のメタ情報
- Google アカウント／Calendar／Meet API との直接連携は行わない：
    - 本部メンバーが自分の Google アカウントで Meet を作成し、
    - 生成された URL を AI-INBOX／SHOP_PORTAL から `meeting_join_url` として登録する運用とする。

### 11.2 プライバシー・ログ

- システムは、Google Meet の録画データや通話内容そのものは一切取得・保存しない。
- MTG に関するログとしては、以下のようなメタ情報のみを sys_events／hq_meetings に記録する。
    - MTG 作成日時／開催日時
    - 参加予定店舗／オーナー
    - MTG 種別（例：初回オリエン／運営相談／契約相談 等）
- MTG 内での個別の発言内容・合意事項は、
    - 必要に応じて HQ が別途議事録として管理する（Google Docs 等）。
    - システム側では「議事録ファイルへのリンク」等を保持するに留める。

### 11.3 通知と参加方法

- MTG 確定時：
    - SHOP_PORTAL の MTG 画面で「このミーティングは Google Meet で実施されます」と明示し、
    - 「Google Meet に参加」ボタンから `meeting_join_url` をそのまま開けるようにする。
- リマインド：
    - `cron_meeting_reminders.php` により、前日／1 時間前等のタイミングでリマインド通知を行う。
    - 通知には必ず `meeting_join_url` を含め、オーナー／スタッフがワンクリックで参加できる前提とする。
- オーナー・スタッフは Google アカウントの有無に応じて、それぞれの環境（ブラウザ／アプリ）から参加する。
    - アカウント作成支援やブラウザ設定などは、本システムのスコープ外とする。

### 11.4 将来拡張

- 将来、Google Calendar／Meet API と連携し、
    - MTG 作成・日程変更・キャンセルを自動で反映する運用に移行する可能性がある。
- その場合も、「MTG の最終責任は HQ にある」ことを維持し、
API エラー時の fallback（手動 URL 共有 等）を必ず用意する。
