---
type: inventory
status: confirmed
date: 2026-08-12
last_verified: 2026-08-12
domain: 前端-API / PMS
source: code-analysis
---

# 前端 PMS 废弃接口清单

## 目的与使用规则

本文记录 xzy-erp-ui 前端中已不被任何页面使用、或后端已无对应 Controller 的 PMS 相关接口封装。用途是让后续 AI 在读取项目、分析需求或编写代码时**直接跳过这些废弃代码**，不将其当作当前有效业务流程的依据。

> 规则：清单中的接口一律不作为业务流程依据；除非用户明确要求，不得恢复调用、不得引用其中的逻辑。

## 判定依据（2026-08-12 全量核对）

- 前端：`src/api/modules/pms/**`、`src/api/modules/scm/**` 导出的函数，在 `src/` 全量范围内无任何有效 import（同名函数按所属模块区分）；或 import 后从未被调用。
- 后端：`xzy-erp-public/xzy-pms` 的 `*Controller.java` 中已无对应 URL 映射。
- 页面代码调用但 import 缺失的函数视为遗留缺陷，需人工确认页面是否仍在使用。
- 页面触发：仅当 API 的宿主方法被有效 UI 触发（模板中未被注释、未被 `v-if="false"` 禁用的按钮/下拉项/表格列配置等）或生命周期钩子调用时，才视为在用；方法存在但触发入口被注释或禁用，按废弃处理。

## A 类：前端无引用，后端 Controller 已不存在（14 个）

这些接口前端零引用、后端也无入口，按废弃处理。

| 模块文件                         | 函数                | 请求       | URL                                  |
| ---------------------------- | ----------------- | -------- | ------------------------------------ |
| pms/product                  | deleteSkuBoxApi   | DELETE   | /pms/product/box/{skuIds}            |
| pms/product                  | cancelApi         | POST     | /pms/product/cancel                  |
| scm/fmsoutbound              | exportApi         | DOWNLOAD | /pms/fmsOutbound/export              |
| scm/pickup                   | exportApi         | DOWNLOAD | /pms/pickup/export                   |
| scm/purchase                 | cancelApi         | POST     | /pms/purchase/cancel                 |
| scm/purchase                 | closeApi          | POST     | /pms/purchase/close                  |
| scm/purchaseChange           | getDtlApi         | GET      | /pms/purchaseChange/get/{id}         |
| scm/purchaseReturn           | deleteItemApi     | DELETE   | /pms/purchaseReturnItem/delete/{id}  |
| scm/purchaseSettlementAdjust | exportApi         | DOWNLOAD | /pms/purchaseSettlementAdjust/export |
| scm/quality                  | exportApi         | DOWNLOAD | /pms/quality/export                  |
| scm/qualityFreeSpu           | exportApi         | DOWNLOAD | /pms/qualityFreeSpu/export           |
| scm/qualityPlan              | getItemListApi    | GET      | /pms/purchaseQcExtension/getItemList |
| scm/qualityPlan              | batchEditApi      | PUT      | /pms/purchaseQcExtension/batchUpdate |
| scm/supply                   | getSupplierDtlApi | GET      | /pms/supplierItem/getItem/{id}       |

## B 类：前端无引用，后端 Controller 仍存在（12 个）

后端仍有映射，可能被其他调用方（SRM、定时任务、外部系统）使用，是否可彻底删除**待人工确认**；前端侧按废弃处理。

| 模块文件 | 函数 | 请求 | URL |
|---|---|---|---|
| pms/product | getProductExListApi | GET | /pms/product/getProductExList |
| pms/product | getProductBoxBySpuIdExApi | GET | /pms/packageInfo/getDetailBySpuId/{spuId} |
| scm/pickup | downloadUnMaintainedApi | DOWNLOAD | /pms/pickup/downloadUnMaintained/{qcIds} |
| scm/pickup | importMaintainedDataApi | POST | /pms/pickup/importMaintainedData |
| scm/pickup | exportErrorMaintainedApi | DOWNLOAD | /pms/pickup/exportErrorMaintained |
| scm/pickup | batchAddQcApi | POST | /pms/pickup/batchAddQc |
| scm/purchase | getCancelPayApplyApi | POST | /pms/purchase/getCancelPayApply/{purchaseId} |
| scm/purchase | getItemCancelPayApplyApi | POST | /pms/purchase/getItemCancelPayApply/{itemId} |
| scm/quotation | getQuotationCurrencyCodeApi | GET | /pms/quotationItem/getCurrencyCodeList/ |
| scm/supply | getTimeNodeApi2 | GET | /pms/supplierSettlement/timeNodeList（与在用 getTimeNodeApi 重复） |
| scm/supply | getSupplierItemPrice | POST | /pms/supplierItemPrice/getSupplierItemPrice（疑被批量版替代） |
| scm/supply | getSupplierItemPriceApi | POST | /pms/supplierItemPrice/getSupplierItemPrice2（疑被批量版替代） |

