---
type: design
status: pending-confirmation
last_verified: 2026-09-02
source: PRD+code+lingxing-api-docs
version: V2
---

# 对接领星-调拨单&FBA发货单同步-技术实现方案（V2）

> 状态：**待确认**（需求与方案确认阶段，未改动任何代码）
> 依据：《对接领星-调拨单&FBA发货单同步-PRD》、领星API文档（20260520爬虫版）、xzy ERP 代码库
> 待确认问题见 [[对接领星-调拨单与FBA发货单同步-待确认问题清单]]
>
> **V2 变更摘要**（基于 2026-09-02 确认结论）：
> 1. 备货单纳入本期范围（移库调拨→非FBA平台仓或第三方仓）
> 2. 平台仓（whType=4）与第三方仓（whType=2）是不同类型，分类条件按"平台仓或第三方仓"表达；FBA仓=平台仓且平台为亚马逊
> 3. 领星客户端落位 **xzy-openapi 服务**（wms 经 Feign 调用）
> 4. **领星同步失败不阻断中台流程**：中台操作照常完成，领星同步异步执行、失败可重试+告警
> 5. 退运支持部分退运：调拨单按中台退运数量在领星收货并反向调拨同数量
> 6. 费用口径：调拨单费用创建时提交、不可更改；FBA发货单、备货单费用可通过覆盖接口重推

## 0. 需求与范围概述

### 0.1 业务目标

物流人员在中台录入【调拨计划】和【调拨发货单】，发货时将单据单向同步到领星。中台发货单与领星执行单据 **1:1**（领星接口不支持多仓发货），仅同步结果、最晚时机（点击【发货】时）创建领星单据：

| 中台场景 | 始发仓 | 目的仓 | 领星执行单据 | 费用规则 |
| --- | --- | --- | --- | --- |
| 集货调拨 | 自营仓或供应商仓 | 自营仓或供应商仓 | 调拨单（type=2 待收货） | 创建时必传（通常0），**不可更改** |
| 海外调拨 | 第三方仓或平台仓 | 第三方仓或非FBA平台仓 | 调拨单（type=2 待收货） | 创建时必传，**不可更改** |
| 海外调拨 | 第三方仓或平台仓 | FBA仓 | FBA发货单（已发货） | 可后补，**可覆盖重推** |
| 移库调拨 | 自营仓或供应商仓 | FBA仓 | FBA发货单（已发货） | 可后补，**可覆盖重推** |
| 移库调拨 | 自营仓或供应商仓 | 非FBA平台仓或第三方仓 | 备货单（待收货） | 创建时随传，**可覆盖重推**（UpdateLogistics，待确认 B-16） |

### 0.2 核心设计原则

1. 单向同步：中台 → 领星，不回拉。
2. 强制覆盖：领星侧字段以中台最新数据为准（费用覆盖重推同理）。
3. 最晚时机同步、仅同步结果：点击【发货】才创建领星执行单据。
4. **领星同步不阻断中台**：中台发货/收货/结束到货/退运等操作按中台事务正常完成；领星同步异步执行，失败进入重试队列并告警（2026-09-02 确认）。
5. 库存口径：中台创建【待发货】发货单时锁中台库存；领星执行单据创建时扣领星库存（CreateSendedOrder/AddAllocationOrder/CreateInbound status=50 均即时扣减始发仓库存）。
6. 异常反馈：日志告警+异常事件，不做领星→中台字段回写。

### 0.3 本期实现范围

- 发货同步：领星【调拨单】（AddAllocationOrder）、【已发货发货单】（CreateSendedOrder+searchProcessResult）、【备货单】（CreateInbound）
- 收货同步：调拨单分批/全部收货、备货单分批收货、收货后领星IB/入库单号回写
- 结束到货：调拨单 finishReceiveAllocationOrder、备货单 inboundCompleteReceipt；中台新增【结束到货】功能；FBA发货单不处理领星侧
- 退运：FBA发货单作废（InvalidShipmentSn）；调拨单"部分收货+反向type=1调拨"；备货单方案待定（B-15）
- 物流费用：费用弹框+分摊计算；调拨单随创建提交；FBA发货单 updateListLogistics 覆盖；备货单 UpdateLogistics 覆盖
- 基础数据映射：店铺、仓库、头程物流渠道、其他费类型、产品ID（sku→product_id，备货单创建与分批收货必用）
- 页面展示：【领星单据】【预估物流费】【实际物流费】列、悬浮详情、费用页签、入库单领星单据列、收货单改名与取消数

## 1. 需求落地实现步骤清单

> M0/M1/M2 为前置；涉及两个仓库：`xzy`（wms 业务侧）与 `xzy-openapi`（领星客户端，源码不在本工作区，需跨仓库协同，见 C-12）。

### M0 准备与确认（0.5周）

| # | 任务 | 类型 | 产出 |
| --- | --- | --- | --- |
| 0.1 | 确认《待确认问题清单》V2 剩余条目（备货单退运、备注幂等、部分退运FBA处理等） | 沟通 | 结论回填 |
| 0.2 | 申请/确认领星 appId、appSecret、签名算法资料（用户后续提供） | 沟通 | 凭据与签名依据 |
| 0.3 | 核对领星基础数据：本地仓/海外仓清单、店铺授权、默认物流商渠道、其他费类型、产品档案（SKU齐全性） | 沟通 | 映射初始数据 |
| 0.4 | xzy-openapi 仓库工作项对齐（排期、负责人、分支策略） | 沟通 | 跨仓库计划 |

### M1 领星 OpenAPI 基础组件 @ xzy-openapi 服务（1.5周）

| # | 任务 | 类型 | 说明 |
| --- | --- | --- | --- |
| 1.1 | 领星客户端：HTTP+签名+公共参数（access_token/timestamp/app_key）+统一响应解析（code=0） | openapi | 包路径建议 `com.xzy.openapi.lingxing` |
| 1.2 | Token 管理：GetToken / RefreshToken（refresh_token 一次性）、提前刷新、并发原子更新；凭据存 `t_auth_config`（serviceProviderCode=LINGXING_ERP） | openapi | |
| 1.3 | 限流：领星接口令牌桶容量多为1，客户端内置串行队列+请求间隔（≥1s） | openapi | |
| 1.4 | 接口日志表 `t_lx_sync_log`（openapi 库），请求/响应全量留痕 | openapi/DB | |
| 1.5 | Feign 契约 @ `xzy-openapi-api`：RemoteLingxingService（建单/收货/作废/费用/基础数据等约18个方法） | api模块 | 见 2.2 |

### M2 基础数据映射（1周，与 M1 并行）

