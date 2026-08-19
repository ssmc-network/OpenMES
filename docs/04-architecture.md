# 04. アーキテクチャ

## 1. 全体構成

```
                         ┌──────────────┐
                         │  API Gateway │  （検証では任意。UC-10 の composition を担う）
                         └───────┬──────┘
             ┌───────────────┬───┴───────────┬──────────────────┐
             ▼               ▼               ▼                  ▼
      ┌────────────┐  ┌────────────┐  ┌──────────────┐  ┌───────────────┐
      │  master    │  │   order    │  │  execution   │  │  inventory    │
      │  service   │  │  service   │  │   service    │  │   service     │
      └─────┬──────┘  └─────┬──────┘  └──────┬───────┘  └───────┬───────┘
            │ role:         │ role:          │ role:            │ role:
            │ mes_master_svc│ mes_order_svc  │ mes_execution_svc│ mes_inventory_svc
            ▼               ▼                ▼                  ▼
      ┌───────────────────────────────────────────────────────────────┐
      │                    PostgreSQL (single instance)               │
      │  mes_master │ mes_order │ mes_execution │ mes_inventory       │
      │  ─ 各ロールは自スキーマ以外に USAGE 権限を持たない ─           │
      └───────────────────────────────────────────────────────────────┘
```

サービス間の実線呼び出し（同期）:

```
order      ──> master      （UC-03: 品目/BOM/工程の検証・取得）
order      ──> inventory   （UC-04: 引当・引当解除）
execution  ──> order       （UC-05/07: オーダー・工程の検証と進捗更新）
execution  ──> inventory   （UC-06/07: 材料払出・完成品入庫）
```

呼び出しは**単方向**に保つ（master は誰も呼ばない、inventory は誰も呼ばない）。
循環依存を作らないことで、障害の波及範囲を限定する。

## 2. データ分離の実現方式

### 2.1 スキーマとロール

```sql
-- スキーマとロールを1対1で作る
CREATE SCHEMA mes_master;
CREATE ROLE mes_master_svc LOGIN PASSWORD '...';

-- 自スキーマのみ利用可能にする
GRANT USAGE ON SCHEMA mes_master TO mes_master_svc;
GRANT SELECT, INSERT, UPDATE, DELETE
  ON ALL TABLES IN SCHEMA mes_master TO mes_master_svc;

-- 他スキーマへのアクセスは付与しない（= permission denied になる）

-- public スキーマの暗黙的な利用も塞ぐ
REVOKE ALL ON SCHEMA public FROM PUBLIC;

-- search_path を固定し、意図しない解決を防ぐ
ALTER ROLE mes_master_svc SET search_path = mes_master;
```

これにより、`mes_order_svc` で接続したセッションが
`SELECT * FROM mes_master.item` を実行すると `permission denied for schema mes_master` となる。

### 2.2 マイグレーション

- マイグレーションはサービスごとに独立したディレクトリで管理する
- マイグレーション実行は**管理用ロール**で行い、サービス実行時のロールとは分ける
  （サービスロールに DDL 権限を与えない）
- サービスのマイグレーションが他サービスのスキーマに触れることを禁止する（レビュー観点かつ、権限で強制）

### 2.3 分離の証明テスト

Phase 1 で以下を自動テスト化し、CI で常時実行する（SC-1）。

1. 各サービスロールで接続し、他の全スキーマへの `SELECT` が失敗することを確認する
2. `information_schema.table_constraints` を検査し、スキーマ跨ぎの外部キーが 0 件であることを確認する
3. 各サービスの `search_path` が自スキーマのみであることを確認する

## 3. サービス間通信

### 3.1 同期通信（採用）

- 検証の第一段階では **REST/JSON による同期呼び出し**に統一する
- 理由：整合性の問題と可用性の結合が**隠れずに露出する**ため、検証として正直である
- タイムアウト・リトライ（指数バックオフ）・サーキットブレーカを最小限入れる

### 3.2 非同期通信（検証範囲外、ただし Outbox は用意する）

- メッセージブローカは導入しない（構成要素を増やさないため）
- ただし各サービスに **Outbox テーブルを用意し、状態変化をイベントとして記録**する
- Outbox は「後からイベント駆動に移行できる余地」を残すためと、
  **Saga のリカバリ（未完了処理の再開）**のために使う

### 3.3 冪等性

全ての更新系APIは `Idempotency-Key` ヘッダを受け付ける。

- キーは DB のユニーク制約で保持する
- 同一キーの再送は、処理を再実行せず**前回の結果を返す**
- 呼び出し側は Saga のリトライ時に**同じキーを再利用する**

## 4. 整合性設計（Saga）

### 4.1 方針

- **オーケストレーション型 Saga** を採用する（コレオグラフィよりも流れが追いやすく、検証で観察しやすい）
- Saga のオーケストレータは、業務の主体となるサービスに置く
  - UC-04 リリース → order-service
  - UC-06/UC-07 実績報告 → execution-service

