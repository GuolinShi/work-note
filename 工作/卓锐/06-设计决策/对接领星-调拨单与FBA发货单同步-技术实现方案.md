---
type: design
status: pending-confirmation
last_verified: 2026-09-01
source: PRD+code+lingxing-api-docs
---

# 对接领星-调拨单&FBA发货单同步-技术实现方案

> 状态：**待确认**（本阶段仅做需求与实现方案确认，未改动任何代码）
> 依据：《对接领星-调拨单&FBA发货单同步-PRD》、领星API文档（20260520爬虫版，本地 `lingxing-api/raw-markdown`）、xzy ERP 代码库
> 待确认问题汇总见 [[对接领星-调拨单与FBA发货单同步-待确认问题清单]]

## 0. 需求与范围概述

### 0.1 业务目标

物流人员在中台录入【调拨计划】和【调拨发货单】，发货时将单据单向同步到领星：

| 中台单据场景 | 领星执行单据 | 费用 |
| --- | --- | --- |
| 集货调拨（自营仓/供应商仓 之间） | 调拨单（type=2 待收货） | 创建时必传（通常为 0） |
| 海外调拨（第三方仓/平台仓 → 非FBA平台仓/第三方仓） | 调拨单（type=2 待收货） | 创建时必传（产生费用） |
| 移库调拨（自营仓/供应商仓 → FBA仓） | FBA发货单（已发货） | 发货后补传（预估+实际） |
| 海外调拨（第三方仓/平台仓 → FBA仓） | FBA发货单（已发货） | 发货后补传（预估+实际） |
| 移库调拨（自营仓/供应商仓 → 非FBA平台仓/第三方仓） | 备货单 | **本期不做（PRD未提供备货单接口流程，待确认）** |

### 0.2 核心设计原则（来自 PRD）

1. 单向同步：中台 → 领星，不提供领星回拉中台的定时任务。
2. 强制覆盖：领星侧字段以中台最新数据为准。
3. 最晚时机同步、仅同步结果：中台【发货】点击时才创建领星执行单据，领星执行单据与中台发货单 **1:1**（领星接口不支持多仓发货）。
4. 库存锁定时机：中台创建【待发货】调拨发货单时锁中台库存；领星创建执行单据时锁/扣领星库存。
5. 异常处理：接口报错通过日志告警/异常事件反馈，不做字段回写。

### 0.3 本期实现范围

- 发货时创建领星【调拨单】（AddAllocationOrder，type=2）
- 发货时创建领星【已发货发货单】（CreateSendedOrder + searchProcessResult 异步结果查询）
- 调拨单收货同步：分批收货（partlyReceiveAllocationOrder）/ 全部收货（receiveAllocationOrder）
- 调拨单结束到货（finishReceiveAllocationOrder）+ 中台收货单【结束到货】功能
- 发货单收货：中台与领星各自收货，无需同步（FBA 自动同步收货数据）
- 退运：FBA发货单作废（InvalidShipmentSn）；调拨单"先收货+反向type=1调拨单"撤销法
- 物流费用：费用录入弹框、按体积重/计费重分摊、费用同步领星（updateListLogistics）
- 基础数据映射：亚马逊店铺（sid）、仓库（wid）、头程物流渠道（默认物流商）、其他费类型
- 页面展示：调拨发货单列表【领星单据】【预估物流费】【实际物流费】列及悬浮详情；入库单列表【领星单据】列；收货单状态改名与【取消数】

## 1. 需求落地实现步骤清单

> 里程碑按依赖顺序排列；M0/M1 为所有后续工作的前置。每个任务标注：后端(BE)/前端(FE)/DB/配置(CFG)。

### M0 准备与确认（0.5周）

| # | 任务 | 类型 | 产出 |
| --- | --- | --- | --- |
| 0.1 | 与产品/业务确认《待确认问题清单》全部条目（尤其：备货单场景处理、费用分摊方式锁定时机、发货失败策略、退运数量口径） | 沟通 | 确认结论回填本方案 |
| 0.2 | 申请领星开放平台 appId/appSecret，确认生产/测试环境与 IP 白名单要求 | 沟通 | 凭据入 Nacos/`t_auth_config` |
| 0.3 | 核对领星侧基础数据：仓库清单（本地仓/海外仓）、店铺授权、"默认物流商"渠道、其他费类型 | 沟通 | 映射初始数据 |
| 0.4 | 获取领星接口签名算法权威说明（本地爬虫文档缺 Guidance/TestSign 章节，需联调校准） | 沟通 | 签名实现依据 |

### M1 领星 OpenAPI 基础组件（1周）

| # | 任务 | 类型 | 说明 |
| --- | --- | --- | --- |
| 1.1 | 新建 `com.xzy.erp.wms.lingxing` 包：client（HTTP+签名+限流）、config、enums、dto | BE | 建议落在 xzy-wms 服务（见 2.1 架构决策） |
| 1.2 | Token 管理：access-token 获取（`/api/auth-server/oauth/access-token`）、refresh 续约（每个 refresh_token 只能用一次）、过期前主动刷新、并发取令牌加锁 | BE | 凭据存 `t_auth_config`（serviceProviderCode=LINGXING_ERP），运行态 token 落库+Redis |
| 1.3 | 签名与公共参数：按领星规则生成 sign，组装 access_token/timestamp/app_key；统一响应解析（code=0 成功） | BE | 签名细节待 0.4 校准 |
| 1.4 | 限流控制：领星多数接口令牌桶容量=1，客户端内置串行队列/信号量+请求间隔控制 | BE | 防止批量操作打爆限流 |
| 1.5 | 同步日志表 `t_lx_sync_log` + 请求/响应全量留痕（脱敏） | DB/BE | 问题排查与对账依据 |
| 1.6 | 重试框架：`t_lx_retry_task`（业务类型+payload+次数+下次执行时间）+ xxl-job 扫描执行 | DB/BE | 收货/结束到货/退运等补偿场景复用 |

### M2 基础数据映射（1周，可与 M1 并行）

