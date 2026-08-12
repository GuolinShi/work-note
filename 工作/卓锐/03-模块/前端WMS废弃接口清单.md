---
type: inventory
status: confirmed
date: 2026-08-12
last_verified: 2026-08-12
domain: 前端-API / WMS
source: code-analysis
---

# 前端 WMS 废弃接口清单

## 目的与使用规则

本文记录 xzy-erp-ui 前端中已不被任何页面有效使用、或后端已无对应 Controller 的 WMS 相关接口封装（URL 以 `/wms/` 开头，含 `src/api/modules/wms/**` 与 `src/api/modules/scm/shipmentPlan`）。用途是让后续 AI 读取项目、分析需求或编写代码时**直接跳过这些废弃代码**，不将其当作当前有效业务流程的依据。

> 规则：清单中的接口一律不作为业务流程依据；除非用户明确要求，不得恢复调用、不得引用其中的逻辑。

## 判定依据（2026-08-12 全量核对）

- 前端：导出函数在 `src/` 全量范围内无任何有效 import（同名函数按所属模块区分），或 import 后从未被调用。
- 后端：`xzy-erp-public/xzy-wms` 的 `*Controller.java` 注解映射（类级前缀 + 方法级路径拼接，`/wms` 由网关统一前缀）中已无对应路径。
- 页面触发：仅当 API 的宿主方法被有效 UI 触发（模板中未被注释、未被 `v-if="false"` 禁用的按钮/下拉项/表格列配置等）或生命周期钩子调用时，才视为在用；方法存在但触发入口被注释或禁用，按废弃处理。

## A 类：前端无引用，后端 Controller 已不存在（13 个）

前端零引用、后端也无入口，按废弃处理，删除封装风险最低。

| 模块文件                  | 函数                                  | 请求       | URL                                                     |
| --------------------- | ----------------------------------- | -------- | ------------------------------------------------------- |
| wms/allocate          | cancelAllocaeApi                    | POST     | /wms/allocate/cancel                                    |
| wms/deliveryPlan      | deleteApi                           | POST     | /wms/deliveryPlan/remove/                               |
| wms/deliveryPlan      | submitApi                           | POST     | /wms/deliveryPlan/submit                                |
| wms/inbound           | deleteApi                           | DELETE   | /wms/inbound/delete/                                    |
| wms/inbound           | deleteItemApi                       | DELETE   | /wms/inboundItem/delete/                                |
| wms/outbound          | deleteApi                           | DELETE   | /wms/outbound/delete/                                   |
| wms/outbound          | deleteItemApi                       | DELETE   | /wms/outboundItem/delete/                               |
| wms/logisticsPlanBach | removeApi                           | POST     | /wms/logisticsPlanBach/remove/                          |
| wms/logisticsPlanBach | importFbaLogisticsPlanByPlatformApi | POST     | /wms/logisticsPlanBach/importFbaLogisticsPlanByPlatform |
| wms/receiptNew        | createInboundOrderApi               | POST     | /wms/receiptNew/createInboundOrder                      |
| wms/shipment          | deleteShipmentItemApi               | DELETE   | /wms/allocateItem/delete/                               |
| wms/shipment          | getFBABoxUrl                        | DOWNLOAD | /wms/shipment/downloadFbaBoxFiles                       |
| wms/shipment          | getFBALabel                         | DOWNLOAD | /wms/shipment/downloadFbaFnSkuLabel                     |

## B 类：前端无引用，后端 Controller 仍存在（10 个）

后端仍有映射，可能被其他调用方（外部系统、定时任务、其他服务）使用，是否可彻底删除**待人工确认**；前端侧按废弃处理。

| 模块文件 | 函数 | 请求 | URL |
|---|---|---|---|
| wms/deliveryPlan | editApi | POST | /wms/deliveryPlan/edit |
| wms/deliveryPlan | logicConfirmApi | POST | /wms/deliveryPlan/logicConfirm |
| wms/deliveryPlan | purchaseConfirmApi | POST | /wms/deliveryPlan/purchaseConfirm |
| wms/logisticsPlanBach | bachConverseAuditApi | POST | /wms/logisticsPlanBach/bachConverseAudit |
| wms/logisticsPlanBach | bachRemoveApi | POST | /wms/logisticsPlanBach/bachRemove |
| wms/shipment | addShipmentApi | POST | /wms/shipment/add |
| wms/shipment | deleteBoxApi | DELETE | /wms/shipment/deleteBox |
| wms/shipment | editShipmentApi | PUT | /wms/shipment/edit |
| wms/shipment | generateReceiptApi | POST | /wms/shipment/generateReceipt |
| wms/thirdProduct | getThirdSystemApi | GET | /wms/thirdProduct/getThirdSystemList |

