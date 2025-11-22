# API_COMM（外部API通信レイヤ） - 仕様設計書 (PHASE_02_API_COMM.md)

## 1. 目的・範囲

### 1.1 フェーズ概要

本フェーズ（Phase 2：API_COMM）は、SYS_CORE による HTTP クライアント／ログ基盤を前提に、

- STORES予約 API
- RemoteLOCK （v1.4 ではスタブ）
- 通知系（ChatWork／メール／SMS／LINE／OpenAI）

との連携を **業務単位の I/O 契約としてラップするレイヤ** を定義する。

上位レイヤ（HELP_UI / LINE_UI / SHOP_PORTAL / AI_INBOX / AI_CORE）は、

- 「顧客を特定する」
- 「顧客詳細を取得する」
- 「予約一覧／予約詳細を取得する」
- 「疎通状態を知る」
- 「エスカレーション通知を飛ばす」

といった抽象 API を利用するだけで済むようにし、

外部 API の仕様・パラメータ・レート制限を API_COMM 内で吸収する。

### 1.2 対象とする STORES 予約 API

STORES 側ドキュメントの v202101 を前提とする。

- Base URL`https://api.stores.dev/reserve/202101`
- 認証方式`Authorization: Bearer <token>`（事前共有クレデンシャル）

本フェーズで利用するエンドポイントは以下とする（参照系のみ）。

1. `GET /customers`
    - 主なクエリ：
        - `merchant_canonical_ids` : マーチャント絞り込み（カンマ区切り）
        - `email` : 顧客メールアドレス（部分一致）
        - `phone_number` : 顧客電話番号（完全一致・ハイフンなし前提）
        - `name` : 顧客名（部分一致）
        - `page` / `sort` / 各種日時フィルタ など
    - レスポンス：
        - `pagination: { page, total }`
        - `customers: Customer[]`（subscriptions, ticket_books 含む）
2. `GET /customers/{customer_canonical_id}`
    - パス：
        - `customer_canonical_id: integer`
    - レスポンス：
        - `customer: Customer`（+ subscriptions, ticket_books, reservations）
3. `GET /reservations`
    - 主なクエリ：
        - `merchant_canonical_ids`
        - `booking_start_from/to`
        - `booking_made_from/to`
        - `status`（ReservationStatus）
        - `page`, `sort` 等
    - レスポンス：
        - `pagination: Pagination`
        - `reservations: Reservation[]`
4. `GET /reservations/{reservation_canonical_id}`
    - パス：
        - `reservation_canonical_id: integer`
    - レスポンス：
        - `reservation: Reservation`

### 1.3 本フェーズで扱うもの

- STORES予約 API の **参照系呼び出し** を API_COMM 経由で提供
- 「顧客特定」「顧客詳細」「予約一覧」「予約詳細」の業務 API 定義
- STORES 側レート制限を踏まえた **呼び出し頻度削減ポリシー**
- `customer_links`（LINE ID ↔ STORES会員ID 等）の DB スキーマ確定
- 共通レスポンス `ApiCommResult` の定義
- 外部 API 向けタイムアウト／リトライのパラメータ設計
- `system.check_connectivity` の I/O 仕様（v1.4 は手動実行前提）

### 1.4 本フェーズで扱わないもの

- STORES 予約の作成／変更／キャンセル等の **更新系 API 呼び出し**
- RemoteLOCK との実 API 連携（PIN 取得／解錠制御）
- HTTP エンドポイントとして公開される REST API の 4xx/5xx 設計・認証・監視
→ Phase3 以降（HELP_UI / LINE_UI / 管理画面）で実施

---

## 2. 前フェーズからの継承ポイント

### 2.1 SoR / DB 方針

- `db_fields.yaml` に定義された Phase0 基本 DB（owners / shops / shop_secrets / fee_* / shop_* / sales_records 等）を SoR とする。
- 既存 DB と意味が重複する `sys_params` キーは作成しない（二重管理禁止）。
- 設定値の解決優先順位：
    1. 既存 DB（shops / shop_secrets / options_master / shop_options 等）
    2. `sys_params`
    3. `.env`
    4. コード内デフォルト