| # | 任务 | 类型 | 说明 |
| --- | --- | --- | --- |
| 2.1 | 店铺：SellerLists → `t_lx_seller`（sid/seller_id/marketplace_id），与 `t_amazon_seller_channel_config` 按 seller_id+marketplace_id 映射；每日刷新 | BE/DB | |
| 2.2 | 仓库：WarehouseLists（type=1本地仓/3海外仓/4平台仓）→ `t_lx_warehouse`；中台仓库↔领星wid映射用 `t_warehouse_mapping`（serviceProviderCode=LINGXING_ERP，extWhCode=领星wid），映射时校验领星仓类型与用途匹配（见 2.3 说明） | BE/DB/CFG | |
| 2.3 | 物流渠道：ChannelList → `t_lx_channel`；定位"默认物流商"渠道（备货单 logistics_id 必传；按计费重分摊时发货单也必传） | BE/DB | |
| 2.4 | 其他费类型：GetHeadLogisticsFeeTypes → `t_lx_fee_type` | BE/DB | |
| 2.5 | 产品ID：ProductLists(sku_list) → `t_lx_product_map`（sku→product_id）；**备货单创建、调拨单分批收货、备货单分批收货均必传 product_id** | BE/DB | |

> 仓库映射用途校验规则：调拨单始发/目的仓须为领星本地仓；FBA发货单始发仓须为领星本地仓；备货单始发仓须为领星本地仓、目的仓须为领星海外仓。映射维护页按用途提示可选仓库类型。

### M3 调拨计划分类标识（0.5周）

| # | 任务 | 类型 | 说明 |
| --- | --- | --- | --- |
| 3.1 | `t_wms_allocate` 增加 `allocate_category`（COLLECT/MOVE/OVERSEA）与 `lx_doc_type`（ALLOCATION/FBA_SHIPMENT/STOCKUP/NONE） | DB/BE | 判定规则见 5.1 |
| 3.2 | 创建/保存调拨计划时自动计算分类；历史数据刷新脚本 | BE | |
| 3.3 | 发货单生成时继承 `lx_doc_type`，驱动同步分支 | BE | |

### M4 物流费用模块（1.5周）

| # | 任务 | 类型 | 说明 |
| --- | --- | --- | --- |
| 4.1 | 费用表 `t_wms_lx_fee`（发货单粒度：预估/实际的物流费、其他费、合计，分摊方式，重量快照，同步状态） | DB | 见 3.2 |
| 4.2 | 重量计算服务：单品体积重=包装长×宽×高/6000/单箱数量；单品计费重=MAX(单品包装重量,体积重)；按发货单明细汇总（数据源 `t_pms_product_spec`/`t_pms_package_info`，单位口径见 B-5/B-6） | BE | |
| 4.3 | 分摊预览接口：物流计划总费用×(本单重量/计划内全部发货单重量合计) | BE | |
| 4.4 | 费用保存接口：覆盖写入；按钮可见性按单据类型（调拨单类仅待发货；备货单类、FBA发货单类待发货+已发货） | BE | |
| 4.5 | 【物流费用】弹框（三种形态共用组件，按类型控制字段） | FE | |
| 4.6 | 发货前置校验：**调拨单类未保存费用禁止发货**（领星侧创建后不可改，PRD 强制）；费用=0 二次确认；备货单类、FBA发货单类不校验 | BE/FE | |
| 4.7 | 费用同步：保存/确认后按类型推送（调拨单不推，随创建提交；备货单 UpdateLogistics；FBA发货单 updateListLogistics），失败进重试 | BE | |

### M5 发货同步 @ wms 侧编排（1.5周）

| # | 任务 | 类型 | 说明 |
| --- | --- | --- | --- |
| 5.1 | 单据关联表 `t_wms_lx_shipment_doc` + 同步任务表 `t_lx_sync_task`（含重试） | DB | 见 3.1/3.4 |
| 5.2 | 发货后置同步编排：`ShipmentServiceImpl.shipping` 事务提交后按 `lx_doc_type` 投递同步任务（不阻断发货） | BE | 统一异步模型见 5.2 |
| 5.3 | 调拨单参数组装与提交（AddAllocationOrder type=2，费用、备注、sku+good_num） | BE | 见 5.3 |
| 5.4 | FBA发货单参数组装与提交（CreateSendedOrder + request_flag），轮询 searchProcessResult | BE | 见 5.4 |
| 5.5 | 备货单参数组装与提交（CreateInbound status=50/40，inbound_order_no=发货单号幂等，product_id 明细） | BE | 见 5.5 |
| 5.6 | 成功后回写领星单号；调拨单回查 getStorageAllocationList 缓存 product_id 与 IB 单号链路 | BE | |

### M6 收货与结束到货同步（1周）

| # | 任务 | 类型 | 说明 |
| --- | --- | --- | --- |
| 6.1 | 手动签收后：调拨单 partlyReceiveAllocationOrder / 备货单 inboundBatchesReceipt（均用 product_id+本次收货量），异步+重试 | BE | |
| 6.2 | 全部收货后：receiveAllocationOrder | BE | |
| 6.3 | 收货成功后回查领星入库单号（IB）回写收货单/入库单 | BE/DB | |
| 6.4 | 中台【结束到货】功能：待收货/部分收货可结束；状态→已收货；取消数=未收货；明细加 cancel_qty；回写发货单 | BE/DB/FE | |
| 6.5 | 结束到货同步：调拨单 finishReceiveAllocationOrder / 备货单 inboundCompleteReceipt；FBA发货单不处理 | BE | |
| 6.6 | 收货单列表：改名【待收货】、状态多选、取消数列 | FE | |

### M7 退运同步（1周）

| # | 任务 | 类型 | 说明 |
| --- | --- | --- | --- |
| 7.1 | 在途退运事务提交后按领星单据类型投递撤销任务（不阻断退运） | BE | |
| 7.2 | FBA发货单：InvalidShipmentSn（整单作废+恢复库存）；部分退运时"作废原单+按剩余数量重建"（待确认 B-11） | BE | |
| 7.3 | 调拨单：按退运数量分批收货（领星）→ 反向 type=1 调拨同数量归还，两步断点续做 | BE | 见 5.8.2 |
| 7.4 | 备货单：方案待定（B-15），本期先仅记录告警不自动处理 | BE | |
| 7.5 | 关联记录置【已作废】/重建记录；列表展示 | BE/FE | |

### M8 前端页面改造（1.5周，与 M5~M7 并行）

| # | 任务 | 类型 |
| --- | --- | --- |
| 8.1 | 调拨发货单列表：【领星单据】列（悬浮弹框：单据类型/单号/同步状态/费用/失败原因）、【预估物流费】【实际物流费】列、操作列【物流费用】按钮（按类型与状态显隐）、同步状态标识（待同步/同步中/失败可重试） | FE |
| 8.2 | 查看弹框【物流费用】页签；新增费用弹框组件 feeDialog.vue | FE |
| 8.3 | 收货单：改名、多选、取消数、【结束到货】按钮与确认弹框 | FE |
| 8.4 | 入库单列表：【备注】右侧【领星单据】列（领星IB单） | FE |
| 8.5 | 同步失败手动重试入口（列表操作或详情按钮） | FE |

