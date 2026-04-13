# Database Architecture

Designed from analysis of 9 CSE quarterly financial reports spanning 6 sectors.

## Companies Analyzed

| # | Company | Sector | Report Type |
|---|---------|--------|-------------|
| 1 | Softlogic Life Insurance PLC | Insurance | Year ended 31 Dec 2025 |
| 2 | LB Finance PLC | Finance | 9 months ended 31 Dec 2025 |
| 3 | Nations Trust Bank PLC | Banking | 12 months ended 31 Dec 2025 |
| 4 | Colombo Dockyard PLC | Manufacturing/Engineering | Year ended 31 Dec 2025 |
| 5 | Dilmah Ceylon Tea Company PLC | Consumer Goods (FMCG) | 9 months ended 31 Dec 2025 |
| 6 | Union Chemicals Lanka PLC | Chemicals/Manufacturing | Year ended 25 Dec 2025 |
| 7 | John Keells Holdings PLC | Diversified Conglomerate | 9 months ended 31 Dec 2025 |
| 8 | Ceylon Hospitals PLC (Durdans) | Healthcare | 9 months ended 31 Dec 2025 |
| 9 | Tangerine Beach Hotels PLC | Hotels/Tourism | 9 months ended 31 Dec 2025 |

## Key Design Challenge

CSE companies fall into two fundamentally different reporting formats:

1. **General companies** (manufacturing, FMCG, hotels, healthcare, conglomerates) — use standard Revenue → Cost of Sales → Gross Profit → Operating Profit → PBT → PAT structure
2. **Financial institutions** (banks, finance, insurance) — use Interest Income → Interest Expense → Net Interest Income, or Premiums → Claims → Underwriting Profit structure

**Solution:** A hybrid approach with:
- Common fields shared across ALL companies (profit, tax, EPS, total assets, total equity, etc.)
- Sector-specific JSONB columns for industry-specific line items
- Separate structured tables for the 4 core financial statements

---

## Entity Relationship Diagram

```
┌─────────────┐       ┌──────────────────────────┐
│  companies   │──1:N──│  financial_reports        │
└─────────────┘       │  (metadata per filing)    │
                      └──────────┬───────────────┘
                                 │
                    ┌────────────┼────────────────┐
                    │            │                │
              ┌─────┴─────┐ ┌───┴────────┐ ┌─────┴──────┐
              │  income    │ │  balance    │ │  cash_flow  │
              │ _statement │ │  _sheet     │ │ _statement  │
              └───────────┘ └────────────┘ └────────────┘
```

---

## Table Definitions

### 1. `companies`

Core company registry.

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| id | UUID | PK | |
| ticker | VARCHAR(20) | UNIQUE, NOT NULL, indexed | CSE ticker symbol |
| name | VARCHAR(255) | NOT NULL | Full legal name (e.g., "Softlogic Life Insurance PLC") |
| sector | VARCHAR(100) | | e.g., "Insurance", "Banking", "Manufacturing" |
| industry | VARCHAR(100) | | More specific (e.g., "Life Insurance", "Hotels") |
| market | VARCHAR(50) | DEFAULT 'CSE' | Stock exchange |
| company_type | VARCHAR(20) | NOT NULL | `general`, `bank`, `finance`, `insurance` |
| financial_year_end | VARCHAR(10) | | e.g., "March 31", "December 31" |
| registration_no | VARCHAR(50) | | e.g., "PQ 118" |
| is_active | BOOLEAN | DEFAULT true | |
| created_at | TIMESTAMPTZ | DEFAULT now() | |
| updated_at | TIMESTAMPTZ | auto-update | |

**Why `company_type`?** Banks, finance companies, and insurance companies have fundamentally different income statement structures. This field tells the API which sector-specific fields to expect/return.

---

### 2. `financial_reports`