### 2.2 SYS_CORE との関係

- SYS_CORE の主な責務（抜粋）：
    - `SysConfig`：DB（options_master / shop_options 等）＋ `.env` から設定値を解決
    - `SysLogger`：`api_call_logs` / `sys_events` / `tickets` へのログ書き込み
    - `SysEscalation`：ChatWork／メール／SMS など通知の実行
    - `HttpClient`＋各種 `Client` クラス：
        - `StoresClient`, `ChatworkClient`, `SmsClient`, `LineClient`, `OpenAiClient`, `RemoteLockClient` 等
- HELP / LINE / SHOP / AI などから外部 API を利用する際は、**Clients 経由でのみ HTTP 通信を行う**（直接の curl 使用は禁止）。

> API_COMM は、これら Clients の 一段上のレイヤ として、
> 
> 
> 「顧客特定」「予約取得」といった業務 API をまとめて提供する。
> 

### 2.3 AI_CORE との関係

- AI_CORE は、外部 API の詳細仕様を知らずに、`AI_CORE_REQUEST.context` に渡された情報を前提に会話を制御する。
- LLM へのリクエストは必ず SYS_CORE の `OpenAiClient` 経由とし、
API キーやベースURLは API_COMM も AI_CORE も直接持たない。
- STORES／RemoteLOCK などの具体的な I/O は AI_CORE の外に切り出し、
本フェーズ（API_COMM）で業務 API として定義する。

### 2.4 STORES／RemoteLOCK 連携の全体構想

- STORES 予約 API は、Web（help.）／LINE（line.）における
    - 本人確認
    - 顧客特定
    - 直近予約情報の取得
    に利用する。
- Web（help.）：
    - `store_code`（= shops.code）／`merchant_canonical_id` から店舗を特定
    - メール＋電話番号で STORES 会員を検索
- LINE（line.）：
    - LIFF + 認証により STORES 会員を特定し、
    - `customer_links` に LINEユーザID ↔ STORES会員ID を保存
- RemoteLOCK：
    - v1.4 では実連携を行わず、`RemoteLockGateway` とスタブレスポンスのみ用意
    （実 API は Phase9 以降で実装予定）

---

## 3. 新規決定事項

### 3.1 API_COMM の責務とレイヤ構造

- API_COMM は **純粋な業務サービスレイヤ** とする。
    - HTTP ステータスコードは「外部 API と SYS_CORE の間」でのみ扱う概念。
    - 上位には `ApiCommResult` のみを返し、HTTP 4xx/5xx へのマッピングは Phase3 以降の HTTP 層で実装する。
- ディレクトリ構造（想定）：

```
/shared/
  api_comm/
    ApiComm.php                … ファサード（エントリポイント）
    StoresGateway.php          … STORES向け業務API群
    RemoteLockGateway.php      … RemoteLOCK向けスタブAPI
    NotificationGateway.php    … 通知系API（SysEscalationラッパ）
    SystemHealthGateway.php    … system.check_connectivity 実装（任意）
    Dto/
      ApiCommResult.php        … 共通レスポンス構造（配列でも可）

```

- 上位レイヤ（HELP_UI / LINE_UI / SHOP_PORTAL / AI_INBOX / AI_CORE）は、
    
    原則として `ApiComm::call($operation, ApiCommRequest)` のような形で利用する想定。
    

### 3.2 STORES API 利用ポリシー（レート制限対策含む）

1. **エンドポイントの限定**
    - 本フェーズで STORES に対して利用するのは以下の 4 つのみ：
        - `GET /customers`
        - `GET /customers/{customer_canonical_id}`
        - `GET /reservations`
        - `GET /reservations/{reservation_canonical_id}`
2. **店舗絞り込みの徹底**
    - すべての STORES 呼び出しには、可能な限り `merchant_canonical_ids` を付与し、
        
        1 店舗（1 merchant）に絞った検索を行う。
        
    - `shops` と STORES merchant の対応：
        - `shops.merchant_public_id` と STORES の `merchant_canonical_id` を
            
            `shop_internal_info` または `shop_secrets` で管理する（どちらを使うかは Phase0 決定に従う）。
            
