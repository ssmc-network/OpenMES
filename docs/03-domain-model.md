# 03. ドメインモデル / データモデル

## 1. 用語集

| 用語 | 英語 | 説明 |
| --- | --- | --- |
| 品目 | Item | 製造・購買・在庫の対象となるもの。原材料 / 仕掛品 / 完成品を区別する |
| 部品表 | BOM | ある品目を1単位作るのに必要な構成品目と員数 |
| 工程 | Operation | 製造の作業単位。ルーティングの1行 |
| ルーティング | Routing | 品目を製造するための工程の並び |
| 作業区 | Work Center | 工程を実施する設備・ラインのグループ |
| 保管場所 | Location | 在庫を保持する物理的な場所 |
| 製造オーダー | Production Order | 「何を・いくつ・いつまでに」作るかの指示 |
| 引当 | Allocation | 特定のオーダー向けに在庫を確保すること。物理的な移動は伴わない |
| 払出 | Issue | 在庫を実際に消費すること |
| 入庫 | Receipt | 在庫を増やすこと |
| ロット | Lot | 同一条件で製造・入荷された数量のまとまり。追跡の単位 |
| ロット系譜 | Lot Genealogy | 材料ロットと製品ロットの親子関係 |
| 作業実績 | Work Record | 工程における作業の開始・完了・数量の記録 |

## 2. サービスと集約の対応

| サービス | スキーマ | 集約 |
| --- | --- | --- |
| master-service | `mes_master` | Item, Bom, Routing, WorkCenter, Location |
| order-service | `mes_order` | ProductionOrder（工程行・所要材料行を含む） |
| execution-service | `mes_execution` | WorkRecord（不良明細・材料消費明細を含む） |
| inventory-service | `mes_inventory` | Lot, Stock, Allocation, LotGenealogy |

分割の判断基準は「**トランザクション整合性が必要な範囲**」とした。
1つの集約は必ず1つのスキーマ内にあり、集約内の更新は単一のローカルトランザクションで完結する。

## 3. スキーマ定義

型は Postgres を前提とする。全テーブルに `created_at` / `updated_at`（`timestamptz`）を持つものとし、以下では省略する。

### 3.1 `mes_master` — マスタ

```
item
  item_code        text PK
  item_name        text NOT NULL
  item_type        text NOT NULL   -- RAW | WIP | FG
  uom              text NOT NULL   -- 単位: PCS, KG, M ...
  lot_managed      boolean NOT NULL
  is_active        boolean NOT NULL

bom_header
  bom_id           bigint PK
  parent_item_code text NOT NULL -> item.item_code
  version          int  NOT NULL
  valid_from       timestamptz NOT NULL
  valid_to         timestamptz
  UNIQUE (parent_item_code, version)

bom_line
  bom_id             bigint  -> bom_header.bom_id
  line_no            int
  component_item_code text NOT NULL -> item.item_code
  qty_per            numeric(18,6) NOT NULL   -- 親1単位あたりの員数
  scrap_rate         numeric(9,6)  NOT NULL DEFAULT 0
  PK (bom_id, line_no)

routing_header
  routing_id       bigint PK
  item_code        text NOT NULL -> item.item_code
  version          int  NOT NULL
  valid_from       timestamptz NOT NULL
  valid_to         timestamptz
  UNIQUE (item_code, version)

routing_operation
  routing_id       bigint -> routing_header.routing_id
  op_no            int
  op_name          text NOT NULL
  work_center_code text NOT NULL -> work_center.work_center_code
  std_time_sec     int NOT NULL
  is_final         boolean NOT NULL   -- 最終工程フラグ（完成品入庫の契機）
  PK (routing_id, op_no)

work_center
  work_center_code text PK
  work_center_name text NOT NULL

location
  location_code    text PK
  location_name    text NOT NULL
  location_type    text NOT NULL   -- WAREHOUSE | LINE_SIDE | WIP
```

### 3.2 `mes_order` — 生産指示