### M9 异常、重试与监控（0.5周，贯穿）

| # | 任务 | 类型 | 说明 |
| --- | --- | --- | --- |
| 9.1 | 重试 job：扫描 `t_lx_sync_task` 执行（退避策略），超次数转人工告警 | BE | |
| 9.2 | 告警：工作台消息（ErrorMsgTypeEnums）+日志；钉钉机器人待确认 C-10 | BE | |
| 9.3 | 定时任务：token 刷新、基础数据每日刷新、对账（中台应同步单据 vs 关联记录） | BE | |

### M10 联调、测试与上线（1.5周）

| # | 任务 | 类型 | 说明 |
| --- | --- | --- | --- |
| 10.1 | 逐接口校准签名、参数、限流（签名资料由用户提供） | 联调 | |
| 10.2 | 场景用例：三类单据×发货/收货/部分收货/结束到货/退运（全部+部分）/费用/重试/并发 | 测试 | 见第 8 节 |
| 10.3 | 上线灰度：调拨单→备货单→FBA发货单；存量不追溯 | 上线 | |

**总工期估算：约 7.5~8.5 周**（含 xzy-openapi 跨仓库开发；若 openapi 侧资源不足将顺延）。

## 2. 总体架构

### 2.1 组件落位（已确认：xzy-openapi）

```text
┌────────────────────────────┐        Feign (R<...>)        ┌──────────────────────────────┐
│ xzy-wms 服务（业务编排）    │ ───────────────────────────▶ │ xzy-openapi 服务（领星客户端） │
│ · 发货/收货/退运/费用 编排   │                              │ · RemoteLingxingController    │
│ · t_lx_sync_task 重试队列   │ ◀─────────────────────────── │ · LingXingClient(签名/限流/日志)│
│ · t_wms_lx_shipment_doc    │          同步返回             │ · LingXingTokenManager        │
│ · t_wms_lx_fee             │                              │ · t_lx_sync_log / t_auth_config│
└────────────────────────────┘                              └──────────────┬───────────────┘
                                                                            │ HTTPS
                                                                    https://openapi.lingxing.com
```

- `xzy-openapi-api`（Feign 契约，属 xzy 仓库）：新增 `RemoteLingxingService` + req/resp DTO。
- `xzy-openapi`（服务实现，独立仓库）：领星 client、token、签名、限流、接口日志。
- `xzy-wms`：所有业务编排、同步任务与重试、单据关联、费用；**中台操作一律不因领星失败而回滚**。

### 2.2 Feign 契约清单（RemoteLingxingService）

| 分组 | 方法 | 对应领星接口 |
| --- | --- | --- |
| 发货 | createAllocationOrder | AddAllocationOrder |
| 发货 | createSendedOrder / searchProcessResult | CreateSendedOrder / searchProcessResult |
| 发货 | createStockupOrder | CreateInbound（备货单） |
| 收货 | partlyReceiveAllocation / receiveAllAllocation / finishReceiveAllocation | 调拨单三个收货接口 |
| 收货 | batchesReceiptStockup / completeReceiptStockup | 备货单分批收货/结束到货 |
| 查询 | getAllocationList | getStorageAllocationList（回查IB/product_id/状态） |
| 退运 | invalidShipment | InvalidShipmentSn |
| 费用 | updateShipmentLogistics | updateListLogistics（FBA发货单） |
| 费用 | updateStockupLogistics | UpdateLogistics（备货单） |
| 基础数据 | sellerLists / warehouseLists / channelList / feeTypes / productLists | 五个基础数据接口 |

### 2.3 异步同步统一模型（wms 侧，核心变更）

所有领星写操作走同一套"任务化"流程，中台事务与领星调用完全解耦：

```text
中台业务事务（发货/收货/结束到货/退运）提交成功
  → 写 t_lx_sync_task（bizType+bizId+payload，status=待执行）
  → 事务后异步立即执行一次（线程池/MQ）：
       Feign 调 xzy-openapi → 领星
       ├─ 成功 → 任务置成功；回写关联记录（领星单号等）
       ├─ 失败(业务错误，如库存不足/映射缺失) → 任务置【待数据补齐】：暂停自动重试，
       │     告警提示原因；数据补齐后支持手动重试（或数据变更触发重试）
       └─ 失败(网络/超时/限流) → 任务置【重试中】，退避重试（1min/5min/30min/2h/6h，
             上限如10次），超限告警转人工
```

任务状态机：`0待执行 → 1执行中 → 2成功 / 3重试中 / 4待数据补齐 / 5放弃(转人工)`；多步骤任务（退运两步法）用 `step` 字段记录断点。

**幂等保障**
- FBA发货单：request_flag 唯一且重试复用；结果经 searchProcessResult 查询，绝不重复提交。
- 备货单：`inbound_order_no`（客户参考号）= 中台发货单号，领星侧唯一约束天然幂等。
- 调拨单：无原生幂等键 → 提交前先将任务置【执行中】（乐观锁唯一占用），超时未应答的重试须先经 getStorageAllocationList 按（始发仓+目的仓+日期+备注）查重，防重复建单；为此备注末尾追加中台发货单号（待确认 B-12）。

### 2.4 一致性策略总览（V2：全部非阻断）

| 场景 | 中台动作 | 领星动作 | 失败处理 |
| --- | --- | --- | --- |
| 发货（三类单据） | 事务正常完成、置已发货 | 异步创建执行单据 | 重试/待数据补齐+告警；列表可查同步状态 |
| 收货（分批/全部） | 事务正常完成 | 异步收货 | 同上 |
| 结束到货 | 事务正常完成 | 异步结束到货 | 同上 |
| 退运 | 事务正常完成 | 异步作废/收货+反向调拨 | 同上（两步断点） |
| 费用保存 | 保存成功 | 异步覆盖推送（备货单/发货单） | 同上；再次编辑保存会触发重推 |

## 3. 数据库设计（DDL 草案，以评审为准）

### 3.1 新增：领星单据关联表 `t_wms_lx_shipment_doc`（wms 库）

```sql
CREATE TABLE t_wms_lx_shipment_doc (
  id              BIGINT AUTO_INCREMENT PRIMARY KEY,
  shipment_id     BIGINT       NOT NULL COMMENT '中台调拨发货单ID',
  allocate_id     BIGINT       NULL COMMENT '调拨计划ID（冗余）',
  doc_type        VARCHAR(32)  NOT NULL COMMENT 'ALLOCATION=调拨单 / FBA_SHIPMENT=FBA发货单 / STOCKUP=备货单',
  lx_order_sn     VARCHAR(64)  NULL COMMENT '领星单号（TF*/SP*/OWS*）',
  request_flag    VARCHAR(64)  NULL COMMENT 'CreateSendedOrder 幂等标识',
  sync_status     TINYINT      NOT NULL DEFAULT 0 COMMENT '0待同步 1同步中 2成功 3失败 4待数据补齐 5已作废',
  lx_ib_no        VARCHAR(64)  NULL COMMENT '领星入库单号（收货后回查）',
  error_msg       VARCHAR(1024) NULL,
  invalid_time    DATETIME     NULL COMMENT '作废时间（退运）',
  create_time DATETIME, update_time DATETIME, create_by VARCHAR(64), update_by VARCHAR(64),
  del_flag TINYINT DEFAULT 0,
  KEY idx_shipment (shipment_id), KEY idx_flag (request_flag), KEY idx_sn (lx_order_sn)
) COMMENT '调拨发货单-领星单据关联表（退运重建时新增记录，旧记录保留）';
```

