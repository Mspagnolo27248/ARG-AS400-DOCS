### List of Use Cases Implemented by the AP200, AP205, and AP210 RPG Programs

The call stack consists of three RPG III programs (`AP200`, `AP205`, `AP210`) invoked by the main OCL program (`AP200.ocl36`) to manage the Accounts Payable (A/P) Purchase Register process, summarize journal entries, and post A/P invoices to inventory receipts. These programs collectively implement a single overarching use case, which can be broken down into three interconnected functions for clarity. Below is the identified use case, treated as a large function that processes inputs programmatically without screen interaction.

**Use Case: Process Accounts Payable Transactions and Post to General Ledger and Inventory**
- **Description**: This use case encompasses the end-to-end processing of A/P transactions, including voucher creation, journal entry generation, summarization, and posting to inventory receipts. It validates and updates A/P records, generates general ledger entries, and produces reports, ensuring accurate financial and inventory tracking.
- **Components**:
  1. **AP200 (Purchase Register)**: Processes A/P transactions, creates vouchers, updates related files, and generates journal entries and a Purchase Register report.
  2. **AP205 (Purchase Journal)**: Summarizes A/P journal entries and generates general ledger entries and a Purchase Journal report.
  3. **AP210 (A/P to Inventory Posting)**: Posts A/P invoice details to inventory receipt records, updating quantities and amounts.

---

### Function Requirement Document: Process Accounts Payable Transactions and Post to General Ledger and Inventory



# Function Requirement: Process Accounts Payable Transactions and Post to General Ledger and Inventory

## Purpose
To process Accounts Payable (A/P) transactions, create and manage vouchers, generate general ledger journal entries, summarize A/P entries, post invoice details to inventory receipts, and produce detailed financial and diagnostic reports without user interface interaction.

## Inputs
- **APTRAN**: Transaction file with header (company, vendor, invoice, dates, hold codes, sales order, carrier ID) and detail (expense G/L, amount, discount, gallons, receipt number, purchase order) records.
- **APCONT**: A/P control file (company name, journal number, next voucher number).
- **APVEND**: Vendor master file (year-to-date purchases, balance).
- **APOPEN, APOPENH, APOPEND, APOPENV**: Open A/P files for voucher records.
- **APINVH**: Invoice header file for invoice details.
- **FRCINH, FRCFBH**: Freight invoice and override header files for A/P status.
- **POFILEH, POFILED**: Purchase order files (disabled).
- **APPJJR**: Journal register file for input to summarization.
- **AP205S**: Sort/index file for `APPJJR`.
- **INFIL1, INTZH1**: Inventory receipt and holding files for invoice posting.
- **System Date and Time**: For date conversions and reporting.
- **Journal Date (JRDATE)**: For Y2K-compliant date processing.

## Outputs
- **APPJJR**: Journal register with G/L entries (A/P, expense, intercompany).
- **APPYTR**: Payment transaction records for prepaid/ACH/wire/employee/utility vouchers.
- **APHISTH, APHISTD, APHISTV**: History files for canceled vouchers.
- **TEMGEN**: General ledger entries (detailed and summarized).
- **APOPENH, APOPEND, APOPENV**: Updated open A/P records.
- **APVEND**: Updated vendor balances.
- **APINVH**: Updated invoice headers.
- **FRCINH, FRCFBH**: Updated freight A/P status.
- **INFIL1, INTZH1**: Updated inventory receipt records.
- **APPRINT (AP200, AP205)**: Purchase Register and Purchase Journal reports.
- **APLIST**: Diagnostic report for inventory posting.

## Process Steps
1. **Initialize System**:
   - Convert journal date (`JRDATE`) to Y2K-compliant format (e.g., `JRYMD`, `JRYM8`).
   - Retrieve company details (`APCONT`) and initialize totals, counters, and separators.

2. **Process A/P Transactions (AP200)**:
   - Read `APTRAN` header and detail records.
   - Assign voucher numbers (`NXTVO`) for non-canceled, non-100% retention vouchers.
   - Update freight files (`FRCINH`, `FRCFBH`) by clearing A/P status (`FRAPST`) for deleted vouchers.
   - Write payment transactions (`APPYTR`) for prepaid, ACH, wire, employee, or utility auto-pay vouchers.
   - Calculate discounts (`ATDISC = ATAMT * (ATDSPC / 100)`) if discount percent is non-zero.
   - Handle retentions: compute retention amount (`ATRTAM = ATAMT * (ATRTPC / 100)`), adjust `ATAMT`, and set retention discount (`ATRTDS`).
   - Generate intercompany journal entries (`APPJJR`) if company numbers differ (`ATCONO ≠ ATEXCO`).
   - Update open A/P (`APOPENH`, `APOPEND`, `APOPENV`), vendor (`APVEND`), and invoice (`APINVH`) files.
   - Write history records (`APHISTH`, `APHISTD`, `APHISTV`) for canceled vouchers.
   - Produce Purchase Register report (`APPRINT`) with voucher details, totals, and special fields (sales order, carrier ID, process type).

