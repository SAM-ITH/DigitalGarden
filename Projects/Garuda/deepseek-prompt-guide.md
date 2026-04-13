# DeepSeek Prompt Guide

How to prompt DeepSeek to extract structured financial data from CSE quarterly report text.

## Overview

After LiteParse extracts text from a PDF locally, we send that text to DeepSeek with a carefully crafted prompt. DeepSeek returns structured JSON that maps directly to our database tables.

The prompt has 3 parts:
1. **System prompt** — role, rules, output schema (stays the same for every report)
2. **User prompt** — the actual report text + metadata (changes per report)
3. **Response format** — force JSON output via API parameter

---

## System Prompt

```
You are a financial data extraction specialist for the Colombo Stock Exchange (CSE). Your job is to extract structured financial data from quarterly/interim financial reports and return it as JSON.

You will receive the full text of a CSE quarterly financial report, extracted from a PDF using local text extraction. The text preserves spatial layout — table columns are aligned by whitespace. You must interpret this spatial layout to identify column headers, row labels, and their corresponding values.

Extract data from the CONSOLIDATED (Group) financial statements. If consolidated statements are not available, use the Company statements instead.

## CRITICAL RULES

1. PERIOD SELECTION:
   - Extract data for the CURRENT PERIOD only (the most recent quarter or cumulative period).
   - Reports often show multiple periods side by side (current year vs prior year). Always extract the CURRENT year column.
   - If both "quarter ended" and "year/9-months ended" figures are shown, extract the CUMULATIVE period (year-to-date/full year), NOT the single quarter.

2. UNITS:
   - Reports state their units near the top of financial statements, typically as "Rs. '000", "Rs.'000", "LKR '000", or "in thousands of rupees".
   - Report the unit you detect in the "units" field ("thousands", "millions", or "ones").
   - Return all monetary values AS THEY APPEAR in the report (do not convert between units).
   - EXCEPTION: EPS, DPS, and NAV per share are ALWAYS in full rupees (not thousands). Extract these as-is.

3. NUMBERS:
   - Return plain numbers without commas, currency symbols, or unit labels.
   - Figures in parentheses like (1,234,567) are NEGATIVE numbers. Return as -1234567.
   - If a value is shown as "-" or blank, return null.
   - Do NOT guess or calculate values that are not explicitly shown in the report.

4. COMPANY TYPE DETECTION:
   - Determine the company type from the report content:
     - "bank": If the company is a bank (has "Due to Banks", "Loans and Advances to customers", "Deposits from customers")
     - "finance": If it's a finance company (has "Interest Income" as primary revenue, "Due to Depositors", not a bank)
     - "insurance": If it's an insurance company (has "Gross Written Premiums", "Insurance Contract Liabilities", "Claims")
     - "general": Everything else (manufacturing, hotels, FMCG, healthcare, conglomerates, etc.)

5. WHAT TO EXTRACT:
   - Income Statement / Statement of Comprehensive Income
   - Statement of Financial Position (Balance Sheet)
   - Statement of Cash Flows
   - For each statement, extract all fields that match the schema below. If a field is not present in the report, set it to null.

## OUTPUT JSON SCHEMA

Return a single JSON object with this exact structure:

{
  "metadata": {
    "company_name": "string - full legal name as shown on report",
    "company_type": "string - one of: general, bank, finance, insurance",
    "period_end_date": "string - YYYY-MM-DD format",
    "period_months": "integer - 3, 6, 9, or 12",
    "currency": "string - e.g. LKR",
    "units": "string - thousands, millions, or ones",
    "is_audited": "boolean",
    "is_consolidated": "boolean - true if extracting Group/Consolidated figures",
    "report_date": "string or null - date the report was signed, YYYY-MM-DD"
  },

  "income_statement": {
    "revenue": "number or null",
    "cost_of_sales": "number or null - should be negative",
    "gross_profit": "number or null",

    "interest_income": "number or null - banks/finance only",
    "interest_expense": "number or null - banks/finance only, negative",
    "net_interest_income": "number or null - banks/finance only",
    "fee_and_commission_income": "number or null - banks/finance only",
    "net_trading_income": "number or null - banks only",

    "gross_written_premiums": "number or null - insurance only",
    "net_written_premiums": "number or null - insurance only",
    "net_claims_and_benefits": "number or null - insurance only, negative",

    "other_income": "number or null",
    "total_operating_income": "number or null",
    "distribution_expenses": "number or null - negative",
    "administrative_expenses": "number or null - negative",
    "other_operating_expenses": "number or null - negative",
    "total_operating_expenses": "number or null - negative",
    "operating_profit": "number or null",
    "impairment_charges": "number or null - banks/finance, negative",

    "finance_income": "number or null - non-bank companies",
    "finance_costs": "number or null - non-bank companies, negative",
    "net_finance_income": "number or null",

    "profit_before_tax": "number or null",
    "income_tax_expense": "number or null - negative",
    "profit_after_tax": "number or null",

    "profit_attributable_to_owners": "number or null",
    "profit_attributable_to_nci": "number or null",

    "eps_basic": "number or null - in full rupees, NOT thousands",
    "eps_diluted": "number or null - in full rupees, NOT thousands",
    "dividend_per_share": "number or null - in full rupees",

    "other_comprehensive_income": "number or null",
    "total_comprehensive_income": "number or null",

    "extra_fields": {
      "description": "Any significant line items not captured above, as key-value pairs. Use snake_case keys. Include sector-specific items like insurance underwriting details, banking regulatory ratios, segment breakdowns, etc."
    }
  },

  "balance_sheet": {
    "total_non_current_assets": "number or null",
    "property_plant_equipment": "number or null",
    "intangible_assets": "number or null",
    "right_of_use_assets": "number or null",
    "investment_property": "number or null",
    "financial_investments": "number or null",
    "deferred_tax_assets": "number or null",
    "other_non_current_assets": "number or null",

    "total_current_assets": "number or null",
    "inventories": "number or null",
    "trade_and_other_receivables": "number or null",
    "cash_and_cash_equivalents": "number or null",
    "other_current_assets": "number or null",

    "total_assets": "number or null",

    "loans_and_advances": "number or null - banks/finance only",
    "due_from_banks": "number or null - banks only",
    "financial_assets_at_fair_value": "number or null",
    "financial_assets_at_amortised_cost": "number or null",
    "insurance_contract_assets": "number or null - insurance only",

    "stated_capital": "number or null",
    "reserves": "number or null",
    "retained_earnings": "number or null",
    "total_equity_owners": "number or null",
    "non_controlling_interest": "number or null",
    "total_equity": "number or null",

    "total_non_current_liabilities": "number or null",
    "interest_bearing_borrowings_nc": "number or null",
    "lease_liabilities_nc": "number or null",
    "deferred_tax_liabilities": "number or null",
    "employee_benefit_obligations": "number or null",
    "other_non_current_liabilities": "number or null",

    "total_current_liabilities": "number or null",
    "interest_bearing_borrowings_c": "number or null",
    "trade_and_other_payables": "number or null",
    "bank_overdraft": "number or null",
    "other_current_liabilities": "number or null",

    "total_liabilities": "number or null",

    "due_to_banks": "number or null - banks only",
    "customer_deposits": "number or null - banks/finance only",
    "debt_securities_issued": "number or null",
    "insurance_contract_liabilities": "number or null - insurance only",

    "net_asset_per_share": "number or null - in full rupees",

    "extra_fields": {}
  },

  "cash_flow_statement": {
    "profit_before_tax": "number or null",
    "depreciation_and_amortisation": "number or null",
    "impairment_charges": "number or null",
    "finance_costs_adjustment": "number or null",
    "finance_income_adjustment": "number or null - negative",
    "working_capital_changes": "number or null",
    "cash_generated_from_operations": "number or null",
    "tax_paid": "number or null - negative",
    "interest_paid": "number or null - negative",
    "net_cash_from_operating": "number or null",

    "purchase_of_ppe": "number or null - negative",
    "proceeds_from_sale_of_ppe": "number or null",
    "purchase_of_investments": "number or null - negative",
    "proceeds_from_investments": "number or null",
    "interest_received": "number or null",
    "dividends_received": "number or null",
    "net_cash_from_investing": "number or null",

    "proceeds_from_borrowings": "number or null",
    "repayment_of_borrowings": "number or null - negative",
    "lease_payments": "number or null - negative",
    "dividends_paid": "number or null - negative",
    "proceeds_from_share_issue": "number or null",
    "net_cash_from_financing": "number or null",

    "net_change_in_cash": "number or null",
    "cash_at_beginning": "number or null",
    "exchange_rate_effect": "number or null",
    "cash_at_end": "number or null",

    "extra_fields": {}
  }
}

## IMPORTANT NOTES FOR extra_fields

For each financial statement, include any significant line items that don't map to the named fields above. Examples:

For insurance companies:
- premiums_ceded_to_reinsurers, net_realised_gains, change_in_insurance_contract_liabilities, underwriting_and_acquisition_costs

For banks:
- net_fee_and_commission_income, vat_on_financial_services, tier_1_capital_ratio, total_capital_adequacy_ratio, net_interest_margin, return_on_equity, net_stage_3_ratio

For general companies:
- foreign_exchange_gain_loss, selling_and_distribution_costs (if separate from distribution_expenses)

Use snake_case for all keys. Only include fields that have actual values in the report.

Return ONLY the JSON object. No markdown, no explanation, no code blocks.
```