| # | 任务 | 类型 | 说明 |
| --- | --- | --- | --- |
| 2.1 | 店铺映射：调 `BasicData/SellerLists` 全量拉取领星店铺 → 缓存 `t_lx_seller`（sid、seller_id、marketplace_id、status）；与 `gzzr_openapi.t_amazon_seller_channel_config` 按 seller_id+marketplace_id 建立映射 | BE/DB | 每日定时刷新 |
| 2.2 | 仓库映射：调 `Warehouse/WarehouseLists`（type=1本地仓/3海外仓/4平台仓）→ 缓存 `t_lx_warehouse`；提供中台仓库→领星wid映射维护（复用 `t_warehouse_mapping`，serviceProviderCode=LINGXING_ERP，extWhCode=领星wid） | BE/DB/CFG | FBA目的仓无需映射（由shipmentId决定），始发仓必须映射 |
| 2.3 | 物流渠道：调 `Logistics/ChannelList` → 缓存 `t_lx_channel`（含 provider、volume_calc_param、billing_type）；按"默认物流商"定位渠道 | BE/DB | 用于按计费重分摊时必传的 logistics_channel_id |
| 2.4 | 其他费类型：调 `FBA/GetHeadLogisticsFeeTypes` → 缓存 `t_lx_fee_type` | BE/DB | 费用上传 other_fee_arr 必传 fee_type_id |
| 2.5 | 产品ID映射：调 `Product/ProductLists`（sku_list 批量）→ 缓存 `t_lx_product_map`（sku↔领星product_id） | BE/DB | 调拨单分批收货接口必传 product_id |

### M3 调拨计划分类标识（0.5周）

| # | 任务 | 类型 | 说明 |
| --- | --- | --- | --- |
| 3.1 | `t_wms_allocate` 增加 `allocate_category`（集货调拨/移库调拨/海外调拨）与 `lx_doc_type`（ALLOCATION/FBA_SHIPMENT/STOCKUP/NONE） | DB/BE | 依据始发仓/目的仓 whType + is_fba 判定 |
| 3.2 | 调拨计划创建/保存时自动计算分类；历史数据刷新脚本 | BE | 分类规则见 5.1 |
| 3.3 | 调拨发货单生成时继承分类（`t_wms_shipment` 冗余 `lx_doc_type`，驱动发货分支） | BE | |

### M4 物流费用模块（1.5周）

| # | 任务 | 类型 | 说明 |
| --- | --- | --- | --- |
| 4.1 | 费用表 `t_wms_lx_fee`（按发货单粒度：预估/实际 物流费+其他费+合计、分摊方式、重量快照、同步状态） | DB | 见 3.2 |
| 4.2 | 重量计算服务：单品体积重=包装长×宽×高/6000/单箱数量；单品计费重=MAX(单品包装重量,体积重)；按发货单明细汇总 | BE | 数据源 `t_pms_product_spec`/`t_pms_package_info`，单位换算注意 |
| 4.3 | 费用分摊预览接口：输入物流计划总费用+分摊方式，按各发货单重量占比输出分摊明细 | BE | 公式：分摊额=总费用×(本单重量/物流计划全部发货单重量合计) |
| 4.4 | 费用保存接口：覆盖式写入；领星调拨单类单据仅【待发货】可保存，FBA发货单类【待发货/已发货】均可 | BE | 保存即记录，同步在发货/确认时触发 |
| 4.5 | 【物流费用】弹框（主单信息+物流计划/SO信息+重量汇总+分摊方式+四项费用编辑+下方发货单分摊预览） | FE | 对应 PRD 两个弹框形态（调拨单类/发货单类） |
| 4.6 | 发货前置校验：走领星调拨单的发货单，未保存费用禁止发货；费用合计=0 时二次确认提示 | BE/FE | FBA发货单类不校验费用 |

### M5 发货同步-领星调拨单（1周）

| # | 任务 | 类型 | 说明 |
| --- | --- | --- | --- |
| 5.1 | 单据关联表 `t_wms_lx_shipment_doc`（中台发货单↔领星单据，1:1 但支持作废后重建的多版本） | DB | 见 3.1 |
| 5.2 | `ShipmentServiceImpl.shipping` 增加领星分支：lx_doc_type=ALLOCATION 时先调 `AddAllocationOrder`（type=2），成功后再执行中台出库事务 | BE | 失败即阻断发货，详见 5.2 时序 |
| 5.3 | 调拨单创建参数组装：仓库映射、费用、备注（排柜计划号+物流计划号+SO/物流订单号）、product_list(sku+good_num) | BE | 无需 product_id/fnsku |
| 5.4 | 创建成功后回查 `getStorageAllocationList` 缓存 item_list 的 product_id 映射 | BE | 供分批收货使用 |
| 5.5 | 领星单据号回写与列表展示联动 | BE | |

### M6 发货同步-FBA发货单（1.5周，本需求最复杂点）

| # | 任务 | 类型 | 说明 |
| --- | --- | --- | --- |
| 6.1 | `shipping` 领星分支：lx_doc_type=FBA_SHIPMENT 时调 `CreateSendedOrder`，带唯一 `request_flag` | BE | 组装规则见 5.3 |
| 6.2 | 同步轮询 `searchProcessResult`（2s间隔、上限60s）；成功→执行中台出库事务；明确失败→发货失败回滚 | BE | |
| 6.3 | 轮询超时处理：发货单置【同步中】锁定态（保持待发货、禁止操作），后台 job 续查；查到成功自动补做出库事务，查到失败解锁 | BE/DB | 新增中间状态，防"领星已扣库存、中台未出库"的不一致 |
| 6.4 | 店铺/市场/shipmentId/fnsku 取值链校验：allocate.shopId→`t_amazon_seller_channel_config`→seller_id+marketplace_id；shipmentId 取 `t_wms_allocate.shipment_id`；fnsku 取 `t_wms_shipment_item.fnsku` | BE | 缺失即拦截并提示 |
| 6.5 | head_fee_type 与 logistics_channel_id 组装（与费用分摊方式联动） | BE | 见 5.3.5 与待确认 C-6 |

### M7 费用同步（0.5周）

| # | 任务 | 类型 | 说明 |
| --- | --- | --- | --- |
| 7.1 | 费用确认时对已发货的 FBA 发货单调 `updateListLogistics` 覆盖式更新（新版头程物流信息） | BE | 预估+实际费用、重量、体积 |
| 7.2 | 待发货阶段保存的费用随 `CreateSendedOrder` 一并提交（estimate/actual_expenses_list） | BE | 减少一次调用 |
| 7.3 | 费用同步失败进重试队列+告警；列表展示同步状态 | BE | |

### M8 收货与结束到货同步（1周）

