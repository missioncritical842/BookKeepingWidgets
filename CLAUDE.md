# BookKeepingWidgets Project Instructions

## Repository Info
- **GitHub**: https://github.com/missioncritical842/BookKeepingWidgets
- **GitHub Pages**: https://missioncritical842.github.io/BookKeepingWidgets/
- **Current Version**: v1.28

## Grist Documents Using These Widgets

### Headwaters Book Keeping
- **Doc ID**: `hcmQR7zX3i5HqNwzCbi1jn`

### Christ the Anchor Book Keeping
- **Doc ID**: `kEfMiMXXMwg5YGYfhrAnvo`

## Widgets
| Widget | GitHub Pages URL |
|--------|------------------|
| Form1096.html | `https://missioncritical842.github.io/BookKeepingWidgets/Form1096.html?v1.28` |
| Form1099NEC.html | `https://missioncritical842.github.io/BookKeepingWidgets/Form1099NEC.html?v1.28` |
| PayrollFilingsHelper.html | `https://missioncritical842.github.io/BookKeepingWidgets/PayrollFilingsHelper.html?v1.28` |
| budgetvactuals.html | `https://missioncritical842.github.io/BookKeepingWidgets/budgetvactuals.html?v1.28` |
| donationenvelopes.html | `https://missioncritical842.github.io/BookKeepingWidgets/donationenvelopes.html?v1.28` |
| donationreceipts.html | `https://missioncritical842.github.io/BookKeepingWidgets/donationreceipts.html?v1.28` |
| envelopes.html | `https://missioncritical842.github.io/BookKeepingWidgets/envelopes.html?v1.28` |
| lastmonthpandl.html | `https://missioncritical842.github.io/BookKeepingWidgets/lastmonthpandl.html?v1.28` |
| lastyearcashflow.html | `https://missioncritical842.github.io/BookKeepingWidgets/lastyearcashflow.html?v1.28` |
| lastyearpandl.html | `https://missioncritical842.github.io/BookKeepingWidgets/lastyearpandl.html?v1.28` |
| pandlalltime.html | `https://missioncritical842.github.io/BookKeepingWidgets/pandlalltime.html?v1.28` |
| quarterlypandl.html | `https://missioncritical842.github.io/BookKeepingWidgets/quarterlypandl.html?v1.28` |

## Commit Requirements
- **Always increment version number** in commit message (e.g., "Description of changes (v1.17)")
- After committing, provide updated GitHub Pages URLs with new version as cache-busting parameter: `?v1.X`
- Push to origin after commit

## Grist MCP Integration - CRITICAL
When working with Grist data:
1. **Always use Grist MCP tools** (`list_tables`, `list_columns`, `list_records`) to verify actual table and column names
2. **Never create fallback or wildcard lookups** - reference exact column/table names from the schema
3. **Never guess column names** - query the schema first using `mcp__grist__list_columns`
4. Both Grist docs have similar schemas but may have slight differences (e.g., AccountsBackup only exists in Christ the Anchor)
5. **Grist MCP pydantic errors**: `update_records`, `delete_records`, and `update_columns` return pydantic validation errors ("Input should be a valid dictionary") because the API returns null on success. **Operations succeed despite the error** — verify by re-reading records afterward.
6. **Record limit**: `list_records` defaults to 100. Christ the Anchor has 111+ accounts — use `limit=120` or higher when fetching Accounts.

### Common Column Lookup
Before referencing any column in widget code, run:
```
mcp__grist__list_columns(doc_id="<doc_id>", table_id="<table_name>")
```

---

## Christ the Anchor — Budget System (2026)

### Overview
The 2026 budget system consists of three components:
1. **Budget table** — Monthly budget amounts per account
2. **Strategic_Fund table** — Tracks monthly actuals and cumulative fund balance across years
3. **budgetvactuals.html widget** — Visual dashboard comparing budget to actuals

### Budget Table
- **Year** (Numeric): Budget year (e.g., 2026)
- **Account** (Ref:Accounts): Links to the chart of accounts
- **Jan–Dec** (Numeric, currency): Monthly budget amounts
- **Annual_Total** (Formula): `$Jan + $Feb + ... + $Dec`
- **Notes** (Text): Description