### 3.2 新增：发货单物流费用表 `t_wms_lx_fee`（wms 库）

```sql
CREATE TABLE t_wms_lx_fee (
  id                    BIGINT AUTO_INCREMENT PRIMARY KEY,
  shipment_id           BIGINT NOT NULL COMMENT '调拨发货单ID',
  allocate_type         VARCHAR(16) NOT NULL COMMENT '分摊方式 VOLUME=体积重 CHARGEABLE=计费重',
  predict_logistics_fee DECIMAL(18,4) DEFAULT 0 COMMENT '物流费用(预估)',
  predict_other_fee     DECIMAL(18,4) DEFAULT 0 COMMENT '其他费用(预估)',
  predict_total_fee     DECIMAL(18,4) DEFAULT 0 COMMENT '费用合计(预估)',
  actual_logistics_fee  DECIMAL(18,4) DEFAULT 0 COMMENT '物流费用(实际)',
  actual_other_fee      DECIMAL(18,4) DEFAULT 0 COMMENT '其他费用(实际)',
  actual_total_fee      DECIMAL(18,4) DEFAULT 0 COMMENT '费用合计(实际)',
  actual_weight         DECIMAL(18,4) DEFAULT 0 COMMENT '总实重KG',
  volume_weight         DECIMAL(18,4) DEFAULT 0 COMMENT '总体积重KG',
  chargeable_weight     DECIMAL(18,4) DEFAULT 0 COMMENT '总计费重KG',
  lx_sync_status        TINYINT DEFAULT 0 COMMENT '0无需同步 1待同步 2成功 3失败',
  lx_sync_time DATETIME NULL, lx_error VARCHAR(1024) NULL,
  create_time DATETIME, update_time DATETIME, create_by VARCHAR(64), update_by VARCHAR(64),
  del_flag TINYINT DEFAULT 0,
  UNIQUE KEY uk_shipment (shipment_id)
) COMMENT '调拨发货单物流费用表';
```

### 3.3 新增：同步任务表 `t_lx_sync_task`（wms 库，编排+重试一体）

```sql
CREATE TABLE t_lx_sync_task (
  id              BIGINT AUTO_INCREMENT PRIMARY KEY,
  biz_type        VARCHAR(48) NOT NULL COMMENT 'CREATE_ALLOCATION/CREATE_FBA_SHIPMENT/POLL_FBA_RESULT/CREATE_STOCKUP/RECEIVE_PART_ALLOCATION/RECEIVE_ALL_ALLOCATION/FINISH_RECEIVE_ALLOCATION/RECEIVE_PART_STOCKUP/FINISH_RECEIVE_STOCKUP/INVALID_SHIPMENT/REBUILD_FBA_SHIPMENT/RETURN_ALLOCATION/UPDATE_FEE_SHIPMENT/UPDATE_FEE_STOCKUP',
  biz_id          BIGINT NOT NULL COMMENT '业务主键（发货单/收货单/退运单ID）',
  doc_id          BIGINT NULL COMMENT '关联 t_wms_lx_shipment_doc.id',
  payload         TEXT COMMENT '执行上下文JSON（请求参数快照）',
  step            VARCHAR(32) DEFAULT 'INIT' COMMENT '多步骤断点（退运：RECEIVED/REVERSED）',
  status          TINYINT DEFAULT 0 COMMENT '0待执行 1执行中 2成功 3重试中 4待数据补齐 5放弃',
  retry_count     INT DEFAULT 0, max_retry INT DEFAULT 10,
  next_retry_time DATETIME NULL,
  error_msg       VARCHAR(2048) NULL,
  create_time DATETIME, update_time DATETIME,
  KEY idx_scan (status, next_retry_time), KEY idx_biz (biz_type, biz_id)
) COMMENT '领星同步任务表（含重试）';
```

### 3.4 新增：领星基础数据缓存表（wms 库，经 openapi Feign 拉取）

```sql
-- 店铺
CREATE TABLE t_lx_seller (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  sid BIGINT NOT NULL COMMENT '领星店铺ID',
  seller_id VARCHAR(64) NOT NULL COMMENT '亚马逊卖家账号',
  marketplace_id VARCHAR(64) NOT NULL COMMENT '市场ID',
  name VARCHAR(128), mid BIGINT, region VARCHAR(16), status TINYINT,
  sync_time DATETIME, del_flag TINYINT DEFAULT 0,
  UNIQUE KEY uk_seller_mkt (seller_id, marketplace_id)
) COMMENT '领星店铺缓存';

-- 仓库（WarehouseLists type=1/3/4 全量）
CREATE TABLE t_lx_warehouse (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  lx_wid BIGINT NOT NULL COMMENT '领星仓库ID',
  name VARCHAR(128), type TINYINT COMMENT '1本地仓 3海外仓 4平台仓 6AWD',
  sub_type TINYINT NULL COMMENT '海外仓子类型 1无API 2有API（type=3时有效）',
  country_code VARCHAR(16), t_warehouse_code VARCHAR(64), t_status TINYINT,
  sync_time DATETIME, del_flag TINYINT DEFAULT 0,
  UNIQUE KEY uk_wid (lx_wid)
) COMMENT '领星仓库缓存';

-- 头程物流渠道
CREATE TABLE t_lx_channel (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  lx_channel_id BIGINT NOT NULL,
  channel_name VARCHAR(128), billing_type TINYINT, volume_calc_param INT,
  provider_id BIGINT, provider_name VARCHAR(128), enabled TINYINT,
  is_default TINYINT DEFAULT 0 COMMENT '默认物流商渠道标记（配置项）',
  sync_time DATETIME, del_flag TINYINT DEFAULT 0,
  UNIQUE KEY uk_channel (lx_channel_id)
) COMMENT '领星头程物流渠道缓存';

-- 其他费类型
CREATE TABLE t_lx_fee_type (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  fee_type_id VARCHAR(64) NOT NULL, name VARCHAR(128), remark VARCHAR(512),
  sync_time DATETIME, del_flag TINYINT DEFAULT 0,
  UNIQUE KEY uk_fee_type (fee_type_id)
) COMMENT '领星其他费类型缓存';

-- 产品映射（备货单创建/分批收货必用）
CREATE TABLE t_lx_product_map (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  sku VARCHAR(128) NOT NULL, lx_product_id BIGINT NOT NULL,
  sync_time DATETIME, del_flag TINYINT DEFAULT 0,
  UNIQUE KEY uk_sku (sku)
) COMMENT '领星产品ID映射缓存';
```

