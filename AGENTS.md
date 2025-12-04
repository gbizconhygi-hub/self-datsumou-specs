# AGENTS.md — v1.4 エージェント定義と読解ガイド

このリポジトリは「加盟店サポート自動化プロジェクト v1.4」の  
**仕様書一式＋デフォルトパラメータ／文言集＋DBスキーマ** をまとめたものです。

ここでは、このリポジトリを読む **AIエージェント／人間設計者** 向けに、

- どのフェーズをどの「役割のエージェント」が読むのか
- 各エージェントが前提として読むべきファイル
- 仕様を読む順番と、出してほしいアウトプット

を整理します。

---

## 0. 共通前提（全エージェント共通）

どのエージェントも、まずは以下のファイルを「土台」として理解してから  
各 PHASE_xx_* を読む前提とします。

### 0-1. MUST 読むファイル

- `README.md`  
  リポジトリ全体の概要・目的・バージョン。
- `spec_v1.4_base.md`  
  v1.4 全体の思想・要素一覧・レイヤ構造。
- `v1.4_timeline.md`  
  フェーズごとの進行と「どの要素がどのフェーズで固まるか」の一覧。
- `PHASE_00_FOUNDATION.md`  
  システム全体の基礎概念／レイヤ構造の詳細（存在する場合）。
- `PHASE_00_DB_CORE.md`  
  DB コア設計の骨格（テーブル群・命名規則・IDポリシー）。
- `db_fields.yaml`  
  各テーブルのフィールド定義（機械可読なスキーマ）。

### 0-2. SHOULD 読む補助ファイル

- `DB_BOX_INTENT.md`  
  DB周りの設計意図・集約の思想。
- 各種 `*_DEFAULT_PARAMS.md` / `*_DEFAULT_TEXTS.md` / `*_POLICY.md`  
  各UI/機能のデフォルト設定／文言／運用ポリシー。

> メモ: LLM にこのリポジトリを読ませる場合は、  
> 「まず上記のファイルを読み込んだうえで、担当フェーズの PHASE_xx_* を読んでください」  
> という起動プロンプトにする。

---

## 1. エージェント一覧（役割と対応ファイル）

### 1-1. DB_CORE エージェント

**役割:**  
DBコアスキーマの設計・変更・参照を担当するエージェント。

**主に読むファイル:**

- `PHASE_00_DB_CORE.md`
- `db_fields.yaml`
- `DB_BOX_INTENT.md`
- `DB_PATCH_07_OP_MAIL.md`（DB差分確認時）

**アウトプット例:**

- 新テーブル／カラム追加提案（DDL）
- 既存テーブル変更時の影響範囲整理
- 各 PHASE からの「この情報を永続化したい」に対する DB 解決案

---

### 1-2. SYS_CORE エージェント（システム基盤）

**役割:**  
API ベースURL／認証／env／ログ／共通ルールなど、  
全要素に共通する「システムの骨組み」を担当。

**主に読むファイル:**

- `PHASE_01_SYS_CORE.md`
- `spec_v1.4_base.md`
- `PHASE_00_FOUNDATION.md`

**アウトプット例:**

- 共通ミドルウェア構成（認証・CSRF・API キー）
- 共通レスポンス形式／エラーコードの標準化ルール
- Webhook / Push 受信（OP_MAIL 含む）の共通ポリシー

---

### 1-3. AI_CORE エージェント

**役割:**  
AIコア（意図分類＋回答生成＋ログ＋チューニング）の「頭脳」を設計。

**主に読むファイル:**

- `PHASE_02_AI_CORE.md`
- `AI_INBOX_DEFAULT_PARAMS.md`
- `HELP_UI_DEFAULT_PARAMS.md`
- `LINE_UI_DEFAULT_PARAMS.md`
- `AI_LABELS_OP_MAIL.md`（OP_MAIL向けラベル仕様）

**アウトプット例:**

- LLM 呼び出し I/O 契約（AI_CORE_REQUEST / RESPONSE）
- チャネル別プロファイル設計（help./line./shop./ai./op_mail）
- ログ・学習サンプルの構造（どのテーブルの何を使うか）

---

### 1-4. API_COMM エージェント（外部API通信）

**役割:**  
STORES予約 / RemoteLOCK / 外部メール / SMS 等との API 契約を整理する担当。

**主に読むファイル:**