#### 2026 Budget Lines (all leaf-level accounts)
| Row | Account | Path | Annual | Notes |
|-----|---------|------|--------|-------|
| 1 | 23 | Direct Public Support:Individ, Business Contributions | $58,000 | Tithes and Offerings (INCOME) |
| 2 | 106 | Payroll Expenses:Joe Salary | $20,000 | Joe Salary |
| 3 | 107 | Payroll Expenses:Tim Salary | $20,000 | Tim Salary |
| 5 | 110 | Program Services:Out:Missions:Global Missions | $1,500 | Global Missions |
| 6 | 75 | Program Services:In:Events:Food and Drink | $1,250 | Food for special events |
| 7 | 109 | Program Services:Out:Missions:Local Missions | $3,600 | Local Missions |
| 8 | 105 | Program Services:In:Benevolence | $1,000 | Benevolence |
| 9 | 100 | Program Services:Up:Worship Expenses | $1,450 | Worship Supplies |
| 10 | 111 | Management and General:Facilities and Equipment:Liability Insurance | $400 | General liability insurance |
| 11 | 59 | Management and General:Travel and Meetings:Conference, Convention, Meeting | $520 | Conference, Convention, Meeting |
| 12 | 60 | Management and General:Travel and Meetings:Travel | $520 | Travel |
| 13 | 102 | Management and General:Travel and Meetings:Food | $520 | Food while traveling |
| 14 | 103 | Management and General:Travel and Meetings:Fuel | $520 | Fuel while traveling |
| 15 | 104 | Management and General:Travel and Meetings:Lodging | $520 | Lodging while traveling |

**Totals**: Income $58,000 | Expenses $51,800 | Net $6,200

**Budget rule**: Always budget at the lowest-level (leaf) account. The widget rolls values up to parent accounts automatically.

### Accounts Added for Budget
These accounts were created to support the 2026 budget:

| Row ID | ACCOUNT_ID | CODE | Name | Parent | Path |
|--------|-----------|------|------|--------|------|
| 106 | 106 | 579 | Joe Salary | 64 (Payroll Expenses) | Payroll Expenses:Joe Salary |
| 107 | 107 | 580 | Tim Salary | 64 (Payroll Expenses) | Payroll Expenses:Tim Salary |
| 109 | 109 | 581 | Local Missions | 85 (Missions) | Program Services:Out:Missions:Local Missions |
| 110 | 110 | 582 | Global Missions | 85 (Missions) | Program Services:Out:Missions:Global Missions |
| 111 | 111 | 583 | Liability Insurance | 46 (Facilities and Equipment) | Management and General:Facilities and Equipment:Liability Insurance |

### Strategic Fund Table
Tracks monthly income/expense actuals and maintains a running cumulative balance **across years**.

#### Columns
| Column | Type | Formula/Description |
|--------|------|---------------------|
| Year | Numeric | Budget year |
| Month | Numeric | 1–12 |
| Month_Name | Formula | `["Jan","Feb",...][int($Month) - 1]` |
| Budgeted_Income | Formula | Sums Budget rows for INCOME accounts for this year/month |
| Budgeted_Expenses | Formula | Sums Budget rows for EXPENSE accounts for this year/month |
| Planned_Contribution | Formula | `$Budgeted_Income - $Budgeted_Expenses` |
| Actual_Income | Formula | Sums Transaction_Categorization INCOME for this year/month |
| Actual_Expenses | Formula | `abs(sum(...))` of Transaction_Categorization EXPENSE for this year/month |
| Actual_Net | Formula | `$Actual_Income - $Actual_Expenses` |
| Cumulative_Balance | Formula | **Cross-year**: `sum(r.Actual_Net for r in Strategic_Fund.lookupRecords() if r.Year < $Year or (r.Year == $Year and r.Month <= $Month))` |

#### Rows
- **2025**: 12 rows (IDs 13–24) — No budget existed, but actual income/expenses are captured from Transaction_Categorization. 2025 total surplus: ~$28,991.
- **2026**: 12 rows (IDs 1–12) — Budget and actuals tracked monthly.
- **Future years**: Add 12 rows per year. Formulas auto-populate from Budget and Transaction_Categorization tables.

