### List of Use Cases Implemented by the AP100, AP110, and AP115 Programs

The RPG programs `AP100`, `AP110`, and `AP115` form a call stack for processing Accounts Payable (A/P) voucher transactions on IBM midrange systems (e.g., AS/400 or iSeries). Together, they implement a single cohesive use case:

1. **Use Case: Process and Validate A/P Voucher Transactions**
   - **Description**: This use case involves the entry, validation, editing, and reporting of A/P voucher transactions, including header and detail records, for various payment types (e.g., prepaid checks, ACH, wire transfers, employee expenses, utility auto-payments). The process ensures data integrity, validates against vendor, company, G/L, and inventory data, and generates reports for errors and totals.
   - **Components**:
     - **AP100**: Handles interactive entry and validation of voucher transactions via screen formats (`AP100S1`, `AP100S2`, `AP100S3`, `AP100S5`), updating transaction files (`APTRAN`, `APCONT`) and calling `AP1011` for freight calculations.
     - **AP110**: Validates voucher transactions from `APTRAN`, checks for errors (e.g., duplicate invoices, invalid G/Ls), updates prepaid check records (`APCHKT`), and produces an edit report (`APLIST`).
     - **AP115**: Validates prepaid checks, ensuring non-void checks are not already open and void checks match the full amount, generating an error report (`APLIST`).

This single use case encompasses the entire workflow of entering, validating, and reporting A/P voucher transactions, with each program handling a specific aspect of the process.

---

### Function Requirement Document: Process and Validate A/P Voucher Transactions



# Function Requirement Document: Process and Validate A/P Voucher Transactions

## Overview
This function processes and validates Accounts Payable (A/P) voucher transactions, including header and detail records, for various payment types (prepaid checks, ACH, wire transfers, employee expenses, utility auto-payments). It ensures data integrity by validating inputs against company, vendor, G/L, and inventory data, calculates due dates and discounts, updates transaction and check files, and generates reports for errors and totals.

## Inputs
- **Company Number (`CONO`)**: 2-digit identifier for the company.
- **Vendor Number (`VEND`)**: 5-digit identifier for the vendor.
- **Entry Number (`ENT#`)**: 5-digit transaction identifier (auto-generated or user-provided).
- **Invoice Number (`INV#`)**: 20-character vendor invoice number.
- **Invoice Amount (`IAMT`)**: 11.2-digit total invoice amount.
- **Invoice Date (`INDT`)**: 6-digit vendor invoice date (MMDDYY).
- **Due Date (`DUDT`)**: 6-digit due date (MMDDYY, calculated or user-provided).
- **Discount Due Date (`DSDT`)**: 6-digit discount due date (MMDDYY, calculated or user-provided).
- **Hold Code (`HOLD`)**: 1-character code (`H`=hold, `A`=ACH, `W`=wire transfer, `E`=employee expense, `U`=utility auto-payment).
- **Prepay Code (`PAID`)**: 1-character code (`P`=prepaid, `A`=ACH, `E`=employee expense, `U`=utility auto-payment).
- **Prepaid Check Number (`PPCK`)**: 6-digit check number for prepaid transactions.
- **Single Check Flag (`SNGL`)**: 1-character flag (`S`=single check).
- **Canceled Voucher (`CNVO`)**: 5-digit canceled voucher number.
- **Purchase Order Number (`PONO`)**: 30-character purchase order number.
- **Sales Order Number (`SORN`)**: 6-digit sales order number.
- **Carrier ID (`CAID`)**: 6-character carrier identifier.
- **Process Type (`PTYP`)**: 6-character process type (`NORMAL`, `PAPER`, `ARGLMS`).
- **Freight Total (`FRTL`)**: 7.2-digit total freight amount to allocate.
- **Detail Lines**:
  - **Sequence Number (`ENSQ`)**: 3-digit line sequence.
  - **Expense G/L (`EXGL`)**: 8-digit expense general ledger account.
  - **Amount (`AMT`)**: 8.2-digit line amount.
  - **Discount Amount (`DISC`)**: 8.2-digit discount amount.
  - **Discount Percentage (`DSPC`)**: 3.2-digit discount percentage.
  - **Gallons (`GALN`)**: 4.2-digit gallons quantity.
  - **Receipt Number (`RCPT`)**: 7-digit receipt number.
  - **Receipt Code (`CLCD`)**: 1-character code (`O`=open, `C`=closed).
  - **Freight Amount (`FRAM`)**: 4.2-digit freight amount per line.
  - **Product Amount (`PRAM`)**: 6.2-digit product amount per line.