| # | 任务 | 类型 | 说明 |
| --- | --- | --- | --- |
| 8.1 | `ReceiptNewController.receive`（手动签收）后：对关联领星调拨单调 `partlyReceiveAllocationOrder`（product_id+本次收货量） | BE | 领星失败不回滚中台收货，进重试队列 |
| 8.2 | `receiveAll`（全部收货）后调 `receiveAllocationOrder`（orderSnMany） | BE | |
| 8.3 | 收货成功后回查 `getStorageAllocationList` 取 `inbound_order_sn`（IB单号）回写收货单/入库单 | BE/DB | 入库单列表【领星单据】列数据源 |
| 8.4 | 新增【结束到货】功能：收货单（待收货/部分收货）可结束；状态→已收货；取消数=未收货；明细加 `cancel_qty`；回写发货单已收/取消数 | BE/DB/FE | PRD：结束到货后发货单无需继续下推收货单 |
| 8.5 | 【结束到货】时对领星调拨单调 `finishReceiveAllocationOrder`；FBA发货单类不调领星 | BE | |
| 8.6 | 收货单列表：【未收货】改名【待收货】、状态筛选改多选、列表与明细增加【取消数】列 | FE | |

### M9 退运同步（1周）

| # | 任务 | 类型 | 说明 |
| --- | --- | --- | --- |
| 9.1 | 在途退运（ReturnOnroad）创建成功后：FBA发货单类调 `InvalidShipmentSn`（isReturnStock=1 恢复库存） | BE | 单据关联置【已作废】，允许后续重新下推发货单再建领星单 |
| 9.2 | 调拨单类退运：`receiveAllocationOrder` 全部收货 → `AddAllocationOrder` type=1 反向调拨（目的仓→始发仓，费用0）归还库存 | BE | 两步均入重试队列，支持断点续做 |
| 9.3 | 退运撤销状态展示（领星单据列标注"已作废"） | BE/FE | |

### M10 前端页面改造（1.5周，可与 M5~M9 并行）

| # | 任务 | 类型 |
| --- | --- | --- |
| 10.1 | 调拨发货单列表：【领星单据】列（悬浮弹框）、【预估物流费】【实际物流费】列；操作列【物流费用】按钮（按单据类型与状态控制显隐） | FE |
| 10.2 | 调拨发货单查看弹框：新增【物流费用】页签 | FE |
| 10.3 | 收货单列表/明细：改名、多选状态、取消数列、【结束到货】按钮与确认弹框 | FE |
| 10.4 | 入库单列表：【备注】右侧【领星单据】列（领星IB单） | FE |

### M11 异常、重试与告警（0.5周，贯穿）

| # | 任务 | 类型 | 说明 |
| --- | --- | --- | --- |
| 11.1 | 同步失败统一告警（复用工作台消息 `ErrorMsgTypeEnums`，是否加钉钉机器人待确认） | BE | |
| 11.2 | 定时任务：searchProcessResult 续查、重试队列执行、基础数据每日刷新、孤儿单据对账 | BE | |
| 11.3 | 对账 job：中台已发货且应同步的单据 vs `t_wms_lx_shipment_doc`，发现缺失即告警 | BE | |

### M12 联调、测试与上线（1.5周）

| # | 任务 | 类型 | 说明 |
| --- | --- | --- | --- |
| 12.1 | 领星沙箱/生产联调：逐接口校准签名、参数、限流 | 联调 | |
| 12.2 | 场景用例覆盖：三类调拨×发货/收货/部分收货/结束到货/退运/费用（预估+实际）/异常重试/并发发货 | 测试 | 用例清单见第 8 节 |
| 12.3 | 上线策略：按调拨计划分类灰度（先集货调拨→海外调拨→FBA），存量单据不追溯，仅新单走新流程 | 上线 | |

**总工期估算：约 6~7 周**（1后端+1前端+0.5测试并行；M0 确认事项若拖延将顺延）。

## 2. 总体架构

### 2.1 组件落位（架构决策，待确认 C-1）

领星 OpenAPI 客户端**直接建在 `xzy-wms` 服务**内（`com.xzy.erp.wms.lingxing`），不经过 xzy-openapi 服务中转。理由：

1. 发货操作需要同步拿到领星结果以决定中台事务走向（成功才出库），走 Feign 中转增加一跳超时与分布式事务复杂度；
2. 本工作区不含 xzy-openapi 服务源码（仅 api 模块），避免跨仓库改动；
3. 与现有"推送第三方仓"（pushToThirdWh 走 openapi）并存：那套面向海外仓出入库指令，本需求面向领星 ERP 单据，语义不同。

> 若团队要求所有外部对接统一收口 xzy-openapi 服务，则改为：wms 定义 Feign 接口，openapi 服务实现领星 client——需另行排期（跨仓库）。

### 2.2 组件结构（xzy-wms 内）

```text
com.xzy.erp.wms.lingxing
├── client
│   ├── LingXingClient.java          // HTTP 执行、签名、公共参数、限流、日志
│   ├── LingXingTokenManager.java    // token 获取/刷新/缓存（Redis+DB）
│   └── LingXingRateLimiter.java     // 令牌桶=1 的串行化控制
├── config
│   └── LingXingProperties.java      // appId/appSecret/baseUrl（Nacos）
├── service
│   ├── LingXingSyncService.java     // 发货/收货/退运/费用 同步编排（对外门面）
│   ├── LingXingBaseDataService.java // 店铺/仓库/渠道/费类/产品 缓存刷新
│   ├── LingXingFeeService.java      // 费用计算、分摊、保存、同步
│   └── LingXingRetryService.java    // 重试队列执行
├── dto/req/resp                     // 各接口出入参
├── job                              // xxl-job：token刷新/结果续查/重试/基础数据/对账
└── enums                            // 单据类型/同步状态/业务类型
```

### 2.3 一致性策略总览

| 场景 | 策略 | 失败处理 |
| --- | --- | --- |
| 发货-调拨单 | 先领星后中台（领星成功→中台出库事务） | 领星失败：阻断发货；中台事务失败：极端场景，告警+人工（调拨单待收货不可撤销，补偿成本高） |
| 发货-FBA发货单 | 先领星（异步接口）轮询结果→成功后中台出库事务 | 明确失败：阻断；轮询超时：【同步中】锁定态+后台续查 |
| 收货/结束到货 | 中台为主：中台事务先成功，领星异步调用 | 领星失败：不回滚中台，进重试队列+告警 |
| 退运 | 中台退运事务先成功，领星撤销异步执行 | 失败重试队列（调拨单两步撤销记录断点） |
| 费用上传 | 中台保存成功→同步领星（覆盖式） | 失败重试队列；允许再次编辑触发重推 |

## 3. 数据库设计（DDL 草案，以评审为准）

### 3.1 新增：领星单据关联表 `t_wms_lx_shipment_doc`

> 中台发货单与领星执行单据 1:1，但退运作废后可重新下推重建，故按"每次同步一条记录"设计。