3. **ページ数制限**
    - `GET /customers`, `GET /reservations` のページング：
        - v1.4 では **常に `page=1` のみ取得** する。
        - `pagination.total > 1` の場合は、「候補が多く自動特定不可」と扱う。
    - 自動ページングや 2ページ目以降の取得は行わない。
4. **リクエスト単位キャッシュ**
    - API_COMM 内部で、1 リクエスト（`request_id`）の間における
        
        同一パラメータでの STORES 呼び出しはメモリキャッシュする。
        
    - キャッシュキー：
        - `(shop_id, operation名, 正規化された payload JSON)`
    - キャッシュヒットかどうかは `ApiCommResult.meta.cached` に true/false で記録する。
5. **HTTP 429 等の扱い**
    - STORES 側のレート制限等により 429 が返った場合：
        - `status = "error"`
        - `code = "STORES_RATE_LIMITED"`
        - `error.type = "external_api"`
        - `error.details.external_status_code = 429`
    - 上位レイヤは **即時リトライを行わない** 前提とし、
        
        AI 側では「時間をおいて再度お試しください」「マイページでの確認を案内」する。
        

### 3.3 STORES 検索ポリシー（完全一致＋理由付き NOT_FOUND）

STORES への顧客検索は、**email / phone の完全一致** を前提としつつ、

STORES 側の仕様（email は部分一致、phone_number は完全一致）を API_COMM 内で補正する。

### ポリシー

- `phone_number`
    - ユーザー入力から数字以外を除去 (`0-9` のみ)。
    - STORES にも数字のみで渡す。
    - レスポンスの `primary_phone_number` も同様に数字のみ抽出して完全一致比較。
- `email`
    - STORES には元入力をそのまま渡す（部分一致検索になる）。
    - レスポンスの `customer.email` を `lowercase + trim` で正規化し、
        - それが `payload.email` の正規化と一致するものだけにフィルタリング。
    - 完全一致するもの以外は候補から除外。

### 結果の分類

- `match_status`：
    - `"unique"` : 完全一致フィルタ後に 1件
    - `"not_found"` : 0件、または「自動特定困難」
    - `"multiple_candidates"` : 2件以上
- `not_found_reason`（`match_status = "not_found"` のとき有効）：
    - `"no_record"` : 条件に完全一致するレコードが 0 件
    - `"too_many_candidates"` : 2件以上ヒットし、自動特定できない
    - `"over_page_limit"` : `pagination.total` が許容値を超えたため検索を打ち切った
    - `"none"` : その他（`match_status != "not_found"` の場合）

v1.4 の UX では **`not_found` と `multiple_candidates` をひとまとめに「自動特定できません」系の扱い** とするが、

AI プロンプトでは `not_found_reason` を使って説明を差し分けられる。

### allow_multi_match

- `payload.allow_multi_match`（bool）を I/O に含める。
    - v1.4 では **常に `false`** として扱う。
    - 将来 `true` としたとき、`candidates` を UI／AI に渡し、
        
        対話的な絞り込みフローを追加実装できるようにする。
        

### 3.4 `customer_links` テーブル（チャネル汎用化）

STORES 顧客とチャネル固有ID（LINE userId など）を紐づけるためのテーブル。

```sql
CREATE TABLE customer_links (
  id                 BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
  shop_id            BIGINT       NOT NULL,    -- 対象店舗（shops.id）
  channel_type       VARCHAR(32)  NOT NULL,    -- 'line', 'help', 'web_account' など
  external_user_id   VARCHAR(255) NOT NULL,    -- 例: LINE userId, Web会員ID
  stores_customer_id VARCHAR(64)  NOT NULL,    -- STORES customers.canonical_id の文字列表現
  created_at_utc     DATETIME     NOT NULL,
  updated_at_utc     DATETIME     NOT NULL,
  UNIQUE KEY uniq_customer_link_1 (shop_id, channel_type, external_user_id),
  UNIQUE KEY uniq_customer_link_2 (shop_id, channel_type, stores_customer_id)
);

```