```
production_order
  order_no          text PK
  item_code         text NOT NULL      -- master への論理参照（FK なし）
  item_name_snap    text NOT NULL      -- 登録時点のスナップショット
  uom_snap          text NOT NULL
  planned_qty       numeric(18,6) NOT NULL
  completed_qty     numeric(18,6) NOT NULL DEFAULT 0
  scrap_qty         numeric(18,6) NOT NULL DEFAULT 0
  due_date          date NOT NULL
  status            text NOT NULL      -- CREATED|RELEASED|IN_PROGRESS|COMPLETED|CLOSED|CANCELLED
  bom_id_snap       bigint NOT NULL
  routing_id_snap   bigint NOT NULL
  released_at       timestamptz
  completed_at      timestamptz

production_order_operation
  order_no          text -> production_order.order_no
  op_no             int
  op_name_snap      text NOT NULL
  work_center_code  text NOT NULL
  std_time_sec_snap int NOT NULL
  is_final          boolean NOT NULL
  planned_qty       numeric(18,6) NOT NULL
  completed_qty     numeric(18,6) NOT NULL DEFAULT 0
  scrap_qty         numeric(18,6) NOT NULL DEFAULT 0
  status            text NOT NULL      -- PENDING|IN_PROGRESS|COMPLETED
  PK (order_no, op_no)

order_material
  order_no             text -> production_order.order_no
  line_no              int
  component_item_code  text NOT NULL
  component_name_snap  text NOT NULL
  required_qty         numeric(18,6) NOT NULL
  allocated_qty        numeric(18,6) NOT NULL DEFAULT 0   -- inventory の結果の写し（参照用）
  consumed_qty         numeric(18,6) NOT NULL DEFAULT 0   -- 同上
  PK (order_no, line_no)

order_outbox
  outbox_id        bigserial PK
  aggregate_id     text NOT NULL
  event_type       text NOT NULL
  payload          jsonb NOT NULL
  occurred_at      timestamptz NOT NULL
  published_at     timestamptz
```

> `*_snap` 列がスナップショットである。マスタが後から改版されても、
> 発行済みオーダーの内容は変わらない。**この列群の存在自体が、
> 「マスタを共有テーブルとして参照していない」ことの証拠**になる。

### 3.3 `mes_execution` — 実績収集

```
work_record
  record_id        uuid PK
  order_no         text NOT NULL      -- order への論理参照（FK なし）
  op_no            int  NOT NULL
  work_center_code text NOT NULL
  operator_id      text
  started_at       timestamptz NOT NULL
  ended_at         timestamptz
  good_qty         numeric(18,6) NOT NULL DEFAULT 0
  scrap_qty        numeric(18,6) NOT NULL DEFAULT 0
  status           text NOT NULL      -- IN_PROGRESS|COMPLETED|CANCELLED
  idempotency_key  text UNIQUE

defect_record
  record_id        uuid -> work_record.record_id
  seq              int
  defect_code      text NOT NULL
  qty              numeric(18,6) NOT NULL
  PK (record_id, seq)

material_consumption
  consumption_id      uuid PK
  record_id           uuid -> work_record.record_id
  component_item_code text NOT NULL
  lot_no              text            -- 非ロット管理品は NULL
  qty                 numeric(18,6) NOT NULL
  issued              boolean NOT NULL DEFAULT false   -- inventory 側の払出が確定したか
  idempotency_key     text UNIQUE

execution_outbox
  （order_outbox と同構造）
```

### 3.4 `mes_inventory` — 在庫・ロット

```
lot
  lot_no           text PK
  item_code        text NOT NULL
  lot_type         text NOT NULL     -- PURCHASED | PRODUCED
  source_order_no  text              -- 製造ロットの場合の生成元オーダー（論理参照）
  produced_at      timestamptz NOT NULL
  initial_qty      numeric(18,6) NOT NULL

stock
  stock_id         bigserial PK
  item_code        text NOT NULL
  lot_no           text -> lot.lot_no    -- 非ロット管理品は NULL
  location_code    text NOT NULL
  qty_on_hand      numeric(18,6) NOT NULL CHECK (qty_on_hand >= 0)
  qty_allocated    numeric(18,6) NOT NULL CHECK (qty_allocated >= 0)
  CHECK (qty_allocated <= qty_on_hand)
  UNIQUE (item_code, lot_no, location_code)

allocation
  allocation_id    uuid PK
  ref_type         text NOT NULL     -- ORDER
  ref_no           text NOT NULL     -- order_no
  item_code        text NOT NULL
  lot_no           text
  location_code    text NOT NULL
  qty              numeric(18,6) NOT NULL
  status           text NOT NULL     -- ALLOCATED | CONSUMED | RELEASED
  idempotency_key  text UNIQUE

stock_transaction
  txn_id           bigserial PK
  txn_type         text NOT NULL     -- RECEIPT|ISSUE|ALLOCATE|DEALLOCATE|ADJUST
  item_code        text NOT NULL
  lot_no           text
  location_code    text NOT NULL
  qty              numeric(18,6) NOT NULL   -- 符号付き
  ref_type         text
  ref_no           text
  idempotency_key  text UNIQUE
  occurred_at      timestamptz NOT NULL

lot_genealogy
  genealogy_id     bigserial PK
  parent_lot_no    text NOT NULL -> lot.lot_no   -- 材料ロット
  child_lot_no     text NOT NULL -> lot.lot_no   -- 製品ロット
  order_no         text NOT NULL
  qty              numeric(18,6) NOT NULL
  UNIQUE (parent_lot_no, child_lot_no, order_no)

inventory_outbox
  （order_outbox と同構造）
```

