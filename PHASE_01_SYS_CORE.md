# PHASE_01_SYS_CORE.md  
加盟店サポート自動化プロジェクト v1.4  
**Phase 1 — SYS_CORE（共通基盤）仕様書**  
Version: v1.4

---

## 1. 目的・範囲

本フェーズでは、加盟店サポート自動化プロジェクト v1.4 の **共通基盤（SYS_CORE）** を定義する。

- 全サブドメイン（help./line./shop./ai./assets.）で共通して利用する基盤層  
- パラメータ化ポリシーに基づき、固定値を **DBパラメータ化** し後から変更可能に  
- 通知（ChatWork / Mail / SMS）、AIコア、外部API（STORES / RemoteLock※拡張ポイント / OpenAI / LINE）に共通する **抽象レイヤー** を確立  
- マルチテナント前提の **shop単位の設定・ログ・権限モデル** を確立  
- 後続フェーズ（HELP_UI / LINE_UI / SHOP_PORTAL / AI_INBOX 等）が **SYS_COREに依存する設計** を可能にする

---

## 2. 前フェーズからの継承ポイント

### 2.1 ドメイン構成

- help.self-datsumou.net（顧客AIチュートリアル / Web）
- line.self-datsumou.net（LINEチュートリアル）
- shop.self-datsumou.net（加盟店ポータル）
- ai.self-datsumou.net（本部AI-INBOX）
- assets.self-datsumou.net（静的アセット / CDN）
- root: self-datsumou.net（メール送信ドメイン）

### 2.2 外部サービス

- STORES予約API（店舗ごとにAPIキー保持）
- RemoteLock API（将来接続するための拡張ポイント）
- OpenAI API（分類 / 回答生成）
- ChatWork API（本部向け通知）
- Twilio（SMS送信）
- LINE Messaging API / LIFF（本人認証）

### 2.3 DB構造の継承

- ai_core_params / ai_core_templates  
- shop_secrets（各種 API Key）  
- customer_links（LINE ID ↔ STORES顧客ID）  
- audit_logs / tutorial_feedbacks / operator_mail_logs  

> 詳細は `DB_FIELDS.md` を正とする。

---

## 3. 新規決定事項（確定）

---

### 3.1 パラメータ管理モデル（A: 決裁済）

#### ■ 方針  
固定値（通知タイミング、AI温度、閾値、リトライ回数、特例フラグ等）は  
**すべて DB で管理し、後から編集可能にする。**

#### ■ パラメータ解決順  
1. shop_params（店舗別上書き）  
2. sys_params（全体共通）  
3. .env（最終デフォルト）

#### ■ テーブル案

```
sys_params(id, key, value, type, description, updated_by, updated_at)
shop_params(id, shop_id, key, value, type, description, updated_by, updated_at)
```

#### 🔧 パラメータ候補

- 保存先: sys_params / shop_params  
- 種別: string / int / float / bool / json  
- 編集権限: hq_admin のみ  
- ルール: **DB管理 + 管理画面編集** 前提

---

### 3.2 通知基盤（Notification Core）  
（B: キュー方式で決裁済）

#### ■ 方針  
ChatWork / Mail / SMS を **共通インターフェースで扱い**  
実際の送信は **notification_queue + cron** による非同期方式。

#### ■ テーブル案

```
notification_queue(
  id, channel, shop_id, to, payload_json,
  priority, status, error_message,
  scheduled_at, sent_at, created_at, updated_at
)
```

#### 🔧 パラメータ候補

- キュー実行間隔: 1分  
- 再送回数: 3回  
- タイムアウト: 3〜10秒  
- 失敗ログ保存期間: 180日  
- デフォルトFromメール: noreply@self-datsumou.net  

---

### 3.3 外部API共通レイヤー  
（C: 共通APIレイヤーで決裁済）

#### ■ 方針  
STORES / RemoteLock（将来）/ OpenAI / LINE / Twilio を  
**共通の入口 `call_api()`** で扱う。

```
call_api($service, $method, $endpoint, $params, $options)
```

#### ■ STORES予約API  
- APIキーは **店舗ごとに保持**  
- call_api('stores') 内で shop_id → APIキーを解決

#### ■ RemoteLock  
- 今回は **未実装**  
- 将来、別エンジニアが作ったアプリを  
  `call_api('remotelock')` に差し込めるように拡張ポイントを確保

#### ■ APIログ（共通）

```
api_call_logs(
  id, service, shop_id, endpoint,
  request_json_masked, response_json_masked,
  http_status, duration_ms, result,
  error_message, created_at
)
```

#### 🔧 パラメータ候補

- ベースURL: sys_params  
- タイムアウト: 5秒（OpenAIのみ10〜15秒）  
- リトライ: 1回  
- 保存期間: 90日  

---

### 3.4 例外・監査ログ  
（E: 保存期間決裁済）

#### ■ 方針  
例外・APIエラーは `system_alerts` に記録し、  
必要に応じて ChatWork / メール通知。

```
system_alerts(
  id, level, category, message,
  context_json, is_notified, created_at
)
```

#### 🔧 保存期間

- APIログ: 90日  
- 通知ログ: 180日  
- システムアラート: 365日  

---

### 3.5 時刻ポリシー（D: 決裁済）

- **DB保存 = UTC**  
- **表示 = Japan Time（JST）**  
- 外部APIレスポンスは保存前にUTCへ変換

---

### 3.6 LINE ID ↔ STORES顧客ID  
（ログではなく基幹データ → 永続保存）

- 削除は「本人からの削除依頼時のみ」  
- 自動削除は行わない  
- 保存期間の議論対象外（ログとは別物）

---

## 4. データ構造 / I/O定義

### 4.1 新規テーブル一覧

- sys_params  
- shop_params  
- notification_queue  
- api_call_logs  
- system_alerts  

### 4.2 resolve_param() I/O

```
resolve_param(shop_id, key):
  1. shop_params から検索
  2. sys_params から検索
  3. .env を参照
  return value
```

---

## 5. 内部フロー

### 5.1 通知フロー

1. notify() が呼ばれる  
2. notification_queue に pending 追加  
3. cron が pending を送信  
4. 成功 → sent  
5. 失敗 → error  
6. critical は system_alerts + ChatWork通知  

---

## 6. 通知 / 外部API インターフェース

### 6.1 notify()

```
notify($channel, $to, $template_key, $vars = [], $options = [])
```

### 6.2 call_api()

```
call_api('stores', 'GET', '/reservations', [...])
call_api('openai', 'POST', 'chat.completions', [...])
call_api('remotelock', 'GET', '/pins', [...])  // 将来追加
```

---

## 7. セキュリティ・権限

- hq_admin … パラメータ編集 + 全ログ閲覧  
- hq_staff … 閲覧のみ  
- shop_owner … 自店舗ログのみ閲覧  
- APIキーは shop_secrets / env に集約  
- ログ出力はマスキング必須  

---

## 8. 未決課題・次フェーズ引き継ぎ

### 8.1 未決課題

- 中間スコープ（brand / region）の必要性  
- アラート通知レベルの初期値微調整  

### 8.2 引き継ぎ

- HELP_UI / LINE_UI：固定値は **すべて param 解決関数から取得**  
- AI_CORE：  
  - モデル系 → ai_core_params  
  - タイムアウト・リトライ系 → SYS_COREパラメータ  

---
