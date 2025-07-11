### List of Use Cases Implemented by the AP200 Call Stack

The `AP200` call stack, consisting of the OCL procedure (`AP200.ocl36.txt`), the parameter prompt program (`AP200P.rpg36.txt`), and the main processing program (`AP200.rpg36.txt`), implements the following primary use case:

1. **Generate Purchase Journal and Update A/P Records**:
   - This use case involves prompting for and validating input parameters (dates and accounting periods), processing Accounts Payable (A/P) transactions, generating journal entries, updating open A/P, vendor, purchase order (P/O), and freight files, and producing a Purchase Register report. It handles various transaction types (e.g., prepaid, ACH, wire transfer, employee expense, utility auto-pay), cancellations, retentions, and intercompany transfers, ensuring accurate financial and inventory tracking.

This is the primary use case, as the programs collectively form a cohesive workflow for managing the A/P Purchase Journal process, from parameter input to transaction processing and reporting.

---

### Function Requirement Document



# Function Requirement Document: Generate Purchase Journal and Update A/P Records

## Purpose
The function processes Accounts Payable (A/P) transactions to generate Purchase Journal entries, update open A/P, vendor, purchase order (P/O), and freight files, and produce a Purchase Register report. It validates input parameters and handles transaction types (prepaid, ACH, wire transfer, employee expense, utility auto-pay), cancellations, retentions, and intercompany transfers.

## Inputs
- **Purchase Journal Date (`JRDATE`)**: MMDDYY format, required.
- **Cash Disbursements Date (`CDDATE`)**: MMDDYY format, required if prepaid/ACH/wire/employee transactions exist.
- **Purchase Journal Period/Year (`KYPD`, `KYPDYY`)**: Required if 13 accounting periods are enabled.
- **Cash Disbursements Period/Year (`CDPD`, `CDPDYY`)**: Required if 13 periods and prepaid transactions exist.
- **A/P Transaction File (`APTRAN`)**: Contains header and detail records with fields like company (`ATCONO`), vendor (`ATVEND`), invoice (`ATINV#`), amount (`ATAMT`), payment type (`ATPAID`), hold code (`ATHOLD`), etc.
- **Control Files**: `APCONT` (control data), `APVEND` (vendor data), `GSCONT` (system controls), `GLCONT` (G/L controls), `GSTABL` (period end dates), `POFILEH`, `POFILED` (P/O data), `FRCINH`, `FRCFBH` (freight data).

## Outputs
- **Purchase Journal (`APPJJR`)**: Journal entries for A/P, expense, retention, and intercompany transactions.
- **Open A/P Files (`APOPENH`, `APOPEND`, `APOPENV`)**: Updated with new or canceled vouchers.
- **History Files (`APHISTH`, `APHISTD`, `APHISTV`)**: Records for canceled vouchers.
- **Vendor File (`APVEND`)**: Updated year-to-date and balance totals.
- **Purchase Order Files (`POFILEH`, `POFILED`)**: Updated amounts and receipt data if P/O integration enabled.
- **Freight Files (`FRCINH`, `FRCFBH`)**: Cleared A/P status for deleted vouchers.
- **Payment Transaction File (`APPYTR`)**: Updated with prepaid/ACH/wire/employee transaction data.
- **Purchase Register Report (`APPRINT`)**: Detailed report with voucher, vendor, invoice, and totals.

## Process Steps
1. **Validate Input Parameters**:
   - Validate `JRDATE` and `CDDATE` for correct format (MMDDYY) and fiscal year compliance using `GLCONT` (`GCLSYR`, `GCFFMO`).
   - If 13 accounting periods enabled (`GX13GL = 'Y'` in `GSCONT`), validate `KYPD` (1–13) and `CDPD` (1–13) against period end dates in `GSTABL` (`TBPDDT`).
   - Check for prepaid/ACH/wire/employee transactions (`ATPAID = 'P', 'A', 'W', 'E'`) in `APTRAN` to determine if `CDDATE` is required.
2. **Initialize Processing**:
   - Convert `JRDATE` to YYYYMMDD (`PJYMD`) with Y2K-compliant century (19xx/20xx).
   - Retrieve next journal (`ACJRNL`) and voucher numbers (`ACNXVO`) from `APCONT`.
   - Set journal ID (`JRNID`) to `'PJ'`, `'WT'`, or `'EE'` based on transaction type.