One row per filing. Acts as the parent record linking the three financial statements.

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| id | UUID | PK | |
| company_id | UUID | FK → companies.id, NOT NULL | |
| year | INTEGER | NOT NULL | Calendar year of period end (e.g., 2025) |
| quarter | SMALLINT | NOT NULL, CHECK 1-4 | Q1=1, Q2=2, Q3=3, Q4=4 |
| period_end_date | DATE | | Exact period end (e.g., 2025-12-31) |
| period_months | SMALLINT | | 3 (quarterly), 9 (YTD), 12 (annual) |
| currency | VARCHAR(3) | DEFAULT 'LKR' | |
| units | VARCHAR(20) | DEFAULT 'thousands' | `thousands`, `millions`, `ones` |
| is_audited | BOOLEAN | DEFAULT false | |
| is_consolidated | BOOLEAN | DEFAULT true | Group vs Company figures |
| report_date | DATE | | Date report was signed/published |
| source_url | TEXT | | Link to original PDF |
| raw_data | JSONB | | Full extracted data before normalization |
| created_at | TIMESTAMPTZ | DEFAULT now() | |
| updated_at | TIMESTAMPTZ | auto-update | |

**Unique constraint:** `(company_id, year, quarter, is_consolidated)` — allows storing both Group and Company figures.

---

### 3. `income_statements`

Stores profit & loss data. Common fields as columns, sector-specific data in JSONB.

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| report_id | UUID | FK → financial_reports.id, UNIQUE, NOT NULL |
| **Revenue fields (general companies)** | | |
| revenue | NUMERIC(20,2) | Total revenue / gross income |
| cost_of_sales | NUMERIC(20,2) | Cost of goods sold / cost of services |
| gross_profit | NUMERIC(20,2) | Revenue - COGS |
| **Revenue fields (financial institutions)** | | |
| interest_income | NUMERIC(20,2) | Banks/finance: interest earned |
| interest_expense | NUMERIC(20,2) | Banks/finance: interest paid |
| net_interest_income | NUMERIC(20,2) | Interest income - interest expense |
| fee_and_commission_income | NUMERIC(20,2) | Banks: fee income |
| net_trading_income | NUMERIC(20,2) | Banks: trading gains/losses |
| **Revenue fields (insurance)** | | |
| gross_written_premiums | NUMERIC(20,2) | Insurance: total premiums |
| net_written_premiums | NUMERIC(20,2) | After reinsurance ceded |
| net_claims_and_benefits | NUMERIC(20,2) | Insurance: claims paid |
| **Common operating fields** | | |
| other_income | NUMERIC(20,2) | Other operating income |
| total_operating_income | NUMERIC(20,2) | All revenue sources combined |
| distribution_expenses | NUMERIC(20,2) | Selling & distribution costs |
| administrative_expenses | NUMERIC(20,2) | Admin/overhead costs |
| other_operating_expenses | NUMERIC(20,2) | Other opex |
| total_operating_expenses | NUMERIC(20,2) | All operating expenses combined |
| operating_profit | NUMERIC(20,2) | Results from operating activities |
| impairment_charges | NUMERIC(20,2) | Banks/finance: credit loss provisions |
| **Finance & tax** | | |
| finance_income | NUMERIC(20,2) | Interest/investment income (non-banks) |
| finance_costs | NUMERIC(20,2) | Interest expense (non-banks) |
| net_finance_income | NUMERIC(20,2) | Finance income - finance costs |
| profit_before_tax | NUMERIC(20,2) | PBT |
| income_tax_expense | NUMERIC(20,2) | Tax charge |
| profit_after_tax | NUMERIC(20,2) | PAT / Net profit |
| **Attribution** | | |
| profit_attributable_to_owners | NUMERIC(20,2) | Owners of the parent |
| profit_attributable_to_nci | NUMERIC(20,2) | Non-controlling interest |
| **Per share** | | |
| eps_basic | NUMERIC(12,4) | Basic EPS (Rs.) |
| eps_diluted | NUMERIC(12,4) | Diluted EPS (Rs.) |
| dividend_per_share | NUMERIC(12,4) | DPS (Rs.) |
| **Comprehensive income** | | |
| other_comprehensive_income | NUMERIC(20,2) | OCI net of tax |
| total_comprehensive_income | NUMERIC(20,2) | PAT + OCI |
| **Sector-specific overflow** | | |
| extra_fields | JSONB | Industry-specific line items not captured above |