3. **Summarize Journal Entries (AP205)**:
   - Read `APPJJR` records via `AP205S` sort file.
   - Summarize A/P entries (`PJTYPE = 'AP      '`) into a single `TEMGEN` record per voucher.
   - Write detailed `TEMGEN` records for non-A/P entries (expense, intercompany).
   - Adjust negative amounts: negate and switch debit (`D`) to credit (`C`) or vice versa.
   - Accumulate company-level debit (`L4DR`) and credit (`L4CR`) totals.
   - Produce Purchase Journal report (`APPRINT`) with journal entries, totals, and gallons/receipt details.

4. **Post to Inventory Receipts (AP210)**:
   - Read `APTRAN` detail records with receipt number (`APREC#`).
   - Skip records with sales order (`ATSORN ≠ *ZERO`) or deleted status (`APHDEL = 'D'`, `APDDEL = 'D'`).
   - Match receipt number in `INFIL1` or `INTZH1`:
     - If exact match (`APGAL = IHNQTY + IHNQTF - IHAPTQ - IHAPTF`), update record.
     - If no exact match, find record with sufficient gallons (`APGAL < RNQTY`).
     - If no sufficient gallons, update first record.
   - Update `INFIL1` or `INTZH1` with invoice date (`JRYMD`), G/L number (`APGL`), journal number (`JR#`), quantities (`IHAPTQ`, `IHAPTF`), dollars (`IHAPTD`), invoice number (`APINVN`), PO number (`IHPONO`), and close code (`IHCLCD = 'O' or 'C'`).
   - Produce diagnostic report (`APLIST`) with input and updated fields.

## Business Rules
1. **Voucher Management**:
   - Assign unique voucher numbers (`NXTVO`) for non-canceled, non-100% retention, and retention vouchers.
   - Support payment types: prepaid (`P`), ACH (`A`), wire (`W`), employee expense (`E`), utility auto-pay (`U`).
   - Mark canceled vouchers (`ATCNVO ≠ *ZEROS`) as deleted in open A/P files and write to history files.

2. **Discounts and Retentions**:
   - Calculate discounts if `ATDSPC ≠ 0` and `ATDISC = 0`.
   - For retentions (`ATRTPC ≠ 0`), compute `ATRTAM` and adjust `ATAMT`; for 100% retention, set `ATAMT = 0`.

3. **Freight Handling**:
   - Clear `FRAPST` in `FRCINH` or `FRCFBH` for deleted vouchers if previously flagged (`FRAPST = 'Y'`).
   - Prioritize `FRCFBH` over `FRCINH` for freight status checks.

4. **Intercompany Transfers**:
   - Generate `APPJJR` entries for intercompany transactions (`ATCONO ≠ ATEXCO`) using intercompany G/L accounts (`ACICGL`).

5. **Journal Summarization**:
   - Summarize A/P entries into a single `TEMGEN` record with fixed description.
   - Negate negative amounts and adjust debit/credit codes accordingly.

6. **Inventory Posting**:
   - Skip posting if sales order exists or records are deleted.
   - Match receipts exactly, by sufficient gallons, or use first record if no match.
   - Update inventory records with A/P details and set close status based on `APCLCD`.

7. **Y2K Compliance**:
   - Convert dates to 8-digit format using century (`Y2KCEN = 19` or `19 + 100`) based on `Y2KCMP = 1980`.

8. **Reporting**:
   - Generate Purchase Register (`AP200`) with voucher, sales order, carrier ID, and process type details.
   - Generate Purchase Journal (`AP205`) with summarized A/P and detailed non-A/P entries, including gallons/receipt.
   - Generate diagnostic report (`AP210`) for inventory posting verification.

## Calculations
- **Discount**: `ATDISC = ATAMT * (ATDSPC / 100)` if `ATDSPC ≠ 0` and `ATDISC = 0`.
- **Retention Amount**: `ATRTAM = ATAMT * (ATRTPC / 100)`; for 100% retention, `ATAMT = 0`, `ATRTAM = original ATAMT`.
- **Remaining Quantity (Inventory)**: `RNQTY = IHNQTY + IHNQTF - IHAPTQ - IHAPTF`.
- **Date Conversion**: `YYYYMMDD = JRDATE * 10000.01`, century set to 19 if year ≥ 1980, else 20.

## Constraints
- Disabled purchase order updates (`POFILEH`, `POFILED`) in `AP200`.
- `APLIST` is temporary for diagnostic purposes and may be removed.
- 13-period accounting supported if `KYYYPD ≠ 0`.

