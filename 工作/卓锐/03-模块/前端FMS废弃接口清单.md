---
type: inventory
status: confirmed
date: 2026-08-12
last_verified: 2026-08-12
domain: 前端-API / FMS
source: code-analysis
---

# 前端 FMS 废弃接口清单

## 目的与使用规则

本文记录 xzy-erp-ui 前端中已不被任何页面有效使用、或后端已无对应 Controller 的 FMS 相关接口封装（URL 以 `/fms/` 开头，位于 `src/api/modules/fms/**`）。用途是让后续 AI 读取项目、分析需求或编写代码时**直接跳过这些废弃代码**，不将其当作当前有效业务流程的依据。

> 规则：清单中的接口一律不作为业务流程依据；除非用户明确要求，不得恢复调用、不得引用其中的逻辑。

## 判定依据（2026-08-12 全量核对）

- 前端：导出函数在 `src/` 全量范围内无任何有效 import（同名函数按所属模块区分），或 import 后从未被调用。
- 后端：`xzy-erp-public/xzy-fms` 的 `*Controller.java` 注解映射（类级前缀 + 方法级路径拼接，`/fms` 由网关统一前缀）中已无对应路径。
- 页面触发：仅当 API 的宿主方法被有效 UI 触发（模板中未被注释、未被 `v-if="false"` 禁用的按钮/下拉项/表格列配置等）或生命周期钩子调用时，才视为在用。
- FMS 未发现"页面引用但 import 缺失"（C 类）与"有调用但无有效 UI 触发"（D 类）案例。
- 人工确认（2026-08-12）：`views/fms/payApply`、`views/fms/payBill`、`views/fms/prePayment` 三个页面目录未在生产使用；代码核查确认仅被这三个目录引用的 FMS 业务接口按废弃处理。

## A 类：前端无引用，后端 Controller 已不存在（5 个）

前端零引用、后端也无入口，按废弃处理，删除封装风险最低。

| 模块文件 | 函数 | 请求 | URL |
|---|---|---|---|
| fms/payApply | deleteApi | DELETE | /fms/payApply/delete/ |
| fms/payApply | exportApi | DOWNLOAD | /fms/payApply/export |
| fms/prePayment | deleteApi | DELETE | /fms/prePayment/delete/ |
| fms/zrPayApply | exportApi | DOWNLOAD | /fms/zrPayApply/export |
| fms/zrPayApply | getPayDataApi | POST | /fms/zrPayApply/getPayData/ |

## B 类：前端无引用，后端 Controller 仍存在（5 个）

后端仍有映射，是否被其他调用方使用**待人工确认**；前端侧按废弃处理。

| 模块文件 | 函数 | 请求 | URL | 备注 |
|---|---|---|---|---|
| fms/payApply | getListApi | GET | /fms/payApply/getList | 请款单列表页已改用 waitingPayment 模块 |
| fms/zrPay | addPayApi | POST | /fms/zrPay/add | zrPay 页面仅用 getPayListApi/cancelPayApi/savePayLogApi |
| fms/zrPay | editPayApi | PUT | /fms/zrPay/edit | 同上 |
| fms/zrPay | deletePayApi | DELETE | /fms/zrPay/cancel/{payId} | 封装疑似错误：URL 与 cancelPayApi 相同，后端删除实为 DELETE /zrPay/delete/{zrPayIds} |
| fms/zrPayApply | editPaApi | PUT | /fms/zrPayApply/edit | 函数名疑似拼写错误（editPaApi） |

## E 类：所属页面已确认未使用（人工确认 2026-08-12）（18 个）

用户确认 `views/fms/payApply`、`views/fms/payBill`、`views/fms/prePayment` 三个页面目录在生产中未使用。以下 FMS 业务接口经代码核查**仅被这三个目录引用**（无其他页面使用），因此按废弃处理；若页面恢复使用，需从本清单移除并更新 `last_verified`。