### 4.2 UC-07（作業完了）の Saga

```
execution-service（オーケストレータ）

 step 1  work_record を COMPLETED にする            [local tx]
           補償: work_record を IN_PROGRESS に戻す

 step 2  inventory へ材料払出を要求                  [remote, idempotent]
           補償: 払出取消（在庫を戻す）

 step 3  最終工程なら inventory へ完成品入庫を要求    [remote, idempotent]
           （入庫時に lot_genealogy も記録される）
           補償: 入庫取消（ロット無効化・在庫減算）

 step 4  order へ工程進捗を通知                      [remote, idempotent]
           補償: 進捗を戻す
```

- 各ステップの前後で Outbox に Saga の進行状態を記録する
- プロセス障害で中断した Saga は、**リカバリワーカが Outbox を走査して再開**する
- 補償が失敗し続ける場合は「要人手介入」状態としてマークし、握りつぶさない

### 4.3 検証で明らかにしたいこと

- 補償コードの実装量（NFR / 棄却基準 R3 の判定材料）
- 「現場が完了ボタンを押してから応答が返るまで」の時間
- 途中失敗時に、業務的に許容できる状態に着地できるか

## 5. サービス内部のレイヤ構成

言語に依存しない形で、各サービスは以下の層を持つ。

```
api/         HTTPハンドラ。DTO変換、バリデーション、冪等キーの取り回し
application/ ユースケース。トランザクション境界。Saga オーケストレーション
domain/      エンティティ・値オブジェクト・ドメインルール。外部依存を持たない
infra/
  repository/  DBアクセス（自スキーマのみ）
  client/      他サービスのHTTPクライアント
  outbox/      Outbox 書き込みとリカバリ
```

**規約**: `domain` 層は他サービスの存在を知らない。
外部サービスへの依存は `application` 層がインターフェース越しに扱い、実装は `infra/client` に置く。
これにより、他サービスをスタブ化した単体テストが可能になる（NFR-11）。

## 6. 実行環境

### 6.1 ローカル（主）

- `docker compose up` で Postgres + 4サービスが起動する
- Postgres は初期化スクリプトでスキーマ・ロールを作成する
- 既存の自宅 Postgres を使う場合は、接続先を環境変数で差し替えられるようにする
  （その場合もスキーマ・ロールの作成スクリプトは同一のものを使う）

### 6.2 Kubernetes（副・後段）

- Phase 5 以降のオプションとして、同じイメージを k8s マニフェストで動かせるようにする
- ただし**検証の主眼は k8s ではない**ため、優先度は低い
- 既存の自宅 k8s / Postgres への接続は環境変数と Secret で吸収する

## 7. リポジトリ構成（予定）

```
/
├── README.md
├── CLAUDE.md
├── docs/
│   ├── 01-verification-plan.md
│   ├── 02-requirements.md
│   ├── 03-domain-model.md
│   ├── 04-architecture.md
│   ├── 90-verification-report.md      （Phase 6 で作成）
│   └── adr/
├── db/
│   ├── init/                          スキーマ・ロール作成
│   └── migrations/
│       ├── master/
│       ├── order/
│       ├── execution/
│       └── inventory/
├── services/
│   ├── master/
│   ├── order/
│   ├── execution/
│   └── inventory/
├── tests/
│   ├── isolation/                     分離の証明テスト（SC-1）
│   ├── e2e/                           ユースケース単位のE2E（SC-2）
│   └── chaos/                         障害注入テスト（SC-4）
├── tools/
│   └── seed/                          性能検証用データ生成
├── docker-compose.yml
└── Makefile
```

## 8. テスト戦略

| 層 | 対象 | 方針 |
| --- | --- | --- |
| 単体 | domain 層のルール | 外部依存なしで実行 |
| 統合 | repository + 実 Postgres | サービスロールで接続し、越境しないことも同時に検証 |
| 契約 | サービス間API | OpenAPI スキーマに対する契約テスト |
| 分離 | ロール権限・FK・search_path | SC-1。CI で常時実行 |
| E2E | UC-01〜UC-10 | SC-2。全サービス起動して通し |
| 障害注入 | 依存サービス停止時の Saga | SC-4 |
| 性能 | UC-09, UC-10 | SC-3, SC-6。シードデータ投入後に計測 |

**重要**: E2E テストは検証の証拠そのものである。
越境アクセスをすればDBが拒否するため、**E2E が通ること自体が「分割が成立していること」の証明**になる。

## 9. 観測

- 全リクエストに相関ID（`X-Correlation-Id`）を付与し、サービス跨ぎで伝播する
- サービス間呼び出しは構造化ログに記録し、**ユースケースあたりの跨ぎ呼び出し回数を集計できる**ようにする
  （これが結合度の定量指標になる）
- Saga の各ステップ・補償はログに残す
