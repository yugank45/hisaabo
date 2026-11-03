# TDS (Tax Deducted at Source) Architecture & Payment Workflow

## Overview

This Phase 1 workflow covers customer-deducted TDS on sales receipts (receivable-side TDS). The shared registry contains current numeric tax codes and legacy section identifiers for historical review; it does not establish the applicable rate or legal eligibility for an individual transaction.

When a customer pays an invoice subject to TDS:
1. The customer remits the **net cash amount** to the business seller.
2. The customer retains the **TDS deducted amount** to deposit with the Central Government (income tax department) in the business's PAN/TAN.
3. The customer issues Form 16A (TDS certificate) to the business.
4. The business records the gross settlement of the invoice: the net cash received plus the TDS receivable asset, fully discharging the customer's invoice debt.

---

## 1. Statutory Tax Code Registry & Auto-Seeding

Hisaabo maintains a standard statutory tax code registry defined in `@hisaabo/shared` (`TDS_TAX_CODES`).

### Codes and configuration:
- Current registry entries use numeric codes such as **1023**, **1024**, **1026**, and **1027**. Use the tenant's active list (`tdsSection.list`) to choose a configured code.
- Codes such as **194C** and **194J** are marked legacy and require review; they are not automatically mapped to current codes.
- The current registry source is [Razorpay's tax-code reference](https://razorpay.com/docs/payroll/tax-verifications/tax-codes). Registry labels and effective dates do not determine a transaction's applicable rate.

### Tenant Provisioning & Lazy Seeding:
- When a new tenant is created via `business.create`, all default statutory sections are auto-seeded into `tds_section_configs` with `active: true`.
- For existing tenants or test fixtures, `tdsSection.list` and `payment.create` employ lazy auto-seeding (`seedTdsSections`) with idempotency (`onConflictDoNothing`), guaranteeing that every tenant has active TDS sections ready in the UI dropdown without manual SQL migrations.

---

## 2. Payment Allocation Model: Gross Settlement vs Net Cash Received

The payment flow distinguishes between:
- **Gross Settlement (`allocations[i].amount`)**: The total amount credited against the invoice debt.
- **TDS Deducted (`allocations[i].tdsAmount`)**: The tax withheld by the customer under a specific section (`tdsSectionCode`).
- **Net Cash Received (`payment.amount`)**: The actual liquid funds received into the business's bank or cash drawer.

### Mathematical Relationship:
$$\text{Gross Settlement} = \text{Net Cash} + \text{TDS Deducted}$$
$$\text{Net Cash} = \text{Gross Settlement} - \text{TDS Deducted}$$

### Example:
- **Invoice Balance Due**: ₹10,000.00
- **Example deduction amount**: ₹1,000.00 (illustrative only; the application does not determine this rate)
- **Net Cash Received**: ₹9,000.00

**Payload Submitted**:
- `payment.amount`: `"9000.00"` (Net cash deposited)
- `allocations[0].amount`: `"10000.00"` (Gross debt settled)
- `allocations[0].tdsAmount`: `"1000.00"` (Tax withheld)
- `allocations[0].tdsSectionCode`: `"1027"` (example current registry code; tenant configuration must include it)
- `allocations[0].tdsRate`: `"10.0"` (stored client-supplied metadata, not independently validated against statutory rules)

**Invoice Balance**:
$$\text{New Amount Paid} = \text{Previous Amount Paid} + \text{Gross Settlement} = ₹10,000.00$$
$$\text{Remaining Balance} = ₹0.00 \implies \text{Status: "paid"}$$

---

## 3. Double-Entry General Ledger Semantics

A TDS payment records balanced double-entry accounting entries:

| Account | Type | Debit (₹) | Credit (₹) | Note |
|---|---|---|---|---|
| **Bank / Cash Account** | Asset | 9,000.00 | — | Net cash received |
| **TDS Receivable** | Asset | 1,000.00 | — | Tax credit available in Form 26AS / AIS |
| **Accounts Receivable (Party)** | Asset | — | 10,000.00 | Customer invoice balance fully cleared |

Total Debits (₹10,000.00) == Total Credits (₹10,000.00). The ledger is strictly balanced.

---

## 4. Customer Statement Accounting (`reports.partyStatement`)

In the customer statement of account:
- Invoices increase the customer's balance due (Debits).
- Payments credit the customer's balance due (Credits).
- Because the customer's liability was discharged by both the cash transfer and the statutory tax withholding on behalf of the seller, the total settlement amount credited to the customer is:
  $$\text{Credit} = \text{pmt.amount} + \text{pmt.tdsAmount} = 9,000.00 + 1,000.00 = ₹10,000.00$$
- The statement displays a clear transaction description:
  `Payment received (incl. TDS ₹1,000.00)`
- The customer closing balance correctly reconciles to **₹0.00**, eliminating false outstanding debts.

---

## 5. UI and UX Controls

1. **Party Context Gating**:
   - TDS controls only appear when receiving payments from **Customers**.
   - TDS inputs are hidden and cleared for **Suppliers** (where TDS deduction by us would be TDS Payable, handled during invoice booking).
2. **Multi-Invoice Guard**:
   - If multiple invoices are selected, TDS input is disabled with a notice requiring single-invoice allocation per TDS payment, preventing ambiguous deduction assignment.
3. **TAN Integration**:
   - Customer Tax Deduction and Collection Account Number (TAN) is recorded on the party profile (`PartyDetailSlideOver`, `AddPartyModal`, `QuickPartyCreate`).

---

## 6. Validation Rules & Invariants

All TDS payments are strictly validated against repository invariants:
1. `money.compare(tdsAmount, "0") > 0` requires `tdsSectionCode` to be present and `tdsRate > 0`.
2. `money.compare(tdsAmount, grossAmount) <= 0`: TDS amount cannot exceed the gross invoice settlement.
3. `money.compare(tdsAmount, "0") === 0`: TDS section code and rate must be omitted or undefined; even a zero rate is rejected as metadata without a deduction.
4. Active section validation: `tdsSectionCode` must exist, be active, and be effective on the payment date in tenant-scoped `tds_section_configs`.
5. `tdsAmount` must equal the gross allocation multiplied by `tdsRate`, rounded to paise; a one-paise tolerance permits normal decimal rounding.
6. All financial arithmetic uses the `@hisaabo/shared` `money` module with 2-decimal-place rounding.

The rate check uses the gross invoice allocation as its base. It does not determine the legally eligible base, thresholds, exemptions, surcharge, or the applicable statutory rate; do not treat the recorded rate as verified tax advice. Historical payments before a code's configured effective date need an explicitly configured historical section; the application does not guess a legacy mapping or rate.

## 7. Supplier-side TDS scope

Supplier/payable-side TDS is not implemented. TDS on customer receipts debits account **1600 – TDS Receivable**; a supplier deduction would instead create a TDS payable liability and requires separate supplier invoice deductions, settlement and remittance records, and reporting/reconciliation. Do not record supplier deductions through the customer-receipt TDS fields or account 1600.