> 仓库映射复用 `t_warehouse_mapping`（whId + serviceProviderCode=LINGXING_ERP + extWhCode=领星wid + authConfigId）。
> 接口级请求/响应日志 `t_lx_sync_log` 落在 **xzy-openapi 服务库**（客户端侧全量留痕），wms 侧只留任务状态与错误摘要。

### 3.5 现有表变更

| 表 | 变更 | 说明 |
| --- | --- | --- |
| `t_wms_allocate` | + `allocate_category` VARCHAR(16)；+ `lx_doc_type` VARCHAR(32) | 分类与下游单据类型 |
| `t_wms_shipment` | + `lx_doc_type` VARCHAR(32)；+ `lx_sync_status` TINYINT（0无需/1待同步/2同步中/3成功/4失败重试中/5待数据补齐） | 驱动分支与列表展示 |
| `t_wms_receipt_new` | + `lx_ib_no` VARCHAR(64) | 领星入库单号 |
| `t_wms_receipt_item` | + `cancel_qty` INT DEFAULT 0 | 结束到货取消数 |
| `t_wms_shipment_item` | + `cancel_qty` INT DEFAULT 0 | 结束到货回写发货单取消数（A-6） |
| `t_wms_inbound` | + `lx_ib_no` VARCHAR(64) | 入库单列表【领星单据】列 |
| `t_auth_config` | 初始化 LINGXING_ERP 凭据（appId/appSecret/token/refreshToken） | openapi 库 |

## 4. 领星接口使用清单（V2）

| 场景 | 接口（本地文档） | API Path | 关键点 |
| --- | --- | --- | --- |
| 令牌 | GetToken / RefreshToken | /api/auth-server/oauth/access-token、/refresh | expires_in≈7199s；refresh_token 一次性 |
| 店铺 | SellerLists | /erp/sc/data/seller/lists | 全量；唯一键 sid |
| 仓库 | WarehouseLists | /erp/sc/data/local_inventory/warehouse | type=1本地/3海外/4平台；海外仓 sub_type=1无API/2有API |
| 渠道 | ChannelList | /erp/sc/data/local_inventory/channelList | 默认物流商渠道（备货单必传、按计费重分摊必传） |
| 其他费 | GetHeadLogisticsFeeTypes | .../getHeadLogisticsFeeTypes | 缓存 |
| 产品 | ProductLists | .../local_inventory/productList | sku_list≤1000；返回 product_id |
| 建调拨单 | AddAllocationOrder | .../StorageAllocation/addAllocationOrder | type=2 待收货；sku+good_num；费用创建后不可改；返回 TF* 单号；限流1 |
| 建FBA发货单 | CreateSendedOrder | /erp/sc/storage/shipment/createSendedOrder | **异步**；扣发货仓库存；需 request_flag；list 需 seller_id/marketplace_id/shipment_id/fnsku/sku/num |
| 查创建结果 | searchProcessResult | .../shipment/searchProcessResult | 0处理中/1成功/2失败；成功返回 SP* |
| 建备货单 | CreateInbound | /erp/sc/routing/owms/inbound/createInbound | **inbound_order_no 唯一（幂等键=发货单号）**；s_wid 仅限本地仓；r_wid 仅限海外仓；product_list 必传 product_id；status 40待发货/50待收货/60已完成；**收货仓为三方海外仓的备货单状态只到待发货（联调验证 B-14）**；返回 OWS* |
| 备货单发货 | SendInbound | .../owms/inbound/sendInbound | 待发货→待收货并扣库存（本期直接 status=50 创建，备用） |
| 调拨单列表 | getStorageAllocationList | .../StorageAllocation/getStorageAllocationList | item_list 含 product_id；inbound_order_sn(IB)；status 19待收货/20已完成 |
| 调拨单收货 | partlyReceiveAllocationOrder / receiveAllocationOrder | 同前缀 | 分批必传 product_id；全部收货传 orderSnMany |
| 调拨单结束到货 | finishReceiveAllocationOrder | 同前缀 | order_sn |
| 备货单收货 | inboundBatchesReceipt | /erp/sc/routing/owms/inbound/batchesReceipt | overseas_order_no + product_id + current_receive_num |
| 备货单结束到货 | inboundCompleteReceipt | .../inbound/completeReceipt | overseas_order_no |
| 更新发货单费用 | updateListLogistics | .../shipment/updateListLogistics | **覆盖式**（不传也置空）；限流2 |
| 更新备货单费用 | UpdateLogistics | .../owms/inbound/updateLogistics | 覆盖式；旧版/新版物流信息均支持 |
| 作废发货单 | InvalidShipmentSn | /basicOpen/openapi/fbaShipment/shipmentSn/invalid | shipmentNos[]；isReturnStock=1 恢复库存 |
| 删除备货单 | DeleteOverSeaStockOrder | /basicOpen/overSeaWarehouse/stockOrder/delete | 仅待审批/待配货/待发货/已驳回可删（**待收货不可删**，退运不能依赖此接口） |
| （备用）编辑发货单 | updateInboundShipmentListMws | .../shipment/updateInboundShipmentListMws | 发货量不允许大于计划量，已发货单适用性存疑，仅备用 |
| （备用）货件状态 | UpdateShipmentActualStatus | 同前缀 | PRD 明确结束到货不调，本期不用 |

## 5. 业务流程详细设计（V2）

### 5.1 调拨计划分类判定

仓库类型取 `t_wms_warehouse.wh_type`：1自营、2第三方、3供应商、**4平台仓**。平台仓中 **平台=亚马逊（platformId=AMAZON）为 FBA 仓，其余为非FBA平台仓**（代码中已有该判定：`PlatformEnums.AMAZON.getId().equals(warehouse.getPlatformId())`）。

> 规则表达遵循确认口径：平台仓与第三方仓是不同类型，处理一致时条件写为"平台仓或第三方仓"。

```text
集货调拨 COLLECT：
  始发仓 ∈ {自营仓, 供应商仓} 且 目的仓 ∈ {自营仓, 供应商仓}
  → lx_doc_type = ALLOCATION（调拨单，费用通常0，创建后不可改）

移库调拨 MOVE：
  始发仓 ∈ {自营仓, 供应商仓} 且 目的仓 = FBA仓（平台仓且平台=亚马逊）
  → lx_doc_type = FBA_SHIPMENT（发货单，费用可覆盖）
  始发仓 ∈ {自营仓, 供应商仓} 且 目的仓 ∈ {非FBA平台仓, 第三方仓}
  → lx_doc_type = STOCKUP（备货单，费用可覆盖【B-16待确认】）

海外调拨 OVERSEA：
  始发仓 ∈ {第三方仓, 平台仓（含FBA仓与非FBA平台仓）} 且 目的仓 = FBA仓
  → lx_doc_type = FBA_SHIPMENT（发货单，产生费用）
  始发仓 ∈ {第三方仓, 平台仓（含FBA仓与非FBA平台仓）} 且 目的仓 ∈ {非FBA平台仓, 第三方仓}
  → lx_doc_type = ALLOCATION（调拨单，产生费用，创建后不可改）
```