3. **Process Transactions**:
   - For each `APTRAN` header:
     - Skip if deleted (`ATHDEL = 'D'`); clear `FRAPST` in `FRCINH`/`FRCFBH` if `'Y'`.
     - Assign voucher number (`VOUCHR`) from `ACNXVO` or `ATCNVO` (canceled).
     - Set flags for prepaid (`P`), ACH (`A`), wire (`W`), employee expense (`E`), utility auto-pay (`U`), single check (`ATSNGL = 'S'`), or hold (`ATHOLD = 'H', 'A', 'W', 'E', 'U'`).
     - Calculate retention amount (`ATRTAM = ATAMT * ATRTPC / 100`) if `ATRTPC ≠ 0`.
     - Write payment records to `APPYTR` for prepaid/ACH/wire/employee transactions.
   - For each detail:
     - Calculate discount (`ATDISC = ATAMT * ATDSPC / 100`) if `ATDSPC ≠ 0`.
     - Adjust `ATAMT` for retention if applicable.
     - Update level 1 totals (`L1AMT`, `L1DISC`, `L1RTAM`, `L1RTDS`, `L1PAMT`, `L1FAMT`).
     - Generate intercompany entries if `ATCONO ≠ ATEXCO`.
     - Update P/O files (`POFILEH`, `POFILED`) if enabled (`ACPOYN = 'Y'`) with amounts (`POAPPU`, `PDAPV$`) and receipt data (`PDRCQT`, `PDRCDT`, `PDCOMP`).
4. **Handle Cancellations**:
   - For canceled vouchers (`ATCNVO ≠ 0`), update `APOPENH`, `APOPEND`, or `APOPENV` and write to history files (`APHISTH`, `APHISTD`, `APHISTV`).
5. **Generate Journal Entries**:
   - Write to `APPJJR` for A/P, expense, retention, and intercompany transactions.
6. **Update Totals**:
   - Update vendor totals (`VN$YTD`, `VNPURC`, `VNCBAL`) in `APVEND`.
   - Update invoice history (`APINVH`) with amounts and dates.
   - Accumulate company totals (`L2AMT`, `L2DISC`, `L2PAMT`, `L2FAMT`).
7. **Produce Report**:
   - Generate Purchase Register (`APPRINT`) with company, voucher, vendor, invoice, discount, gallons, receipt, and totals (voucher, retention, company).

## Business Rules
- **Validation**:
  - Dates must be valid and within the current fiscal year (`GCLSYR`, `GCFFMO`).
  - Periods (1–13) must match `GSTABL` boundaries if 13 periods enabled.
  - `CDDATE` required only if prepaid/ACH/wire/employee transactions exist.
- **Transaction Types**:
  - Supports prepaid (`P`), ACH (`A`), wire (`W`), employee expense (`E`), and utility auto-pay (`U`).
  - Hold codes (`H`, `A`, `W`, `E`, `U`) affect voucher processing.
- **Retention**:
  - Calculate retention (`ATRTAM`) if `ATRTPC ≠ 0`; create separate retention voucher if not 100%.
- **Intercompany**:
  - Generate debit/credit entries for `ATCONO ≠ ATEXCO` using `ACICGL`.
- **Discounts**:
  - Apply discount (`ATDISC`) if `ATDSPC ≠ 0`.
- **Purchase Orders**:
  - Update `POFILEH`/`POFILED` if `ACPOYN = 'Y'` and `ATPONO ≠ blanks`.
- **Freight**:
  - Clear `FRAPST` in `FRCINH`/`FRCFBH` for deleted vouchers if `'Y'`.
- **Cancellations**:
  - Update open A/P and write history for canceled vouchers.
- **Reporting**:
  - Include sales order (`ATSORN`), sequence (`ATSSRN`), carrier ID (`ATCAID`), process type (`ATPTYP`), and discount due date (`ATDSDT`).

## Calculations
- **Date Conversion**: `YYYYMMDD = MMDDYY * 10000.01`, with century (19xx if year ≥ 80, else 20xx).
- **Retention Amount**: `ATRTAM = ATAMT * (ATRTPC / 100)`.
- **Discount Amount**: `ATDISC = ATAMT * (ATDSPC / 100)`.
- **Totals**:
  - Voucher: `L1AMT += ATAMT`, `L1DISC += ATDISC`, `L1RTAM += ATRTAM`, `L1RTDS += ATRTDS`, `L1PAMT += ATPRAM`, `L1FAMT += ATFRAM`.
  - Company: `L2AMT += L1AMT`, `L2DISC += L1DISC`, `L2PAMT += L1PAMT`, `L2FAMT += L1FAMT`.