## 4. スキーマ跨ぎ参照の一覧（検証の核心）

以下が「本来なら外部キーで繋ぎたいが、分割により繋げない」参照である。
**この表が短いほど分割は素直で、長く複雑なほど分割の無理が大きい。**

| 参照元 | 参照先 | 解決方式 | リスク |
| --- | --- | --- | --- |
| `mes_order.production_order.item_code` | `mes_master.item` | 登録時にAPI検証 + スナップショット保持 | 削除されたマスタを参照し続ける可能性 |
| `mes_order.production_order.bom_id_snap` | `mes_master.bom_header` | 同上（展開結果を保持するため実行時参照は不要） | 低 |
| `mes_order.order_material.component_item_code` | `mes_master.item` | 展開時に検証 + 名称スナップショット | 低 |
| `mes_execution.work_record.order_no` | `mes_order.production_order` | 作業開始時にAPI検証、以降はキーのみ保持 | 存在しないオーダーへの実績（開始時検証で防止） |
| `mes_execution.material_consumption.lot_no` | `mes_inventory.lot` | 払出APIの応答で妥当性を担保 | 中：払出失敗時の補償が必要 |
| `mes_inventory.lot.item_code` | `mes_master.item` | 入庫時にAPI検証 | 低 |
| `mes_inventory.allocation.ref_no` | `mes_order.production_order` | オーダー番号を不透明なキーとして扱う | 低 |
| `mes_inventory.lot_genealogy.order_no` | `mes_order.production_order` | 同上 | 低 |

**設計原則**: 他サービスのIDは「意味を持たない文字列キー」として保持し、
そのIDから他サービスのデータを都度取りに行く設計は避ける（必要な属性はスナップショットで持つ）。

## 5. ロット系譜の設計判断

系譜（どの材料ロットからどの製品ロットができたか）を**どのサービスが持つか**が最大の設計分岐である。

| 案 | 内容 | 評価 |
| --- | --- | --- |
| A | execution が持つ（消費を記録するのは execution だから） | トレース時に inventory のロット情報と突き合わせが必要になり、サービス跨ぎの再帰が発生する → **H4 棄却リスク** |
| B | inventory が持つ（ロットの正本は inventory だから） | execution は「消費した」という事実を inventory に伝えるだけ。トレースは inventory 内で再帰CTE 1回で完結する |
| C | トレース専用サービスを作る | サービスが増え、ミニマム構成の趣旨から外れる。イベント駆動が前提になる |

**採用：案B。** 理由は以下。
- ロット番号を採番するのは inventory であり、ロットの正本も inventory にある
- 系譜は「在庫取引の副産物」であり、払出（ISSUE）と入庫（RECEIPT）の両方を知る inventory が最も自然に記録できる
- トレースが単一スキーマ内の再帰CTEで閉じるため、性能面で有利

**検証で確認すること**: 案B により UC-09 が本当に inventory 内で閉じるか。
閉じない場合（例：トレース結果に品目名やオーダー品目を含めたい）、どの程度の追加参照が必要になるか。

## 6. 状態遷移

### 製造オーダー

```
CREATED ──release──> RELEASED ──first work start──> IN_PROGRESS
                        │                                │
                        │                          all ops done
                        │                                ▼
                        └──────cancel──────>          COMPLETED ──close──> CLOSED
                                                          
CANCELLED  ← cancel（RELEASED までのみ可。引当済なら引当解除を伴う）
```

### 作業実績

```
IN_PROGRESS ──complete──> COMPLETED
     │
     └──cancel──> CANCELLED（消費済み材料があれば在庫へ戻す補償を伴う）
```

### 引当

```
ALLOCATED ──consume──> CONSUMED
    │
    └──release──> RELEASED（オーダー取消時などに在庫へ戻す）
```
