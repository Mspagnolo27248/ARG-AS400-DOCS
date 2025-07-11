# Function Requirement Document: Process Accounts Payable Transactions and Post to General Ledger and Inventory

## Purpose
To process Accounts Payable (A/P) transactions, create and manage vouchers, generate general ledger journal entries, summarize A/P entries, post invoice details to inventory receipts, and produce detailed financial and diagnostic reports. This encompasses the end-to-end processing of A/P transactions, including voucher creation, journal entry generation, summarization, and posting to inventory receipts.

## Components
The system consists of three interconnected RPG programs:
1. **AP200 (Purchase Register)**: Processes A/P transactions, creates vouchers, updates related files, and generates journal entries and a Purchase Register report.
2. **AP205 (Purchase Journal)**: Summarizes A/P journal entries and generates general ledger entries and a Purchase Journal report.
3. **AP210 (A/P to Inventory Posting)**: Posts A/P invoice details to inventory receipt records, updating quantities and amounts.

## Inputs
- **Purchase Journal Date (`JRDATE`)**: MMDDYY format, required.
- **Cash Disbursements Date (`CDDATE`)**: MMDDYY format, required if prepaid/ACH/wire/employee transactions exist.
- **Purchase Journal Period/Year (`KYPD`, `KYPDYY`)**: Required if 13 accounting periods are enabled.
- **Cash Disbursements Period/Year (`CDPD`, `CDPDYY`)**: Required if 13 periods and prepaid transactions exist.
- **APTRAN**: Transaction file with header (company, vendor, invoice, dates, hold codes, sales order, carrier ID) and detail (expense G/L, amount, discount, gallons, receipt number, purchase order) records.
- **APCONT**: A/P control file (company name, journal number, next voucher number).
- **APVEND**: Vendor master file (year-to-date purchases, balance).
- **APOPENH, APOPEND, APOPENV**: Open A/P files for voucher records.
- **APINVH**: Invoice header file for invoice details.
- **FRCINH, FRCFBH**: Freight invoice and override header files for A/P status.
- **POFILEH, POFILED**: Purchase order files.
- **APPJJR**: Journal register file for input to summarization.
- **AP205S**: Sort/index file for `APPJJR`.
- **INFIL1, INTZH1**: Inventory receipt and holding files for invoice posting.
- **Control Files**: `GSCONT` (system controls), `GLCONT` (G/L controls), `GSTABL` (period end dates).
- **System Date and Time**: For date conversions and reporting.

## Outputs
- **APPJJR**: Journal register with G/L entries (A/P, expense, intercompany).
- **APPYTR**: Payment transaction records for prepaid/ACH/wire/employee/utility vouchers.
- **APHISTH, APHISTD, APHISTV**: History files for canceled vouchers.
- **TEMGEN**: General ledger entries (detailed and summarized).
- **APOPENH, APOPEND, APOPENV**: Updated open A/P records.
- **APVEND**: Updated vendor balances.
- **APINVH**: Updated invoice headers.
- **FRCINH, FRCFBH**: Updated freight A/P status.
- **POFILEH, POFILED**: Updated purchase order files (if enabled).
- **INFIL1, INTZH1**: Updated inventory receipt records.
- **APPRINT (AP200, AP205)**: Purchase Register and Purchase Journal reports.
- **APLIST**: Diagnostic report for inventory posting.

## Process Steps

### 1. Validate Input Parameters
- Validate `JRDATE` and `CDDATE` for correct format (MMDDYY) and fiscal year compliance using `GLCONT` (`GCLSYR`, `GCFFMO`).
- If 13 accounting periods enabled (`GX13GL = 'Y'` in `GSCONT`), validate `KYPD` (1–13) and `CDPD` (1–13) against period end dates in `GSTABL` (`TBPDDT`).
- Check for prepaid/ACH/wire/employee transactions (`ATPAID = 'P', 'A', 'W', 'E'`) in `APTRAN` to determine if `CDDATE` is required.

### 2. Initialize System
- Convert journal date (`JRDATE`) to Y2K-compliant format (e.g., `JRYMD`, `JRYM8`).
- Convert `JRDATE` to YYYYMMDD (`PJYMD`) with Y2K-compliant century (19xx if year ≥ 80, else 20xx).
- Retrieve company details (`APCONT`) and initialize totals, counters, and separators.
- Retrieve next journal (`ACJRNL`) and voucher numbers (`ACNXVO`) from `APCONT`.
- Set journal ID (`JRNID`) to `'PJ'`, `'WT'`, or `'EE'` based on transaction type.

