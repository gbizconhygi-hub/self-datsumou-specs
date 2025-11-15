# PHASE_02_AI_CORE.md
加盟店サポート自動化プロジェクト v1.4  
フェーズ2：AIコア（AI_CORE）仕様設計書 — Final

---

## 1. 目的・範囲

本フェーズでは、help./line./shop./ai. の全チャネルで共通利用する  
AIコア（AI_CORE）＝共通AI頭脳 の仕様を定義する。

対象：
- 意図分類（インテント）
- 回答生成（LLM）
- ナレッジ検索
- 自動回答／エスカレーション判定
- STORES予約API参照
- AIログ保存
- 本部側チューニングUIとの接続

対象外：
- UIデザイン（HELP_UI / LINE_UI / SHOP_UI）
- STORESポーリング同期（次フェーズ）
- RemoteLOCK 連携（今回非対応）

---

## 2. 前フェーズからの継承ポイント（PHASE_01_SYS_CORE）

- help/line/shop/ai はすべて同じ AI_CORE を利用する。
- 外部APIは AI_CORE から直接呼ばず SYS_CORE の共通APIレイヤー経由とする。
- 通知（Mail／ChatWork／SMS）はすべてキュー方式。
- 設定値は原則 DB パラメータとして持ち、本部UIから編集可能とする。
  ※危険領域は AI_CORE 固定ロジックとする。
- 問い合わせは `tickets` / `ticket_messages` に統一管理する。

---

## 3. 新規決定事項

### 3-1. Super Category（最上位カテゴリ）

AI誤分類防止のため、最上位にユーザ種別を固定。

super_category:

customer

owner

hq

yaml
コードをコピーする

チャネル側 input として必須化する。

---

### 3-2. Parent Category（大分類）

リスク判定の基盤。以下 7 区分を固定。

| parent_category      | 説明                           | 自動回答 |
|----------------------|--------------------------------|----------|
| faq_basic            | 基本FAQ                        | ◎        |
| reservation_related  | 予約・キャンセル               | ◎        |
| operation_store      | 店舗運営（加盟店向け）         | ◎        |
| trouble_technical    | 技術トラブル                   | △        |
| trouble_customer     | 顧客トラブル・危険行為         | ×        |
| contract_fc          | 契約・解約・ロイヤリティ       | ×        |
| others               | 判別不能                       | ×        |

---

### 3-3. Detail Category（サブカテゴリ）

- 本部管理UIから追加・編集可能
- AIは候補提案のみ行い、最終確定は本部

例：  
`reservation_related/no_show`  
`operation_store/marketing_consulting`  
`trouble_customer/payment_issue`

---

### 3-4. 自動回答の許可範囲

自動回答 可：
- FAQに答えがある
- 回答テンプレートに一致
- マニュアル参照で機械的に回答可能
- 過去ログでパターン化できている

自動回答 禁止（AI_CORE 固定ロジック）：
- クレーム
- 危険行為・破損・損害
- 未払い・料金トラブル
- 契約・違約金・FC契約
- 法律関連
- 自信度が極端に低い
- super_category と parent_category に矛盾がある

---

### 3-5. チャネル別方針

help/line（顧客）：
- 原則 AI 自動回答
- 店舗判断が必要 → 店舗へ自動チケット化
- 非会員の重いクレーム → 本部へエスカレーション
- 攻撃的文言 → 本部へアラート

shop（加盟店 → 本部）：
- マニュアル回答可能 → AI自動回答
- 緊急案件のみ本部へ
- マーケ相談は予約誘導

ai（本部）：
- 内部レビュー・校閲用

---

### 3-6. STORES予約 API（参照のみ）

AIコアが取得するデータ：
- 予約ID
- 会員判定
- ステータス
- キャンセル可否（1時間ルール）
- 店舗ID

判断ロジック：
- キャンセル可 → 操作案内（AI自動回答）
- キャンセル不可 → 店舗へチケット化
- 整合性エラー → 店舗へアラート

今回対象外：
- STORESポーリング同期
- 自動修正ロジック
- RemoteLOCK連携

---

### 3-7. LLMチューニング

- モデル：OpenAI（本番・検証）
- トーン・温度・文字量は本部UIから変更可能
- 店舗オーナーは編集不可

---

## 4. I/O 定義

### 4-1. AI_CORE_REQUEST

```jsonc
{
  "channel": "help | line | shop | ai",
  "user_role": "customer | owner | hq",
  "tenant_id": "STORE_XXXX",
  "ticket_id": "TCK_YYYY",
  "message_id": "MSG_ZZZZ",
  "message_text": "問い合わせ本文",
  "locale": "ja-JP",
  "meta": {
    "user_id": "USER_XXX",
    "store_plan": "standard | premium",
    "previous_intent": "reservation_related/no_show",
    "is_new_ticket": true,
    "channel_specific": {
      "line_user_id": "abc123",
      "email": "foo@example.com"
    }
  }
}
4-2. AI_CORE_RESPONSE
js
コードをコピーする
{
  "status": "ok | escalated | error",
  "ticket_id": "TCK_YYYY",
  "message_id": "MSG_RESP",
  "reply_text": "AI回答本文",
  "reply_type": "auto | draft | human_only",
  "super_category": "customer | owner | hq",
  "intent": {
    "parent": "reservation_related",
    "detail": "reservation_related/no_show",
    "confidence": 0.83
  },
  "escalation": {
    "required": true,
    "level": "normal | high",
    "target": "owner | hq",
    "note": "補足"
  },
  "logs": {
    "log_id": "LOG_XXXX",
    "used_kb": ["KB_001", "KB_002"]
  }
}
4-3. テーブル定義
ai_intents
field	note
intent_code	例：reservation_related/no_show
parent_code	固定7分類
super_category	customer / owner / hq
is_core_parent	親カテゴリの場合
is_custom	サブカテゴリ
routing_policy	auto_ok / human_required

ai_profiles（本部編集可）
tone

temperature

max_tokens

禁止ワード

アラートワード

store_ai_profiles（店舗は閲覧のみ）
呼称/トーン（必要なら）
※ 店舗オーナーは編集不可

ai_logs
field	note
log_id	
ticket_id	
user_role	
parent_intent	
detail_intent	
confidence	
reply_type	
feedback	like/dislike/修正
summary	長期保存
raw_text	30日後自動削除

5. AIコア処理フロー
チャネルがメッセージ受信

ticket_id を割当

super_category（user_role）を強制付与

AI_CORE_REQUEST 送信

AIコア処理

入力正規化

STORES参照

親カテゴリ分類（固定棚）

サブカテゴリ候補推定

ナレッジ検索

回答生成

自信度判定

エスカレーション判断（role矛盾含む）

AI_CORE_RESPONSE 返却

auto / draft / human_only に応じて分岐

6. 通知 / 外部連携
ChatWork：危険文言／クレーム／予約不整合

Mail・SMS：緊急通知

STORES予約：参照のみ

RemoteLOCK：今回非対応

7. セキュリティ / 権限
個人情報は AI に直接渡さず抽象化

raw_text は 30日で自動削除

summary は 1〜3年保存

LLM設定編集可：本部のみ

店舗オーナーは AI 設定編集不可

8. 未決課題 / 次フェーズへ
STORES ポーリング同期

ナレッジ編集UI

AIチューニングUI

店舗別プロファイルUI

マーケ相談誘導フロー

ステータス
本ファイルはオーナーの「確定」指示により、
PHASE_02_AI_CORE として正式採用とする。