```sql
CREATE TABLE t_wms_lx_shipment_doc (
  id              BIGINT AUTO_INCREMENT PRIMARY KEY,
  shipment_id     BIGINT       NOT NULL COMMENT '中台调拨发货单ID',
  allocate_id     BIGINT       NULL COMMENT '调拨计划ID（冗余）',
  doc_type        VARCHAR(32)  NOT NULL COMMENT '领星单据类型 ALLOCATION=调拨单 / FBA_SHIPMENT=FBA发货单',
  lx_order_sn     VARCHAR(64)  NULL COMMENT '领星单号（TF*/SP*）',
  request_flag    VARCHAR(64)  NULL COMMENT 'CreateSendedOrder 幂等标识',
  sync_status     TINYINT      NOT NULL DEFAULT 0 COMMENT '0待同步 1同步中(轮询) 2成功 3失败 4已作废',
  lx_ib_no        VARCHAR(64)  NULL COMMENT '领星入库单号（调拨单收货后回查）',
  error_msg       VARCHAR(1024) NULL,
  invalid_time    DATETIME     NULL COMMENT '作废时间（退运）',
  create_time     DATETIME, update_time DATETIME,
  create_by       VARCHAR(64), update_by VARCHAR(64),
  del_flag        TINYINT DEFAULT 0,
  KEY idx_shipment (shipment_id), KEY idx_flag (request_flag), KEY idx_sn (lx_order_sn)
) COMMENT '调拨发货单-领星单据关联表';
```

### 3.2 新增：发货单物流费用表 `t_wms_lx_fee`

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
  lx_sync_time          DATETIME NULL, lx_error VARCHAR(1024) NULL,
  create_time DATETIME, update_time DATETIME, create_by VARCHAR(64), update_by VARCHAR(64),
  del_flag TINYINT DEFAULT 0,
  UNIQUE KEY uk_shipment (shipment_id)
) COMMENT '调拨发货单物流费用表';
```

### 3.3 新增：同步日志表 `t_lx_sync_log`

```sql
CREATE TABLE t_lx_sync_log (
  id           BIGINT AUTO_INCREMENT PRIMARY KEY,
  biz_type     VARCHAR(48) NOT NULL COMMENT 'CREATE_ALLOCATION/CREATE_FBA_SHIPMENT/POLL_RESULT/RECEIVE_PART/RECEIVE_ALL/FINISH_RECEIVE/INVALID_SHIPMENT/CANCEL_ALLOCATION/UPDATE_FEE/BASE_DATA',
  biz_no       VARCHAR(64) NULL COMMENT '中台单号',
  lx_order_sn  VARCHAR(64) NULL,
  request_json MEDIUMTEXT, response_json MEDIUMTEXT,
  status       TINYINT COMMENT '0失败 1成功',
  error_msg    VARCHAR(2048), cost_time BIGINT,
  create_time  DATETIME,
  KEY idx_biz (biz_type, biz_no), KEY idx_time (create_time)
) COMMENT '领星接口同步日志';
```

### 3.4 新增：重试任务表 `t_lx_retry_task`

```sql
CREATE TABLE t_lx_retry_task (
  id             BIGINT AUTO_INCREMENT PRIMARY KEY,
  biz_type       VARCHAR(48) NOT NULL,
  biz_id         BIGINT NOT NULL COMMENT '业务主键（发货单/收货单/退运单ID）',
  payload        TEXT COMMENT '执行所需上下文JSON',
  step           VARCHAR(32) DEFAULT 'INIT' COMMENT '多步骤断点（如调拨单撤销：RECEIVED/REVERSED）',
  status         TINYINT DEFAULT 0 COMMENT '0待执行 1执行中 2成功 3放弃(超次数)',
  retry_count    INT DEFAULT 0, max_retry INT DEFAULT 10,
  next_retry_time DATETIME,
  error_msg      VARCHAR(2048),
  create_time DATETIME, update_time DATETIME,
  KEY idx_scan (status, next_retry_time)
) COMMENT '领星同步重试任务表';
```

### 3.5 新增：领星基础数据缓存表

```sql
-- 店铺（SellerLists）
CREATE TABLE t_lx_seller (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  sid BIGINT NOT NULL COMMENT '领星店铺ID',
  seller_id VARCHAR(64) NOT NULL COMMENT '亚马逊卖家账号',
  marketplace_id VARCHAR(64) NOT NULL COMMENT '市场ID',
  name VARCHAR(128), mid BIGINT, region VARCHAR(16), status TINYINT,
  sync_time DATETIME, del_flag TINYINT DEFAULT 0,
  UNIQUE KEY uk_seller_mkt (seller_id, marketplace_id)
) COMMENT '领星店铺缓存';

-- 仓库（WarehouseLists type=1/3/4）
CREATE TABLE t_lx_warehouse (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  lx_wid BIGINT NOT NULL COMMENT '领星仓库ID',
  name VARCHAR(128), type TINYINT COMMENT '1本地仓 3海外仓 4平台仓 6AWD',
  country_code VARCHAR(16), t_warehouse_code VARCHAR(64), t_status TINYINT,
  sync_time DATETIME, del_flag TINYINT DEFAULT 0,
  UNIQUE KEY uk_wid (lx_wid)
) COMMENT '领星仓库缓存';

-- 头程物流渠道（ChannelList）
CREATE TABLE t_lx_channel (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  lx_channel_id BIGINT NOT NULL,
  channel_name VARCHAR(128), billing_type TINYINT, volume_calc_param INT,
  provider_id BIGINT, provider_name VARCHAR(128), enabled TINYINT,
  sync_time DATETIME, del_flag TINYINT DEFAULT 0,
  UNIQUE KEY uk_channel (lx_channel_id)
) COMMENT '领星头程物流渠道缓存';

-- 其他费类型（GetHeadLogisticsFeeTypes）
CREATE TABLE t_lx_fee_type (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  fee_type_id VARCHAR(64) NOT NULL, name VARCHAR(128), remark VARCHAR(512),
  sync_time DATETIME, del_flag TINYINT DEFAULT 0,
  UNIQUE KEY uk_fee_type (fee_type_id)
) COMMENT '领星其他费类型缓存';