#### Key Behavior
- The Cumulative_Balance carries forward across years — the 2025 surplus flows into 2026
- Budgeted_Income/Expenses will be $0 for years without a Budget table entry (e.g., 2025)
- Actual_Income/Expenses always pull from real transaction data regardless of whether a budget exists

### Budget vs Actuals Widget (budgetvactuals.html)
Custom Grist widget that displays a budget-to-actuals comparison dashboard.

#### Data Sources
Fetches 6 tables via `grist.docApi.fetchTable()`:
- `Budget` — budget line items
- `Transaction_Categorization` — actual transactions
- `Accounts` — chart of accounts (hierarchy)
- `Account_Types` — INCOME/EXPENSE classification
- `Organization_Settings` — org name for header
- `Strategic_Fund` — cumulative fund data

#### Features
- **Hierarchical tree display**: Accounts shown with indented hierarchy; parent rows are bold subtotals, leaf rows are regular weight
- **Automatic rollup**: Leaf-level budgets and actuals roll up to parent account subtotals
- **Complete coverage**: Shows ALL accounts with budget or actual activity, even if $0 budgeted
- **Income section**: Hierarchical income accounts with Annual Budget, YTD Budget, YTD Actual, Variance, % Used
- **Expense section**: Same columns for expense accounts with hierarchy
- **Summary section**:
  - Net Income (Budget) — annual
  - Net Income (Actual YTD)
  - Strategic Fund Prior Year Carryover — accumulated surplus from all previous years
  - Strategic Fund Cumulative Balance — total balance including current year through current month
- **Account hierarchy rollup**: Actual transactions on child accounts roll up to ancestor budget accounts
- **Color coding**: Green for favorable variances, red for unfavorable
- **Auto-detects current year and month** for YTD calculations

#### How Account Rollup Works
Budget lines may reference parent accounts (e.g., "Travel and Meetings") while actual transactions hit child accounts (e.g., "Travel:Food", "Travel:Fuel"). The widget traverses the PARENT_ACCOUNT chain to roll up child actuals to the budgeted ancestor account.

---

## Christ the Anchor — Schema Reference

### Account Types
| ID | TYPE_NAME |
|----|-----------|
| 1 | INCOME |
| 2 | EXPENSE |
| 3 | BANK |
| 4 | AR |
| 5 | ASSET |
| 6 | LIABILITY |
| 7 | EQUITY |
| 10 | TRANSFER |

### Key Account Hierarchy (Expenses)
```
Payroll Expenses (64)
  ├── Joe Salary (106)
  └── Tim Salary (107)
Management and General (34)
  ├── Business Expenses (35)
  │   ├── Accounting Costs, Bank Fee, Business Registration Fees, etc.
  ├── Facilities and Equipment (46)
  │   ├── Property Insurance (50)
  │   ├── Liability Insurance (111)
  │   ├── Rent, Parking, Utilities (51)
  │   └── ...
  ├── Office Expenses (52)
  │   ├── Internet Service, Software, etc.
  ├── Travel and Meetings (58)
  │   ├── Conference/Convention (59), Travel (60), Food (102), Fuel (103), Lodging (104)
  └── Website Design and Hosting (61)
Program Services (65)
  ├── In (66) — Community
  │   ├── Benevolence (105)
  │   ├── Education (68)
  │   ├── Events (72) → Food and Drink (75), Insurance (76), etc.
  │   └── Planning Meetings (77)
  ├── Out (79) — Mission
  │   ├── Events (80)
  │   └── Missions (85) → Local Missions (109), Global Missions (110), Rent (86)
  ├── Resources and Training (87)
  └── Up (99) — Worship
      └── Worship Expenses (100)
```

### Key Account Hierarchy (Income)
```
Direct Public Support (20)
  ├── Corporate Contributions (21)
  ├── Gifts in Kind - Goods (22)
  └── Individ, Business Contributions (23) ← Main tithes/offerings
Education Services (24)
Investments (27)
Product Sales (29)
```

### Ministry Dimensions
- **Up**: Worship
- **In**: Community
- **Out**: Mission