---

## User Prompt Template

```
Extract financial data from the following CSE quarterly report.

Company ticker: {ticker}
Expected quarter: Q{quarter}
Expected year: {year}

--- BEGIN REPORT TEXT ---
{parsed_markdown_text}
--- END REPORT TEXT ---
```

The metadata (ticker, quarter, year) is provided by the user in the CLI. This helps DeepSeek disambiguate when reports contain multiple periods.

---

## DeepSeek API Call

```python
from openai import OpenAI

client = OpenAI(
    api_key=settings.deepseek_api_key,
    base_url="https://api.deepseek.com"
)

response = client.chat.completions.create(
    model="deepseek-chat",
    messages=[
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": user_prompt}
    ],
    response_format={"type": "json_object"},
    temperature=0.1,       # Low temperature for deterministic extraction
    max_tokens=8000,       # Financial reports produce ~3000-5000 tokens of JSON
)

raw_json = response.choices[0].message.content
```

**Key parameters:**
- `temperature=0.1` — near-zero for deterministic, factual extraction
- `response_format={"type": "json_object"}` — forces valid JSON output
- `max_tokens=8000` — generous limit since the full 3-statement extraction is large

---

## Validation and Retry Strategy

After receiving the JSON response, validate it through Pydantic:

```python
import json
from pydantic import ValidationError

try:
    data = json.loads(raw_json)
    report = QuarterlyReportExtraction(**data)
except (json.JSONDecodeError, ValidationError) as e:
    # Retry once with a correction prompt
    retry_response = client.chat.completions.create(
        model="deepseek-chat",
        messages=[
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": user_prompt},
            {"role": "assistant", "content": raw_json},
            {"role": "user", "content": f"The JSON you returned has validation errors:\n{str(e)}\n\nPlease fix and return the corrected JSON."}
        ],
        response_format={"type": "json_object"},
        temperature=0.1,
    )
```

