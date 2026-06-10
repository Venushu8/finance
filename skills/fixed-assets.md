---
name: fixed-assets
description: "固定资产智能体：负责资产全生命周期管理、折旧计提、资产处置及盘点，对接金蝶固定资产模块数据库"
user-invocable: true
allowed-tools: Bash(multica *), Bash(mysql *), Bash(python3 *)
depends-on: skills/kingdee-integration.md
---

# Fixed Assets (固定资产)

## 核心流程

1. **资产入账** — 资产购置、在建工程转固、融资租赁资产入账
2. **折旧计提** — 支持直线法、双倍余额递减法、年数总和法，自动按月计提
3. **资产变动** — 资产调拨、改良、减值测试、折旧方法变更
4. **资产处置** — 报废、出售、捐赠、投资转出等处置方式及损益计算
5. **资产盘点** — 生成盘点清单，录入实盘数据，输出盘盈亏报告

## 金蝶数据表映射

| 金蝶表名 | 用途 | 关键字段 |
|---------|------|---------|
| t_FACard | 固定资产卡片 | FAssetId, FCode, FName, FCategory, FOrgValue, FDate |
| t_FADeprEntry | 折旧记录 | FEntryId, FAssetId, FPeriod, FDeprAmt, FCumDepr |
| t_FAChange | 资产变动 | FChangeId, FAssetId, FChangeType, FChangeDate, FNewValue |
| t_FAInven | 盘点记录 | FInvenId, FAssetId, FInvenDate, FQtyAct, FQtyBook |
| t_FACategory | 资产类别 | FCatId, FCode, FName, FDeprMethod, FUsefulLife |

## 数据模型

```python
@dataclass
class FixedAsset:
    asset_code: str         # 资产编码
    name: str
    category: str           # 房屋、设备、车辆、电子设备等
    original_value: Decimal
    accumulated_depr: Decimal
    net_value: Decimal
    useful_life_months: int
    depr_method: str        # straight_line / double_declining / sum_of_years
    status: str             # in_use / idle / disposed / scrapped

@dataclass
class DepreciationResult:
    asset_id: str
    period: str
    depr_amount: Decimal
    book_value_before: Decimal
    book_value_after: Decimal
    cumulative_depr: Decimal

@dataclass
class InventoryResult:
    total_assets: int
    scanned: int
    matched: int
    surplus: list[str]      # 盘盈资产编码
    deficit: list[str]      # 盘亏资产编码
    variance_value: Decimal
```

## 错误处理

| 错误场景 | 处理方式 |
|---------|---------|
| 资产卡片重复 | 按资产编码去重，合并差异信息 |
| 折旧后净值低于残值 | 停止计提，标记资产已提足折旧 |
| 处置金额与账面差异大 | 生成处置损益凭证，标记异常审核 |
| 盘点差异超过阈值 | 自动发起复盘，通知资产管理岗 |