-- 产品映射（ProductLists：sku→product_id）
CREATE TABLE t_lx_product_map (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  sku VARCHAR(128) NOT NULL, lx_product_id BIGINT NOT NULL,
  sync_time DATETIME, del_flag TINYINT DEFAULT 0,
  UNIQUE KEY uk_sku (sku)
) COMMENT '领星产品ID映射缓存';
```

> 仓库映射关系复用现有 `t_warehouse_mapping`（whId + serviceProviderCode=LINGXING_ERP + extWhCode=领星wid + authConfigId），不新建表；如需页面化维护，在现有第三方仓映射页扩展。

### 3.6 现有表变更

| 表 | 变更 | 说明 |
| --- | --- | --- |
| `t_wms_allocate` | + `allocate_category` VARCHAR(16)（COLLECT集货/MOVE移库/OVERSEA海外）；+ `lx_doc_type` VARCHAR(32) | 调拨计划分类与下游领星单据类型 |
| `t_wms_shipment` | + `lx_doc_type` VARCHAR(32)；+ `lx_sync_status` TINYINT（0无需/1待同步/2同步中/3成功/4失败） | 驱动发货分支与列表展示 |
| `t_wms_receipt_new` | + `cancel_qty` INT DEFAULT 0（主单取消数汇总）；+ `lx_ib_no` VARCHAR(64)（领星入库单号） | 结束到货与领星单据列 |
| `t_wms_receipt_item` | + `cancel_qty` INT DEFAULT 0 | 明细取消数 |
| `t_wms_inbound` | + `lx_ib_no` VARCHAR(64) | 入库单列表【领星单据】列 |
| `t_auth_config` | 初始化一条 LINGXING_ERP 凭据（appId/appSecret/token/refreshToken） | 复用现有授权配置表 |

## 4. 领星接口使用清单

| 场景 | 接口（本地文档路径） | API Path | 关键参数 | 注意事项 |
| --- | --- | --- | --- | --- |
| 令牌 | Authorization/GetToken | /api/auth-server/oauth/access-token | appId/appSecret（form-data） | expires_in≈7199s，提前刷新 |
| 令牌 | Authorization/RefreshToken | /api/auth-server/oauth/refresh | appId/refreshToken | **每个 refresh_token 只能用一次**，刷新后必须落库新值 |
| 店铺 | BasicData/SellerLists | /erp/sc/data/seller/lists | 无（GET） | 全量返回；唯一键 sid |
| 仓库 | Warehouse/WarehouseLists | /erp/sc/data/local_inventory/warehouse | type=1/3/4 | 分页拉全量 |
| 渠道 | Logistics/ChannelList | /erp/sc/data/local_inventory/channelList | offset/length | 取 provider=默认物流商 的渠道 |
| 其他费 | FBA/GetHeadLogisticsFeeTypes | /erp/sc/routing/fba/shipment/getHeadLogisticsFeeTypes | 无 | 缓存 |
| 产品 | Product/ProductLists | /erp/sc/routing/data/local_inventory/productList | sku_list（≤1000） | 返回 data>>id=product_id |
| 建调拨单 | Warehouse/AddAllocationOrder | /erp/sc/routing/inventoryReceipt/StorageAllocation/addAllocationOrder | type=2、sys_wid/sys_to_wid、freight_fee/other_fee/fee_part_type、product_list(sku/good_num)、remark | 限流1；成功返回 order_sn（TF*）；**创建后费用不可改** |
| 调拨单列表 | Warehouse/getStorageAllocationList | .../getStorageAllocationList | wid/to_wid/日期/分页 | 返回 item_list(含product_id)、inbound_order_sn(IB单)、status(19待收货/20已完成/30已删除) |
| 分批收货 | Warehouse/partlyReceiveAllocationOrder | .../partlyReceiveAllocationOrder | order_sn、product_list(**product_id**、received_good_num) | 必须用领星 product_id，非 SKU |
| 全部收货 | Warehouse/receiveAllocationOrder | .../receiveAllocationOrder | orderSnMany（逗号分隔） | 待收货→已完成，入库仓加库存 |
| 结束到货 | Warehouse/finishReceiveAllocationOrder | .../finishReceiveAllocationOrder | order_sn | 关闭未收部分 |
| 撤销调拨单 | Warehouse/CancelStorageAllocationList | /basicOpen/storageAllocationList/cancel | order_sn | **仅待审批/待配货/待调拨可用；待收货不可用**（本方案不用，采用收货+反向法） |
| 删除调拨单 | Warehouse/DeleteStorageAllocationList | /basicOpen/storageAllocationList/delete | orderSn[] | 同上，仅待审批/待配货/待调拨 |
| 建FBA发货单 | FBA/CreateSendedOrder | /erp/sc/storage/shipment/createSendedOrder | sys_wid、list(seller_id/marketplace_id/shipment_id/fulfillment_network_sku/sku/num)、head_fee_type、remark、request_flag、head_logistics_list | **异步接口**：扣发货仓库存并创建【已发货】单；需 request_flag 查结果 |
| 查创建结果 | FBA/searchProcessResult | /erp/sc/routing/storage/shipment/searchProcessResult | request_flag | process_status：0处理中/1成功/2失败；成功返回 order_sn（SP*） |
| 更新物流费用 | FBA/updateListLogistics | /erp/sc/routing/storage/shipment/updateListLogistics | data[](order_sn、logistics_list_type=1、head_logistics_list) | **覆盖式更新**：tracking_list/预估/实际费用不传也会置空；限流2 |
| 作废发货单 | FBA/InvalidShipmentSn | /basicOpen/openapi/fbaShipment/shipmentSn/invalid | shipmentNos[]、isReturnStock=1、isReturnStockAux=0、cancelReason | 作废并恢复产品库存 |
| （备用）货件状态 | FBA/UpdateShipmentActualStatus | .../updateShipmentActualStatus | is_closed、list(sid/shipment_id) | PRD 明确"结束到货领星侧无需处理"，本期不调 |

> 限流：上述业务接口令牌桶容量多为 1（updateListLogistics 为 2）。客户端按 2.2 的 RateLimiter 串行化；批量场景（如收货重试队列）按每请求 ≥1s 间隔执行。

## 5. 业务流程详细设计

### 5.1 调拨计划分类判定

在调拨计划创建/保存时按始发仓、目的仓类型计算（仓库类型取 `t_wms_warehouse.wh_type`：1自营、2第三方、3供应商、4FBA）：

```text
始发仓 ∈ {自营(1), 供应商(3)} 且 目的仓 ∈ {自营(1), 供应商(3)}
    → 集货调拨 COLLECT    → lx_doc_type = ALLOCATION（调拨单，费用通常0）
始发仓 ∈ {自营(1), 供应商(3)} 且 目的仓 = FBA(4)
    → 移库调拨 MOVE       → lx_doc_type = FBA_SHIPMENT（发货单）
始发仓 ∈ {自营(1), 供应商(3)} 且 目的仓 = 第三方(2，非FBA)
    → 移库调拨 MOVE       → lx_doc_type = STOCKUP（备货单，本期不同步，仅打标，待确认 A-1）