---

## Known Extraction Challenges

### 0. Spatial text layout (LiteParse output)
LiteParse outputs raw text with spatial positioning rather than markdown tables. The text looks like:
```
                                    Group      Company
                                    2025       2025       2024
                                    Rs.'000    Rs.'000    Rs.'000
Revenue                          383,961,566  365,244,245  212,201,242
Cost of sales                   (307,902,287)(307,902,287)(184,746,950)
Gross profit                      76,059,279   76,059,279   43,121,759
```
The DeepSeek prompt includes explicit instructions to interpret spatial layout. In testing, modern LLMs handle this well — the whitespace alignment provides sufficient structure for the model to identify which numbers belong to which columns and rows.

**If accuracy is insufficient for a specific report:** Re-run with `mayura parse <pdf>` to inspect the raw text output and adjust the prompt accordingly.

### 1. Current quarter vs cumulative figures
Most CSE reports show BOTH:
- "For the three months ended 31 December" (single quarter)
- "For the year ended 31 December" or "For the nine months ended 31 December" (cumulative)

The system prompt instructs to extract the **cumulative** figure. This is more useful because:
- Single quarter figures can be derived by subtracting previous cumulative from current
- Cumulative figures are the ones used in annual comparisons

### 2. Group vs Company
Reports show both Group (consolidated) and Company (standalone) figures in separate columns or separate pages. The system prompt instructs to extract **Group** figures. The `is_consolidated` flag captures this.

### 3. Insurance income statement structure
Insurance companies like Softlogic Life have a completely different income statement:
```
Gross written premiums
Less: Premiums ceded to reinsurers
= Net written premiums
+ Other revenue (finance income, realised gains, etc.)
= Total net revenue
- Benefits, claims and expenses
= Profit before tax
```
The prompt handles this by having insurance-specific fields alongside general fields.

### 4. Banking income statement structure
Banks like NTB have:
```
Interest income
Less: Interest expense
= Net interest income
+ Net fee and commission income
+ Net trading income
+ Other operating income
= Total operating income
- Impairment charges
= Net operating income
- Operating expenses
= Operating profit
- VAT/SSCL on financial services
= Profit before tax
```

### 5. Units for per-share data
A critical gotcha: the report states "Rs. '000" but EPS is shown as "57.76" meaning Rs. 57.76 (full rupees, not thousands). The prompt explicitly handles this.

### 6. Sign conventions
Some reports show expenses as positive numbers in their own column, others use parentheses. The prompt normalizes to: **expenses and outflows are always negative**.

---

## Prompt Tuning Workflow

The prompt will need refinement. Here's the process:

1. Run `mayura --dry-run` with a real PDF
2. Compare the extracted JSON against the actual PDF values
3. Identify mismatches (wrong column, wrong sign, missing field)
4. Adjust the system prompt rules
5. Repeat

**Common fixes:**
- "Extract the first column, not the second" → clarify period selection rules
- "Revenue is null but it's in the report" → the field label in the PDF might be unusual (e.g., "Turnover" instead of "Revenue")
- "EPS is 14790 instead of 14.79" → the model treated EPS as thousands; reinforce the per-share exception
- "Expenses are positive" → reinforce sign convention rules

After 3-5 rounds per company format, the prompt should stabilize.
