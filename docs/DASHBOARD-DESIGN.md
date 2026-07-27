# Contract Management Dashboard — Design Document

**Project**: powerbi-contract-uaepl
**Source data**: `contracts_data_collection_template.xlsx` (3 sheets: Agreements, Payment Schedule, Deliverables Rights)
**Build scope (Iteration 1)**: 3 pages — Portfolio Overview, Contract Lifecycle, Payment & Cashflow.

## Semantic Model

```
                          ┌───────────────┐
                          │      Date      │  Generated (2025-01-01 → 2030-12-31)
                          │ (dimension)    │  Marked as date table
                          └───────┬───────┘
                                  │
                ┌─────────────────┼─────────────────┐
                │ (active)        │ (inactive)       │ (active after USERELATIONSHIP)
                ▼                 ▼                  ▼
   'Payment Schedule'      Agreements         'Deliverables Rights'
   [Due_Date]              [Start_Date]       [Due_Date] (optional)
                ▲                 ▲
                │  1:M           │  1:M
                │                │
                └─── Agreements[Agreement_ID] ────┘
                          (hub table)
```

Relationships:
- `Agreements[Agreement_ID]` 1:M `'Payment Schedule'[Agreement_ID]` (active, single-direction)
- `Agreements[Agreement_ID]` 1:M `'Deliverables Rights'[Agreement_ID]` (active, single-direction)
- `'Date'[Date]` 1:M `'Payment Schedule'[Due_Date]` (active)
- `'Date'[Date]` 1:M `Agreements[Start_Date]` (inactive — used with USERELATIONSHIP for time intelligence)
- `'Date'[Date]` 1:M `'Deliverables Rights'[Due_Date]` (inactive)

## Visuals per Page

> Field syntax: `Table[Column]` for columns, `[Measure Name]` for measures (server picks them up automatically).

### Page 1 — Portfolio Overview

| # | Visual type | Position (x, y, w, h) | Fields / purpose |
|---|-------------|------------------------|------------------|
| 1 | card | (20, 20, 290, 120) | Values: `[Total Contract Value]` — Headline portfolio value |
| 2 | card | (320, 20, 290, 120) | Values: `[Active Contracts]` — Active count |
| 3 | card | (620, 20, 290, 120) | Values: `[Overdue Amount]` — Late payments (red) |
| 4 | card | (920, 20, 290, 120) | Values: `[Completion Rate]` — Deliverables completion % |
| 5 | barChart | (20, 160, 590, 320) | Category: `Agreements[Agreement_Type]`, Y: `[Total Contract Value]` — Value by type |
| 6 | donutChart | (620, 160, 590, 320) | Category: `Agreements[Status]`, Y: `[Total Agreements]` — Status mix |
| 7 | tableEx | (20, 500, 1190, 220) | Values: `Agreements[Counterparty_Name]`, `[Total Contract Value]`, `[Active Contracts]`, `Agreements[Status]`, `Agreements[End_Date]` — Top counterparties table |

Slicers (across top or left):
- `Agreements[Status]`, `Agreements[Agreement_Type]`, `Agreements[Department_Owner]`, `'Payment Schedule'[Season]`

### Page 2 — Contract Lifecycle

| # | Visual type | Position (x, y, w, h) | Fields / purpose |
|---|-------------|------------------------|------------------|
| 1 | card | (20, 20, 290, 120) | Values: `[Contracts Expiring 90d]` — Within 90 days |
| 2 | card | (320, 20, 290, 120) | Values: `[Contracts Expiring 180d]` — Within 180 days |
| 3 | card | (620, 20, 290, 120) | Values: `[Contracts Expiring 365d]` — Within 1 year |
| 4 | card | (920, 20, 290, 120) | Values: `[Expired Still Active Flag]` — Data quality flag |
| 5 | barChart | (20, 160, 590, 320) | Category: `Agreements[Department_Owner]`, Y: `[Total Agreements]` — Contracts by dept |
| 6 | columnChart | (620, 160, 590, 320) | Category: `Agreements[Agreement_Type]`, Y: `[Avg Term Months]` — Avg term by type |
| 7 | tableEx | (20, 500, 1190, 220) | Values: `Agreements[Agreement_ID]`, `Agreements[Agreement_Title]`, `Agreements[Counterparty_Name]`, `Agreements[Start_Date]`, `Agreements[End_Date]`, `Agreements[Term_Months]`, `Agreements[Status]`, `Agreements[Renewal_Option]`, `Agreements[Notice_Period_Days]` — Lifecycle detail |

Slicers:
- `Agreements[Status]`, `Agreements[Agreement_Type]`, `Agreements[Department_Owner]`, `Agreements[Renewal_Option]`

### Page 3 — Payment & Cashflow

| # | Visual type | Position (x, y, w, h) | Fields / purpose |
|---|-------------|------------------------|------------------|
| 1 | card | (20, 20, 290, 120) | Values: `[Payment Collection Rate]` — % collected |
| 2 | card | (320, 20, 290, 120) | Values: `[Total Payment Amount]` — Total scheduled |
| 3 | card | (620, 20, 290, 120) | Values: `[Outstanding Amount]` — Yet to collect |
| 4 | card | (920, 20, 290, 120) | Values: `[Upcoming Payments 30d]` — Next 30 days |
| 5 | barChart | (20, 160, 590, 320) | Category: `'Payment Schedule'[Status]`, Y: `[Total Payment Amount]` — Cash by status |
| 6 | lineChart | (620, 160, 590, 320) | Category: `'Date'[Date]` (month), Y: `[Total Payment Amount]` — Cashflow forecast |
| 7 | tableEx | (20, 500, 1190, 220) | Values: `'Payment Schedule'[Payment_ID]`, `'Payment Schedule'[Agreement_ID]`, `'Payment Schedule'[Milestone]`, `'Payment Schedule'[Due_Date]`, `'Payment Schedule'[Amount_AED]`, `'Payment Schedule'[Days_To_Due]`, `'Payment Schedule'[Status]` — Upcoming & overdue detail |

Slicers:
- `'Payment Schedule'[Status]`, `'Payment Schedule'[Season]`, `'Payment Schedule'[Currency]`, `'Payment Schedule'[Payment_Type]`

## Description and Data-Category Wiring

All measures get `description` (already in the catalog) and `formatString` for AI-readiness (Copilot / Q&A). The Date table's primary key gets `dataCategory: Time` and `isKey`, which the `pbip_create_date_table` tool handles automatically.

## Visual Theme

Default Power BI theme for iteration 1. A custom corporate theme (UAEPL colors) can be added via `pbir_set_report_theme` in iteration 2.

## Naming Conventions Applied

- **Tables**: PascalCase with spaces where natural (`Agreements`, `Payment Schedule`, `Deliverables Rights`, `Date`).
- **Measures**: Title Case (e.g., `Total Contract Value`, `YoY Growth %`).
- **Display folders**: `Base`, `Counts`, `Lifecycle`, `Counterparties`, `Time Intelligence`, `Payments`, `Deliverables`.

## Out-of-Scope for Iteration 1 (deferred)

- Season Analysis page
- Deliverables & Obligations page (full)
- Counterparty & Relationship page
- Compliance & Governance page
- Cross-filtering between pages (bookmarks / sync-slicers)
- Mobile layout
- Custom UAEPL theme
- RLS roles

These are defined in the original plan and can be added in iteration 2 by reusing the same `pbir_add_page` + `pbir_add_visual` workflow.