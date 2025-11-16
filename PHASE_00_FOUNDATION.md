# PHASE_00_FOUNDATION.md

## フェーズ0: 基盤アーキテクチャ / 運用設計 - 仕様設計書（v1.4 Final）

## 1. 目的・範囲

本フェーズ0は、加盟店サポート自動化プロジェクト v1.4 の全フェーズに共通する

「システム基盤・共通ルール・環境構成」を定義する。

本書の目的：

- サーバー構成・サブドメイン・ディレクトリ配置の統一
- `.env` の配置場所と機密情報・運用パラメータの管理方針
- Owner / Shop（オーナー vs 店舗）の上位下位概念の明確化
- STORES連携（店舗別APIキー）の扱い
- RemoteLock の扱い（v1.4では実装外）
- フィーチャーフラグ・権限ロール・バックアップ方針
- ログの保持・AI学習データの取り扱い
- エスカレーション通知の土台（ChatWork/SMS）
- ID採番ルール、AIモデルバージョン管理
- **基本DBテーブル・カラムの詳細定義は `PHASE_00_DB_CORE.md` / `db_fields.yaml` を正とし、本書では概念レベルの前提のみ扱う**

これにより、後続フェーズが機能仕様に集中できる環境を作る。

---

## 2. 前提・既存仕様の継承

### 2.1 サーバー環境

- 新規レンタルサーバーをプロジェクト専用として使用（他システム稼働なし）
- 本番環境＝構築環境（prod のみ）
- dev 環境は将来追加予定だが、v1.4 時点では運用なし
- コード内部は将来の dev 対応ができるよう `env_type` を保持（現状固定値：`prod`）

### 2.2 サブドメインと物理パス

