# Data Pipeline Flow

Step-by-step walkthrough of how data moves from a parsed PDF through DeepSeek, the API, and into the database.

---

## Full Pipeline Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                        MAYURA CLI                                │
│                                                                  │
│  ┌─────────┐    ┌────────────┐    ┌───────────┐    ┌─────────┐  │
│  │ User    │───→│ LiteParse  │───→│ DeepSeek  │───→│ Review  │  │
│  │ Input   │    │ (local)    │    │ API       │    │ & Confirm│  │
│  │         │    │            │    │           │    │         │  │
│  │ ticker  │    │ PDF →      │    │ text →    │    │ show    │  │
│  │ quarter │    │ text       │    │ JSON      │    │ table   │  │
│  │ year    │    │            │    │           │    │         │  │
│  │ pdf     │    │            │    │           │    │ [y/N]   │  │
│  └─────────┘    └────────────┘    └───────────┘    └────┬────┘  │
│                                                         │       │
│                                                    ┌────▼────┐  │
│                                                    │ POST to │  │
│                                                    │ API     │  │
│                                                    └────┬────┘  │
└─────────────────────────────────────────────────────────┼───────┘
                                                          │
┌─────────────────────────────────────────────────────────┼───────┐
│                     GARUDA REST API                      │       │
│                                                         ▼       │
│  ┌──────────────┐    ┌──────────────┐    ┌────────────────┐     │
│  │ Authenticate │───→│ Validate     │───→│ Write to DB    │     │
│  │ X-Internal-  │    │ Pydantic     │    │                │     │
│  │ Key header   │    │ schemas      │    │ 1. Upsert co.  │     │
│  └──────────────┘    └──────────────┘    │ 2. Upsert rpt  │     │
│                                          │ 3. Upsert IS   │     │
│                                          │ 4. Upsert BS   │     │
│                                          │ 5. Upsert CF   │     │
│                                          └────────┬───────┘     │
│                                                   │             │
│                                          ┌────────▼───────┐     │
│                                          │  PostgreSQL    │     │
│                                          │  (Coolify)     │     │
│                                          └────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Step 1: User Input (CLI)

The user provides 4 inputs:
```
Company ticker: JKH
Quarter: Q3
Year: 2025
PDF file: /path/to/jkh_q3_2025.pdf
```

The CLI validates:
- Ticker is non-empty
- Quarter is Q1-Q4
- Year is reasonable (2015-current)
- PDF file exists and has .pdf extension

---

## Step 2: PDF Parsing (LiteParse — Local)

```python
from liteparse import LiteParse

parser = LiteParse()
result = parser.parse(str(pdf_path))
extracted_text = result.text
```

**Input:** PDF file path
**Output:** Plain text with spatial layout preserved (~5,000-20,000 characters)

**Prerequisites:** Node.js >= 18 must be installed.

LiteParse runs entirely locally — no API keys, no cloud, no network needed. It preserves the spatial layout of text, so table columns stay aligned:
```
Revenue                     125,053,144    81,254,270     54
Cost of sales              (98,830,601)  (65,060,634)     52
Gross profit                26,222,543    16,193,636      62
```

This is raw text (not markdown tables), but DeepSeek can still interpret the spatial layout and extract the correct values. The prompt is designed to handle this format.

---

## Step 3: DeepSeek Extraction

### 3a. Build the API request

```python
from openai import OpenAI

client = OpenAI(
    api_key=settings.deepseek_api_key,
    base_url="https://api.deepseek.com"
)

# System prompt is loaded from prompts.py (see deepseek-prompt-guide.md)
system_prompt = load_system_prompt()

# User prompt includes the parsed text + metadata
user_prompt = f"""Extract financial data from the following CSE quarterly report.

Company ticker: {ticker}
Expected quarter: Q{quarter}
Expected year: {year}

--- BEGIN REPORT TEXT ---
{extracted_text}
--- END REPORT TEXT ---"""
```

### 3b. Call the API

```python
response = client.chat.completions.create(
    model="deepseek-chat",
    messages=[
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": user_prompt}
    ],
    response_format={"type": "json_object"},
    temperature=0.1,
    max_tokens=8000,
)

raw_json_string = response.choices[0].message.content
```

### 3c. What DeepSeek returns

A JSON string like this (example for John Keells Holdings PLC):