- `is_fba`（目的仓是否FBA）与分类互为校验；分类冗余到发货单，发货时直接分支。
- 海外调拨以 FBA 仓为始发仓时（如滞销转移），领星调拨单能否以平台仓为出库仓需联调验证（B-17）。

### 5.2 发货同步总流程（三类共用骨架）

```text
【发货点击】
1. 现有校验与中台事务：装箱校验 → 始发仓出库 → 发货单置已发货 → 重算物流计划（与现状一致）
   · 调拨单类额外前置校验：费用已保存（未保存则拦截发货——这是数据完备性校验，非领星同步失败；
     费用合计=0 时前端二次确认）
2. 事务提交后按 lx_doc_type 创建 t_lx_sync_task（CREATE_*）+ 预写 t_wms_lx_shipment_doc（待同步）
3. 异步执行器领取任务 → Feign 调 openapi → 领星（细节见 5.3/5.4/5.5）
4. 成功：回写领星单号，发货单 lx_sync_status=成功
   失败：按 2.3 策略重试/待数据补齐+告警；中台单据保持已发货，不受影响
5. 列表始终展示同步状态；失败提供手动重试入口
```

### 5.3 发货 → 领星调拨单（ALLOCATION）

**参数组装（AddAllocationOrder）**

| 领星参数 | 取值 |
| --- | --- |
| type | 2（完整调拨→待收货） |
| sys_wid / sys_to_wid | 始发/目的仓映射的领星本地仓ID |
| freight_fee / other_fee | 费用表实际物流费 / 实际其他费 |
| fee_part_type | 0（不分摊：中台已分摊到发货单粒度，B-3） |
| remark | 排柜计划号+物流计划号+SO/物流订单号 [+中台发货单号，B-12待确认] |
| product_list[].sku / good_num | 发货明细 sku / shipmentQty（本期不传次品，B-4） |

**执行与幂等**
- 任务置执行中（乐观锁唯一占用）→ 调用 → 成功写 TF* 单号。
- 超时未应答的重试：先按（始发仓+目的仓+日期+备注）查 getStorageAllocationList 判重，已存在则直接取单号置成功，否则重新提交。
- 成功后异步回查该单 item_list，缓存 sku→product_id 到 `t_lx_product_map`（分批收货用）。

**常见失败原因处理**
- 映射缺失（仓库未映射）→ 待数据补齐：维护映射后手动重试。
- 领星库存不足 → 告警人工核对领星库存（中台已出库，领星账差异需人工决策，见 7.1 对账）。

### 5.4 发货 → 领星FBA发货单（FBA_SHIPMENT）

**前置数据链（缺失则任务置"待数据补齐"，不阻塞中台发货）**
- 始发仓→领星本地仓映射；`allocate.shopId`→`t_amazon_seller_channel_config`→seller_id+marketplace_id；`allocate.shipmentId`（FBA货件号）非空；明细 fnsku 非空。

**参数组装（CreateSendedOrder）**

| 领星参数 | 取值 |
| --- | --- |
| sys_wid | 始发仓映射的领星本地仓ID |
| actual_shipment_time | 发货当日 |
| expected_arrival_date | allocate.planArrivalDate |
| head_fee_type | 与费用分摊方式联动：体积重→2；计费重→0且传 logistics_channel_id（默认物流商渠道）；未录费用默认 2（体积重）。**分摊方式在创建时锁定，费用金额可后续覆盖**（C-6已确认口径） |
| remark | 排柜计划号+物流计划号+SO/物流订单号 |
| request_flag | `{shipmentNo}_{时间戳}_{rand}`，重试复用同一标识 |
| list[].seller_id / marketplace_id | t_amazon_seller_channel_config |
| list[].shipment_id | allocate.shipmentId（=发货单 inboundNo） |
| list[].fulfillment_network_sku / sku / num | shipment_item.fnsku / sku / shipmentQty |
| head_logistics_list | 发货前已录费用则随单提交（预估+实际）；否则不传 |

**结果处理（任务内完成轮询）**
```text
调用 CreateSendedOrder
 ├─ 受理前失败（参数/网络）→ 重试策略
 └─ 已受理 → 任务转 POLL_FBA_RESULT，轮询 searchProcessResult（2s间隔，单轮上限60s；
      未出结果则任务重新入队，下一轮继续，最长24h）
      ├─ process_status=1 → 写 SP* 单号，成功
      ├─ process_status=2 → 记录 error_details；按错误性质转【待数据补齐】或继续重试
      └─ 24h 未出结果 → 放弃+告警，人工到领星后台按 request_flag 核对
```
> 因 request_flag 幂等+只查不重发，轮询期间不存在重复建单风险；中台发货单始终可正常操作。

### 5.5 发货 → 领星备货单（STOCKUP）

**前置数据链**
- 始发仓→领星**本地仓**映射（s_wid 仅限本地仓；供应商仓是否存在于领星本地仓见 B-13）；
- 目的仓→领星**海外仓**映射（r_wid 仅限海外仓；注意海外仓 sub_type 影响状态上限，见 B-14）；
- 明细 sku→product_id（`t_lx_product_map`，缺失即待数据补齐）；
- 默认物流商渠道 logistics_id（必传）。

**参数组装（CreateInbound）**

| 领星参数 | 取值 |
| --- | --- |
| inbound_order_no | **中台发货单号（唯一幂等键）** |
| s_wid / r_wid | 始发本地仓 / 目的海外仓映射ID |
| status | 50（待收货）；若目的为三方海外仓且联调确认状态只到待发货，则 40（B-14） |
| logistics_id | 默认物流商渠道ID |
| share_id | 与费用分摊方式联动：0计费重/2体积重（同 5.4 口径） |
| estimated_time | allocate.planArrivalDate |
| remark | 排柜计划号+物流计划号+SO/物流订单号 |
| logistics_list（旧版） | 已录费用则随传：logistics_order_no=SO/物流订单号、logistics_money=预估物流费、other_money=预估其他费、real_*=实际费用；币种 CNY（C-7） |
| product_list[].product_id / stock_num / receive_num | product_id 映射 / shipmentQty / 0 |

**执行与幂等**：inbound_order_no 唯一约束保证重试安全；成功回写 OWS* 单号。

### 5.6 物流费用模块（V2：三类单据三种行为）

**5.6.1 重量计算与分摊（同 V1）**