- `PHASE_02_API_COMM.md`
- `CRON_SYS_DEFAULT_PARAMS.md`
- `SALES_LINK_DEFAULT_PARAMS.md`

**アウトプット例:**

- 各外部API の I/O 契約（URL / メソッド / 認証 / レート制限）
- OP_MAIL から STORES 会員情報照合の際の API 仕様（将来）
- cron ジョブから外部APIを叩くときのリトライ戦略

---

### 1-5. HELP_UI エージェント（Webヘルプチャット）

**役割:**  
help.self-datsumou.net 上のヘルプUIと、  
メール返信（noreply + HELP誘導）周りを設計する担当。

**主に読むファイル:**

- `PHASE_03_HELP_UI.md`
- `HELP_UI_DEFAULT_PARAMS.md`
- `HELP_UI_DEFAULT_TEXTS.md`
- `HELP_UI_POLICY.md`

**OP_MAIL との関係:**

- OP_MAIL 由来の案件で「メールで返信」を選んだ場合、  
  noreply メールから HELP チャットに誘導する流れを設計・維持する。

---

### 1-6. LINE_UI エージェント（LINEチャット）

**役割:**  
LINE チャネルのフロント（line.self-datsumou.net / LIFFなど）と  
LINE↔STORES 会員名寄せを担当。

**主に読むファイル:**

- `PHASE_03_LINE_UI.md`
- `LINE_UI_DEFAULT_PARAMS.md`
- `LINE_UI_DEFAULT_TEXTS.md`
- `LINE_UI_POLICY.md`

**OP_MAIL との関係:**

- SHOP_PORTAL から STORES 会員番号で特定された顧客について、  
  `customer_links` に LINE 紐づきがあるか判定 → 「LINEで返信」ボタンを解放する。

---

### 1-7. SHOP_PORTAL エージェント（店舗ポータル）

**役割:**  
`shop.self-datsumou.net` 上のオーナー／店舗向け UI 全般を設計。

**主に読むファイル:**

- `PHASE_04_SHOP_PORTAL.md`
- `SHOP_PORTAL_DEFAULT_PARAMS.md`
- `SHOP_PORTAL_DEFAULT_TEXTS.md`
- `SHOP_PORTAL_POLICY.md`

**OP_MAIL との関係（重要）:**

- OP_MAIL 由来のチケット表示
  - 電話代行報告メール本文の表示
  - フォロー緊急度バッジ（high/normal）
- 顧客特定フロー
  - 加盟店が STORES で会員を特定 → 会員番号/予約ID をポータルに入力
  - バックエンドで会員情報を照合し、LINE紐づきの有無を返す
- 対応ボタン
  - [LINEで返信]（紐づきがある場合のみ）
  - [メールで返信]（noreply + HELP誘導）
  - [対応不要で完了]（AI誤判定・その場解決の時）

---

### 1-8. AI_INBOX エージェント（HQ INBOX）

**役割:**  
`ai.self-datsumou.net` 側の HQ 用 INBOX（AI-INBOX）の設計担当。

**主に読むファイル:**

- `PHASE_05_AI_INBOX.md`
- `AI_INBOX_DEFAULT_PARAMS.md`
- `AI_INBOX_DEFAULT_TEXTS.md`
- `AI_INBOX_POLICY.md`

**OP_MAIL との関係:**

- `source='operator_mail'` のチケットの優先表示 / フィルタリング
- `close_reason='op_mail_resolved_before'`（店舗が「対応不要で完了」した案件）の表示
- HQ が OP_MAIL 系のチケットを横断的に監視・ラベリングする UI の設計

---

### 1-9. KPI_ANALYTICS エージェント

**役割:**  
売上 / 問い合わせ / SLA / 解約理由など、  
各種KPIの集計・可視化を設計する担当。

**主に読むファイル:**

- `PHASE_06_KPI_ANALYTICS.md`
- `KPI_ANALYTICS_DEFAULT_PARAMS.md`
- `SALES_LINK_DEFAULT_PARAMS.md`

**OP_MAIL との関係:**

- `tickets.source='operator_mail'` や `operator_mail_logs` を基に、
  - 「電話代行経由のトラブル発生件数」
  - 「OP_MAIL から補償に繋がった件数」
  - 「加盟店が電話に出なかった件数」
  などを KPI として可視化する。

---

### 1-10. SALES_LINK エージェント

**役割:**  
売上データ連携（STORES CSV 取込等）と KPI とのブリッジ部分を担当。

