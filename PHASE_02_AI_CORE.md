# PHASE_02_AI_CORE.md  
**加盟店サポート自動化プロジェクト v1.4**  
**フェーズ2：AIコア（AI_CORE）仕様設計書 — Final Draft**

---

# 1. 目的・範囲

本仕様は、help./line./shop./ai. を含む全チャネルに共通する  
**AIコア（AI_CORE）＝単一の頭脳** の設計仕様を定義する。

対象範囲：

- 顧客・加盟店・本部の全問い合わせを共通AIコアで処理  
- 意図分類（カテゴリ分岐）  
- サブカテゴリ推定  
- ナレッジ検索  
- 回答生成（OpenAI）  
- STORES予約API参照  
- エスカレーション制御  
- ログ・分析指標

対象外：

- UI設計（HELP_UI / LINE_UI / SHOP_UI / AI_UI）  
- RemoteLOCK連携（PHASE01にて今回非対応と決定）  
- STORES予約の自動ポーリング（次フェーズ）

---

# 2. 前フェーズからの継承ポイント（PHASE_01_SYS_CORE）

- **チャネル統合設計**：help./line./shop./ai. → すべて同じ AIコアを呼び出す  
- **共通APIレイヤー**：AIコアから外部APIを直接叩かず、SYS_COREのAPIレイヤー経由  
- **通知はキュー方式**：Mail / Chatwork / SMS はすべて非同期処理  
- **設定値は原則パラメータ管理**  
  ※あなたの決裁で「危険カテゴリの自動回答禁止」等は AIコア固定ルール化  
- **すべての問い合わせは tickets / ticket_messages に集約**

---

# 3. 新規決定事項（あなたの決裁反映）

---

## 3-1. Super Category（最上位カテゴリ・固定）

AIコアは最初に「誰向けの質問か」を判別し、ロール別の応答方針を適用する。

```
super_category:
  - customer（顧客）
  - owner（加盟店）
  - hq（本部）
```

※ **AI任せにせず、チャネル側で必ず指定すること。**

---

## 3-2. Parent Category（大分類・AIコア固定）

AIの初動判断・自動回答可否・エスカレーション判断に使う“固定棚”。

| parent_category | 説明 | 自動回答 |
|----------------|------|-----------|
| faq_basic | 基本FAQ・利用方法 | ◎可能 |
| reservation_related | 予約・キャンセル | ◎可能（STORES参照） |
| operation_store | 店舗運営・売上相談 | ◎可能 |
| trouble_technical | 技術トラブル | △状況次第 |
| trouble_customer | 顧客トラブル | ×禁止 |
| contract_fc | 契約・解約・ロイヤリティ | ×禁止 |
| others | 判定不能 | ×確認必須 |

---

## 3-3. Detail Category（サブカテゴリ・柔軟追加可）

- サブカテゴリは **本部管理UIで追加・修正可能**  
- AIは「サブカテゴリ候補」を提案  
- 最終確定は人間が行う

例：
- `reservation_related/no_show`  
- `operation_store/marketing_consulting`  
- `trouble_customer/payment_issue`

---

## 3-4. 自動回答の可否ルール（AIコア固定ロジック）

### ◎ 自動回答「可能」
- FAQに答えがあるもの  
- テンプレートに該当  
- マニュアル参照で完結  
- 過去ログからパターン化済み  

### ✕ 自動回答「禁止」
- クレーム  
- 危険行為  
- 破損・損害  
- 未払い  
- 法律・炎上リスク  
- 契約・解約・違約金  
- AIの自信度が極端に低い  
- 親カテゴリが `trouble_customer` / `contract_fc`

※ 本部管理UIから変更不可（固定ルールとして実装）

---

## 3-5. チャネル別の応答方針（あなたの戦略を反映）

### help. / line.（顧客向け）
- 原則100％ AI自動回答  
- STORES予約IDがあれば必ず該当店舗を特定  
- 店舗固有判断が必要 → 自動で店舗チケット化  
- 非会員の重いクレーム → 本部へエスカレーション  
- 攻撃的文言 → Chatworkへアラート

### shop.（加盟店 → 本部）
- マニュアル内で完結するものはAI 100％回答  
- 緊急案件のみ本部へ  
- マーケ相談 → 予約誘導

### ai.（本部内部）
- 内部向けのレビュー・チューニング用

---

## 3-6. STORES予約API連携（今回のフェーズで確定）

### 今回実施する内容（参照のみ）
- 予約ID・会員判定  
- キャンセル可否（1時間ルール）  
- 予約ステータス  
- 店舗ID判定

### AIの自動案内
- キャンセル可能 → 会員自身で操作案内  
- キャンセル不可 → 店舗へ手動キャンセル指示（チケット化）  
- 予約整合性エラー → 店舗アラート

### 次フェーズ以降（今回非対象）
- STORES自動ポーリング機能  
- RemoteLOCK連携  
- 予約補正ロジック

---

## 3-7. LLMパラメータ（本部管理UIで可変）

- temperature  
- max_tokens  
- 丁寧さ  
- 絵文字使用頻度  
- 禁止ワード  
- 文体（formal / friendly）

※ 店舗オーナーは編集不可（あなたの決裁）

---

## 3-8. AI固定応答方針（Bot感NG・あなたの追加要望）