### 3. Process A/P Transactions (AP200)
- Read `APTRAN` header and detail records.
- For each header:
  - Skip if deleted (`ATHDEL = 'D'`); clear `FRAPST` in `FRCINH`/`FRCFBH` if `'Y'`.
  - Assign voucher numbers (`NXTVO`) for non-canceled, non-100% retention vouchers.
  - Set flags for prepaid (`P`), ACH (`A`), wire (`W`), employee expense (`E`), utility auto-pay (`U`), single check (`ATSNGL = 'S'`), or hold (`ATHOLD = 'H', 'A', 'W', 'E', 'U'`).
  - Write payment transactions (`APPYTR`) for prepaid, ACH, wire, employee, or utility auto-pay vouchers.
  - Calculate retention amount (`ATRTAM = ATAMT * ATRTPC / 100`) if `ATRTPC ≠ 0`.
- For each detail:
  - Calculate discounts (`ATDISC = ATAMT * (ATDSPC / 100)`) if discount percent is non-zero AND `ATDISC = 0`.
  - Handle retentions: compute retention amount (`ATRTAM = ATAMT * (ATRTPC / 100)`), adjust `ATAMT`, and set retention discount (`ATRTDS`). For 100% retention, set `ATAMT = 0` and `ATRTAM = original ATAMT`.
  - Update level 1 totals (`L1AMT`, `L1DISC`, `L1RTAM`, `L1RTDS`, `L1PAMT`, `L1FAMT`).
  - Generate intercompany journal entries (`APPJJR`) if company numbers differ (`ATCONO ≠ ATEXCO`).
  - Update P/O files (`POFILEH`, `POFILED`) if enabled (`ACPOYN = 'Y'`) with amounts (`POAPPU`, `PDAPV$`) and receipt data (`PDRCQT`, `PDRCDT`, `PDCOMP`).
- Update open A/P (`APOPENH`, `APOPEND`, `APOPENV`), vendor (`APVEND`), and invoice (`APINVH`) files.
- Write history records (`APHISTH`, `APHISTD`, `APHISTV`) for canceled vouchers.
- Update vendor totals (`VN$YTD`, `VNPURC`, `VNCBAL`) in `APVEND`.
- Accumulate company totals (`L2AMT`, `L2DISC`, `L2PAMT`, `L2FAMT`).
- Produce Purchase Register report (`APPRINT`) with voucher details, totals, and special fields (sales order, carrier ID, process type).

### 4. Summarize Journal Entries (AP205)
- Read `APPJJR` records via `AP205S` sort file.
- Summarize A/P entries (`PJTYPE = 'AP      '`) into a single `TEMGEN` record per voucher.
- Write detailed `TEMGEN` records for non-A/P entries (expense, intercompany).
- Adjust negative amounts: negate and switch debit (`D`) to credit (`C`) or vice versa.
- Accumulate company-level debit (`L4DR`) and credit (`L4CR`) totals.
- Produce Purchase Journal report (`APPRINT`) with journal entries, totals, and gallons/receipt details.

### 5. Post to Inventory Receipts (AP210)
- Read `APTRAN` detail records with receipt number (`APREC#`).
- Skip records with sales order (`ATSORN ≠ *ZERO`) or deleted status (`APHDEL = 'D'`, `APDDEL = 'D'`).
- Match receipt number in `INFIL1` or `INTZH1`:
  - If exact match (`APGAL = IHNQTY + IHNQTF - IHAPTQ - IHAPTF`), update record.
  - If no exact match, find record with sufficient gallons (`APGAL < RNQTY`).
  - If no sufficient gallons, update first record.
- Update `INFIL1` or `INTZH1` with invoice date (`JRYMD`), G/L number (`APGL`), journal number (`JR#`), quantities (`IHAPTQ`, `IHAPTF`), dollars (`IHAPTD`), invoice number (`APINVN`), PO number (`IHPONO`), and close code (`IHCLCD = 'O' or 'C'`).
- Produce diagnostic report (`APLIST`) with input and updated fields.

## Business Rules