## Outputs
- **Updated Files**:
  - `APTRAN`: Transaction header and detail records.
  - `APCHKT`: Prepaid check records.
  - `APCONT`: Updated with next entry number.
  - `APSTAT`: Error status (`Y` for errors, `N` otherwise).
- **Report (`APLIST`)**: Printed report listing transaction details, errors, warnings, and totals (invoices, prepaid amounts, vendor hash).

## Process Steps
1. **Initialize**:
   - Retrieve system date and time.
   - Validate company number (`CONO`) against `APCONT`. If invalid or deleted, return error.
   - Initialize accumulators for invoice counts, amounts, discounts, and prepaid totals.
   - Set process type (`PTYP`) to `NORMAL` if blank.

2. **Validate Header**:
   - Ensure `CONO` exists in `APCONT`, not deleted (`ACDEL ≠ 'D'`).
   - Validate `VEND` against `APVEND`, ensuring not deleted (`VNDEL ≠ 'D'`), not inactive (`VNDEL ≠ 'I'`), and name not blank.
   - Verify `INV#` is non-blank and unique (check `APTRNX`, `APINVH`, `APOPNHC` for non-prepaid, non-canceled, non-wire-transfer transactions).
   - Ensure `INDT` is non-zero and not older than one year (warning only).
   - Ensure `IAMT` is non-zero and matches sum of detail amounts (`AMT` + `FRAM`).
   - Validate `HOLD` (`H`, `A`, `W`, `E`, `U`) and `PAID` (`P`, `A`, `E`, `U`).
   - If `PAID` is set, ensure `PPCK` is provided.
   - Calculate `DUDT` and `DSDT` using terms (`VNTERM`) from `GSTABL` (net days, discount days).
   - Validate `ATAPGL` and `ATBKGL` against `GLMAST`, ensuring not deleted or inactive.
   - If `SORN ≠ 0`, prohibit `RCPT` and `GALN`.
   - For freight invoices (`FRTL ≠ 0`), allocate amounts to detail lines (call `AP1011`).

3. **Validate Detail Lines**:
   - Validate `EXGL` against `GLMAST`, ensuring not deleted or inactive.
   - If `GLPOCD = 'Y'`, ensure `PONO` is non-blank.
   - Validate gallons/receipts:
     - If `VNGRRQ = 'Y'`, require `GALN` and `RCPT`.
     - If `VNGRRQ = 'N'`, prohibit `GALN` and `RCPT`.
     - If `GLAPCD = 'Y'` and `AMT > 0`, require `GALN`.
     - If `GALN > 0`, ensure `GLAPCD = 'Y'`.
     - If `RCPT ≠ 0`, validate against `INFIL1` or `INTZH1` for sufficient quantity (`GALN ≤ IHNQTY + IHNQTF - IHAPTQ - IHAPTF`) and no prior A/P postings.
     - Ensure `CLCD` is `'O'` or `'C'`.
   - Validate discounts:
     - Ensure `DISC` and `DSPC` are not both non-zero.
     - If `DISC` or `DSPC` is non-zero, require `ACDSGL ≠ 0`.
     - If `TBDISC = 0`, prohibit `DISC` and `DSPC`.
     - If `TBDISC ≠ 0`, ensure `DSPC = TBDISC` or both `DISC` and `DSPC` are non-zero.
     - If `DSDT ≠ 0` and `DSDT ≤ system date`, issue warning.
     - Calculate `DISC = AMT * (DSPC / 100)` if `DSPC ≠ 0`.
   - Calculate net amount: `NETAMT = AMT - DISC`.

4. **Validate Prepaid Checks**:
   - For `PAID` transactions, update `APCHKT` with check amount (`ACCKAM = AMT - DISC`).
   - Ensure non-void checks are not already open (`AMCODE ≠ 'O'`).
   - Ensure void checks are open (`AMCODE = 'O'`) and match full amount (`L1CKAM = AMCKAM`).

