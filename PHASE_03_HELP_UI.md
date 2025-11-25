# PHASE_03_HELP_UI — 顧客向け Web チュートリアル仕様（v1.4｜HELP_UI コア）

## 1. ゴール・スコープ

### 1.1 ゴール

HELP_UI（[help.self-datsumou.net](http://help.self-datsumou.net/)）は、以下を満たす顧客向け一次対応レイヤとする。

- FAQカード＋AIチャットを通じて、顧客の大半の問い合わせを **自己解決** させる。
- STORES予約・会員情報、店舗運営ルール（shops / shop_options / sys_params）、店舗別リンク（shop_links）を参照し、
店舗ごとに適切な案内を行う。
- 人手対応（店舗・本部）は「二次対応」とし、チャット内フォームから tickets 経由でエスカレーションする。

### 1.2 スコープ

本仕様は以下を対象とする。

- [help.self-datsumou.net](http://help.self-datsumou.net/) の UI 構造（スマホ／PC）
- HELP_UI から tutorial_api を通じて行うやりとりの I/O 構造
- セッション／ステートマシン構造
- DB BOX（shops / shop_links / shop_options / shop_internal_info / shop_secrets / sys_params / tickets）との責務分担

業務ルール・文面・各種しきい値は、別ドキュメント：

- `HELP_UI_POLICY.md`
- `HELP_UI_DEFAULT_TEXTS.md`
- `HELP_UI_DEFAULT_PARAMS.md`

にて定義・管理する。

---

## 2. 関係コンポーネントと依存関係

### 2.1 レイヤ構造

- HELP_UI（本ドキュメント）
    - スマホ前提チャット UI（HELP_TOP / HELP_CHAT）
    - tutorial_api：`/api/tutorial/message` 単一エンドポイント
- 共通レイヤ
    - SYS_CORE（sys_params / api_call_logs / sys_events / tickets 等）
    - AI_CORE（intent分類・回答生成・decisionフラグ）
    - API_COMM（STORES・RemoteLOCK Stub・通知クライアント）
- DB_CORE（Phase0）
    - owners / shops / shop_secrets / shop_links / shop_internal_info / options_master / shop_options / sys_params / sales_records / ai_* など

HELP_UI は、AI_CORE／API_COMM／SYS_CORE／DB_CORE を**直接意識しない**。
全て tutorial_api 経由でやりとりする。

---

## 3. UI 構造（骨）

### 3.1 レイアウト

- スマホ（メイン利用デバイス）：
    - 1カラム構成
    - 上から
        - ヘッダー
        - コンテキスト帯（店舗／代表FAQ）
        - メッセージタイムライン
        - 下部入力バー（テキスト＋メディア添付ボタン）
- PC・タブレット：
    - 2カラム構成
    - 右：スマホと同じチャットUI
    - 左：FAQカテゴリ／代表質問／店舗サマリなどのメニュー領域

### 3.2 コンポーネント

- ヘッダー
    - 戻るボタン
    - タイトル（店舗名など）
    - サブタイトル（オンラインサポート説明）
- コンテキスト帯
    - 状況ラベル（例：店舗確定／未特定、STORES経由 等）
    - 代表 FAQ カード（タップで faq_shortcut イベント発火）
- メッセージタイムライン
    - user バブル
    - ai バブル
    - system バナー（AI停止／夜間案内 等）
    - form_bubble（本人確認／エスカレーション／忘れ物 など）
- 下部入力バー
    - テキスト入力欄
    - 送信ボタン
    - メディア添付ボタン
    - （任意）クイックリプライチップ

---

## 4. 店舗特定とスコープ

### 4.1 店舗特定の考え方

- URL から店舗コードが取得できる場合：
    - 正規URL：`https://help.self-datsumou.net/shop/{shop_code}`
    - shop_code から `shops` 行を解決し、以降の問い合わせはその店舗コンテキストで扱う。
- URL から店舗が分からない場合：
    - **ただちに店舗特定を強制しない。**
    - AI はまず質問内容を把握し、店舗に依存しない一般的な質問であれば、
    「店舗未特定」のまま global な FAQ／運用ルールに基づいて回答してよい。

### 4.2 店舗特定が必要になるタイミング

- intent／内容的に店舗依存の回答が必要になった時点で、初めて店舗特定を行う。
    - 例：
        - 特定店舗のアクセス／設備／営業時間 等
        - 利用店舗に紐づく忘れ物／設備クレーム 等
- 店舗特定の手段：
    - URLパラメータ／STORES からの遷移情報／ユーザー入力（店舗名自由入力＋候補チップ）など。

具体的な「どの intent で店舗必須とするか」は `HELP_UI_POLICY.md` 側で管理する。

---

## 5. 本人確認フロー（構造）

### 5.1 本人確認が必要なケース

- STORES予約／会員情報の参照が必要なケース（予約内容確認／解錠番号確認／設備クレームなど）は、
AI_CORE の decision とポリシーに従い **本人確認必須** とする。

### 5.2 フロー構造

1. AI が「本人確認が必要」と判断
2. tutorial_api → form_bubble（form_type="identity"）を返却
    - 項目（氏・名・電話・メールなど）の構造のみ定義
    - 必須／任意・バリデーションルールは `HELP_UI_POLICY.md` ＋ `HELP_UI_DEFAULT_PARAMS.md` で決定
3. フォーム送信 → API_COMM（STORES 顧客特定）
4. 成功 → identity_token 発行 → 以降のリクエストに同 token を持たせる
5. 失敗 → リトライ or エスカレーション誘導

本人確認に何回トライするか／どの場面でエスカレーションへ切り替えるかは、パラメータで制御する。

---

## 6. セッションとステートマシン（骨）

### 6.1 セッション

- `session_id` により HELP_UI の会話を 1 セッションとして識別
- セッション開始：
    - HELP_CHAT 初回表示時
- セッション終了：
    - self_resolved
    - escalated
    - idle_timeout

セッションの開始／終了イベントは `sys_events` に記録する。

### 6.2 ステート一覧

- STATE_NEW
- STATE_GREETING
- STATE_NORMAL_AI
- STATE_AWAIT_IDENTITY
- STATE_AWAIT_FORM
- STATE_FALLBACK
- STATE_ESCALATED
- STATE_SELF_RESOLVED
- STATE_ENDED

各ステートと遷移は tutorial_api 内のステートマシンで実装する。
「何ターンでエスカレーション提案に切り替えるか」等の閾値はパラメータとする。

---

## 7. tutorial_api I/O 構造（骨）

### 7.1 エンドポイント

- URL: `/api/tutorial/message`
- Method: `POST`
- 認証・CSRF 対策は SYS_CORE の共通仕様に準拠。

### 7.2 リクエスト構造（概要）

```json
{
  "channel": "help",
  "version": "v1",
  "shop_code": "ngy_kanayama" or null,
  "session_id": "uuid",
  "event_type": "user_message" | "faq_shortcut" | "form_submitted" | "system",
  "payload": { ... },      // event_type ごとに異なる
  "client_meta": { ... },
  "identity_token": "..."  // 任意
}

```

### 7.3 レスポンス構造（概要）

```json
{
  "session_id": "uuid",
  "state": "STATE_NORMAL_AI",
  "mode": "ai" | "fallback",
  "messages": [
    // ai_message / system_banner / form_bubble 等
  ],
  "meta": {
    "ai_available": true,
    "api_comm_available": true,
    "ai_decision": {
      "intent": "...",
      "super_category": "...",
      "parent_category": "...",
      "escalation_required": false,
      "identity_required": false
    }
  }
}

```

`messages` の具体的な type（ai_message / form_bubble 等）の JSON 形は、

UI 実装との I/O 契約として別途 JSON Schema で定義する。

---

## 8. エスカレーション（構造）

- エスカレーション発生条件：
    - AI_CORE decision（escalation_required）
    - ルールベースのしきい値（ターン数等）
    - ユーザーからの「人に対応してほしい」明示
- エスカレーション時：
    - `form_bubble (form_type="escalation" or "lost_item")` を表示
    - フォーム送信時に tickets レコードを作成
    - チケット詳細：
        - source_channel = "help"
        - source_type = "customer_help_tutorial"
        - shop_id / owner_id / customer_id / priority / subject / summary 等
- 具体的なルーティング（店舗／本部／両方）や priority 判定ロジックは `HELP_UI_POLICY.md` 側で定義する。

---

## 9. メディア添付（構造）

- HELP_UI は画像・動画の添付をサポートできる構造を持つ。
- ファイル実体は assets. 配下の API でアップロードし、`media_id` として扱う。
- tutorial_api には `media_ids` 配列で参照を渡す。
- `tickets` や AI_INBOX では `media_ids` を元に閲覧リンクを生成する。

1 メッセージ・1 セッションあたりのメディア上限や保存日数は、

すべてパラメータとして管理する（`HELP_UI_DEFAULT_PARAMS.md`）。

---

## 10. 障害時 fallback（構造）

- AI_CORE 障害／API_COMM 障害検知：
    - per-request エラー
    - `help_ui.force_ai_off` / `help_ui.force_api_comm_off` フラグ
- fallback モード：
    - mode = "fallback"
    - system_banner で現状を明示
    - 直チケット用 form_bubble を提示
- チャット入力欄は原則閉じない。
    - 送信された内容は tickets 経由で人手対応へ回す。

具体的なメッセージ文面は `HELP_UI_DEFAULT_TEXTS.md` にて定義する。

---

## 11. BOX マッピング（責務）

- shops
    - 店舗基本情報（名前・住所・電話・営業時間など）
- shop_links
    - 店舗別外部リンク（Googleマップ／STORESマイページ／LINE追加／忘れ物フォームなど）
- options_master + shop_options
    - 店舗ごとの運用フラグ（解錠ルール／再入室ルール／夜間電話可否など）
- shop_internal_info
    - 店舗内部向けメモ（鍵／緊急連絡先等）※顧客には出さない
- shop_secrets
    - 機密情報（外部サービスの親アカ／APIキーなど）
- sys_params
    - グローバル・店舗別の HELP_UI パラメータ（挙動・しきい値・メッセージキーなど）
- tickets
    - 二次対応（人手対応）の SO R

---

## 12. 依存ドキュメント

- 業務ルール・トーン・NG表現：`HELP_UI_POLICY.md`
- 初期テキスト：`HELP_UI_DEFAULT_TEXTS.md`
- 初期パラメータ：`HELP_UI_DEFAULT_PARAMS.md`

本ファイルは「構造」と「責務分担」に限定し、

運用により変更されうる内容は他の md に切り出す。