### Validation
- Dates must be valid and within the current fiscal year (`GCLSYR`, `GCFFMO`).
- Periods (1–13) must match `GSTABL` boundaries if 13 periods enabled.
- `CDDATE` required only if prepaid/ACH/wire/employee transactions exist.

### Voucher Management
- Assign unique voucher numbers (`NXTVO`) for non-canceled, non-100% retention, and retention vouchers.
- Support payment types: prepaid (`P`), ACH (`A`), wire (`W`), employee expense (`E`), utility auto-pay (`U`).
- Mark canceled vouchers (`ATCNVO ≠ *ZEROS`) as deleted in open A/P files and write to history files.
- Hold codes (`H`, `A`, `W`, `E`, `U`) affect voucher processing.

### Discounts and Retentions
- Calculate discounts if `ATDSPC ≠ 0` and `ATDISC = 0` (both conditions must be true).
- For retentions (`ATRTPC ≠ 0`), compute `ATRTAM` and adjust `ATAMT`; for 100% retention, set `ATAMT = 0` and `ATRTAM = original ATAMT`.
- Create separate retention voucher if not 100% retention.

### Freight Handling
- Clear `FRAPST` in `FRCINH` or `FRCFBH` for deleted vouchers if previously flagged (`FRAPST = 'Y'`).
- Prioritize `FRCFBH` over `FRCINH` for freight status checks.

### Intercompany Transfers
- Generate `APPJJR` entries for intercompany transactions (`ATCONO ≠ ATEXCO`) using intercompany G/L accounts (`ACICGL`).
- Generate debit/credit entries for `ATCONO ≠ ATEXCO`.

### Purchase Orders
- Update `POFILEH`/`POFILED` if `ACPOYN = 'Y'` and `ATPONO ≠ blanks`.

### Journal Summarization
- Summarize A/P entries into a single `TEMGEN` record with fixed description.
- Negate negative amounts and adjust debit/credit codes accordingly.

### Inventory Posting
- Skip posting if sales order exists or records are deleted.
- Match receipts exactly, by sufficient gallons, or use first record if no match.
- Update inventory records with A/P details and set close status based on `APCLCD`.

### Y2K Compliance
- Convert dates to 8-digit format using century (19xx if year ≥ 80, else 20xx) based on `Y2KCMP = 1980`.

### Reporting
- Generate Purchase Register (`AP200`) with voucher, sales order, carrier ID, and process type details.
- Generate Purchase Journal (`AP205`) with summarized A/P and detailed non-A/P entries, including gallons/receipt.
- Generate diagnostic report (`AP210`) for inventory posting verification.
- Include sales order (`ATSORN`), sequence (`ATSSRN`), carrier ID (`ATCAID`), process type (`ATPTYP`), and discount due date (`ATDSDT`) in reports.

## Calculations

### Date Conversion
- `YYYYMMDD = JRDATE * 10000.01`, century set to 19 if year ≥ 1980, else 20.
- Alternative: `YYYYMMDD = MMDDYY * 10000.01`, with century (19xx if year ≥ 80, else 20xx).

### Financial Calculations
- **Discount**: `ATDISC = ATAMT * (ATDSPC / 100)` if `ATDSPC ≠ 0` and `ATDISC = 0`.
- **Retention Amount**: `ATRTAM = ATAMT * (ATRTPC / 100)`.
- **Remaining Quantity (Inventory)**: `RNQTY = IHNQTY + IHNQTF - IHAPTQ - IHAPTF`.

### Totals
- **Voucher Level**: `L1AMT += ATAMT`, `L1DISC += ATDISC`, `L1RTAM += ATRTAM`, `L1RTDS += ATRTDS`, `L1PAMT += ATPRAM`, `L1FAMT += ATFRAM`.
- **Company Level**: `L2AMT += L1AMT`, `L2DISC += L1DISC`, `L2PAMT += L1PAMT`, `L2FAMT += L1FAMT`.

## Constraints
- Purchase order updates (`POFILEH`, `POFILED`) are controlled by `ACPOYN = 'Y'` flag.
- `APLIST` is temporary for diagnostic purposes and may be removed.
- 13-period accounting supported if `KYYYPD ≠ 0`.
- System supports various transaction types including utility auto-pay transactions.

## Error Handling
- Validate all input parameters before processing.
- Handle missing or invalid data gracefully.
- Provide diagnostic reporting for inventory posting verification.
- Maintain audit trail through history files for canceled vouchers.