始发仓 ∈ {第三方(2)} 且 目的仓 = FBA(4)
    → 海外调拨 OVERSEA    → lx_doc_type = FBA_SHIPMENT（发货单，产生费用）
始发仓 ∈ {第三方(2)} 且 目的仓 = 第三方(2，非FBA)
    → 海外调拨 OVERSEA    → lx_doc_type = ALLOCATION（调拨单，产生费用）
```

- `is_fba` 沿用现有字段（目的仓是否FBA），与分类互为校验。
- 分类结果冗余到下游发货单（`t_wms_shipment.lx_doc_type`），发货时直接据此分支，不再重算。
- 平台仓（非FBA，如 Wayfair/Lazada 平台仓）在中台 `wh_type` 的归类口径待确认（A-2）。

### 5.2 发货同步-领星调拨单（ALLOCATION）

**前置校验（发货点击时）**
1. 发货单状态=待发货、装箱信息齐全（现有逻辑）。
2. 费用已保存（强制，因领星调拨单创建后费用不可改）；若 `actual_total_fee=0` 前端二次确认："当前发货单物流费用为0，领星调拨单创建后不支持修改费用，请确认费用是否正确？"。
3. 始发仓/目的仓已完成领星仓库映射，否则拦截提示去维护映射。

**时序（两阶段，避免领星孤儿单）**

```text
[阶段1 领星创建（事务外）]
1. t_wms_lx_shipment_doc 预写一条 sync_status=1（同步中）
2. 调 AddAllocationOrder：
   - type=2（完整调拨→待收货）
   - sys_wid / sys_to_wid = 映射的领星仓库ID
   - freight_fee = actual_logistics_fee，other_fee = actual_other_fee，fee_part_type=0（不分摊，中台已分摊到发货单粒度，待确认 B-3）
   - remark = 排柜计划号 + 物流计划号 + SO/物流订单号（无则留空拼接）
   - product_list[]：sku、good_num=shipmentQty（本期不传次品，待确认 B-4）
3. 成功：得到 order_sn；失败：预写记录置失败+错误信息 → 抛业务异常，发货终止（中台未出库，可整改后重发）

[阶段2 中台出库事务（现有发货逻辑）]
4. 同一事务内：始发仓出库 → 发货单置已发货 → 回写关联记录(sync_status=2, lx_order_sn) → 重算物流计划
5. 事务成功后异步：调 getStorageAllocationList 缓存该单 item_list 的 product_id（供分批收货）

[异常分支]
- 阶段2事务失败（概率极低）：领星调拨单已成【待收货】，无法撤销 → 记录日志+告警，
  人工处理口径=退运流程（先收货再反向调拨），见 5.6.2
```

### 5.3 发货同步-领星FBA发货单（FBA_SHIPMENT）

**前置校验**
1. 状态=待发货、装箱齐全（现有）。费用**不校验**（可后补）。
2. 始发仓已映射领星仓库；店铺映射齐全：`allocate.shopId` → `t_amazon_seller_channel_config` 取得 seller_id、marketplace_id；`allocate.shipment_id`（FBA货件号）非空；明细 `fnsku` 非空。任一缺失即拦截。

**组装 CreateSendedOrder**

| 领星参数 | 中台来源 |
| --- | --- |
| sys_wid | 始发仓映射的领星仓库ID |
| actual_shipment_time | 发货当日 |
| expected_arrival_date | allocate.planArrivalDate |
| remark | 排柜计划号+物流计划号+SO/物流订单号 |
| head_fee_type | 与费用分摊方式联动：体积重→2；计费重→0 且必传 logistics_channel_id（默认物流商渠道，见 4）；未录费用时默认传 2（待确认 C-6） |
| request_flag | `{shipmentNo}_{yyyyMMddHHmmss}_{rand}`，保证唯一（超时续查依赖） |
| list[].seller_id | t_amazon_seller_channel_config.sellerId |
| list[].marketplace_id | t_amazon_seller_channel_config.marketPlaceId |
| list[].shipment_id | allocate.shipmentId（=发货单 inboundNo） |
| list[].fulfillment_network_sku | shipment_item.fnsku |
| list[].sku / num | shipment_item.sku / shipmentQty |
| head_logistics_list | 若发货前已录费用：带预估+实际费用（见 5.4.5）；否则不传 |

**结果轮询与状态机**

```text
发货点击 → 预写关联记录(同步中, request_flag) → CreateSendedOrder
   ├─ 同步调用失败(网络/参数错) → 发货终止，可重试
   └─ 已受理 → 轮询 searchProcessResult（间隔2s，上限60s）
        ├─ process_status=1 → 中台出库事务（同 5.2 阶段2）→ 完成
        ├─ process_status=2 → 发货终止，展示 error_details
        └─ 超时未出结果 → 发货单置【同步中】锁定态：
              · shipment_status 保持待发货，前端禁用发货/截单/编辑按钮
              · 后台 job 每 30s 续查（最长 24h）
              · 查到成功 → 自动执行中台出库事务并解锁
              · 查到失败 → 解锁，允许重新发货
              · 24h 仍处理中 → 告警人工介入（核对领星后台该 request_flag 单据）