**Why both general and financial institution fields?** Every company uses a subset:
- Dilmah/Dockyard/Hotels use: revenue, cost_of_sales, gross_profit
- NTB/LB Finance use: interest_income, interest_expense, net_interest_income
- Softlogic Life uses: gross_written_premiums, net_claims_and_benefits

Unused fields are simply NULL. This avoids needing separate tables per sector while keeping the most-queried fields as proper columns (not buried in JSONB).

---

### 4. `balance_sheets`

Statement of financial position.

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| report_id | UUID | FK → financial_reports.id, UNIQUE, NOT NULL |
| **Assets** | | |
| total_non_current_assets | NUMERIC(20,2) | |
| property_plant_equipment | NUMERIC(20,2) | PPE |
| intangible_assets | NUMERIC(20,2) | |
| right_of_use_assets | NUMERIC(20,2) | IFRS 16 lease assets |
| investment_property | NUMERIC(20,2) | |
| financial_investments | NUMERIC(20,2) | Long-term financial assets |
| deferred_tax_assets | NUMERIC(20,2) | |
| other_non_current_assets | NUMERIC(20,2) | Catch-all |
| total_current_assets | NUMERIC(20,2) | |
| inventories | NUMERIC(20,2) | |
| trade_and_other_receivables | NUMERIC(20,2) | |
| cash_and_cash_equivalents | NUMERIC(20,2) | |
| other_current_assets | NUMERIC(20,2) | Catch-all |
| total_assets | NUMERIC(20,2) | **Key field** |
| **Financial institution assets** | | |
| loans_and_advances | NUMERIC(20,2) | Banks/finance: loan book |
| due_from_banks | NUMERIC(20,2) | Banks: interbank placements |
| financial_assets_at_fair_value | NUMERIC(20,2) | FVTPL/FVOCI instruments |
| financial_assets_at_amortised_cost | NUMERIC(20,2) | Debt instruments at AC |
| insurance_contract_assets | NUMERIC(20,2) | Insurance-specific |
| **Equity** | | |
| stated_capital | NUMERIC(20,2) | Share capital |
| reserves | NUMERIC(20,2) | All reserves combined |
| retained_earnings | NUMERIC(20,2) | Accumulated profits |
| total_equity_owners | NUMERIC(20,2) | Attributable to parent |
| non_controlling_interest | NUMERIC(20,2) | NCI |
| total_equity | NUMERIC(20,2) | **Key field** |
| **Liabilities** | | |
| total_non_current_liabilities | NUMERIC(20,2) | |
| interest_bearing_borrowings_nc | NUMERIC(20,2) | Long-term debt |
| lease_liabilities_nc | NUMERIC(20,2) | |
| deferred_tax_liabilities | NUMERIC(20,2) | |
| employee_benefit_obligations | NUMERIC(20,2) | Retirement/gratuity |
| other_non_current_liabilities | NUMERIC(20,2) | Catch-all |
| total_current_liabilities | NUMERIC(20,2) | |
| interest_bearing_borrowings_c | NUMERIC(20,2) | Short-term debt |
| trade_and_other_payables | NUMERIC(20,2) | |
| bank_overdraft | NUMERIC(20,2) | |
| other_current_liabilities | NUMERIC(20,2) | Catch-all |
| total_liabilities | NUMERIC(20,2) | **Key field** |
| **Financial institution liabilities** | | |
| due_to_banks | NUMERIC(20,2) | Banks: interbank borrowings |
| customer_deposits | NUMERIC(20,2) | Banks/finance: deposits |
| debt_securities_issued | NUMERIC(20,2) | |
| insurance_contract_liabilities | NUMERIC(20,2) | Insurance-specific |
| **Per share** | | |
| net_asset_per_share | NUMERIC(12,4) | NAV per share (Rs.) |
| **Overflow** | | |
| extra_fields | JSONB | Additional line items |

---