```json
{
  "metadata": {
    "company_name": "John Keells Holdings PLC",
    "company_type": "general",
    "period_end_date": "2025-12-31",
    "period_months": 9,
    "currency": "LKR",
    "units": "thousands",
    "is_audited": false,
    "is_consolidated": true,
    "report_date": "2026-01-28"
  },
  "income_statement": {
    "revenue": 383961566,
    "cost_of_sales": -307902287,
    "gross_profit": 76059279,
    "interest_income": null,
    "interest_expense": null,
    "net_interest_income": null,
    "fee_and_commission_income": null,
    "net_trading_income": null,
    "gross_written_premiums": null,
    "net_written_premiums": null,
    "net_claims_and_benefits": null,
    "other_income": 3871488,
    "total_operating_income": null,
    "distribution_expenses": -10217745,
    "administrative_expenses": -35725884,
    "other_operating_expenses": -7678798,
    "total_operating_expenses": null,
    "operating_profit": 26308340,
    "impairment_charges": null,
    "finance_income": 17176919,
    "finance_costs": -18448318,
    "net_finance_income": -1271399,
    "profit_before_tax": 23793133,
    "income_tax_expense": -10370735,
    "profit_after_tax": 13422398,
    "profit_attributable_to_owners": 7328714,
    "profit_attributable_to_nci": 6093684,
    "eps_basic": 0.41,
    "eps_diluted": 0.41,
    "dividend_per_share": 0.15,
    "other_comprehensive_income": 5389309,
    "total_comprehensive_income": 18811707,
    "extra_fields": {
      "change_in_insurance_contract_liabilities": -12352335,
      "change_in_fair_value_investment_property": 2299828,
      "share_of_equity_accounted_investees": 8808699,
      "revenue_from_contracts_with_customers": 365244245,
      "revenue_from_insurance_contracts": 18717321
    }
  },
  "balance_sheet": {
    "...": "...full balance sheet fields..."
  },
  "cash_flow_statement": {
    "...": "...full cash flow fields..."
  }
}
```

### 3d. Validate with Pydantic

```python
import json
from mayura.models import QuarterlyReportExtraction

# Parse JSON string
data = json.loads(raw_json_string)

# Validate through Pydantic model
# This catches: wrong types, missing required fields, invalid values
report = QuarterlyReportExtraction(**data)
```

If validation fails, retry once with the error message (see deepseek-prompt-guide.md).

---

## Step 4: User Review

The CLI displays the extracted data as a Rich table:

```
┌─────────────────────────────────────────────────────────────┐
│  JOHN KEELLS HOLDINGS PLC (JKH) - Q3 2025                   │
│  Period: 9 months ended 2025-12-31 | Units: Rs.'000         │
│  Type: General | Consolidated: Yes | Audited: No             │
├─────────────────────────────────────────────────────────────┤
│  INCOME STATEMENT                                            │
│  Revenue                           383,961,566              │
│  Cost of Sales                    (307,902,287)             │
│  Gross Profit                       76,059,279              │
│  Operating Profit                   26,308,340              │
│  Finance Income                     17,176,919              │
│  Finance Costs                     (18,448,318)             │
│  Profit Before Tax                  23,793,133              │
│  Income Tax                        (10,370,735)             │
│  Profit After Tax                   13,422,398              │
│  EPS (Basic)                        Rs. 0.41                │
├─────────────────────────────────────────────────────────────┤
│  BALANCE SHEET                                               │
│  Total Assets                      XXX,XXX,XXX              │
│  Total Equity                       XX,XXX,XXX              │
│  Total Liabilities                 XXX,XXX,XXX              │
│  NAV per Share                      Rs. X.XX                │
├─────────────────────────────────────────────────────────────┤
│  CASH FLOW                                                   │
│  Operating Activities               XX,XXX,XXX              │
│  Investing Activities              (XX,XXX,XXX)             │
│  Financing Activities              (XX,XXX,XXX)             │
│  Cash at End                        XX,XXX,XXX              │
└─────────────────────────────────────────────────────────────┘

Submit this data? [y/N]:
```

The user compares the table against the original PDF. If correct, they confirm.

If `--dry-run` is used, the CLI dumps the full JSON to stdout and exits:
```bash
mayura --dry-run > jkh_q3_2025.json
```

---

## Step 5: POST to Garuda REST API

### 5a. Build the API request body

The CLI transforms the DeepSeek response into the API's expected format:

```python
# The validated Pydantic model is serialized to the API format
api_payload = {
    "ticker": "JKH",
    "company_name": report.metadata.company_name,
    "company_type": report.metadata.company_type,
    "sector": None,  # Can be set manually or left for API to resolve
    "year": 2025,
    "quarter": 3,
    "period_end_date": "2025-12-31",
    "period_months": 9,
    "currency": "LKR",
    "units": "thousands",
    "is_audited": False,
    "is_consolidated": True,
    "report_date": "2026-01-28",
    "income_statement": {
        "revenue": 383961566,
        "cost_of_sales": -307902287,
        "gross_profit": 76059279,
        # ... all income statement fields ...
        "extra_fields": {
            "change_in_insurance_contract_liabilities": -12352335,
            # ...
        }
    },
    "balance_sheet": {
        "total_assets": ...,
        # ... all balance sheet fields ...
        "extra_fields": {}
    },
    "cash_flow_statement": {
        "net_cash_from_operating": ...,
        # ... all cash flow fields ...
        "extra_fields": {}
    }
}
```