```text
单品体积重(KG) = 包装长(cm)×包装宽(cm)×包装高(cm) / 6000 / 单箱数量
单品计费重(KG) = MAX(单品包装重量, 单品体积重)
单发货单分摊费用 = 物流计划总费用 × (本发货单重量 / 计划内全部发货单重量合计)
分摊方式：调拨单类仅【体积重】；备货单类、FBA发货单类支持【体积重/计费重】，默认体积重
无物流计划的发货单只计算自身（PRD）
```

**5.6.2 费用弹框与按钮规则**

| 单据类型 | 按钮可见状态 | 可编辑字段 | 发货前校验 |
| --- | --- | --- | --- |
| 调拨单类（集货/海外调拨→非FBA） | 仅【待发货】 | 物流费(实际)、其他费(实际) | **未保存费用禁止发货**；合计=0 二次确认 |
| 备货单类（移库→非FBA平台仓或第三方仓） | 待发货+已发货 | 预估两项+实际两项 | 不校验（可后补，建议发货前录入） |
| FBA发货单类 | 待发货+已发货 | 预估两项+实际两项 | 不校验（可后补，PRD） |

弹框结构同 V1（主单信息/物流计划与SO/重量汇总/分摊方式/费用编辑/下方发货单分摊预览；【确定】覆盖写入；批量保存范围见 A-5）。

**5.6.3 费用同步领星**

| 单据类型 | 同步方式 |
| --- | --- |
| 调拨单类 | 费用仅在 AddAllocationOrder 创建时随单提交，**创建后不可更改**（中台侧费用编辑仅限发货前） |
| 备货单类 | 创建时随传（旧版 logistics_list）；发货后每次【确定】调 **UpdateLogistics 覆盖重推**（支持预估+实际） |
| FBA发货单类 | 创建时随传（head_logistics_list）；发货后每次【确定】调 **updateListLogistics 覆盖重推** |

覆盖式接口注意事项：预估/实际两组必须每次完整传（不传即置空）；费用同步失败进重试，中台费用记录不受影响；费用同步任务与单据创建任务独立（单据未创建成功时费用同步任务等待其完成）。

**5.6.4 updateListLogistics / UpdateLogistics 参数组装（FBA发货单）**

```text
order_sn / overseas_order_no = 领星单号
logistics_list_type = 1（新版；备货单 UpdateLogistics 同理，新版字段一致）
head_logistics_list:
  tracking_list = []（SO号是否作为跟踪号同步：C-8 维持本期不同步，仅备注携带）
  estimate_expenses_list: logistics_fee=预估物流费，other_fee_arr=[{fee_type_id=默认其他费类型,
      other_amount=预估其他费}]，price/币种按 C-7 口径
  actual_expenses_list: logistics_fee=实际物流费，weight=总实重，volume=总体积(m³)，
      tax_fee=0，other_fee_arr=实际其他费
```

**5.6.5 页面展示**：列表【预估物流费】【实际物流费】列取费用表合计；查看弹框【物流费用】页签含同步状态与失败原因。

### 5.7 收货同步（V2：三类）

中台收货事务一律先行完成，领星调用异步+重试（失败不回滚中台）。

| 中台动作 | 调拨单类 | 备货单类 | FBA发货单类 |
| --- | --- | --- | --- |
| 手动签收（分批） | partlyReceiveAllocationOrder：order_sn + product_list(product_id, received_good_num=本次签收数) | inboundBatchesReceipt：overseas_order_no + product_list(product_id, current_receive_num) | 不调领星（FBA自动收货） |
| 全部收货 | receiveAllocationOrder(orderSnMany) | （累计分批至全量即可；如需一次性全部收货无对应接口，维持分批） | 不调领星 |
| 结束到货 | finishReceiveAllocationOrder(order_sn) | inboundCompleteReceipt(overseas_order_no) | 不调领星 |
| 领星入库单号回写 | 收货后查 getStorageAllocationList 取 inbound_order_sn(IB) 回写 | 备货单本身即入库单（OWS*），直接展示 | — |

- product_id 来源：建单时缓存的 `t_lx_product_map`（缺失则任务待数据补齐，补齐后重试）。
- 分批收货数量以中台本次签收数为准；领星侧累计收货数与中台对账（对账 job）。

**中台【结束到货】功能**（同 V1，补充备货单分支）：
```text
入口：收货单（状态=待收货/部分收货）【结束到货】
中台事务：receive_status→已收货(10)；明细 cancel_qty=quantity-receivedQty（未收货清零）；
  已收货数与取消数回写发货单明细；该收货链路关闭（无需继续下推）
领星异步：调拨单→结束到货；备货单→结束到货；FBA发货单→不处理
```

### 5.8 退运同步（V2：支持部分退运）

触发：在途退运单（ReturnOnroad）事务提交后，按发货单关联的领星单据类型投递撤销任务。退运支持部分（整箱倍数），领星侧按**实际退运数量**处理。

**5.8.1 FBA发货单类**
```text
全部退运（退运数=发货数）：
  InvalidShipmentSn(shipmentNos=[SP*], isReturnStock=1, isReturnStockAux=0,
      cancelReason=中台退运单号) → 关联记录置已作废；后续可由货件重新下推发货单重建
部分退运（B-11 待确认，建议方案）：
  1) InvalidShipmentSn 作废原单（领星全量恢复库存）
  2) 按剩余数量（发货数-退运数）走 CREATE_FBA_SHIPMENT 重建新发货单（复用 5.4 流程，
     新 request_flag、新关联记录；备注附"退运重建：原单SP*"）
  3) 库存净效应：恢复全量-扣减剩余=恢复退运部分，与中台一致
```

**5.8.2 调拨单类（按退运数量部分处理）**
```text
步骤1（断点 RECEIVED）：partlyReceiveAllocationOrder
   order_sn=原调拨单，product_list=退运SKU及退运数量（received_good_num=退运数）
   —— 即"中台退运多少就在领星收货多少"（目的仓入账该部分）
步骤2（断点 REVERSED）：AddAllocationOrder type=1（简易调拨，创建即完成）
   sys_wid=原目的仓、sys_to_wid=原始发仓（领星本地仓映射）
   product_list=退运SKU及退运数量；费用0；remark="退运归还：退运单号，原单：TF*"
   —— 反向调拨同数量，把该部分库存归还始发仓
两步同一任务断点续做；完成后关联记录标注退运处理完成
剩余未退运数量：原调拨单仍为待收货，继续走正常收货流程
```

**5.8.3 备货单类（方案待定，B-15）**
领星无"海外仓→本地仓"反向接口，待收货备货单亦不可删除。候选方案：
1. 领星侧按退运数量 batchesReceipt 收进海外仓，其后物理退回由业务在领星线下处理（领星账暂挂海外仓）；
2. 中台侧正常退运，领星侧不处理并告警人工。
**本期实现**：备货单退运仅记录+告警转人工，不做自动同步（待业务确认方案后补充）。

### 5.9 发货单截单