**主に読むファイル:**

- `PHASE_06_SALES_LINK.md`
- `SALES_LINK_DEFAULT_PARAMS.md`

**OP_MAIL との関係:**

- 直接の関係は薄いが、OP_MAIL 発生時点の売上・予約情報を参照する場合は  
  SALES_LINK 側のスキーマ理解が必要。

---

### 1-11. CRON_SYS エージェント

**役割:**  
バッチ／cron ジョブの設計（売上取込・KPI集計・SMS追撃・バックアップ等）。

**主に読むファイル:**

- `PHASE_07_CRON_SYS.md`
- `CRON_SYS_DEFAULT_PARAMS.md`

**OP_MAIL との関係:**

- OP_MAIL 自体は cron 対象外（Push型）が前提。  
- ただし以下は CRON_SYS 側で扱う余地がある:
  - OP_MAIL のヘルスチェック（エラー/未処理ログの監視）
  - `source='operator_mail'` + 高緊急度の `tickets` に対する SLA 追撃

---

### 1-12. OP_MAIL エージェント（電話代行メール）

**役割:**  
今回新規で定義した、電話代行オペレーターからの緊急連絡メールを  
AI で分類し、タスク化＋ログ化する専任エージェント。

**主に読むファイル:**

- `PHASE_07_OP_MAIL.md`
- `AI_LABELS_OP_MAIL.md`
- `DB_PATCH_07_OP_MAIL.md`

**責務（要約）:**

- `/op_mail/ingest` の I/O 契約どおりにメールを受信
- `operator_mail_logs` に生メール＋AI判定＋最終判定を保存
- 「解決済み vs 未解決」を判定
  - 解決済み → SHOP_PORTAL に履歴カードのみ
  - 未解決 → `tickets.source='operator_mail'` でタスク起票
- SHOP_PORTAL からの「対応不要で完了」操作を受けて、  
  `final_*` ラベルを更新し、学習サンプルに活かす

---

## 2. 読み込み順ガイド（LLM用の起動パターン）

### 2-1. フェーズ別に仕様を詰めるときの推奨順

1. **全体理解**
   - `README.md`
   - `spec_v1.4_base.md`
   - `v1.4_timeline.md`
2. **DB/土台**
   - `PHASE_00_FOUNDATION.md`
   - `PHASE_00_DB_CORE.md`
   - `db_fields.yaml`
3. **基盤**
   - `PHASE_01_SYS_CORE.md`
   - `PHASE_02_AI_CORE.md`
   - `PHASE_02_API_COMM.md`
4. **UI系**
   - `PHASE_03_HELP_UI.md`
   - `PHASE_03_LINE_UI.md`
   - `PHASE_04_SHOP_PORTAL.md`
5. **HQ/分析**
   - `PHASE_05_AI_INBOX.md`
   - `PHASE_06_KPI_ANALYTICS.md`
   - `PHASE_06_SALES_LINK.md`
6. **バッチ/メール**
   - `PHASE_07_CRON_SYS.md`
   - `PHASE_07_OP_MAIL.md`

### 2-2. LLM 起動テンプレートの典型パターン

エージェントごとのチャットを立ち上げるときは、  
以下のような共通ルールを使うと安定します。

- 最初のプロンプトで:
  - 「このチャットは v1.4 の ○○ フェーズ専用です」
  - 「まず以下のファイルを GitHub コネクタ経由で読み込んでください」  
    （→ `spec_v1.4_base.md` と担当フェーズの `PHASE_xx_*` 等を列挙）
  - 「既存仕様を上書きせず、差分が必要な場合は“差分”として明示してください」

- 例（OP_MAILフェーズ起動用）:
  - `PHASE_07_OP_MAIL.md` の「要素チャット起動テンプレート」の方針に従う。

---

## 3. AGENTS.md の運用ポリシー

- 本ファイルは **仕様書ではなく運用ガイド**。  
  実際の仕様のソース・オブ・トゥルースは各 `PHASE_xx_*` と `*_POLICY.md` にある。
- 新しい要素／フェーズを追加した場合は、
  - 対応エージェントの追加
  - 対応ファイルの一覧更新
  を本ファイルに追記する。
- LLM にリポジトリを読み込ませるときは、  
  本ファイルを軽く読ませたうえで、担当フェーズの PHASE ファイルに飛ばすと  
  「どの範囲まで触るべきか」のブレが減る。

---