## C 类：页面代码有调用但 import 缺失（4 个，遗留缺陷）

页面仍写着调用语句，但对应 import 被注释或缺失，属于未定义引用。**页面是否还在生产菜单使用需人工确认**；确认下线前，不把这两页当作有效入口依据。

| 函数 | 请求 | URL | 调用位置 |
|---|---|---|---|
| scm/purchase::getPayApplyApi | POST | /pms/purchase/getPayApply/{purchaseId} | views/scm/purchase/index.vue:764（import 在 396 行被注释） |
| scm/quality::downUnQcListApi | DOWNLOAD | /pms/quality/downUnQcList/{qcIds} | views/scm/quality/index.vue:429（无 import） |
| scm/quality::importDataApi | POST | /pms/quality/importData | views/scm/quality/index.vue:549（无 import） |
| scm/quality::exportErrorApi | DOWNLOAD | /pms/quality/exportError | views/scm/quality/index.vue:550（无 import） |

## D 类：页面有 import 且有调用，但无有效 UI 触发（10 个）

脚本中方法仍存在并调用接口，但触发入口被注释或整块禁用，页面实际操作无法到达，按废弃处理。

| 模块           | 函数                  | 请求       | URL                                 | 无有效触发的原因（位置）                                                                   |
| ------------ | ------------------- | -------- | ----------------------------------- | ------------------------------------------------------------------------------ |
| pms/spu      | cancelSpuCgApi      | POST     | /pms/spuChange/cancel               | spuChange/index.vue 作废按钮被 HTML 注释（58–67 行），command.type="4" 分支不可达              |
| scm/purchase | inboundApi          | PUT      | /pms/purchase/inbound               | purchase/index.vue「快速入库」菜单项被注释（292–300 行）；在用入口为「提货」→ inboundV2Api              |
| scm/purchase | ordersApi           | POST     | /pms/purchase/orders                | purchase/index.vue「下单」菜单项被注释（246–255 行）                                        |
| scm/quality  | deleteApi           | DELETE   | /pms/quality/delete/                | quality/index.vue 整组操作下拉 `v-if="false"`（114 行），删除分支（deletePickup，506–508 行）不可达 |
| scm/quality  | inboundApi          | PUT      | /pms/quality/inbound                | quality/index.vue「入库」菜单项被注释且整组下拉 `v-if="false"`（114、120–130 行）                 |
| scm/supply   | deleteItemApi       | POST     | /pms/supplierItem/delete            | supplierItem/index.vue「编辑/删除」按钮整体被 `<span v-if="false">` 禁用                    |
| scm/supply   | deleteSettleItemApi | DELETE   | /pms/supplierSettlementItem/delete/ | settlement/dialog.vue 删除按钮 `v-if="false"`（160 行）                               |
| scm/supply   | exportItemErrorApi  | DOWNLOAD | /pms/supplierItem/exportError       | supplierItem/index.vue「新增/导入」按钮整体被 `<span v-if="false">` 禁用                    |
| scm/supply   | importItemApi       | POST     | /pms/supplierItem/importData        | 同上（导入功能被禁用）                                                                    |
| scm/supply   | templeItemApi       | DOWNLOAD | /pms/supplierItem/importTemplate    | 同上（导入模板下载被禁用）                                                                  |

## 其他易误导的模块归类

- scm/shipmentPlan 整个模块（27 个函数）实际调用 /wms/platformPlan/**，是 WMS 接口且均在使用；不要按 scm 目录名推断其归属。
- pms/warehouseProductMap 调用 /openapi/wareProdMapping/**，其中仅 wareProdMappingGetList 在使用，其余 6 个（add/edit/export/importTemplate/importData/exportError）废弃。
- pms/product 的 weightChangeFormula、setBoxList 是单位换算辅助函数而非接口，未使用。

## 维护约定

- 接口被重新启用或删除时，同步更新本文并更新 `last_verified`。
- 新增接口不写入本文；只有确认"前端无引用且（或）后端无入口"时才登记。
- D 类判定依赖"有效 UI 触发"，若未来按钮/菜单项被恢复启用，应从 D 类移除并更新 `last_verified`。

## 相关知识

[[采购]]
[[采购生命周期与状态机]]
[[采购数量与状态约束]]
[[采购流程分叉待确认]]

## 相关代码

- xzy-erp-ui/src/api/modules/scm/purchase/index.ts
- xzy-erp-ui/src/api/modules/scm/quality/index.ts
- xzy-erp-ui/src/api/modules/pms/product/index.ts
- xzy-erp-public/xzy-pms/.../controller/*.java
