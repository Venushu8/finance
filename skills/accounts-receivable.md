---
name: accounts-receivable
description: "应收管理智能体：负责发票管理、收款核销、账龄分析及对账处理，对接金蝶应收模块数据库"
user-invocable: true
allowed-tools: Bash(multica *), Bash(mysql *), Bash(python3 *)
depends-on: skills/kingdee-integration.md
---

# Accounts Receivable (应收管理)

## 核心流程

1. **发票管理** — 根据销售订单自动生成应收发票，支持批量开票、红字冲销
2. **收款核销** — 自动匹配银行回单与应收发票，按先进先出法核销
3. **账龄分析** — 按 30/60/90/120+ 天分段输出应收账款账龄表
4. **对账处理** — 与客户定期对账，自动标记差异项

## 金蝶数据表映射

| 金蝶表名 | 用途 | 关键字段 |
|---------|------|---------|
| t_ARInvoice | 应收发票主表 | FBillNo, FDate, FCustId, FAmount, FStatus |
| t_ARInvoiceEntry | 应收发票分录 | FEntryId, FItemId, FQty, FPrice, FTax |
| t_ARRcpt | 收款单 | FRcptNo, FDate, FCustId, FAmount |
| t_ARRcptMatch | 核销记录 | FRcptId, FInvoiceId, FMatchAmt, FMatchDate |
| t_Customer | 客户档案 | FCustId, FName, FCode, FCreditLimit |

## 数据模型

```python
@dataclass
class ReceivableInvoice:
    bill_no: str          # 单据编号
    date: date            # 开票日期
    customer_id: str      # 客户ID
    amount: Decimal       # 应收金额
    tax: Decimal          # 税额
    status: str           # 0=未核销, 1=部分核销, 2=已核销

@dataclass
class AgingBucket:
    bucket: str           # 30天, 60天, 90天, 120天+
    total_amount: Decimal
    invoice_count: int
```

## 错误处理

| 错误场景 | 处理方式 |
|---------|---------|
| 发票重复创建 | 按 FBillNo + FDate 去重，日志记录跳过行 |
| 核销金额超出发票余额 | 拒绝核销，返回错误码 ERR_OVERMATCH |
| 客户不存在 | 自动创建客户档案草稿，标记人工审核 |