### 5b. Send the HTTP request

```python
import httpx

async with httpx.AsyncClient() as client:
    response = await client.post(
        f"{settings.api_base_url}/api/v1/internal/reports",
        json=api_payload,
        headers={
            "X-Internal-Key": settings.api_internal_key,
            "Content-Type": "application/json"
        },
        timeout=30.0
    )

    if response.status_code == 200:
        result = response.json()
        # {"id": "uuid", "status": "created", "company_id": "uuid", "report_id": "uuid"}
    elif response.status_code == 409:
        # Report already exists — ask user to confirm overwrite
        pass
    else:
        raise ApiError(f"API returned {response.status_code}: {response.text}")
```

---

## Step 6: API Receives and Processes

### 6a. Authentication

```python
# FastAPI dependency
async def verify_internal_key(x_internal_key: str = Header()):
    if x_internal_key != settings.internal_api_key:
        raise HTTPException(status_code=401, detail="Invalid internal key")
```

### 6b. Validation

The API validates the incoming JSON through its own Pydantic schemas:

```python
class FinancialReportCreate(BaseModel):
    ticker: str
    company_name: str
    company_type: Literal["general", "bank", "finance", "insurance"]
    year: int
    quarter: int  # 1-4
    period_end_date: date
    period_months: int
    currency: str = "LKR"
    units: str = "thousands"
    is_audited: bool = False
    is_consolidated: bool = True
    report_date: date | None = None
    income_statement: IncomeStatementCreate
    balance_sheet: BalanceSheetCreate
    cash_flow_statement: CashFlowStatementCreate
```

### 6c. Database Write (Upsert)

The API performs a multi-step upsert within a single database transaction:

```python
@router.post("/api/v1/internal/reports")
async def create_report(
    payload: FinancialReportCreate,
    db: AsyncSession = Depends(get_db),
    _: None = Depends(verify_internal_key)
):
    async with db.begin():
        # STEP 1: Get or create the company
        company = await crud.company.get_or_create(
            db,
            ticker=payload.ticker,
            name=payload.company_name,
            company_type=payload.company_type,
            sector=payload.sector
        )

        # STEP 2: Upsert the financial_report record
        # (unique on company_id + year + quarter + is_consolidated)
        report = await crud.financial_report.upsert(
            db,
            company_id=company.id,
            year=payload.year,
            quarter=payload.quarter,
            is_consolidated=payload.is_consolidated,
            defaults={
                "period_end_date": payload.period_end_date,
                "period_months": payload.period_months,
                "currency": payload.currency,
                "units": payload.units,
                "is_audited": payload.is_audited,
                "report_date": payload.report_date,
                "raw_data": payload.model_dump(mode="json"),
            }
        )

        # STEP 3: Upsert income statement
        await crud.income_statement.upsert(
            db,
            report_id=report.id,
            data=payload.income_statement.model_dump()
        )

        # STEP 4: Upsert balance sheet
        await crud.balance_sheet.upsert(
            db,
            report_id=report.id,
            data=payload.balance_sheet.model_dump()
        )

        # STEP 5: Upsert cash flow statement
        await crud.cash_flow_statement.upsert(
            db,
            report_id=report.id,
            data=payload.cash_flow_statement.model_dump()
        )

    return {
        "status": "created",
        "company_id": str(company.id),
        "report_id": str(report.id),
        "ticker": payload.ticker,
        "period": f"Q{payload.quarter} {payload.year}"
    }
```

### 6d. How the upsert works (SQLAlchemy)

```python
from sqlalchemy.dialects.postgresql import insert

async def upsert(self, db: AsyncSession, company_id, year, quarter, is_consolidated, defaults):
    stmt = insert(FinancialReport).values(
        company_id=company_id,
        year=year,
        quarter=quarter,
        is_consolidated=is_consolidated,
        **defaults
    ).on_conflict_do_update(
        constraint="uq_reports_company_year_quarter_consolidated",
        set_=defaults
    )
    result = await db.execute(stmt)
    await db.flush()

    # Fetch the upserted row
    report = await db.execute(
        select(FinancialReport).where(
            FinancialReport.company_id == company_id,
            FinancialReport.year == year,
            FinancialReport.quarter == quarter,
            FinancialReport.is_consolidated == is_consolidated,
        )
    )
    return report.scalar_one()
```