## D 类：页面有 import 且有调用，但无有效 UI 触发（11 个）

脚本方法仍存在并调用接口，但触发入口被注释或整块禁用，页面实际操作无法到达；后端 Controller 均仍存在。按废弃处理。

| 模块 | 函数 | 请求 | URL | 无有效触发的原因（位置） |
|---|---|---|---|---|
| wms/allocate | auditAllocateApi | POST | /wms/allocate/audit | allocate/index.vue「审核」菜单项被 HTML 注释（约 119–123 行） |
| wms/allocate | exportAllocateApi | DOWNLOAD | /wms/allocate/export | allocate/index.vue「导出」按钮 `v-if="false"`（约 60–68 行） |
| wms/allocate | submitAllocateApi | POST | /wms/allocate/submit | allocate/index.vue「提交」菜单项被 HTML 注释（约 112–119 行） |
| wms/shipment | auditShipmentApi | POST | /wms/shipment/audit | shipment/index.vue「审核」按钮被注释（操作列） |
| wms/shipment | batchAuditApi | POST | /wms/shipment/batchAudit | shipment/index.vue 批量下拉「审核」项被注释（约 115–123 行） |
| wms/shipment | batchDeleteApi | POST | /wms/shipment/batchDelete | shipment/index.vue 批量下拉「删除」项被注释（约 158–166 行） |
| wms/shipment | batchDownloadCheckApi | POST | /wms/shipment/checkFileExists | shipment/index.vue 下载FBA箱唛/标签入口全部被注释（操作列与批量下拉） |
| wms/shipment | batchSubmitApi | POST | /wms/shipment/batchSubmit | shipment/index.vue 批量下拉「提交」项被注释（约 106–114 行） |
| wms/shipment | deleteShipmentApi | DELETE | /wms/shipment/delete/ | shipment/index.vue「删除」按钮被注释（操作列） |
| wms/shipment | downloadErrorApi | DOWNLOAD | /wms/shipment/downloadBatchError | 仅随批量审核流程使用，批量审核入口已被注释 |
| wms/shipment | submitShipmentApi | POST | /wms/shipment/submit | shipment/index.vue「提交」按钮被注释（操作列） |

## 其他易误导的模块归类

- `src/api/modules/scm/shipmentPlan`（27 个函数）全部调用 `/wms/platformPlan/**`，是 WMS 接口且均在使用；不要按 scm 目录名推断其归属。
- wms/amzInbound::getAmzInboundListApi 实际调用 `/amazon/inboundPlan/getList`，不在本次 `/wms/` 判定范围。
- wms/thirdWarehouseMap 五个函数调用 `/openapi/**`（warehouseMapping、authConfig），不在本次判定范围。
- WMS 目录未发现"页面引用但 import 缺失"的 C 类案例。

## 维护约定

- 接口被重新启用或删除时，同步更新本文并更新 `last_verified`。
- 新增接口不写入本文；只有确认"前端无有效引用且（或）后端无入口"时才登记。
- D 类判定依赖"有效 UI 触发"，若未来按钮/菜单项被恢复启用，应从 D 类移除并更新 `last_verified`。

## 相关知识

[[前端PMS废弃接口清单]]
[[仓储]]
[[采购]]
[[采购生命周期与状态机]]

## 相关代码

- xzy-erp-ui/src/api/modules/wms/allocate/index.ts
- xzy-erp-ui/src/api/modules/wms/shipment/index.ts
- xzy-erp-ui/src/api/modules/wms/deliveryPlan/index.ts
- xzy-erp-ui/src/views/wms/allocate/index.vue
- xzy-erp-ui/src/views/wms/shipment/index.vue
- xzy-erp-public/xzy-wms/.../controller/*.java