5. **Update Files**:
   - Write/update `APTRAN` with header and detail records.
   - Update `APCONT` with next entry number (`ACNXTE`).
   - Write/update `APCHKT` with prepaid check records.
   - Write `APSTAT` with error status (`Y` or `N`).

6. **Generate Report**:
   - Produce `APLIST` report with:
     - Headers: Company, date, time, workstation, process type.
     - Details: Entry, vendor, invoice number, description, amounts, G/L, status.
     - Errors/Warnings: List entry numbers with issues (e.g., invalid vendor, duplicate invoice).
     - Totals: Invoice count, amounts, discounts, prepaid totals, vendor hash.

## Business Rules
1. **Company and Vendor**:
   - `CONO` must exist in `APCONT`, not deleted.
   - `VEND` must exist in `APVEND`, not deleted or inactive, with non-blank name.

2. **Invoice**:
   - `INV#` must be non-blank and unique (unless prepaid, canceled, or wire transfer).
   - `IAMT` must be non-zero and match detail totals.
   - `INDT` must be non-zero; warn if older than one year.

3. **Payment Types**:
   - `HOLD`: `'H'`, `'A'`, `'W'`, `'E'`, `'U'`.
   - `PAID`: `'P'`, `'A'`, `'E'`, `'U'`, with `PPCK` required if set.
   - `SNGL = 'S'` for single check processing.

4. **Gallons and Receipts**:
   - If `VNGRRQ = 'Y'`, require `GALN` and `RCPT`.
   - If `VNGRRQ = 'N'`, prohibit `GALN` and `RCPT`.
   - If `GLAPCD = 'Y'` and `AMT > 0`, require `GALN`.
   - If `GALN > 0`, require `GLAPCD = 'Y'`.
   - `RCPT` must exist in `INFIL1` or `INTZH1` with sufficient quantity.
   - `CLCD` must be `'O'` or `'C'`.

5. **Purchase Orders**:
   - If `GLPOCD = 'Y'`, require `PONO`.

6. **Discounts**:
   - `DISC` and `DSPC` cannot both be non-zero.
   - Require `ACDSGL` if `DISC` or `DSPC` is non-zero.
   - If `TBDISC = 0`, prohibit `DISC` and `DSPC`.
   - If `TBDISC ≠ 0`, ensure `DSPC = TBDISC` or both `DISC` and `DSPC` are non-zero.
   - Warn if `DSDT ≤ system date`.

7. **Prepaid Checks**:
   - Non-void checks must not be open.
   - Void checks must be open and match full amount.
   - Update `ACCKAM = AMT - DISC`.

8. **Freight**:
   - If `FRTL ≠ 0`, allocate amounts to detail lines via `AP1011`.

## Calculations
- **Due Date (`DUDT`)**: Calculated from `INDT` + `TBNETD` (net days from `GSTABL`), adjusted for holidays/weekends.
- **Discount Due Date (`DSDT`)**: Calculated from `INDT` + `TBDISD` (discount days from `GSTABL`).
- **Discount Amount (`DISC`)**: If `DSPC ≠ 0`, `DISC = AMT * (DSPC / 100)`.
- **Net Amount (`NETAMT`)**: `NETAMT = AMT - DISC`.
- **Check Amount (`ACCKAM`)**: `ACCKAM = AMT - DISC` for prepaid checks.
- **Receipt Quantity**: `RNQTY = IHNQTY + IHNQTF - IHAPTQ - IHAPTF` (from `INFIL1` or `INTZH1`).
- **Totals**:
  - `L1AMT = Σ(AMT)`, `L1PAMT = Σ(PRAM)`, `L1FAMT = Σ(FRAM)`, `L1DISC = Σ(DISC)`, `L1NET = L1AMT - L1DISC` (per entry).
  - `L2AMT`, `L2PAMT`, `L2FAMT`, `L2DISC`, `L2NET`, `L2PPD`, `L2PPA`, `L2PPW`, `L2PPE` accumulate `L1` totals.

## External Dependencies
- **Program Called**: `AP1011` (for freight amount allocation to detail lines when `FRTL ≠ 0`).
- **Files**:
  - Input: `APCONT`, `APVEND`, `GLMAST`, `GSTABL`, `INFIL1`, `INTZH1`, `APTRNX`, `APOPNHC`, `APINVH`.
  - Update: `APTRAN`, `APCHKT`, `APSTAT`.
  - Output: `APLIST` (report).

