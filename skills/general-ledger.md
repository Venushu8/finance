---
name: general-ledger
description: "总账管理智能体：负责科目表维护、日记账凭证处理、自动结转及月末结账，对接金蝶总账模块数据库"
user-invocable: true
allowed-tools: Bash(multica *), Bash(mysql *), Bash(python3 *)
depends-on: skills/kingdee-integration.md
---

# General Ledger (总账管理)

## 核心流程

1. **科目表管理** — 多级科目体系维护，支持按部门/项目辅助核算
2. **凭证处理** — 自动生成记账凭证，支持批量审核、过账
3. **自动结转** — 月末自动结转损益、汇兑损益，支持自定义结转模板
4. **月末结账** — 结账前检查、凭证断号检测、余额平衡验证

## 金蝶数据表映射

| 金蝶表名 | 用途 | 关键字段 |
|---------|------|---------|
| t_Account | 科目表 | FAcctId, FCode, FName, FLevel, FParentId |
| t_Voucher | 凭证主表 | FVchId, FDate, FPeriod, FAttachments |
| t_VoucherEntry | 凭证分录 | FEntryId, FAcctId, FDebit, FCredit, FSummary |
| t_PeriodStatus | 期间状态 | FYear, FPeriod, FStatus |
| t_AuxAcct | 辅助核算 | FAuxId, FAuxType, FAuxCode, FAuxName |

## 数据模型

```python
@dataclass
class Voucher:
    voucher_id: str        # 凭证号
    date: date             # 日期
    period: str            # 会计期间 YYYYMM
    entries: list[VoucherEntry]
    total_debit: Decimal   # 借方总额
    total_credit: Decimal  # 贷方总额
    status: str            # draft / audited / posted

@dataclass
class VoucherEntry:
    account_code: str      # 科目编码
    debit: Decimal         # 借方金额
    credit: Decimal        # 贷方金额
    summary: str           # 摘要
    aux_accounts: dict     # 辅助核算 {type: code}

@dataclass
class ClosingCheck:
    check_name: str        # 检查项名称
    passed: bool
    detail: str            # 检查详情
```

## 错误处理

| 错误场景 | 处理方式 |
|---------|---------|
| 借贷不平衡 | 拒绝过账，返回差异金额，标记错误分录 |
| 科目不存在/已禁用 | 返回可用科目列表建议，拒绝过账 |
| 期间已结账 | 拒绝操作，提示反结账流程 |
| 凭证断号 | 生成断号报告，提供自动补号选项 |