- v1.4 運用：
    - `channel_type = 'line'` のみを使用。
- 将来的には、Web アカウントなど他チャネルにも対応可能。

### 3.5 ApiCommResult（共通レスポンス形式）

API_COMM は HTTP を意識せず、すべて `ApiCommResult` 形式の PHP 配列（または DTO）を返す。

```json
{
  "status": "success | error",
  "code": "string",              // 'STORES_CUSTOMER_FOUND', 'STORES_CUSTOMER_NOT_FOUND' など
  "message": "string | null",    // ログ／デバッグ向け説明
  "data": { ... },               // 業務データ（APIごとに異なる）
  "error": {                     // status=error の場合のみ
    "type": "validation | external_api | system",
    "details": {
      "field_errors": [
        { "field": "email", "message": "メール形式が不正です" }
      ],
      "external_status_code": 500,
      "external_body": "(マスク済みレスポンス)",
      "retryable": true
    }
  },
  "meta": {
    "request_id": "uuid-string",
    "component": "StoresGateway.resolve_customer_by_contact",
    "latency_ms": 123,
    "cached": false
  }
}

```

- Phase3 以降の HTTP 層では、この `status` / `code` / `error.type` を見て 4xx/5xx にマッピングする。

### 3.6 タイムアウト／リトライ パラメータ

### v1.4 で実装するパラメータ

- 共通（全外部 API 向け）
    - `api_timeout_sec`（int）
    - `api_retry_count`（int）
- STORES 専用上書き枠
    - `stores_api_timeout_sec`（int）
    - `stores_api_retry_count`（int）

解決ルール：

- STORES 呼び出し時：
    - `stores_api_timeout_sec` が定義されていればそれを使用、なければ `api_timeout_sec`
    - `stores_api_retry_count` が定義されていればそれを使用、なければ `api_retry_count`
- その他（ChatWork／SMS／LINE／OpenAI／RemoteLOCKスタブ）：
    - `api_timeout_sec` / `api_retry_count` のみ使用

スコープ：

- v1.4 では `sys_params.scope = 'global'` を基本とし、
    
    将来必要に応じて `scope='shop'` で店舗ごとの個別チューニングを許容する。
    

### 将来候補（v1.4では未実装）

- `stores_api_max_calls_per_request`（int）
    - 1 アプリケーションリクエストの中で STORES を叩ける回数の上限
- `stores_api_read_only`（bool）
    - 将来、更新系 API を導入する際のブロックフラグ

### 3.7 system.check_connectivity（安全寄り STORES 健康診断）

- API_COMM 内に `system.check_connectivity` を実装する。
- 役割：
    - STORES／ChatWork／SMS／LINE／OpenAI／RemoteLOCKスタブ に対して疎通確認を行い、結果を一括で返す。
- STORES に対する健康診断ポリシー：
    1. 1回の `system.check_connectivity` 呼び出しにつき STORES へのリクエストは **1回のみ**
    2. 「代表店舗」の `merchant_canonical_id` を1つ決め、そこに対してのみ `GET /reservations?page=1&merchant_canonical_ids=...` 等の軽量なリクエストを行う
    3. 頻度は運用上「手動実行」レベルに留め、定期監視は将来フェーズで検討

---

## 4. データ構造 / I/O 定義

### 4.1 共通リクエスト: ApiCommRequest

```json
{
  "request_id": "uuid-string | null",
  "channel": "help | line | shop | ai | cron",
  "actor_type": "customer | merchant | hq | system",
  "shop_id": 123,
  "payload": { ... }   // 各 API 固有の入力
}

```

- `request_id` が null の場合は API_COMM 側で生成し、`ApiCommResult.meta.request_id` に入れる。
- 1 つの `request_id` 内で STORES 呼び出しキャッシュを行う。

---

