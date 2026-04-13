# System 1: Mayura CLI

Bloomberg-style Python CLI terminal for extracting structured financial data from CSE quarterly report PDFs.

**Tech stack:** Python, Rich, LiteParse (local PDF parsing), DeepSeek API (via OpenAI SDK), httpx, pydantic-settings

**Prerequisites:** Node.js >= 18 (required by LiteParse)

## Data Flow

```
User Input (ticker, quarter, year, PDF)
        ↓
   LiteParse (local)
        ↓
   Extracted Text (spatial layout preserved)
        ↓
   DeepSeek API (structured JSON)
        ↓
   User Confirmation
        ↓
   POST to Garuda REST API
```

## Final Directory Structure

```
Garuda/
├── pyproject.toml
├── .env.example
├── .gitignore
└── src/
    └── mayura/
        ├── __init__.py
        ├── cli.py                    # Entry point, orchestrates the flow
        ├── config.py                 # Pydantic settings
        ├── constants.py              # Version, app name, color codes
        ├── exceptions.py             # Custom exception hierarchy
        ├── logging.py                # Log configuration
        ├── models.py                 # All Pydantic data models
        ├── api/
        │   ├── __init__.py
        │   └── client.py             # Garuda REST API client
        ├── parsers/
        │   ├── __init__.py
        │   └── pdf_parser.py         # LiteParse integration (local)
        ├── structuring/
        │   ├── __init__.py
        │   ├── deepseek_client.py    # DeepSeek API integration
        │   └── prompts.py            # Prompt templates
        └── ui/
            ├── __init__.py
            ├── theme.py              # Rich theme/console
            ├── header.py             # Banner rendering
            ├── menu.py               # Menu loop
            ├── prompts.py            # Input collection
            └── display.py            # Output formatting helpers
```

## Phase Dependencies

```
Phase 0 (Scaffolding)
  ├── Phase 1 (Terminal UI)     ← can run in parallel with Phase 2
  ├── Phase 2 (PDF Parsing)
  │     └── Phase 3 (AI Structuring)
  │           └── Phase 4 (API Integration) ← needs Phase 1 + Phase 3
  └── Phase 5 (Polish) ← needs all above
```

---

## Phase 0: Project Scaffolding

**Goal:** A properly structured Python project with dependency management, config handling, and a runnable (but empty) CLI entry point.

### Tasks

1. Create `pyproject.toml` using a modern build system (Hatchling or setuptools). Define the package as `mayura` with a console script entry point: `mayura = "mayura.cli:main"`

2. Create the directory structure under `src/mayura/`

3. `config.py` — Use `pydantic-settings` (BaseSettings) to manage configuration:
   ```python
   class MayuraSettings(BaseSettings):
       model_config = SettingsConfigDict(env_file=".env", env_prefix="MAYURA_")

       deepseek_api_key: str
       deepseek_model: str = "deepseek-chat"
       api_base_url: str = "http://localhost:8000"
       api_internal_key: str = ""
       api_timeout: int = 30
   ```

4. `cli.py` — Minimal `main()` that prints "Mayura v0.1.0" and exits

5. `.env.example` with all required keys documented

### Dependencies (pyproject.toml)

- `pydantic-settings` — config management
- `rich` — terminal UI (Phase 1)
- `httpx` — async HTTP client (Phase 4)
- `liteparse` — local PDF parsing (Phase 2), requires Node.js >= 18
- `openai` — DeepSeek API (Phase 3)

### Files to Create

- `pyproject.toml`
- `.env.example`
- `.gitignore`
- `src/mayura/__init__.py`
- `src/mayura/cli.py`
- `src/mayura/config.py`
- `src/mayura/constants.py`

### Done When

`pip install -e .` succeeds, then running `mayura` in the terminal prints the version string and exits cleanly. A `.env` file with dummy keys loads without error.

---

## Phase 1: Terminal UI

**Goal:** A styled, interactive terminal interface that collects user inputs and displays a Bloomberg-style header/menu. No data processing yet — just the shell.

### Tasks

1. Create `src/mayura/ui/` package:
   - `theme.py` — Rich theme with Bloomberg-style colors (amber/orange on black, green for success, red for errors). Custom `Console` instance.
   - `header.py` — Render the Mayura banner using `rich.panel.Panel` with ASCII art or styled text, tagline, and current date/time.
   - `menu.py` — Numbered menu: `[1] Quarterly Report Data Extractor`, `[2] Exit`. Use `rich.prompt.IntPrompt`.
   - `prompts.py` — Input collection flow:
     - **Company ticker** — string, non-empty
     - **Financial quarter** — choice of Q1/Q2/Q3/Q4
     - **Financial year** — integer, validated range (2015-current year)
     - **PDF file path** — string, validated that file exists and has `.pdf` extension