---

## Step 7: API Response → CLI Display

### Success response (200)

```json
{
  "status": "created",
  "company_id": "a1b2c3d4-...",
  "report_id": "e5f6g7h8-...",
  "ticker": "JKH",
  "period": "Q3 2025"
}
```

CLI displays:
```
✓ Successfully saved JKH Q3 2025 financial report
  Report ID: e5f6g7h8-...
```

### Duplicate/update response (200)

If the same ticker+quarter+year already exists, the upsert updates it:
```json
{
  "status": "updated",
  "company_id": "a1b2c3d4-...",
  "report_id": "e5f6g7h8-...",
  "ticker": "JKH",
  "period": "Q3 2025"
}
```

### Error responses

| Status | Meaning | CLI Display |
|--------|---------|-------------|
| 401 | Invalid internal key | "Authentication failed. Check your API key." |
| 422 | Validation error | "Invalid data: {field}: {error}" |
| 500 | Server error | "API server error. Check logs." |
| Connection refused | API not running | "Cannot reach API at {url}. Is the server running?" |

---

## What Ends Up in the Database

After a successful pipeline run for JKH Q3 2025:

### `companies` table
```
id:             a1b2c3d4-...
ticker:         JKH
name:           John Keells Holdings PLC
sector:         Diversified
company_type:   general
financial_year_end: March 31
```

### `financial_reports` table
```
id:              e5f6g7h8-...
company_id:      a1b2c3d4-...
year:            2025
quarter:         3
period_end_date: 2025-12-31
period_months:   9
currency:        LKR
units:           thousands
is_audited:      false
is_consolidated: true
report_date:     2026-01-28
raw_data:        {full JSON payload}
```

### `income_statements` table
```
id:              i1j2k3l4-...
report_id:       e5f6g7h8-...
revenue:         383961566.00
cost_of_sales:   -307902287.00
gross_profit:    76059279.00
operating_profit: 26308340.00
profit_before_tax: 23793133.00
income_tax_expense: -10370735.00
profit_after_tax: 13422398.00
eps_basic:       0.4100
extra_fields:    {"change_in_insurance_contract_liabilities": -12352335, ...}
```

### `balance_sheets` table
```
id:              m1n2o3p4-...
report_id:       e5f6g7h8-...
total_assets:    ...
total_equity:    ...
total_liabilities: ...
cash_and_cash_equivalents: ...
net_asset_per_share: ...
```

### `cash_flow_statements` table
```
id:              q1r2s3t4-...
report_id:       e5f6g7h8-...
net_cash_from_operating: ...
net_cash_from_investing: ...
net_cash_from_financing: ...
cash_at_end:     ...
```

---

## Reading the Data Back (Public API)

Once data is saved, customers query it through the public read API:

```bash
# Get JKH's Q3 2025 report
curl -H "X-API-Key: garuda_abc123..." \
  "https://api.garuda.lk/api/v1/reports/JKH/2025/3"
```

Response:
```json
{
  "company": {
    "ticker": "JKH",
    "name": "John Keells Holdings PLC",
    "sector": "Diversified",
    "company_type": "general"
  },
  "report": {
    "year": 2025,
    "quarter": 3,
    "period_end_date": "2025-12-31",
    "period_months": 9,
    "currency": "LKR",
    "units": "thousands",
    "is_audited": false
  },
  "income_statement": {
    "revenue": 383961566,
    "cost_of_sales": -307902287,
    "gross_profit": 76059279,
    "profit_before_tax": 23793133,
    "profit_after_tax": 13422398,
    "eps_basic": 0.41,
    "...": "..."
  },
  "balance_sheet": {
    "total_assets": "...",
    "total_equity": "...",
    "...": "..."
  },
  "cash_flow_statement": {
    "net_cash_from_operating": "...",
    "...": "..."
  }
}
```

---

## Error Handling Summary

| Stage | Error | Recovery |
|-------|-------|----------|
| LiteParse | Node.js not installed | Show error: "Node.js >= 18 required. Install from nodejs.org" |
| LiteParse | Empty/corrupt PDF | Show error, ask user to check file |
| DeepSeek | Invalid JSON response | Retry once with correction prompt |
| DeepSeek | Pydantic validation failure | Retry once with error details |
| DeepSeek | Missing critical fields (PBT, total assets) | Warn user, allow submission anyway |
| API POST | 401 Unauthorized | Check API key config |
| API POST | 422 Validation | Show which fields failed, allow user to fix |
| API POST | Connection refused | Check if API is running |
| API POST | 500 Server error | Log and show generic error |
| Database | Unique constraint violation | Upsert handles this automatically |
| Database | Connection failure | API returns 500, CLI shows error |