### 4.2 STORES: 顧客特定 `stores.resolve_customer_by_contact`

### 4.2.1 目的

- email／phone_number（＋任意の name）から、**完全一致ポリシー** で顧客を 1件特定する。

### 4.2.2 リクエスト

```json
{
  "request_id": "uuid-or-null",
  "channel": "help | line",
  "actor_type": "customer",
  "shop_id": 123,
  "payload": {
    "email": "user@example.com",          // 任意
    "phone_number": "09012345678",        // 任意（数字のみ推奨）
    "name": "山田 太郎",                  // 任意（ログ用途）

    "allow_multi_match": false,
    "merchant_canonical_id": "mc_xxx"     // 任意：明示指定あれば優先
  }
}

```

- `email` / `phone_number` のいずれかは必須（両方 null は validation error）。

### 4.2.3 内部処理フロー

1. `merchant_canonical_id` の解決
    - `payload.merchant_canonical_id` が null の場合：
        - `shop_id` → `shops` / `shop_internal_info` / `shop_secrets` 等から解決。
2. `/customers` への GET
    - クエリ：
        - `merchant_canonical_ids`: 解決された 1件のみ（文字列）
        - `page`: 1
        - `email`: 指定されていればそのまま
        - `phone_number`: 指定されていれば数字のみ抽出して渡す
3. レスポンスフィルタ
    - `customers` 配列に対して：
        - email 指定あり：
            - `normalized(customer.email) == normalized(payload.email)` のみ残す
        - phone 指定あり：
            - `数字のみ(customer.primary_phone_number) == 数字のみ(payload.phone_number)` のみ残す
4. 件数による判定
    - フィルタ後の件数と `pagination.total` に応じて
        - `match_status` / `not_found_reason` を決定

### 4.2.4 レスポンス（成功時）

```json
{
  "status": "success",
  "code": "STORES_CUSTOMER_FOUND | STORES_CUSTOMER_NOT_FOUND | STORES_CUSTOMER_MULTIPLE_CANDIDATES",
  "data": {
    "match_status": "unique | not_found | multiple_candidates",
    "not_found_reason": "none | no_record | too_many_candidates | over_page_limit",

    "customer": {
      "stores_customer_id": "123",
      "canonical_id": 123,
      "name": "山田 太郎",
      "email": "user@example.com",
      "primary_phone_number": "09012345678",
      "merchant_canonical_id": "mc_xxx",

      "raw": { "...": "..." }  // STORES Customer のサブセット（将来用）
    },
    "candidates": [
      {
        "stores_customer_id": "456",
        "canonical_id": 456,
        "name": "山田 花子",
        "email": "user@example.com",
        "primary_phone_number": "09099998888"
      }
    ],
    "pagination": {
      "page": 1,
      "total": 1
    }
  }
}

```

- `match_status` と `code` の対応：
    - `unique` → `STORES_CUSTOMER_FOUND`
    - `not_found` → `STORES_CUSTOMER_NOT_FOUND`
    - `multiple_candidates` → `STORES_CUSTOMER_MULTIPLE_CANDIDATES`
- v1.4 の UX では `STORES_CUSTOMER_NOT_FOUND` と `STORES_CUSTOMER_MULTIPLE_CANDIDATES` を
    
    ほぼ同様に扱い、「自動特定できない」旨のメッセージを返す。
    

### 4.2.5 エラー時

- validation error（必須項目不足 等）：
    - `status = "error"`
    - `code = "INVALID_REQUEST"`
    - `error.type = "validation"`
- STORES 側 4xx/5xx：
    - `status = "error"`
    - `code = "STORES_CUSTOMER_LOOKUP_FAILED" または "STORES_RATE_LIMITED"`
    - `error.type = "external_api"`

---

### 4.3 STORES: LINE ID から特定 `stores.resolve_customer_by_line_user`

### 4.3.1 目的

- LINE ユーザID（`line_user_id`）から `customer_links` を用いて STORES 顧客を特定する。

### 4.3.2 リクエスト