```

> 该设计目的：CreateSendedOrder 一旦受理就会扣领星库存，绝不能出现"领星已扣、中台未出库"或"重复创建"。request_flag 保证超时重查幂等；锁定态禁止二次发货防止重复建单。

### 5.4 物流费用模块

**5.4.1 重量计算（分摊基础）**

数据源：`t_pms_product_spec`（包装长/宽/高、净重/毛重、单品体积重）与 `t_pms_package_info`（默认箱规：包装尺寸、单箱数量）。计算口径（与 PRD 一致）：

```text
单品体积重(KG) = 包装长(cm) × 包装宽(cm) × 包装高(cm) / 6000 / 单箱数量
单品计费重(KG) = MAX(单品包装重量, 单品体积重)
发货单总实重   = Σ(单品包装重量 × 发货数量)
发货单总体积重 = Σ(单品体积重 × 发货数量)
发货单总计费重 = Σ(单品计费重 × 发货数量)
```

> 待确认：单品包装重量取净重还是毛重（B-5）；无箱规/无尺寸的产品按 0 计还是拦截（B-6）；尺寸单位以哪个表为准（spec 标注 mm、package_info 标注 cm，需统一口径）。
> 无物流计划的发货单，只计算发货单自身（PRD 已说明）。

**5.4.2 分摊算法**

```text
单发货单分摊费用 = 物流计划总费用 × ( 本发货单重量 / 物流计划下全部发货单重量合计 )
分摊方式：调拨单类仅【体积重】；FBA发货单类支持【体积重/计费重】，默认体积重
```

**5.4.3 费用弹框（前端交互）**

- 弹框上：发货单主单信息；中：物流计划单+SO/物流订单号（无则留空）+总实重/体积重/计费重；分摊方式选择；四项费用编辑（物流费预估/其他费预估/物流费实际/其他费实际，调拨单类仅两项实际）；自动算合计；下：物流计划下全部发货单及各重量，录入总费用时实时预览分摊结果。
- 【确定】：覆盖写入 `t_wms_lx_fee`。
- 按钮显隐：调拨单类仅【待发货】；FBA发货单类【待发货+已发货】。

**5.4.4 费用同步领星**

| 单据类型 | 同步时机与方式 |
| --- | --- |
| 调拨单类 | 费用只在 `AddAllocationOrder` 创建时随单提交（freight_fee/other_fee），创建后**不可再改**；因此发货后弹框隐藏，中台侧仅留记录 |
| FBA发货单类（待发货时录入） | 随 `CreateSendedOrder` 的 head_logistics_list 提交 |
| FBA发货单类（已发货后录入/修改） | 调 `updateListLogistics`（覆盖式），失败进重试队列；每次【确定】都重推（覆盖上次费用，与 PRD 一致） |

**5.4.5 updateListLogistics 参数组装**

```text
data[0].order_sn = lx_order_sn
data[0].logistics_list_type = 1（新版）
data[0].head_logistics_list:
  tracking_list = []（SO号是否作为跟踪号同步，待确认 C-8）
  estimate_expenses_list:
    logistics_fee = 物流费用(预估)，logistics_fee_currency = CNY（待确认 C-7）
    price = 单价（口径待确认 C-7，建议 预估物流费/计费重，无法计算时传0）
    other_fee_arr = [{ fee_type_id=默认"其他费用"类型, other_amount=其他费用(预估), other_currency=CNY }]
  actual_expenses_list:
    logistics_fee = 物流费用(实际)，weight = 总实重，volume = 总体积(m³)，
    tax_fee = 0（本期无关税），price/other_fee_arr 同上（实际口径）
```

> 注意覆盖式语义：每次必须把预估与实际两组都完整传，否则未传部分被置空。

**5.4.6 页面展示**
- 列表新增【预估物流费】【实际物流费】列，取 `t_wms_lx_fee` 合计值。
- 查看弹框新增【物流费用】页签，展示费用明细+分摊方式+同步状态。

### 5.5 收货同步

**5.5.1 分批收货（手动签收）**

```text
中台：ReceiptNewController.receive 事务成功（中台为主）
  → 发货单存在有效领星调拨单（doc_type=ALLOCATION, sync_status=2, 未作废）？
      是 → 调 partlyReceiveAllocationOrder：
           order_sn = lx_order_sn
           product_list[]：product_id（t_lx_product_map / 建单时缓存）、
                           received_good_num = 本次该SKU签收数
      失败 → 写重试队列（不回滚中台收货）+ 告警
      否（FBA发货单类）→ 不调领星（FBA自动收货）
```

**5.5.2 全部收货**：`receiveAll` 后调 `receiveAllocationOrder(orderSnMany=lx_order_sn)`；成功后回查 `getStorageAllocationList` 取 `inbound_order_sn` 写入 `t_wms_receipt_new.lx_ib_no` 与对应 `t_wms_inbound.lx_ib_no`（按收货单→入库单链路）。

**5.5.3 结束到货（新增功能）**

```text
入口：收货单列表【结束到货】按钮（状态=待收货/部分收货 可见）
中台处理（事务）：
  1. receive_status → 已收货(10)
  2. 明细 cancel_qty = quantity - receivedQty（未收货部分），原"未收货"清零
  3. 已收货数、取消数回写发货单（shipment_item.receivedQty 不变，记录取消口径待确认 A-6）
  4. 发货单不再需要继续下推收货单（该收货链路关闭）
领星处理：
  调拨单类 → finishReceiveAllocationOrder(order_sn)；失败进重试队列
  FBA发货单类 → 不处理（PRD：领星上无需处理）
```

**5.5.4 收货单列表调整**：【未收货】文案改【待收货】；状态筛选改多选；列表与明细弹框增加【取消数】列。

### 5.6 退运同步

触发点：在途退运单创建成功（`ReturnOnroadServiceImpl.createReturnOnroad` 事务提交后异步执行领星撤销；退运单记录关联的发货单→领星单据）。

**5.6.1 FBA发货单类**

```text
调 InvalidShipmentSn：
  shipmentNos = [lx_order_sn]
  isReturnStock = 1（恢复产品库存）、isReturnStockAux = 0（辅料不恢复）
  cancelReason = "中台退运：" + 退运单号
成功 → 关联记录 sync_status=4（已作废）
失败 → 重试队列 + 告警
作废后：货件可再次下推新发货单 → 发货时创建新的领星单（新关联记录，旧记录保留已作废）
```

**5.6.2 调拨单类（待收货无法直接撤销 → 两步法）**

```text
步骤1（断点 RECEIVED）：receiveAllocationOrder(order_sn) 将原调拨单全部收货
步骤2（断点 REVERSED）：AddAllocationOrder type=1（简易调拨，创建即完成）：
   sys_wid = 原目的仓（领星wid）、sys_to_wid = 原始发仓（领星wid）
   product_list = 原调拨单 SKU 与数量（退运数量口径待确认 B-7：按原单数量还是退运数量）
   freight_fee/other_fee = 0，remark = "退运归还：" + 退运单号 + "，原单：" + lx_order_sn