| サブドメイン | 物理ディレクトリ |
| --- | --- |
| [help.self-datsumou.net](http://help.self-datsumou.net/) | /self-datsumou.net/public_html/help |
| [line.self-datsumou.net](http://line.self-datsumou.net/) | /self-datsumou.net/public_html/line |
| [shop.self-datsumou.net](http://shop.self-datsumou.net/) | /self-datsumou.net/public_html/shop |
| [ai.self-datsumou.net](http://ai.self-datsumou.net/) | /self-datsumou.net/public_html/ai |
| [assets.self-datsumou.net](http://assets.self-datsumou.net/) | /self-datsumou.net/public_html/assets |

### 2.3 .env の配置（確定）

- `.env` は **公開ディレクトリより上** に配置
    - **パス： `/self-datsumou.net/.env`**
- 各アプリ（help/line/shop/ai）は固定パス参照で `.env` を読む

### 2.4 Owner / Shop（オーナーと店舗）

- Owner（加盟店主）1：N Shop（店舗）
→ オーナー＝上位、店舗＝下位
- ログインについて：
    - オーナー・店舗スタッフは同ID共有前提
    → ロールは `SHOP` で統一
    - 本部スタッフは同ID共有前提
    → ロールは `HQ` で統一

### 2.5 STORES API キーの前提

- STORES API キーは **店舗単位で異なる**
- エンドポイントURLは全店舗共通
- 店舗別の STORES API キーは、**`shop_secrets` テーブルのKV形式**（例：`k='STORES_API_KEY'`）で保持し、`.env` には置かない
（詳細なスキーマは `PHASE_00_DB_CORE.md` を参照）

### 2.6 RemoteLock の扱い

- RemoteLock連携は v1.4 の実装対象外
- 既存の「キー自動発行アプリ」（他エンジニア作成）を将来参照可能にするため
    - **拡張ポイント（Interface）だけ用意**
    - デフォルト実装は「何もしない」

---

## 3. 環境構成・パラメータ管理・権限

### 3.1 環境レベル

| 環境 | 説明 |
| --- | --- |
| prod | 本番（唯一の運用環境） |
| dev | 将来追加予定（v1.4では未運用） |
| local | 開発者用（今回運用なし） |
- `env_type` はコード内で保持（v1.4は `"prod"` 固定）

### 3.2 `.env` に入れるもの（機密情報）

| 項目 | 理由 |
| --- | --- |
| DB接続情報 | 完全機密 |
| ChatWork APIキー | 完全機密 |
| LINE APIキー・Channel Secret | 完全機密 |
| SMS送信キー | 完全機密 |
| メール送信のSMTP認証情報 | 完全機密 |
| 本部用ChatWorkルームID（用途別） | 店舗情報とは無関係なため.envに集約 |
| STORES共通設定（BaseURL・Timeout等） | 全体共通・非機密 |

※ STORES API **キーは `.env` に入れない**（店舗ごとに異なるため。`shop_secrets` に保持）

### 3.3 DBで管理するもの（運用値）

- 店舗別AIチューニング（温度・文体・ボット感NGなど）
- システム全体パラメータ（通知回数・閾値）
- フィーチャーフラグ
- 店舗別／オーナー別の設定値

> 実際のテーブル・カラム定義は PHASE_00_DB_CORE.md / db_fields.yaml を正とし、本書では「何をDB側に持つか」の方針のみを定義する。
> 

### 3.4 権限ロール（シンプル運用）

| ロール | 説明 |
| --- | --- |
| HQ | 本部スタッフ（ai.） |
| SHOP | 加盟店オーナー／スタッフ（shop.） |
| SYSTEM | 内部処理用 |

---

## 4. 日付・時刻・タイムゾーン

### 4.1 保存

- DB保存：**UTC**（業界標準）
- アプリ表示：**Asia/Tokyo** に変換して出力

### 4.2 フォーマット

| 用途 | 形式 |
| --- | --- |
| DB／ログ内部 | `YYYY-MM-DD HH:mm:ss` |
| UI表示 | `YYYY-MM-DD HH:mm` または `YYYY-MM-DD`（機能フェーズで最適化） |

---

## 5. エラー表示ポリシー

### 5.1 エンドユーザー（help/LINE）

- 技術情報は出さない
- 補償・再試行案内のみ
- 例：「システムエラーが発生しました。しばらくしてからもう一度お試しください。」

### 5.2 加盟店（shop）

- 「原因カテゴリ＋エラーコード」のみ表示
- 例：「設定情報不足（ERR-20251116-000123）」

### 5.3 本部（ai）

- 原因のヒント＋エラーコードを表示
- 詳細（SQL・APIレスポンス）はログにのみ記録

---

## 6. フィーチャーフラグ

- 目的：機能の段階的リリース、店舗ごとのON/OFF

### 6.1 管理の考え方

- フィーチャーフラグは **「スコープ付きパラメータ」** として管理する。
    - 全体フラグ → scope=`global`
    - 店舗別フラグ → scope=`shop`, shop_id をキーに含める
- 実際の保持テーブル（例：`sys_params` のようなパラメータストア）は **Phase1（SYS_CORE）で定義** し、
Phase0ではテーブル名や物理構造を固定しない。

### 6.2 キー例（概念）

- `feature_ai_tutorial_enabled`
- `feature_auto_notification_enabled`
- `feature_owner_portal_v2_enabled`

> 実装では「キー名＋スコープ＋値（bool/int/json）」の形式で管理し、テーブル定義は PHASE_01 で確定させる。
> 

---

## 7. バックアップ・リストア（最低限の前提）

業界標準に基づく推奨運用：

### 7.1 DBバックアップ

- 最低：**1日1回のフルバックアップ**（mysqldump など）
- 推奨保持期間：30日

### 7.2 ファイルバックアップ

- コードは Git/ZIP で復元可能
- 優先バックアップ対象：
    - アップロードファイルが発生した場合（現時点ほぼなし）
    - ログファイル（必要に応じて）
- 必要に応じてサーバースナップショット併用

### 7.3 リストア方針

- **原則：システム全体を特定日時に戻す**
- レコード単体の復元は想定しない

---

## 8. ID採番ルール（内部IDと外向けID）

### 8.1 内部ID

- `INT AUTO_INCREMENT`
- 全テーブル基準の業界標準方式

### 8.2 外向けID（人間向け管理用）

プレフィックス＋日付＋連番の形式：

例）

- チケットID：`TKT-20251116-000001`
- AIセッションID：`AIS-20251116-XYZ123`

---

## 9. AIモデル・チューニングのバージョン管理

### 9.1 本部側（ai.）で一括管理

- `ai_model_version`
- `prompt_profile_id`
- `default_temperature`

本部はこれらを **一括設定** し、必要があれば店舗別に微調整。

### 9.2 店舗別チューニング

- 店舗別のAIチューニング値は、Phase1で定義するパラメータテーブル（例：`sys_params` の shop スコープ）で管理する想定。
- 本部が代理編集する前提。

### 9.3 ログへの記録

AIログには必ず：

- 実際に使用した `ai_model_version`
- `prompt_profile_id`
- `temperature`（最終値）

を記録して追跡可能にする。

---

## 10. ログと AI学習データの取り扱い

### 10.1 生ログ（個人情報含む）

用途：障害対応・個別調査

保持：短〜中期間（例：180日）

形式：個人情報保持あり

※ 具体的な保持期間・テーブル設計は Phase1/2 で詳細定義する。

### 10.2 学習データ（個人情報マスク済み）

用途：AI精度向上・FAQ改善

保持：長期（無期限推奨）

加工：

- 電話番号・メール・氏名 → マスク or ハッシュ
- 会員ID → マッピングID化

---

## 11. エスカレーション通知（ChatWork / SMS）

### 11.1 エスカレーション種類

1. **顧客 → 加盟店（オーナー）**
2. **顧客 → 本部**
3. **加盟店 → 本部**

### 11.2 通知チャネル

- 加盟店（オーナー）向け：
    - オーナーごとの ChatWork ルームID
    - 条件により「時間差SMS」または「重要案件は同時SMS」
- 本部向け：
    - 用途別 ChatWork ルーム（`.env` に用途別で保持）
    - 例）`HQ_CHATWORK_ROOM_TUTORIAL`, `HQ_CHATWORK_ROOM_ALERT`, …

### 11.3 通知ペイロード形式（共通）

```
チケットID
質問内容
AIの判断結果（カテゴリ＋自信度など）
返信URL（shopまたはaiの返信画面）

```

### 11.4 ルームIDの保持場所

- **オーナー用ChatWorkルームID： `owners.chatwork_room_id` に保持する**
- 本部用ChatWorkルームID：`.env` に用途別で保持

### 11.5 通知ルーティング

1. shop_id を受け取り
2. owner_id に辿る
3. `owners.chatwork_room_id` から room_id を取得
4. ChatWorkに通知

---

## 12. 外部連携（STORES / RemoteLock / ChatWork / LINE / SMS）

### 12.1 STORES

- 認証：`shop_secrets` テーブルのKV（例：`k='STORES_API_KEY'`）から店舗別 STORES API キーを取得
- 共通設定：`.env` の `STORES_API_BASE_URL`
- 接続レイヤー：共通クライアント `StoresClient`

### 12.2 RemoteLock（v1.4実装外）

- `KeyInfoProviderInterface` のみ定義
- デフォルト実装は「常に情報なし」

### 12.3 ChatWork

- 加盟店通知：ownerごとの room_id（`owners.chatwork_room_id`）へ通知
- 本部通知：`.env` の用途別 room_id へ通知

### 12.4 LINE

- help/LINE UI フェーズで定義
- APIキー・シークレットは `.env`

### 12.5 SMS

- 緊急通知・重要チケットのSMS送信
- 送信APIキーは `.env`

---

## 13. データ構造（基盤部分の考え方）

- 実際のテーブル／カラム定義は **`PHASE_00_DB_CORE.md` / `db_fields.yaml` を正** とし、本節では「どの種類の情報をどのレイヤーで持つか」の考え方のみ整理する。

### 13.1 システム全体パラメータ（グローバル）

- 通知回数・閾値、AIデフォルト設定、全体フィーチャーフラグなどを管理する。
- Phase1（SYS_CORE）で定義するパラメータテーブルの **scope=`global`** として保持する想定。

### 13.2 店舗別パラメータ

- 店舗別AIチューニング、店舗別機能ON/OFFなどを管理する。
- 同じくパラメータテーブルの **scope=`shop`, shop_id** で管理する想定。
- ON/OFF で足りない項目は `options_master` + `shop_options`（DB_CORE参照）で扱う。

### 13.3 店舗別機密情報

- RemoteLOCK親アカウント、STORESログイン情報、各種APIキーなど、オーナーに開示しない情報は
    
    **`shop_secrets` テーブルに KV 形式で保持** する（例：`k='REMOTELOCK_PARENT_EMAIL'` 等）。
    
- 値は暗号化して保存し、復号はサーバ側ロジックのみで行う。

### 13.4 オーナー別通知情報

- オーナー用ChatWorkルームIDなどの通知先情報は **`owners.chatwork_room_id` をSoR** とする。
- 将来、通知先が増える場合は Phase1 以降で拡張テーブルを追加する。

---

## 14. マイグレーション方針

### 14.1 設計は各フェーズで書く

- 各フェーズで新規テーブル・カラムが必要になった場合
    
    → そのフェーズの仕様書（`PHASE_xx_*.md`）に **必ずDDLを明記**
    

### 14.2 実行は最終ZIP出力時に**まとめて一括**

- v1.4リリース時点で
    - `db_migration_v1_4.sql`（フルセット）
    - `db_migration_v1_4_diff.sql`（差分用）
- を ChatGPT が一括生成し、あなたが prod に適用する。

> Phase0 で確定した PHASE_00_DB_CORE.md / db_fields.yaml の内容を前提とし、
> 
> 
> 後続フェーズは「追加分」だけをマイグレーションに乗せる。
> 

---

## 15. 未決課題 / 次フェーズへ引き継ぎ

- 生ログ保持期間の具体値（例：90日 or 180日）
- SMS送信の条件（重要案件の定義）
- RemoteLock参照インターフェースのI/O仕様
- AI-INBOXとの統合方針（返信URL部分）
- パラメータテーブル（例：`sys_params`）の具体スキーマ設計（Phase1 SYS_COREで確定）
