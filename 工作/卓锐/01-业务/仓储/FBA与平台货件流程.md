---
type: process
status: confirmed
last_verified: 2026-09-04
source: code
---

# FBA 与平台货件流程

> 代码依据：`PlatformPlanServiceImpl`、`AmzInboundBindServiceImpl`、`AmzInboundBindImportServiceImpl`、`AmazonReceiptExceptionServiceImpl`、`AmazonInventoryFlowServiceImpl`、`ShipmentServiceImpl`（FBA 相关方法）。
> 平台货件 = 亚马逊 FBA 入库计划（Inbound Plan），通过 SP-API 与亚马逊交互；WMS 侧把货件与箱、调拨发货单绑定。

## 一、平台货件状态机（PlatformPlanStatusEnum）

| code | 状态 | 说明 |
| --- | --- | --- |
| 0 | 待提交 | 新建 |
| 1 | 入库计划创建中 | 调 SP-API 创建 |
| 2 | 待确认打包信息 | 亚马逊返回后待确认 |
| 3 | 打包信息确认中 | 确认请求处理中 |
| 4 | 待确认放置地 | 分仓结果待确认 |
| 5 | 入库计划分仓中 | 分仓请求处理中 |
| 6 | 待确认运输方式 | |
| 7 | 运输方式确认中 | |
| 8 | 待审核 | 内部审核 |
| 9 | 待预报 | |
| 10 | 已预报 | 完成 |
| -1 | 取消 | |

## 二、主要流程

1. **新增（add）**：选择店铺、目的仓（FBA 仓）、SKU 与数量（可导入，`validImportSku` 校验 SKU 与仓库/店铺归属）；
2. **提交（submit）**：状态→入库计划创建中，调 SP-API 创建入库计划；亚马逊回调（`fbaCallback`）更新打包/分仓/运输各阶段数据与状态；
3. **确认打包（confirmPack）**/ **重新生成打包（reGeneratePack）**：确认或重算装箱方案；
4. **确认放置地（confirmPlacement）**/ **重新分仓（reGeneratePlacement）**：确认亚马逊分仓结果（多个目的地仓）；
5. **确认运输方式（confirmTransport）**；
6. **审核（audit）** → **完成/预报（finish）**：进入待预报→已预报；
7. **取消（delete）**：取消货件并记录原因。

每个确认动作都要求当前状态匹配（如"待确认打包信息"才能确认打包），状态不符即拒绝。

## 三、FBA 绑定（AmzInboundBind）

把平台货件的箱子与中台调拨发货单/调拨计划绑定：
- **addBind / addBindV2**：建立 货件→箱→发货明细 绑定（含导入）；校验箱号、数量与货件计划匹配；
- **removeBind**：解绑；
- **getDeliveryList / getPlanInfoByBachNo**：按批次号查货件与箱信息，供生成发货单使用；
- FBA 调拨发货单生成时（见 [[发货与出库流程]]）：按绑定取箱号，发货数量不能超过计划数量。

## 四、FBA 收货与差异

- **amazonReceiptJobHandler**：拉取亚马逊收货数据（FBA 收货报告），按箱明细比对：
  - 生成 `AmazonInventoryFlow`（FBA 库存流水）；
  - 差异（多收/少收/错收）记入 `AmazonReceiptException`（收货异常）待处理；
  - `replenishSkuJobHandler`：补充缺失的 SKU 映射；
- **downloadFbaBoxFiles / downloadFbaFnSkuLabel**：下载 FBA 箱唛/标签文件用于贴标。

## 五、边界汇总

- 平台货件状态机单向推进，且每个确认动作都校验前置状态；
- 与亚马逊的交互异步化：提交后进入"xx中"状态，由回调/轮询推进；
- FBA 调拨发货受货件计划数量约束（不能超发）；
- 收货差异不直接改账，先落异常表人工/任务处理。