两步写入同一重试任务，支持断点续做；全部完成后关联记录置已作废
```

> 该两步法与人工 SOP 一致（PRD 20260819 新增章节）。若领星后续开放待收货调拨单撤销接口，可简化为单接口。

### 5.7 发货单截单（与领星无交互）

截单（cancel）仅发生在【待发货】状态，此时领星执行单据尚未创建（最晚时机同步原则），故**截单无需调用领星**。现有 `resetAllocateToWaitPush`（截单后调拨计划回退待推单）逻辑已实现，PRD 该项需求确认是否已满足（C-9）。

## 6. 前端改造清单（xzy-erp-ui）

| 页面/组件 | 改造内容 |
| --- | --- |
| `views/wms/shipment/index.vue`（调拨发货单列表） | ①【目的仓】右侧新增【领星单据】列：展示最新有效领星单号（已作废单灰色标注），hover 弹框展示：单据类型、领星单号、同步状态、领星单据状态、预估/实际物流费、失败原因；②【领星单据】右侧新增【预估物流费】【实际物流费】列；③操作列新增【物流费用】按钮（按 5.4.3 规则显隐）；④【同步中】锁定态禁用发货/截单/编辑按钮 |
| `views/wms/shipment/dialog.vue`（查看弹框） | 新增【物流费用】页签 |
| 新增 `views/wms/shipment/feeDialog.vue` | 物流费用弹框（两种形态：调拨单类/发货单类） |
| `views/wms/receiptNew/**`（收货单） | 状态文案【未收货】→【待收货】；状态筛选改多选；列表+明细新增【取消数】列；新增【结束到货】按钮+确认弹框 |
| `views/wms/inbound/**`（入库单列表） | 【备注】右侧新增【领星单据】列（领星IB单号，可点击/悬浮展示） |
| `src/api/wms/shipment.ts` 等 | 新增费用查询/保存/分摊预览、结束到货、领星单据详情接口 |

## 7. 异常处理、重试与监控

### 7.1 异常分级与处理

| 级别 | 场景 | 处理 |
| --- | --- | --- |
| 阻断级 | 发货时领星建单失败、映射缺失、关键数据缺失（店铺/shipmentId/fnsku） | 发货操作直接失败并提示原因；中台不做任何状态变更 |
| 补偿级 | 收货/结束到货/退运/费用同步领星失败 | 中台操作正常完成；失败写入 `t_lx_retry_task`，job 按退避策略重试（1min/5min/30min/2h/6h...），超次数告警转人工 |
| 告警级 | CreateSendedOrder 轮询 24h 无结果、阶段2事务失败产生领星孤儿单、token 刷新失败、对账差异 | 即时告警（工作台消息；钉钉机器人待确认 C-10）+ 日志留痕 |

### 7.2 Token 异常

- access_token 失效（响应鉴权错误码）→ 立即用 refresh_token 刷新并重放一次原请求；
- refresh_token 只能使用一次：刷新成功后新 token 对原子写库（加锁防并发双刷新）；
- 刷新仍失败 → 重新走 GetToken（appId/appSecret），并告警提示检查凭据。

### 7.3 幂等设计

| 操作 | 幂等保障 |
| --- | --- |
| CreateSendedOrder | request_flag 唯一；超时续查只查不重发；锁定态禁止二次发货 |
| AddAllocationOrder | 无原生幂等参数 → 预写关联记录(同步中)+发货分布式锁(现有 ALLOCATE_SHIPMENT_LOCK_KEY)保证同一发货单串行；调用前先查该发货单是否已有成功记录 |
| 收货/结束到货/退运/费用 | 重试任务按 (biz_type,biz_id) 唯一；执行前校验领星侧单据当前状态（查 getStorageAllocationList）防重复操作 |

### 7.4 定时任务（xxl-job）

| Job | 频率 | 职责 |
| --- | --- | --- |
| lxTokenRefreshJob | 每 100 分钟 | access_token 提前刷新（有效期约2小时） |
| lxResultPollJob | 每 30 秒 | 续查【同步中】发货单的 searchProcessResult |
| lxRetryJob | 每 1 分钟 | 扫描重试任务执行 |
| lxBaseDataSyncJob | 每日 02:00 | 刷新店铺/仓库/渠道/其他费类型缓存 |
| lxReconcileJob | 每日 06:00 | 对账：应同步单据 vs 关联记录；领星调拨单状态与中台收货状态比对 |

## 8. 测试要点

1. **三类调拨全链路**：集货调拨（0费用）/ 海外调拨（含费用）/ FBA 各走一遍：发货→领星建单→（部分）收货→结束到货→退运→重建。
2. **费用**：体积重/计费重两种分摊；预估+实际费用；发货前录入随单提交；发货后修改触发 updateListLogistics 覆盖；费用=0 二次确认；领星侧核对费用值。
3. **异常与重试**：发货时领星报错（映射缺失/库存不足/参数错）→ 阻断且中台无脏状态；CreateSendedOrder 超时→锁定态→续查成功自动出库 / 续查失败解锁；收货后领星宕机→重试队列补偿成功；退运两步法中断→断点续做。
4. **幂等与并发**：同一发货单双击发货；重试期间人工再次触发；refresh_token 并发刷新。
5. **限流**：连续多单发货/收货，验证串行队列不触发领星限流报错。
6. **展示**：领星单据列/悬浮框/费用列/页签/收货单取消数/入库单IB单号 与数据一致。
7. **数据核对**：领星侧库存增减与中台出库/入库/退运逐笔核对（对账脚本）。

## 9. 上线方案

1. **配置先行**：凭据、仓库映射、店铺核对、默认物流商渠道、其他费类型在上线前配置完毕并走一次冒烟（手工触发一次建调拨单+作废）。
2. **灰度顺序**：先集货调拨（0费用、流程最简）→ 海外调拨（费用链路）→ FBA发货单（异步链路最复杂）；按调拨计划分类开关控制。
3. **存量策略**：上线前已存在的调拨计划/发货单不追溯同步；仅上线后新下推的发货单走新流程（以 `lx_doc_type` 是否有值区分）。
4. **回滚**：同步开关关闭后发货回退为纯中台流程（不建领星单）；已建领星单按人工 SOP 处理。
5. **观察期**：上线后一周每日人工核对对账 job 输出；确认无差异后转常态监控。

## 10. 附：关键代码落点（改造参考，不含代码实现）

| 模块 | 位置 | 说明 |
| --- | --- | --- |
| 发货分支 | `xzy-erp-public/xzy-wms/.../service/impl/ShipmentServiceImpl.java#shipping` | 按 `lx_doc_type` 增加领星同步分支 |
| 截单 | 同上 `#cancel` | 无需调领星（最晚时机同步） |
| 收货 | `.../controller/ReceiptNewController.java#receive/receiveAll` + `ReceiptNewServiceImpl` | 收货后异步触发领星调拨单收货 |
| 结束到货 | `ReceiptNewController` 新增 `finishReceive` | 中台状态+取消数+领星结束到货 |
| 退运 | `.../service/impl/ReturnOnroadServiceImpl.java#createReturnOnroad` | 事务提交后触发领星撤销 |
| 调拨计划 | `AllocateServiceImpl` 创建/保存处 | 分类打标 |
| 发货单生成 | `createShipmentByAllocate` | 继承分类、冗余字段 |
| 前端 | `xzy-erp-ui/src/views/wms/shipment/**、receiptNew/**、inbound/**` | 见第 6 节 |

## 相关知识

[[退货与退运流程]]、[[发货与出库流程]]、[[收货与入库流程]]、[[库存]]、[[系统架构]]

## 变更记录

- 2026-09-01 初稿（需求与方案确认阶段，未改代码）