### 5. `cash_flow_statements`

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| report_id | UUID | FK → financial_reports.id, UNIQUE, NOT NULL |
| **Operating activities** | | |
| profit_before_tax | NUMERIC(20,2) | Starting point |
| depreciation_and_amortisation | NUMERIC(20,2) | D&A adjustment |
| impairment_charges | NUMERIC(20,2) | |
| finance_costs_adjustment | NUMERIC(20,2) | |
| finance_income_adjustment | NUMERIC(20,2) | |
| working_capital_changes | NUMERIC(20,2) | Net change |
| cash_generated_from_operations | NUMERIC(20,2) | Before tax & interest paid |
| tax_paid | NUMERIC(20,2) | |
| interest_paid | NUMERIC(20,2) | |
| net_cash_from_operating | NUMERIC(20,2) | **Key field** |
| **Investing activities** | | |
| purchase_of_ppe | NUMERIC(20,2) | Capex |
| proceeds_from_sale_of_ppe | NUMERIC(20,2) | |
| purchase_of_investments | NUMERIC(20,2) | |
| proceeds_from_investments | NUMERIC(20,2) | |
| interest_received | NUMERIC(20,2) | |
| dividends_received | NUMERIC(20,2) | |
| net_cash_from_investing | NUMERIC(20,2) | **Key field** |
| **Financing activities** | | |
| proceeds_from_borrowings | NUMERIC(20,2) | |
| repayment_of_borrowings | NUMERIC(20,2) | |
| lease_payments | NUMERIC(20,2) | |
| dividends_paid | NUMERIC(20,2) | |
| proceeds_from_share_issue | NUMERIC(20,2) | |
| net_cash_from_financing | NUMERIC(20,2) | **Key field** |
| **Summary** | | |
| net_change_in_cash | NUMERIC(20,2) | |
| cash_at_beginning | NUMERIC(20,2) | |
| exchange_rate_effect | NUMERIC(20,2) | |
| cash_at_end | NUMERIC(20,2) | **Key field** |
| **Overflow** | | |
| extra_fields | JSONB | Additional line items |

---

### 6. `api_keys`

For API monetization (unchanged from original plan).

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| key_hash | VARCHAR(64) | UNIQUE, SHA-256 |
| key_prefix | VARCHAR(8) | First 8 chars for display |
| customer_name | VARCHAR(255) | NOT NULL |
| customer_email | VARCHAR(255) | |
| plan | VARCHAR(50) | DEFAULT 'free' |
| rate_limit_per_minute | INTEGER | DEFAULT 60 |
| is_active | BOOLEAN | DEFAULT true |
| is_internal | BOOLEAN | DEFAULT false |
| expires_at | TIMESTAMPTZ | |
| created_at | TIMESTAMPTZ | DEFAULT now() |

### 7. `usage_logs`

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| api_key_id | UUID | FK → api_keys.id |
| endpoint | VARCHAR(255) | |
| method | VARCHAR(10) | |
| status_code | SMALLINT | |
| response_time_ms | INTEGER | |
| ip_address | VARCHAR(45) | |
| created_at | TIMESTAMPTZ | DEFAULT now(), indexed |

---

## Indexes

```sql
-- Core lookups
CREATE UNIQUE INDEX idx_companies_ticker ON companies(ticker);
CREATE UNIQUE INDEX idx_reports_unique ON financial_reports(company_id, year, quarter, is_consolidated);
CREATE INDEX idx_reports_year_quarter ON financial_reports(year, quarter);
CREATE INDEX idx_reports_company ON financial_reports(company_id);

-- Financial statement lookups
CREATE UNIQUE INDEX idx_income_report ON income_statements(report_id);
CREATE UNIQUE INDEX idx_balance_report ON balance_sheets(report_id);
CREATE UNIQUE INDEX idx_cashflow_report ON cash_flow_statements(report_id);

-- API usage
CREATE INDEX idx_usage_key_date ON usage_logs(api_key_id, created_at);
CREATE INDEX idx_api_keys_hash ON api_keys(key_hash);
```

---

## How the `extra_fields` JSONB Works

Each sector has line items unique to it. Rather than adding hundreds of nullable columns, sector-specific details go into `extra_fields`:

**Insurance example** (Softlogic Life):
```json
{
  "premiums_ceded_to_reinsurers": -2737583,
  "net_realised_gains": 1245304,
  "net_fair_value_gains": 267447,
  "insurance_benefits_and_claims": -19318180,
  "change_in_insurance_contract_liabilities": -4292556,
  "underwriting_and_acquisition_costs": -7239551
}
```

**Banking example** (Nations Trust Bank):
```json
{
  "net_fee_and_commission_income": 4632708,
  "net_gain_from_trading": 4538409,
  "net_gain_from_derecognition": 1602870,
  "vat_on_financial_services": -8083580,
  "sscl_on_financial_services": -859865,
  "tier_1_capital_ratio": 19.61,
  "total_capital_adequacy_ratio": 20.72,
  "net_stage_3_ratio": 0.91,
  "return_on_equity": 21.86,
  "net_interest_margin": 6.05
}
```

**General company example** (Dilmah):
```json
{
  "foreign_exchange_gain_loss": 1101532,
  "selling_and_distribution_costs": -2774101
}
```

This approach ensures:
- The 80% most-queried fields are proper indexed columns
- The 20% sector-specific fields are still stored and queryable via JSONB
- No schema changes needed when adding a new industry

---

## Data Flow: PDF → Database

```
PDF Report
    ↓
LiteParse (local text extraction)
    ↓
DeepSeek (structured JSON extraction)
    ↓
Mayura CLI validates against Pydantic models
    ↓
POST /api/v1/internal/reports
    ↓
API resolves company (get_or_create by ticker)
    ↓
Creates financial_report record
    ↓
Creates income_statement, balance_sheet, cash_flow_statement
    ↓
PostgreSQL
```

---

## Observations from Report Analysis

### Reporting period variations
- **Standard calendar year:** Colombo Dockyard, Softlogic Life, NTB
- **March financial year (9-month interim):** Dilmah (FY ends 31 March), LB Finance (FY ends 31 March), Ceylon Hospitals (FY ends 31 March)
- **Non-standard year-end:** Union Chemicals (25 December)
- **Implication:** The `period_end_date` and `period_months` fields are critical. Quarter numbering must be relative to the company's financial year, not the calendar year.

### Unit variations
- All 9 reports use Rs. '000 (thousands)
- But some EPS/DPS/NAV values are in full rupees — these per-share fields use `NUMERIC(12,4)` separately

### Group vs Company
- 7 of 9 reports show both **Group (consolidated)** and **Company (standalone)** figures
- The `is_consolidated` flag on `financial_reports` handles this
- Most API consumers will want Group figures, so default API queries should filter to `is_consolidated = true`

### Financial institution differences
| Field | Banks (NTB) | Finance (LB Finance) | Insurance (Softlogic Life) |
|-------|-------------|---------------------|---------------------------|
| Primary revenue | Interest income | Interest income | Gross written premiums |
| Primary expense | Interest expense | Interest expense | Claims & benefits |
| Key metric | Net interest income | Net interest income | Net written premiums |
| Unique assets | Loans & advances | Loans & receivables | Financial investments, Policyholder loans |
| Unique liabilities | Customer deposits | Due to depositors | Insurance contract liabilities |
| Regulatory ratios | Capital adequacy, NIM | - | - |

### Common fields across ALL 9 companies
These fields appear in every single report regardless of sector:
- **Income statement:** profit_before_tax, income_tax_expense, profit_after_tax, eps_basic
- **Balance sheet:** total_assets, total_equity, total_liabilities, cash_and_cash_equivalents, property_plant_equipment, stated_capital, retained_earnings
- **Cash flow:** net_cash_from_operating, net_cash_from_investing, net_cash_from_financing, net_change_in_cash

---

## Sample API Queries

```
# Get latest quarterly report for a company
GET /api/v1/reports/NTB?latest=true

# Get all Q4 2025 reports across all companies
GET /api/v1/reports?quarter=4&year=2025

# Get banking sector companies
GET /api/v1/companies?sector=Banking

# Compare profitability across sectors
GET /api/v1/reports?year=2025&fields=ticker,profit_after_tax,eps_basic

# Get full financial statement breakdown
GET /api/v1/reports/JKH/2025/3?include=income_statement,balance_sheet,cash_flow
```