```json
{
  "request_id": "uuid-or-null",
  "channel": "line",
  "actor_type": "customer",
  "shop_id": 123,
  "payload": {
    "line_user_id": "Uxxxxxxxxxxxx",
    "allow_multi_match": false
  }
}

```

### 4.3.3 内部処理フロー

1. `customer_links` を検索：
    - `SELECT stores_customer_id FROM customer_links WHERE shop_id=? AND channel_type='line' AND external_user_id=?`
2. レコードなし：
    - `link_status = "none"`
    - STORES API は叩かず、そのまま NOT_FOUND 系コードを返す。
3. レコードあり：
    - `stores_customer_id` を用いて `GET /customers/{customer_canonical_id}` を 1回呼び出す。
    - 200 で Customer が取得できた場合：
        - `link_status = "exists_valid"`
    - 404 / 権限エラー等で取得できない場合：
        - `link_status = "exists_broken"`

### 4.3.4 レスポンス

```json
{
  "status": "success | error",
  "code": "STORES_CUSTOMER_FOUND_BY_LINK | STORES_CUSTOMER_LINK_NOT_FOUND | STORES_CUSTOMER_LINK_BROKEN | STORES_CUSTOMER_LOOKUP_FAILED",
  "data": {
    "link_status": "none | exists_valid | exists_broken",
    "link": {
      "stores_customer_id": "123",
      "channel_type": "line",
      "external_user_id": "Uxxxxxxxxxxxx"
    },
    "customer": { ... }   // link_status=exists_valid のときのみ
  }
}

```

- AI／UI 側での推奨メッセージ：
    - `STORES_CUSTOMER_LINK_NOT_FOUND`（link_status=none）：
        - 「まだ連携されていない／初回連携が必要」という前提の案内。
    - `STORES_CUSTOMER_LINK_BROKEN`（link_status=exists_broken）：
        - 「以前の連携が無効になった可能性があるため、再連携が必要」という案内。

---

### 4.4 STORES: 顧客詳細取得 `stores.get_customer`

### 4.4.1 目的

- `stores_customer_id`（Customer.canonical_id）から顧客詳細を取得。

### 4.4.2 リクエスト

```json
{
  "request_id": "uuid-or-null",
  "channel": "help | line | shop | ai",
  "actor_type": "customer | merchant | hq",
  "shop_id": 123,
  "payload": {
    "stores_customer_id": "123"
  }
}

```

### 4.4.3 レスポンス

```json
{
  "status": "success | error",
  "code": "STORES_CUSTOMER_DETAIL_FOUND | STORES_CUSTOMER_DETAIL_NOT_FOUND | STORES_CUSTOMER_LOOKUP_FAILED",
  "data": {
    "customer": { ... Customer フル情報 ... }
  }
}

```

---

### 4.5 STORES: 予約一覧取得 `stores.list_reservations`

### 4.5.1 目的

- 店舗単位で指定条件に合致する予約一覧を取得（将来 RemoteLOCK 実装を見据えた設計）。

### 4.5.2 リクエスト

```json
{
  "request_id": "uuid-or-null",
  "channel": "help | line | shop | ai | cron",
  "actor_type": "customer | merchant | hq | system",
  "shop_id": 123,
  "payload": {
    "booking_start_from": "2025-11-22T00:00:00+09:00",
    "booking_start_to": "2025-11-23T00:00:00+09:00",
    "status": "accepted | pending | canceled_by_host | ...",
    "page": 1
  }
}

```

### 4.5.3 レスポンス

```json
{
  "status": "success | error",
  "code": "STORES_RESERVATIONS_LISTED | STORES_RESERVATIONS_LOOKUP_FAILED",
  "data": {
    "pagination": { "page": 1, "total": 3 },
    "reservations": [
      {
        "canonical_id": 111,
        "booking_service_canonical_id": "bs_xxx",
        "booking_start": "2025-11-22T10:00:00+09:00",
        "customer_name": "山田 太郎",
        "status": "accepted",
        "merchant_canonical_id": "mc_xxx",
        "raw": { "...": "..." }
      }
    ]
  }
}

```

