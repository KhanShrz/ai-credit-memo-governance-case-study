# Workflow logic (Foundry workflow, tested version 9)

> The exported workflow YAML will be added as `workflow.yaml`. This file documents the same logic in plain form.

## Node order

```
Start
→ Set variable: Local.SendBackCount = 0
→ Intake-Agent
→ Data-Retrieval-Agent (output saved to Local.Fin)
→ Send message: CALCULATOR TIE-OUT
→ Send message: GATE 1 REVIEW CHECKLIST
→ Ask a question: Gate 1 decision (saved to Local.Gate1Decision, always ask)
→ If / else:
    If       APPROVE and no figure missing → Send "Gate 1 APPROVED" → Send RATIO CALCULATOR
    Else if  Local.SendBackCount >= 2       → Send "ESCALATED ..."
    Else                                    → SendBackCount + 1 → Go to Data-Retrieval-Agent
```

## Gate 1 conditions (Power Fx)

Approve path:
```
StartsWith(Upper(Local.Gate1Decision), "APPROVE") && CountRows(Local.Fin.not_found) = 0
```

Escalation path (after two send-backs):
```
Local.SendBackCount >= 2
```

Send-back path: `Local.SendBackCount = Local.SendBackCount + 1`, then Go to Data-Retrieval-Agent.

## Gate 1 question and checklist

Question: `Gate 1 decision: reply APPROVE, or SEND BACK followed by the reason.`

```
GATE 1 REVIEW CHECKLIST
1. Borrower in DOCUMENT CHECK matches the intake request
2. Fiscal year is the most recent filing
3. No items marked NOT FOUND
4. CALCULATOR TIE-OUT difference is 0 for BOTH years
5. Spot-check 2 figures against the cited page in the 10-K
```

Escalation message:
```
ESCALATED: data failed Gate 1 three times. Routed to senior credit officer. Last reason: {Local.Gate1Decision}
```

## Calculator tie-out (runs before Gate 1)

```
CALCULATOR TIE-OUT (formula, not AI)
FY2026: {If(Local.Fin.gross_margin.cy = 0 || Local.Fin.operating_income.cy = 0, "INPUT MISSING - FAIL", If(Local.Fin.gross_margin.cy - Local.Fin.research_and_development.cy - Local.Fin.sales_and_marketing.cy - Local.Fin.general_and_administrative.cy - Local.Fin.operating_income.cy = 0, "PASS", "FAIL"))}
FY2025: {If(Local.Fin.gross_margin.py = 0 || Local.Fin.operating_income.py = 0, "INPUT MISSING - FAIL", If(Local.Fin.gross_margin.py - Local.Fin.research_and_development.py - Local.Fin.sales_and_marketing.py - Local.Fin.general_and_administrative.py - Local.Fin.operating_income.py = 0, "PASS", "FAIL"))}
```

Note: the tie-out checks that the figures are consistent, not that each figure is the right line item.

## Ratio calculator (runs only after approval)

```
RATIO CALCULATOR (formula, not AI)
Interest coverage: FY2026 {If(Local.Fin.interest_expense.cy = 0, "INPUT MISSING", Text(Local.Fin.operating_income.cy / Local.Fin.interest_expense.cy, "0.00") & "x")} | FY2025 {If(Local.Fin.interest_expense.py = 0, "INPUT MISSING", Text(Local.Fin.operating_income.py / Local.Fin.interest_expense.py, "0.00") & "x")}
Current ratio: FY2026 {Text(Local.Fin.total_current_assets.cy / Local.Fin.total_current_liabilities.cy, "0.00")} | FY2025 {Text(Local.Fin.total_current_assets.py / Local.Fin.total_current_liabilities.py, "0.00")}
Debt-to-equity: FY2026 {Text((Local.Fin.current_portion_ltd.cy + Local.Fin.long_term_debt.cy) / Local.Fin.stockholders_equity.cy, "0.00")} | FY2025 {Text((Local.Fin.current_portion_ltd.py + Local.Fin.long_term_debt.py) / Local.Fin.stockholders_equity.py, "0.00")}
Total debt / EBITDA: FY2026 {Text((Local.Fin.current_portion_ltd.cy + Local.Fin.long_term_debt.cy) / (Local.Fin.operating_income.cy + Local.Fin.depreciation_amortization.cy), "0.00")}x | FY2025 {Text((Local.Fin.current_portion_ltd.py + Local.Fin.long_term_debt.py) / (Local.Fin.operating_income.py + Local.Fin.depreciation_amortization.py), "0.00")}x
Net margin: FY2026 {Text(Local.Fin.net_income.cy / Local.Fin.total_revenue.cy, "0.0%")} | FY2025 {Text(Local.Fin.net_income.py / Local.Fin.total_revenue.py, "0.0%")}
```

## Definitions used

- **Total debt** = current portion of long-term debt + long-term debt (leases excluded)
- **EBITDA** = operating income + "depreciation, amortization, and other" from the cash flow statement
- These definitions are documented here but not yet formally approved (see case study, section 8).
