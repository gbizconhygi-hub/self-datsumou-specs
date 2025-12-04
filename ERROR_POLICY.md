# ERROR_POLICY.md — v1.4 エラー設計ポリシー

このドキュメントは、v1.4 システム全体で共通利用する

- エラーレスポンスの形式
- エラーコードと HTTP ステータスの対応
- ログ／監査との関係

を定義します。

---

## 1. 共通レスポンス形式

### 1.1 成功レスポンス

原則として、成功時は以下の形式とする。

```json
{
  "ok": true,
  "data": { ... }   // 任意、エンドポイントごとのペイロード
}

```

- `ok`:
    - `true` 固定。
- `data`:
    - エンドポイント固有のデータ。
    - 必要がなければ省略可（`{ "ok": true }`）。

### 1.2 エラーレスポンス

エラー時は以下の形式とする。

```json
{
  "ok": false,
  "error_code": "VALIDATION_ERROR",
  "message": "入力内容に誤りがあります。",
  "details": {
    "field_errors": {
      "email": "メールアドレスの形式が正しくありません。"
    },
    "context": {
      "ticket_id": 1234
    }
  }
}

```

- `ok`:
    - `false` 固定。
- `error_code`:
    - 2. エラーコード一覧 のいずれか。
- `message`:
    - ユーザー／オペレーター向けの簡潔な説明文（日本語）。
    - 内部的なスタックトレースや機密情報は含めない。
- `details`（任意）:
    - `field_errors`: フォーム入力の場合の、フィールド別エラー。
    - `context`: デバッグ・ログ用に補足したい情報（IDなど）。
        
        ※ APIレスポンスに含める範囲は慎重に。詳細はサーバログへ。
        

---

## 2. エラーコード一覧

### 2.1 汎用エラー

| error_code | HTTP | Description |
| --- | --- | --- |
| `INTERNAL_ERROR` | 500 | 予期しないサーバ内部エラー |
| `SERVICE_UNAVAILABLE` | 503 | メンテナンス中／外部サービス障害 |
| `NOT_IMPLEMENTED` | 501 | 未実装の機能 |

### 2.2 認証・権限

| error_code | HTTP | Description |
| --- | --- | --- |
| `AUTH_REQUIRED` | 401 | 認証が必要 |
| `AUTH_FAILED` | 401 | 認証失敗（トークン不正／期限切れなど） |
| `PERMISSION_DENIED` | 403 | 認証済だが権限不足 |

### 2.3 リクエスト／バリデーション

| error_code | HTTP | Description |
| --- | --- | --- |
| `VALIDATION_ERROR` | 422 | 入力バリデーションエラー |
| `BAD_REQUEST` | 400 | リクエスト形式不正／想定外のパラメータ |
| `UNSUPPORTED_MEDIA` | 415 | サポートされていない Content-Type |

### 2.4 リソース系

| error_code | HTTP | Description |
| --- | --- | --- |
| `NOT_FOUND` | 404 | リソースが見つからない |
| `CONFLICT` | 409 | 競合（重複登録など） |
| `GONE` | 410 | 既に削除済み／利用不可 |

### 2.5 外部API・通信系

| error_code | HTTP | Description |
| --- | --- | --- |
| `EXTERNAL_API_ERROR` | 502 | STORES / RemoteLOCK / SMS 等、外部API呼び出しでのエラー全般 |
| `EXTERNAL_API_TIMEOUT` | 504 | 外部API呼び出しのタイムアウト |
| `RATE_LIMITED` | 429 | レート制限超過 |

---

## 3. OP_MAIL 専用エラーコード

OP_MAIL モジュールでは、電話代行メールの Webhook 受信と分類に関する

専用のエラーコードを定義する。

| error_code | HTTP | Description |
| --- | --- | --- |
| `OP_MAIL_INVALID_TOKEN` | 401 | `X-Op-Mail-Token` が不正／設定値と異なる |
| `OP_MAIL_INVALID_PAYLOAD` | 400 | 必須フィールド欠落（provider_message_id / subject / to など） |
| `OP_MAIL_DUPLICATE` | 200 | `provider_message_id` が既に処理済み（idempotentとして扱う） |
| `OP_MAIL_SHOP_UNKNOWN` | 200 | `to[]` から店舗を一意に特定できず、`shop_id=NULL` でログのみ作成 |
| `OP_MAIL_CLASSIFY_ERROR` | 500 | AI_CORE 呼び出し失敗／応答不正 |
- `OP_MAIL_DUPLICATE` / `OP_MAIL_SHOP_UNKNOWN` は HTTP 的には 200 を返し、
    
    `{"ok": false, "error_code": "..."}` のように論理エラーとして扱ってもよい。
    
    （Webhook 側に再送させないため）
    

---

## 4. HTTP ステータスとの対応方針

### 4.1 基本方針

- 2xx: 正常系
    - 200: 成功
    - 201: 作成成功（必要であれば）
- 4xx: クライアント起因のエラー
    - 400: 形式不正 / 意味不明なリクエスト
    - 401: 認証されていない
    - 403: 認証済だが権限がない
    - 404: リソースが存在しない
    - 409: 競合（例: 重複登録など）
    - 422: 入力バリデーション不正
    - 429: レート制限超過
- 5xx: サーバ起因 or 外部要因
    - 500: 予期しないサーバ内部エラー
    - 502: 外部APIエラー
    - 503: メンテナンス／一時停止
    - 504: 外部APIタイムアウト

### 4.2 Webhook系特有の扱い

- 再送を避けたい論理エラー（`OP_MAIL_DUPLICATE` など）は、
    
    HTTP 200 で受けつつ `{"ok": false, "error_code": "..."}`
    
    を返す運用も許容する。
    
- 再送してほしい場合（ネットワークエラー／一時障害）は、
    
    明確に 5xx を返す。
    

---

## 5. ログ・監査との関係

### 5.1 sys_events

- エラー発生時は、HTTP レスポンスだけでなく `sys_events` にも記録する。
- 推奨フィールド例:
    - `event_type` = `api_error`, `op_mail.ingest_failed` など
    - `severity` = `warning` / `error` / `critical`
    - `error_code` = 上記 error_code
    - `context` = endpoint, ticket_id, shop_id 等

### 5.2 audit_logs

- ユーザー操作に伴うエラーや重要な状態変更（例: 「対応不要で完了」による close）は、
    
    `audit_logs` に操作履歴として残す。
    
- ログには「誰が」「いつ」「どのチケットに対して」「どのエラーを確認したか」までを含める。

---

## 6. フロントエンドでの扱い（簡単なガイド）

- `error_code` ごとに表示方針の例:

| error_code | UIレベルでの扱い例 |
| --- | --- |
| `VALIDATION_ERROR` | フォームの各フィールドにエラーメッセージ表示 |
| `AUTH_REQUIRED` | ログイン画面へのリダイレクト |
| `PERMISSION_DENIED` | 「権限がありません」メッセージ＋行動提案 |
| `NOT_FOUND` | 「該当データが見つかりません」 |
| `OP_MAIL_SHOP_UNKNOWN` | 「店舗を特定できません。対象店舗を選択してください。」 |
| `INTERNAL_ERROR` | 「エラーが発生しました。時間をおいて再度お試しください。」+ ログID表示 |

---

## 7. 運用ポリシー

- 新しいエラーケースを追加する場合は、
    - ここ（ERROR_POLICY.md）に `error_code` と意味を追記
    - 対応する PHASE_xx_* で「どの場面でそのエラーを返すか」を記述
- 既存の `error_code` を変更／削除する場合は、
    - フロント／Cron／ログ分析などで利用している箇所への影響を確認すること。
