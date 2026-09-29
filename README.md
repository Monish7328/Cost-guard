# CostGuard — Cloud Infrastructure Cost Impact Predictor & Budget Guardrail

> **“The Financial Firewall Before Cloud Deployment”**

CostGuard inspects planned Azure infrastructure modifications before deployment—either directly from **Terraform Plan JSON** in CI/CD pipelines via CLI or through an intuitive **Enterprise Infrastructure Request Portal**—queries live Azure Retail Prices backed by an embedded SQLite write-through cache, calculates 730-hour monthly cost impacts, and enforces company budget guardrails to either **APPROVE** or **BLOCK** changes.

---

## 🏛️ System Architecture

```text
               COMPANY USER / DEVOPS ENGINEER
                             │
          ┌──────────────────┴──────────────────┐
          ▼                                     ▼
  [CostGuard Web Dashboard]             [CostGuard CLI Engine]
  (React + Tailwind + Lucide)           (Piped stdin / --plan)
          │                                     │
          ▼                                     ▼
  [FastAPI Backend REST API]                    │
          │                                     │
          └──────────────────┬──────────────────┘
                             ▼
                 [CostGuard Core Engine]
                             │
                ┌────────────┴────────────┐
                ▼                         ▼
      [Terraform Plan Parser]    [Resource Extractors]
      (Extracts change actions:   - Virtual Machines
       create, update, delete)    - Managed Disks
                                  - Gateways
                                  - Extensible Registry
                             │
                             ▼
                 [Tiered Pricing Engine]
                             │
                ┌────────────┴────────────┐
                ▼                         ▼
      [Embedded SQLite Cache]   [Live Azure Retail Prices API]
      (Write-through cache)     (Public endpoint, Consumption,
                                 non-spot, no API key needed)
                             │
                             ▼
                 [Monthly Cost Calculator]
                 (Hourly rate × 730 hrs + Monthly meters)
                             │
                             ▼
                 [Budget Guardrail Policy]
                 (Delta vs Max Allowed Increase Limit)
                             │
                ┌────────────┴────────────┐
                ▼                         ▼
         [ ✅ APPROVED ]           [ ❌ BLOCKED ]
          (Exit Code 0)             (Exit Code 1)
```

---

## 🚀 Key Features

1. **Unified Core Engine**: The CLI and Web dashboard share the exact same Python core engine (`costguard.core`), guaranteeing 100% calculation and policy parity.
2. **Real Azure Retail Pricing**: Live queries against Azure Retail Prices API (`https://prices.azure.com/api/retail/prices`) with strict filtering (Consumption, non-spot, non-low-priority). No fake or invented prices.
3. **Write-Through SQLite Cache**: Transparent caching in `costguard.db` to prevent redundant network calls, accelerate repeated evaluations, and provide resilient offline cache reads.
4. **Standard 730-Hour Cost Formula**: Accurate billing calculation (`Hourly Rate × 730` or direct monthly block pricing) supporting resource creation, deletion, resizing (updates), and zero-cost tag modifications.
5. **Budget Guardrail & Exit Codes**:
   - `0`: Approved / within allowed budget increase limit.
   - `1`: Blocked / budget breached.
   - `2`: Invalid input / processing error.
6. **Extensible Resource Architecture**: Built on a modular registry pattern (`BaseResourceExtractor`), allowing new Azure resources (SQL Database, App Service, AKS) to be plugged in effortlessly.
7. **Enterprise Web Portal**:
   - **Executive Dashboard**: Cloud spend progress, headroom, guardrail status, and utilization bars.
   - **Infrastructure Request Builder**: Multi-category resource picker (Compute, Storage, Networking) with SKU discovery.
   - **Terraform Plan Analyzer**: Interactive plan upload or copy-paste with simulated CLI exit codes.
   - **Audit Trail & Request History**: Complete searchable history of all evaluated requests and decisions.
   - **Budget Guardrail Settings**: Custom enterprise budget and maximum allowed increase thresholds.

---

## 💻 CLI Usage

### Option 1: Piped via Stdin (CI/CD Pipeline)
```bash
# Example in bash / CI/CD
terraform show -json tfplan | python -m costguard.cli --max-increase 1000

# PowerShell equivalent
Get-Content examples/tfplan_approved.json | python -m costguard.cli --max-increase 1000
```

### Option 2: Via File Argument
```bash
python -m costguard.cli --plan examples/tfplan_approved.json --max-increase 1000
```