---

### 4.6 STORES: 予約詳細取得 `stores.get_reservation`

### 4.6.1 目的

- `reservation_canonical_id` から予約詳細を取得。

### 4.6.2 リクエスト

```json
{
  "request_id": "uuid-or-null",
  "channel": "help | line | shop | ai",
  "actor_type": "customer | merchant | hq",
  "shop_id": 123,
  "payload": {
    "reservation_canonical_id": "111"
  }
}

```

### 4.6.3 レスポンス

```json
{
  "status": "success | error",
  "code": "STORES_RESERVATION_DETAIL_FOUND | STORES_RESERVATION_DETAIL_NOT_FOUND | STORES_RESERVATION_LOOKUP_FAILED",
  "data": {
    "reservation": { ... Reservation ... }
  }
}

```

---

### 4.7 RemoteLOCK: スタブ `remotelock.get_key_info_stub`

### 4.7.1 目的

- v1.4 時点では RemoteLOCK 実連携は行わないため、
    
    「常にスタブである」ことを上位に伝える。
    

### 4.7.2 レスポンス

```json
{
  "status": "error",
  "code": "REMOTELOCK_STUB_MODE",
  "message": "v1.4ではRemoteLOCK API連携は実装されていません。",
  "data": null,
  "error": {
    "type": "external_api",
    "details": {
      "reason": "stub_only"
    }
  }
}

```

---

### 4.8 通知ラッパ `notify.escalation`

### 4.8.1 リクエスト

```json
{
  "request_id": "uuid-or-null",
  "channel": "help | line | shop | ai | cron",
  "actor_type": "customer | merchant | hq | system",
  "shop_id": 123,
  "payload": {
    "rule_code": "API_ERROR_CRITICAL",
    "summary": "STORES予約APIが3回連続で失敗しました。",
    "details": {
      "component": "StoresGateway.resolve_customer_by_contact",
      "last_status_code": 500
    }
  }
}

```

### 4.8.2 レスポンス

```json
{
  "status": "success | error",
  "code": "ESCALATION_TRIGGERED | ESCALATION_FAILED",
  "data": {
    "rule_code": "API_ERROR_CRITICAL",
    "channels_used": ["chatwork", "sms"],
    "ticket_id": 9876
  }
}

```

---

### 4.9 システム疎通 `system.check_connectivity`

### 4.9.1 リクエスト

```json
{
  "request_id": "uuid-or-null",
  "channel": "ai",
  "actor_type": "system",
  "shop_id": null,
  "payload": {
    "targets": ["stores", "chatwork", "sms", "line", "openai", "remotelock"]
  }
}

```

### 4.9.2 レスポンス

```json
{
  "status": "success | error",
  "code": "HEALTH_OK | HEALTH_WARN | HEALTH_ERROR",
  "data": {
    "stores": {
      "status": "ok | warn | fail",
      "latency_ms": 200,
      "message": "正常に疎通しました"
    },
    "chatwork": {
      "status": "ok | warn | fail",
      "latency_ms": 120,
      "message": "..."
    },
    "sms": { "...": "..." },
    "line": { "...": "..." },
    "openai": { "...": "..." },
    "remotelock": {
      "status": "stub",
      "message": "v1.4ではスタブモードです"
    }
  }
}

```

- STORES の疎通確認は代表店舗 1件に対してのみ行い、1回のリクエストに限定する。

---

## 5. 共通処理フロー

### 5.1 Web（help.）本人確認フロー

1. ユーザーが STORES の予約ページから `help.` に遷移（`store_code` 等で `shop_id` を特定）。
2. メール／電話番号／氏名を入力 → メールアドレスの認証リンク配信（HELP_UI フェーズで定義）。
3. 認証完了後、`stores.resolve_customer_by_contact` を 1回呼び出し、顧客特定。
    - `match_status = unique` → `stores_customer_id` を取得し、必要に応じて `stores.get_customer` も呼ぶ。
    - `not_found` / `multiple_candidates` → AI 側で「自動特定できない」旨を案内し、
        
        よくある原因と STORES マイページ確認を促すテンプレートを利用。
        