2. Create `src/mayura/models.py` with Pydantic model:
   ```python
   class ExtractionRequest(BaseModel):
       ticker: str
       quarter: Literal["Q1", "Q2", "Q3", "Q4"]
       year: int
       pdf_path: Path

       @field_validator("pdf_path")
       def validate_pdf_exists(cls, v):
           if not v.exists():
               raise ValueError(f"File not found: {v}")
           if v.suffix.lower() != ".pdf":
               raise ValueError("File must be a PDF")
           return v
   ```

3. Wire UI into `cli.py`: show header → menu loop → input prompts → confirmation panel summarizing inputs

### Files to Create

- `src/mayura/ui/__init__.py`
- `src/mayura/ui/theme.py`
- `src/mayura/ui/header.py`
- `src/mayura/ui/menu.py`
- `src/mayura/ui/prompts.py`
- `src/mayura/models.py`
- Modify: `src/mayura/cli.py`

### Done When

Running `mayura` shows a styled terminal header, presents a menu, collects inputs with validation (rejects missing files, bad quarters), and displays a confirmation panel summarizing what was entered.

---

## Phase 2: PDF Parsing (LiteParse)

**Goal:** Given a PDF file path, extract its text content locally using LiteParse and return it as a string.

### Prerequisites

- Node.js >= 18 must be installed (LiteParse runs on Node.js under the hood)
- Install: `pip install liteparse`

### Tasks

1. Create `src/mayura/parsers/pdf_parser.py`:
   ```python
   from liteparse import LiteParse

   class PdfParser:
       def __init__(self):
           self.parser = LiteParse()

       def parse(self, pdf_path: Path) -> str:
           result = self.parser.parse(str(pdf_path))
           return result.text
   ```
   - Runs entirely locally — no API keys, no cloud uploads, no network needed
   - Preserves spatial layout of text (table columns stay aligned)
   - Returns full text from all pages
   - Raises `ParseError` on failure

2. Create `src/mayura/exceptions.py`:
   ```python
   class MayuraError(Exception): ...
   class ParseError(MayuraError): ...
   class StructuringError(MayuraError): ...
   class ApiError(MayuraError): ...
   ```

3. CLI integration: show `rich.status.Status` spinner ("Parsing PDF..."), display preview of first 500 chars of extracted text

4. Debug mode: `mayura parse <path-to-pdf>` — standalone parser test command

### Key Design Decisions

**Why LiteParse over LlamaParse:**
- **Free and local** — no API key, no cloud dependency, no per-page cost
- **Privacy** — PDFs never leave the machine
- **Speed** — no network latency, parses locally in seconds
- **Offline** — works without internet

**Trade-off:** LiteParse outputs raw text with spatial layout preservation rather than clean markdown tables. However, since the text goes to DeepSeek for AI structuring anyway, the LLM can interpret the spatial layout. The DeepSeek prompt is designed to handle both formats.

### Files to Create

- `src/mayura/parsers/__init__.py`
- `src/mayura/parsers/pdf_parser.py`
- `src/mayura/exceptions.py`
- Modify: `src/mayura/cli.py`

### Done When

Running `mayura` with a real CSE quarterly report PDF parses it locally via LiteParse, shows a spinner during processing, and prints the extracted text. Errors (corrupt PDF, Node.js not installed) are caught and displayed as styled error panels.

---

## Phase 3: AI Structuring (DeepSeek)

**Goal:** Take the raw text from LiteParse and use DeepSeek to extract structured financial fields into validated Pydantic models.

### Tasks

1. Extend `src/mayura/models.py` with financial data models:
   ```python
   class IncomeStatement(BaseModel):
       revenue: Decimal | None = None
       cost_of_sales: Decimal | None = None
       gross_profit: Decimal | None = None
       operating_profit: Decimal | None = None
       finance_income: Decimal | None = None
       finance_costs: Decimal | None = None
       profit_before_tax: Decimal | None = None
       income_tax_expense: Decimal | None = None
       net_profit: Decimal | None = None

   class BalanceSheet(BaseModel):
       total_assets: Decimal | None = None
       total_non_current_assets: Decimal | None = None
       total_current_assets: Decimal | None = None
       total_equity: Decimal | None = None
       total_liabilities: Decimal | None = None
       total_non_current_liabilities: Decimal | None = None
       total_current_liabilities: Decimal | None = None

   class PerShareData(BaseModel):
       eps_basic: Decimal | None = None
       eps_diluted: Decimal | None = None
       nav_per_share: Decimal | None = None

   class QuarterlyReport(BaseModel):
       ticker: str
       quarter: str
       year: int
       currency: str = "LKR"
       units: str = "thousands"
       income_statement: IncomeStatement
       balance_sheet: BalanceSheet
       per_share: PerShareData
   ```

   All numeric fields are `Decimal | None` — None because some reports may omit certain fields. Use `Decimal` not `float` for financial precision.

