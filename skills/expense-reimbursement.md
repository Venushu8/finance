---
name: expense-reimbursement
description: "费用报销智能体：处理员工费用报销申请、审批流程、OCR 收据识别及财务入账，对接金蝶费用模块数据库"
user-invocable: true
allowed-tools: Bash(multica *), Bash(mysql *), Bash(python3 *)
depends-on: skills/kingdee-integration.md
---

# Expense Reimbursement (费用报销)

## 核心流程

1. **报销申请** — 员工提交费用报销单，支持差旅、招待、办公等多种费用类型
2. **OCR 收据识别** — 自动识别收据中的金额、日期、商户、税号等信息
3. **审批流程** — 按金额和部门自动路由审批，支持多级审批
4. **财务入账** — 审批通过后自动生成记账凭证，安排付款

## 金蝶数据表映射

| 金蝶表名 | 用途 | 关键字段 |
|---------|------|---------|
| t_ERExpense | 报销单主表 | FExpId, FDate, FEmpId, FDeptId, FTotalAmt, FStatus |
| t_ERExpenseEntry | 报销单分录 | FEntryId, FExpTypeId, FAmount, FDesc |
| t_ERApproval | 审批记录 | FApprovalId, FExpId, FApprover, FAction, FComment |
| t_ExpType | 费用类型 | FTypeId, FCode, FName, FBudgetCtrl |
| t_Employee | 员工档案 | FEmpId, FCode, FName, FDeptId |

## 数据模型

```python
@dataclass
class ExpenseReport:
    report_id: str         # 报销单号
    employee_id: str
    department: str
    expense_type: str      # travel / entertainment / office / other
    total_amount: Decimal
    items: list[ExpenseItem]
    status: str            # draft / pending_approval / approved / rejected / paid

@dataclass
class ExpenseItem:
    expense_date: date
    category: str          # 费用类别
    amount: Decimal
    currency: str
    receipt_url: str
    description: str

@dataclass
class OCRReceipt:
    merchant_name: str
    transaction_date: date
    total_amount: Decimal
    tax_amount: Decimal | None
    tax_id: str | None
    items: list[dict]
    confidence: float      # 0-1
```

## 错误处理

| 错误场景 | 处理方式 |
|---------|---------|
| OCR 识别置信度 < 0.7 | 标记人工复核，不自动入账 |
| 超出预算 | 通知申请人和审批人，暂停审批 |
| 费用类型与部门不匹配 | 退回修改，提示可用费用类型 |
| 收据重复报销 | 按收据号/日期+金额去重检测，拒绝重复单 |
