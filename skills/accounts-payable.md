---
name: accounts-payable
description: "应付管理智能体：负责供应商发票处理、付款排程、三单匹配及应付账款管理，对接金蝶应付模块数据库"
user-invocable: true
allowed-tools: Bash(multica *), Bash(mysql *), Bash(python3 *)
depends-on: skills/kingdee-integration.md
---

# Accounts Payable (应付管理)

## 核心流程

1. **供应商发票处理** — 接收并验证供应商发票，自动录入应付系统
2. **付款排程** — 按账期和付款条件自动生成付款计划，支持批量支付
3. **三单匹配** — 采购订单、入库单、供应商发票三方自动匹配
4. **应付账款管理** — 应付余额查询、账龄分析、预付款冲销

## 金蝶数据表映射

| 金蝶表名 | 用途 | 关键字段 |
|---------|------|---------|
| t_APPayable | 应付主表 | FBillNo, FDate, FSuppId, FAmount, FStatus |
| t_APPayableEntry | 应付分录 | FEntryId, FItemId, FQty, FPrice |
| t_APPayment | 付款单 | FPayNo, FDate, FSuppId, FPayAmt |
| t_POOrder | 采购订单 | FOrderNo, FDate, FSuppId, FTotalAmt |
| t_StkReceive | 入库单 | FRecNo, FDate, FOrderNo, FQty |
| t_Supplier | 供应商档案 | FSuppId, FName, FCode, FPayTerms |

## 数据模型

```python
@dataclass
class ThreeWayMatch:
    purchase_order: str      # 采购订单号
    receipt_no: str          # 入库单号
    invoice_no: str          # 发票号
    po_qty: Decimal          # 采购数量
    rec_qty: Decimal         # 入库数量
    inv_qty: Decimal         # 发票数量
    match_status: str        # matched / qty_mismatch / price_mismatch / not_found

@dataclass
class PaymentSchedule:
    supplier_id: str
    due_date: date
    amount: Decimal
    priority: int            # 1=紧急, 2=正常, 3=可延后
    status: str              # pending / scheduled / paid / overdue
```

## 错误处理

| 错误场景 | 处理方式 |
|---------|---------|
| 三单匹配数量不一致 | 标记差异，发送通知给采购+仓库，暂停付款 |
| 供应商信用不足 | 跳过自动付款，标记人工审核 |
| 发票重复提交 | 按发票号+供应商去重，记录警告日志 |