### Output: Professional Terminal ASCII Report
```text
============================================================
                   COSTGUARD ANALYSIS
============================================================
Current Monthly Cost       $        0.00
Projected Monthly Cost     $      332.88
Monthly Increase           $      332.88
Company Limit              $    1,000.00
------------------------------------------------------------

STATUS:
             [APPROVED]

Reason:
Projected monthly increase of $332.88 is within the company's allowed budget limit of $1,000.00.

------------------------------------------------------------
RESOURCE BREAKDOWN

* Compute: azurerm_linux_virtual_machine
  SKU: Standard_D2s_v5
  Region: East Us
  Action: CREATE | Quantity: 1
  Unit Rate: $0.0960/1 Hour ($70.08/mo) [CACHE]
  Monthly Cost: $70.08 (Delta: $+70.08)

* Compute: azurerm_linux_virtual_machine
  SKU: Standard_D2s_v5
  Region: East Us
  Action: CREATE | Quantity: 1
  Unit Rate: $0.0960/1 Hour ($70.08/mo) [CACHE]
  Monthly Cost: $70.08 (Delta: $+70.08)

* Storage: azurerm_managed_disk
  SKU: Premium LRS (128 GB)
  Region: East Us
  Action: CREATE | Quantity: 1
  Unit Rate: $19.7100/1/Month ($19.71/mo) [CACHE]
  Monthly Cost: $19.71 (Delta: $+19.71)

* Storage: azurerm_managed_disk
  SKU: Premium LRS (128 GB)
  Region: East Us
  Action: CREATE | Quantity: 1
  Unit Rate: $19.7100/1/Month ($19.71/mo) [CACHE]
  Monthly Cost: $19.71 (Delta: $+19.71)

* Networking: azurerm_virtual_network_gateway
  SKU: VpnGw1
  Region: East Us
  Action: CREATE | Quantity: 1
  Unit Rate: $0.2100/1 Hour ($153.30/mo) [CACHE]
  Monthly Cost: $153.30 (Delta: $+153.30)

============================================================
```

### JSON Output
```bash
python -m costguard.cli --plan examples/tfplan_approved.json --max-increase 1000 --json
```

---

## 🌐 Running the Web Application

### 1. Start the Backend REST API
```bash
cd backend
python -m uvicorn api.app:app --host 127.0.0.1 --port 8000 --reload
```
- API Base: `http://localhost:8000`
- Interactive Swagger API Docs: `http://localhost:8000/docs`

### 2. Start the Frontend Dashboard
```bash
cd frontend
npm run dev
```
- Web Application: `http://localhost:5173`

---

## 🧪 Automated Test Suite

Run the full pytest suite covering parser, extractors, live pricing, write-through cache, calculator, guardrail policy, and CLI exit codes:

```bash
cd backend
python -m pytest tests -v
```

All 13 tests execute and pass in under 3 seconds!

---

## 📁 Project Structure

```text
costguard/
├── backend/
│   ├── costguard/
│   │   ├── __init__.py
│   │   ├── __main__.py            # python -m costguard entrypoint
│   │   ├── cli.py                 # Standalone CLI tool
│   │   ├── models.py              # Pydantic schemas & domain models
│   │   ├── parser.py              # Terraform Plan JSON parser & plan generator
│   │   ├── calculator.py          # 730-hour monthly cost calculator
│   │   ├── guardrail.py           # Budget guardrail policy engine
│   │   ├── catalog.py             # Curated SKU discovery catalog
│   │   ├── db.py                  # SQLite persistence & audit repository
│   │   ├── service.py             # Unified facade service
│   │   ├── extractors/
│   │   │   ├── base.py            # BaseResourceExtractor interface
│   │   │   ├── registry.py        # ExtractorRegistry
│   │   │   ├── vm.py              # Azure VM extractor
│   │   │   ├── disk.py            # Azure Managed Disk extractor
│   │   │   └── gateway.py         # Azure Gateway extractor
│   │   ├── pricing/
│   │   │   ├── azure_client.py    # Live Azure Retail Prices API client
│   │   │   └── region_mapper.py   # Region normalization map
│   │   └── cache/
│   │       └── sqlite_cache.py    # Write-through SQLite cache
│   ├── api/
│   │   ├── app.py                 # FastAPI application
│   │   └── routes.py              # REST API endpoints
│   └── tests/
│       ├── test_parser.py
│       ├── test_extractors.py
│       ├── test_cache.py
│       ├── test_calculator.py
│       ├── test_guardrail.py
│       └── test_cli.py
├── frontend/
│   ├── src/
│   │   ├── App.tsx
│   │   ├── components/
│   │   │   ├── Navbar.tsx
│   │   │   ├── DecisionBanner.tsx
│   │   │   ├── BreakdownTable.tsx
│   │   │   └── MetricCard.tsx
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx
│   │   │   ├── NewRequest.tsx
│   │   │   ├── TerraformAnalyzer.tsx
│   │   │   ├── RequestHistory.tsx
│   │   │   └── Settings.tsx
│   │   └── services/
│   │       └── api.ts
│   └── package.json
├── examples/
│   ├── tfplan_approved.json       # Realistic approved plan ($332/mo)
│   ├── tfplan_blocked.json        # Realistic over-budget plan ($1,749/mo)
│   └── tfplan_update.json         # VM resize update plan
└── README.md
```