2. Create `src/mayura/structuring/deepseek_client.py`:
   ```python
   class FinancialStructurer:
       def __init__(self, settings: MayuraSettings): ...
       async def structure(self, raw_text: str, ticker: str, quarter: str, year: int) -> QuarterlyReport: ...
   ```
   - Uses `openai` library pointed at `https://api.deepseek.com`
   - Requests `response_format={"type": "json_object"}`
   - Parses JSON response through `QuarterlyReport` Pydantic model
   - Retries once on validation failure with a corrective prompt

3. Create `src/mayura/structuring/prompts.py` — The system prompt is the most critical piece:
   ```
   You are a financial data extraction specialist for the Colombo Stock Exchange (CSE).

   CRITICAL RULES:
   - Extract data for the CURRENT QUARTER only (not cumulative/year-to-date)
   - Identify the unit scale (thousands, millions, or ones) from report headers
   - All monetary values must be numbers without commas or currency symbols
   - If a value is not present in the report, use null
   - For negative values (losses, expenses), use negative numbers
   - EPS values are typically in rupees (not thousands), extract as-is

   Return ONLY valid JSON matching this schema:
   {schema}
   ```

4. Display structured data as a `rich.table.Table` for user review before submission

### Prompt Design Notes

- **Current quarter vs. cumulative:** CSE reports show both columns side by side. The prompt must specify which to extract. This is the most common error.
- **Units detection:** Reports state units near table headers (e.g., "Rs. '000"). Getting this wrong means every number is off by 1000x.
- **Sign conventions:** Some reports use parentheses for negatives, others use minus signs. Normalize to negative numbers.
- **Expect 3-5 rounds of prompt refinement** against real PDFs per company format variation.

### Files to Create

- `src/mayura/structuring/__init__.py`
- `src/mayura/structuring/deepseek_client.py`
- `src/mayura/structuring/prompts.py`
- Modify: `src/mayura/models.py`, `src/mayura/cli.py`

### Done When

Full flow works: PDF → parse → structure → formatted Rich table shown in terminal. User can visually verify numbers match the PDF. Handles DeepSeek API errors and validation failures gracefully.

---

## Phase 4: API Integration

**Goal:** After the user confirms the structured data looks correct, submit it to the Garuda REST API.

### Tasks

1. Create `src/mayura/api/client.py`:
   ```python
   class GarudaApiClient:
       def __init__(self, settings: MayuraSettings): ...
       async def submit_quarterly_report(self, report: QuarterlyReport) -> dict: ...
       async def ping(self) -> bool: ...
   ```
   - Uses `httpx.AsyncClient` with base URL and internal API key from settings
   - POSTs to `/api/v1/internal/reports`
   - Handles HTTP errors (4xx, 5xx) mapped to `ApiError`
   - Health check via `GET /health`

2. Confirmation step: after displaying structured data table, prompt `"Submit this data? [y/N]"`

3. `--dry-run` flag: runs entire pipeline, dumps JSON to stdout, skips API call

4. Display results: green success panel on success (with record ID), red error panel on failure

### Files to Create

- `src/mayura/api/__init__.py`
- `src/mayura/api/client.py`
- Modify: `src/mayura/cli.py`

### Done When

End-to-end flow completes: PDF → parse → structure → confirm → submit. `--dry-run` outputs JSON without needing System 2 running. API connection failures handled gracefully.

---

## Phase 5: Polish and Error Handling

**Goal:** Production-quality error handling, logging, and UI polish for a complete v1.0.

### Tasks

1. Create `src/mayura/logging.py`:
   - Python `logging` writing to `~/.mayura/logs/mayura.log`
   - `RotatingFileHandler` (5MB, 3 backups)
   - Log every pipeline step with timestamps
   - Full stack traces for exceptions

2. Create `src/mayura/ui/display.py`:
   - `display_extraction_summary(report)` — Rich table with formatted numbers, color-coded profit/loss
   - `display_error(error)` — Styled error panel
   - `display_success(message)` — Styled success panel

3. Top-level error handling in `cli.py`:
   - Catch `MayuraError` subclasses → user-friendly error panels
   - Catch unexpected exceptions → log traceback, show generic message
   - Never show raw Python tracebacks (unless `--verbose`)

4. Add `--verbose` flag (debug output to terminal)
5. Add `--version` flag
6. Handle `Ctrl+C` gracefully (catch `KeyboardInterrupt`, print "Aborted.", exit cleanly)

### Files to Create

- `src/mayura/logging.py`
- `src/mayura/ui/display.py`
- Modify: `src/mayura/cli.py`

### Done When

All error scenarios handled gracefully. Operations logged to file. No raw tracebacks reach the user. CLI feels polished and professional.
