---
name: financial-reporting
description: "财务报表智能体：负责三大报表编制、管理报告生成及预算分析，对接金蝶报表模块数据库"
user-invocable: true
allowed-tools: Bash(multica *), Bash(mysql *), Bash(python3 *)
depends-on: skills/kingdee-integration.md
---

# Financial Reporting (财务报表)

## 核心流程

1. **三大报表** — 自动生成资产负债表、利润表、现金流量表（直接法/间接法）
2. **管理报告** — 按部门/项目维度的收入成本分析，同比环比
3. **预算分析** — 实际 vs 预算对比分析，预警超预算科目
4. **报表发布** — 支持 PDF/Excel 导出，自动发送管理层审阅

## 金蝶数据表映射

| 金蝶表名 | 用途 | 关键字段 |
|---------|------|---------|
| t_RptBalance | 试算平衡表 | FPeriod, FAcctId, FBeginDebit, FBeginCredit, FEndDebit, FEndCredit |
| t_RptGL | 总分类账 | FPeriod, FAcctId, FSumDebit, FSumCredit, FEndBalance |
| t_Budget | 预算主表 | FBgtId, FPeriod, FDeptId, FAcctId, FBgtAmt |
| t_BudgetExec | 预算执行 | FBgtId, FPeriod, FActualAmt, FDiffAmt |
| t_RptTemplate | 报表模板 | FTplId, FTplName, FFormula |

## 数据模型

```python
@dataclass
class FinancialStatement:
    report_type: str       # balance_sheet / income / cashflow
    period: str            # YYYYMM
    company_id: str
    items: list[ReportItem]
    
@dataclass
class ReportItem:
    code: str              # 项目编码，如 1001
    name: str              # 项目名称，如 货币资金
    formula: str           # 取数公式
    current_period: Decimal
    previous_period: Decimal
    yoy_change: float      # 同比变动 %

@dataclass
class BudgetAnalysis:
    account_code: str
    budget_amount: Decimal
    actual_amount: Decimal
    variance: Decimal
    variance_pct: float
    alert_level: str       # normal / warning / critical
```

## 错误处理

| 错误场景 | 处理方式 |
|---------|---------|
| 模板公式解析失败 | 跳过该行，记录错误公式，报告模板维护人员 |
| 期间数据不完整 | 列出缺失科目，提示反结账补录 |
| 预算未录入 | 使用上期数据作为参考线，标记无预算 |