| 模块                 | 函数                  | 请求       | URL                                             | 引用页面（已确认未使用）                | 后端映射 |
| ------------------ | ------------------- | -------- | ----------------------------------------------- | --------------------------- | ---- |
| fms/payApply       | generatePayApplyApi | POST     | /fms/payApply/generatePayApply                  | payApply/waitingPayment.vue | 存在   |
| fms/payApply       | addApi              | POST     | /pms/purchase/add（异常封装，后端实际有 /fms/payApply/add） | payApply/waitingPayment.vue | 存在   |
| fms/payBill        | getListApi          | GET      | /fms/payBill/getList                            | payBill/index.vue           | 存在   |
| fms/payBill        | deleteApi           | DELETE   | /fms/payBill/delete/                            | payBill/index.vue           | 已不存在 |
| fms/payBill        | exportApi           | DOWNLOAD | /fms/payBill/export                             | payBill/index.vue           | 存在   |
| fms/payBill        | getDetailApi        | GET      | /fms/payBill/getDetail/                         | payBill/index.vue           | 已不存在 |
| fms/payBill        | payApi              | POST     | /fms/payBill/pay                                | payBill/index.vue           | 存在   |
| fms/payBill        | getBusinessNoApi    | GET      | /fms/payBill/getBusinessNo/                     | payBill/dialog.vue          | 已不存在 |
| fms/prePayment     | getListApi          | GET      | /fms/prePayment/getList                         | prePayment/index.vue        | 存在   |
| fms/prePayment     | addApi              | POST     | /fms/prePayment/add                             | prePayment/index.vue        | 存在   |
| fms/prePayment     | editApi             | PUT      | /fms/prePayment/edit                            | prePayment/index.vue        | 存在   |
| fms/prePayment     | submitApi           | POST     | /fms/prePayment/submit                          | prePayment/index.vue        | 存在   |
| fms/prePayment     | auditApi            | POST     | /fms/prePayment/audit                           | prePayment/index.vue        | 存在   |
| fms/prePayment     | cancelApi           | POST     | /fms/prePayment/cancel                          | prePayment/index.vue        | 存在   |
| fms/prePayment     | exportApi           | DOWNLOAD | /fms/prePayment/export                          | prePayment/index.vue        | 存在   |
| fms/prePayment     | recoverApi          | POST     | /fms/prePayment/recover                         | prePayment/index.vue        | 存在   |
| fms/waitingPayment | getListApi          | GET      | /fms/waitingPayment/getList                     | payApply/waitingPayment.vue | 存在   |
| fms/waitingPayment | exportApi           | DOWNLOAD | /fms/waitingPayment/export                      | payApply/waitingPayment.vue | 已不存在 |

## 其他发现（非废弃，但读取代码时需注意）

- `fms/payApply::addApi` 指向 `/pms/purchase/add`，疑似复制粘贴错误（后端实际存在 `/fms/payApply/add`）。该封装所在页面目录已确认未使用，当前不会触发；若未来恢复请款单页面，必须先修正此 URL。
- `fms/zrPay::deletePayApi` 即使被恢复使用也会因 URL/方法错误而失效（后端 DELETE 删除接口为 `/zrPay/delete/{zrPayIds}`）。
- FMS 前端模块中未发现被注释/`v-if="false"` 禁用的接口调用（与 PMS/WMS 不同）。

## 维护约定

- 接口被重新启用或删除时，同步更新本文并更新 `last_verified`。
- 新增接口不写入本文；只有确认"前端无有效引用且（或）后端无入口"时才登记。
- E 类依赖"页面未使用"的人工确认：若 payApply/payBill/prePayment 页面恢复使用，需同步将对应接口移出 E 类并更新 `last_verified`。
- 若确认 addApi 的错误 URL 是线上缺陷，应在本笔记"其他发现"中更新结论并关联 [[采购]]。

## 相关知识

[[财务]]
[[前端PMS废弃接口清单]]
[[前端WMS废弃接口清单]]
[[采购]]
[[采购生命周期与状态机]]

## 相关代码

- xzy-erp-ui/src/api/modules/fms/payApply/index.ts
- xzy-erp-ui/src/api/modules/fms/zrPay/index.ts
- xzy-erp-ui/src/api/modules/fms/zrPayApply/index.ts
- xzy-erp-ui/src/views/fms/payApply/waitingPayment.vue
- xzy-erp-public/xzy-fms/.../controller/*.java