AIコアは **常に人間味を最優先し、ボット感を厳禁とする**（ハードルール）。

### （1）ボット感の禁止項目
- 無機質・事務的・冷たい文章  
- 「私はAIです」などの自己開示  
- テンプレ棒読み調  
- 文語体連投  
- 命令文の羅列

### （2）ロール別トーン（super_category別）

#### 顧客向け（customer）
- 柔らかく丁寧  
- 若干の口語表現  
- 安心感を与える  
- 適度に顔文字・絵文字🙂  
- 例）  
  「大丈夫ですよ！こちらで確認してみますね✨」

#### 加盟店向け（owner）
- 上から目線禁止  
- マニュアル文体禁止  
- 親身・寄り添いトーン  
- 例）  
  「いつも運営ありがとうございます。状況を整理すると…」

#### 本部向け（hq）
- 端的で論理的だが無機質禁止  
- 内部メモに向いた文章

### （3）チューニング可否
- **可**：tone, temperature, 丁寧さ, 絵文字頻度, 禁止ワード  
- **不可**：ボット感NG・人間味最優先・ロール別方針

---

# 4. データ構造 / I/O形式

---

## 4-1. AI_CORE_REQUEST（JSON）

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
      "line_user_id": "LINE_123",
      "email": "foo@example.com"
    }
  }
}
```

---

## 4-2. AI_CORE_RESPONSE（JSON）

```jsonc
{
  "status": "ok | escalated | error",
  "ticket_id": "TCK_YYYY",
  "message_id": "MSG_RESP",
  "reply_text": "AIが生成した本文",
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
    "note": "人間向け補足"
  },
  "logs": {
    "log_id": "LOG_XXXX",
    "used_kb": ["KB_001", "KB_002"],
    "self_resolved": true,
    "resolution_path": "auto"
  }
}
```

---

## 4-3. テーブル構造（追加分含む）

### ai_intents（固定＋拡張カテゴリ）
| field | description |
|-------|-------------|
| intent_code | `reservation_related/no_show` |
| parent_code | `reservation_related` |
| super_category | `customer` / `owner` / `hq` |
| is_core_parent | boolean |
| is_custom | boolean |
| routing_policy | `auto_ok` / `human_required` |

---

### ai_profiles（本部編集可能）
- tone  
- temperature  
- emoji_level  
- max_tokens  
- banned_phrases

---

### store_ai_profiles（店舗プロファイル）
- 店舗固有設定  
※ オーナーは閲覧のみ、編集不可

---

### ai_logs（セルフ解決記録含む）

| field | description |
|-------|-------------|
| log_id | PK |
| ticket_id | FK |
| user_role | customer / owner / hq |
| parent_intent | |
| detail_intent | |
| confidence | |
| reply_type | auto / draft / human_only |
| feedback | like/dislike/修正 |
| summary | マスク済要約 |
| raw_text | 30日で自動削除 |
| self_resolved | true/false |
| resolution_path | auto / auto_with_store / auto_with_hq / draft_fixed_by_owner / draft_fixed_by_hq / unresolved |

---

# 5. AIコア処理フロー（共通）

1. チャネル層がメッセージ受信  
2. 既存チケットへ紐付け or 新規チケット作成  
3. `super_category (user_role)` を必ず付与  
4. AI_CORE_REQUEST を送信  
5. AIコア処理  
   - 正規化  
   - STORES予約参照  
   - 親カテゴリ分類  
   - サブカテゴリ候補推定  
   - ナレッジ検索  
   - LLM回答生成（ボット感NGルール適用）  
   - 自信度判定  
   - エスカレーション判定  
6. AI_CORE_RESPONSE返却  
7. 店舗・本部へのルーティング  
8. self_resolved 判定 → ai_logs 保存

---

# 6. 通知 / 外部連携

- **Chatwork**：危険行為・破損・攻撃的文言  
- **Mail/SMS**：緊急連絡・重要通知  
- **STORES予約API（参照のみ）**  
- **RemoteLOCK**：今回非対象

---

# 7. セキュリティ・権限

- 個人情報は AI に直接渡さない  
  - 名前 → “A様”  
  - 電話番号は “電話番号あり” 程度に抽象化  
- rawログ：30日で削除  
- summaryログ：1〜3年保存  
- AI設定編集権限は本部のみ  
- 店舗オーナーは閲覧のみ

---

# 8. 未決課題 / 引き継ぎ

- STORESポーリング機能（フェーズ外）  
- ナレッジ編集UIの細部  
- 絵文字頻度の最適化ロジック（顧客向け）  
- 加盟店向け“寄り添い文例ライブラリ”の整備  
- AIチュートリアル層（Phase3）でUI/UX統合

---

# ✔ 決裁済み要点まとめ

- 最上位カテゴリ（顧客/加盟店/本部） → **導入確定**  
- 親カテゴリ7分類 → **固定確定**  
- サブカテゴリ → **本部UIで自由追加**  
- 危険カテゴリの自動回答禁止 → **AIコア固定**  
- 自動化を最大化する基本方針 → **確定**  
- Bot感NG（人間味最優先） → **AIコア固定**  
- self_resolved / resolution_path 追加 → **確定**  
- rawログ30日削除 → **確定**  
- STORES参照のみ → **確定**  
- 店舗オーナーのAI設定編集不可 → **確定**

---