4. 取得した `customer` / `stores_customer_id` / 代表予約情報は
    
    `ai_sessions.state_json` 等に保存し、セッション内では再度 STORES を叩かずに使い回す。
    

### 5.2 LINE（line.）本人確認フロー

1. 友だち追加 → 「本人確認」ボタン → LIFF で STORES アカウントに連携。
2. STORES 側で `customer_canonical_id` を取得できたタイミングで、
    - `customer_links (shop_id, 'line', line_user_id) → stores_customer_id` を登録／更新。
3. 通常のトークでは、
    - `stores.resolve_customer_by_line_user` → `stores.get_customer` の順で 1回呼び出し、顧客情報を context に乗せる。
4. リンクが存在しない／壊れている場合は、
    - それぞれ `STORES_CUSTOMER_LINK_NOT_FOUND` / `STORES_CUSTOMER_LINK_BROKEN` をもとに AI が再連携案内。

### 5.3 セッション内での STORES 再照会ルール

- AI_CORE／UI 側の運用ルールとして：
    1. **最初の本人確認時だけ STORES を叩き、以後はセッション内スナップショットを利用する** のを原則とする。
    2. セッション内で STORES の再照会が必要になるのは、
        - ユーザーが「今このチャット中に予約を変更／取り直した」ことを明示した場合など、
            
            状態が変わったと判断できるときに限定。
            
    3. STORES 再照会の頻度は、`stores_api_timeout_sec` / `stores_api_retry_count`、
        
        将来候補の `stores_api_max_calls_per_request` で制御可能な設計とする。
        

---

## 6. 通知 / 外部連携

- ChatWork／メール／SMS／LINE／OpenAI／RemoteLOCK への実 HTTP 呼び出しは、
    
    Phase1 で定義した Clients ＋ `HttpClient` が担当する。
    
- API_COMM の責務：
    - どの業務イベントで `notify.escalation` を呼び出すかを決める。
    - `rule_code`・`summary`・`details` を組み立てて `SysEscalation` を呼ぶ。
    - 結果を `ApiCommResult` に載せて上位に返す。

---

## 7. セキュリティ・権限

- API_COMM は、サーバ内部からのみ呼び出される PHP ライブラリとし、
    
    直接インターネットに公開される HTTP エンドポイントとしては露出しない。
    
- STORES 認証情報：
    - 店舗単位で `shop_secrets` に保存し、API_COMM 内でのみ参照。
    - `Authorization: Bearer <token>` ヘッダで利用。
- PII（個人情報）の扱い：
    - `api_call_logs.response_body` にはメールアドレス・電話番号等を **マスク済み** で保存する。
    - `ApiCommResult.data` 内では
        - マスク済みの `email_masked` / `phone_last4` 等を利用するのが望ましい。
- RemoteLOCK：
    - v1.4 はスタブのみであり、実クレデンシャルの管理方法は Phase9 で設計する。

---

## 8. 未決課題 / 次フェーズへの引き継ぎ

1. `stores_api_max_calls_per_request` 等の具体的な上限値
    - 実運用のログを見つつ後日決定する。
2. `stores_customer_snapshot_ttl_min` の扱い
    - AI セッション上の STORES情報スナップショットを何分間有効と見なすかの目安。
    - 実装は AI_CORE／UI 側の責務とし、本仕様では「候補」として記載に留める。
3. LINE ID → STORES 顧客の自動再照合（全件スキャン）の是非
    - レート制限とトレードオフになるため、v1.4 では**実装しない**。
4. RemoteLOCK 実 API 連携時の I/O 詳細
    - どのキー情報をどの粒度で返すか（PINのみ／予約メタ情報を含むか）。
5. HTTP 層の 4xx/5xx マッピングと監視設計
    - HELP_UI / LINE_UI / 管理画面フェーズで、
        - `ApiCommResult.code` → HTTPステータス／モニタリングイベントへのマッピングルールを定義する。