截单仅发生在【待发货】，此时领星执行单据尚未创建（最晚时机同步），**截单无需调领星**；截单产生的同步任务（如有未执行的）随单据删除一并作废。现有 `resetAllocateToWaitPush`（截单后调拨计划回退待推单）已实现，本期仅回归（C-9）。

## 6. 前端改造清单（xzy-erp-ui，V2）

| 页面/组件 | 改造内容 |
| --- | --- |
| `views/wms/shipment/index.vue` | ①【领星单据】列：最新有效领星单号（类型图标：调拨单/FBA发货单/备货单；已作废灰色），hover 弹框展示类型、单号、同步状态、预估/实际物流费、失败原因；②【预估物流费】【实际物流费】列；③操作列【物流费用】按钮（按 5.6.2 显隐）；④同步状态标识（待同步/同步中/同步失败-重试中/待数据补齐），失败提供【重试】按钮 |
| `views/wms/shipment/dialog.vue` | 新增【物流费用】页签 |
| 新增 `views/wms/shipment/feeDialog.vue` | 费用弹框（三类形态共用，字段按类型控制） |
| `views/wms/receiptNew/**` | 【未收货】→【待收货】；状态筛选多选；列表+明细【取消数】列；【结束到货】按钮+确认弹框 |
| `views/wms/inbound/**` | 【备注】右侧【领星单据】列（领星IB单/备货单号） |
| `src/api/wms/*.ts` | 新增：费用查询/保存/分摊预览、结束到货、领星单据详情、手动重试 |

## 7. 异常处理、重试与监控（V2）

### 7.1 异常分级（全部不阻断中台）

| 级别 | 场景 | 处理 |
| --- | --- | --- |
| 自动重试 | 网络/超时/限流/领星内部错误 | 退避重试（1min/5min/30min/2h/6h...），上限次数后告警转人工 |
| 待数据补齐 | 仓库/店铺/产品映射缺失、fnsku/shipmentId 缺失、领星库存不足、参数业务错误 | 暂停自动重试；告警展示原因；数据补齐后手动重试（或随基础数据刷新自动重试一次） |
| 告警转人工 | 重试超限、FBA结果24h未出、对账差异、token 刷新失败 | 工作台消息+日志（钉钉机器人待确认 C-10） |

### 7.2 Token 与幂等（同 V1，落 openapi 侧）

- access_token 提前刷新；refresh_token 一次性、原子更新；失败重走 GetToken+告警。
- 幂等：request_flag（FBA）、inbound_order_no=发货单号（备货单）、任务乐观锁+查重（调拨单）。

### 7.3 定时任务（xxl-job）

| Job | 频率 | 职责 | 所在服务 |
| --- | --- | --- | --- |
| lxTokenRefreshJob | 100分钟 | token 提前刷新 | openapi |
| lxSyncTaskJob | 1分钟 | 扫描同步任务执行/续查/重试 | wms |
| lxBaseDataSyncJob | 每日02:00 | 店铺/仓库/渠道/费类/产品映射刷新 | wms（经Feign） |
| lxReconcileJob | 每日06:00 | 对账：应同步单据 vs 关联记录；领星单状态抽查（调拨单列表接口）；库存差异提示 | wms |

## 8. 测试要点（V2）

1. **三类单据全链路**：调拨单（集货0费用/海外含费用）、备货单、FBA发货单 各走：发货→领星建单→分批收货→全部收货/结束到货→退运（全部+部分）→重建。
2. **费用**：体积重/计费重分摊；调拨单发货前强制+0费用确认；备货单/发货单后补费用触发覆盖重推（领星侧核对金额与覆盖语义）；费用同步失败重试。
3. **非阻断验证**：领星接口全部失败场景下（停服/错参/无映射），中台发货/收货/退运均正常完成；同步状态正确展示；补齐后手动重试成功。
4. **幂等与并发**：任务重复执行不重复建单（request_flag/inbound_order_no/查重）；双击发货；重试与手动重试并发。
5. **部分退运**：调拨单退运部分数量→领星收货该数量+反向调拨同数量；剩余数量正常收货；FBA部分退运作废+重建后数量正确。
6. **限流与批量**：连续多单串行通过，不触发领星限流。
7. **展示一致性**：领星单据列/悬浮框/费用列/页签/取消数/IB单。
8. **对账**：领星库存增减与中台出入库逐笔核对。

## 9. 上线方案

1. **配置先行**：凭据、签名、仓库映射（本地仓/海外仓）、店铺核对、默认物流商渠道、其他费类型；冒烟一次三类建单+作废/删除。
2. **灰度顺序**：调拨单（流程最简）→ 备货单 → FBA发货单（异步链路最复杂）；按分类开关控制。
3. **存量策略**：仅上线后新下推发货单走新流程；存量不追溯。
4. **回滚**：关闭同步开关后发货回退纯中台流程；已建领星单按人工 SOP 处理。
5. **观察期**：一周每日核对对账 job 输出，无差异转常态监控。

## 10. 附：关键代码落点（改造参考，不含实现）

| 模块 | 位置 | 说明 |
| --- | --- | --- |
| Feign 契约 | `xzy-erp-private/xzy-openapi-api/.../remote/RemoteLingxingService.java`（新增）+ req/resp DTO | 约18方法 |
| 领星客户端 | `xzy-openapi` 服务（独立仓库）：controller + lingxing client/token/签名/限流 + `t_lx_sync_log` | 跨仓库，见 C-12 |
| 发货编排 | `xzy-wms .../service/impl/ShipmentServiceImpl.java#shipping` 事务后投递同步任务 | 不改中台事务本身 |
| 收货编排 | `ReceiptNewController#receive/receiveAll` + 新增 `finishReceive` | 事务后异步 |
| 退运编排 | `ReturnOnroadServiceImpl#createReturnOnroad` 事务后异步 | |
| 调拨计划 | `AllocateServiceImpl` 创建/保存处分类打标；`createShipmentByAllocate` 继承 | |
| 同步执行 | 新增 `LingXingSyncService`（任务领取/Feign调用/回写）+ `LingXingSyncTaskJob` | wms |
| 前端 | `xzy-erp-ui/src/views/wms/shipment/**、receiptNew/**、inbound/**` | 第 6 节 |

## 相关知识

[[退货与退运流程]]、[[发货与出库流程]]、[[收货与入库流程]]、[[库存]]、[[系统架构]]

## 变更记录

- 2026-09-01 V1 初稿
- 2026-09-02 V2：备货单纳入范围；架构改为 xzy-openapi 客户端+Feign；一致性策略改为"领星失败不阻断中台+异步重试"；退运支持部分数量（调拨单部分收货+反向调拨同数量）；分类条件明确平台仓（FBA=平台仓且平台=亚马逊）与第三方仓表达；费用三类行为（调拨单不可改/备货单可覆盖/发货单可覆盖）；新增备货单全套接口与流程、新增待确认 B-11~B-17
