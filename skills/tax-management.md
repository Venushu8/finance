---
name: tax-management
description: "税务管理智能体：负责增值税、企业所得税、代扣代缴税的计算、申报及合规管理，对接金蝶税务模块数据库"
user-invocable: true
allowed-tools: Bash(multica *), Bash(mysql *), Bash(python3 *)
depends-on: skills/kingdee-integration.md
---

# Tax Management (税务管理)

## 核心流程

1. **增值税管理** — 进项/销项税额自动归集，增值税申报表生成
2. **企业所得税** — 季度预缴+年度汇算清缴，纳税调整项自动识别
3. **代扣代缴** — 个税、境外预提税自动计算与申报
4. **税务合规** — 税务日历管理，申报期限提醒，发票认证状态跟踪

## 金蝶数据表映射

| 金蝶表名 | 用途 | 关键字段 |
|---------|------|---------|
| t_TaxInvoice | 发票主表 | FInvId, FType, FCode, FDate, FAmount, FTaxAmt |
| t_TaxVATReturn | 增值税申报表 | FPeriod, FTaxType, FOutputTax, FInputTax, FNetPayable |
| t_TaxCITReturn | 企业所得税申报 | FPeriod, FTaxType, FProfit, FAdjAmt, FTaxPayable |
| t_TaxCalendar | 税务日历 | FEventId, FName, FDeadline, FStatus |
| t_TaxWithholding | 代扣代缴 | FWithholdId, FEmployee, FIncome, FTaxAmt, FDate |

## 数据模型

```python
@dataclass
class VATReturn:
    period: str                    # YYYYMM
    output_tax: Decimal            # 销项税额
    input_tax: Decimal             # 进项税额
    input_credit: Decimal          # 进项转出
    deductible: Decimal            # 可抵扣税额
    net_payable: Decimal           # 应纳增值税
    prepaid: Decimal               # 已预缴

@dataclass
class CITReturn:
    period: str                    # 税款所属期
    accounting_profit: Decimal     # 会计利润
    tax_adjustments: list[TaxAdjust]
    taxable_income: Decimal        # 应纳税所得额
    tax_rate: float                # 适用税率
    tax_payable: Decimal           # 应纳所得税额

@dataclass
class TaxAdjust:
    description: str
    amount: Decimal
    direction: str                 # increase / decrease
    regulation_ref: str            # 税法依据
```

## 错误处理

| 错误场景 | 处理方式 |
|---------|---------|
| 进项税与销项税期间不匹配 | 标记异常进项票，移入待认证池 |
| 申报数据与账务不一致 | 输出差异明细，提供调整建议 |
| 申报期限已过 | 自动发起补申报流程，标记滞纳金计算 |
| 发票认证失败 | 记录失败原因，通知税务专员处理 |
