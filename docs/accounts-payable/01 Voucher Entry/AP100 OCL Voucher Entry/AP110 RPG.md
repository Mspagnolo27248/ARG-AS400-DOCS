The `AP110.rpg36.txt` is an RPG III program for IBM midrange systems (e.g., AS/400 or iSeries), called by an OCL program (e.g., `AP110.ocl36.txt`). It performs validation and editing of Accounts Payable (A/P) voucher transactions, ensuring data integrity for headers and detail lines. The program generates a printed report (`APLIST`) listing transaction details, errors, and totals for invoice amounts, discounts, and prepaid checks. It includes modifications for ACH payments, employee expenses, utility auto-payments, FlexiCapture invoice uploads, and validations for gallons, receipts, and purchase orders. Below is a detailed explanation of the process steps, business rules, tables used, and external programs called.

---

### Process Steps of the RPG Program (AP110)

The `AP110` program processes A/P voucher transactions from the `APTRAN` file, validates them against various files (e.g., `APVEND`, `GLMAST`, `APCONT`), and produces a detailed error report via `APLIST`. The key steps are as follows:

1. **Initialization (Lines 0092–0107)**:
   - Executes at the total level (`L2`) to initialize variables:
     - Retrieves system date and time (`TIME`) and stores them in `TIMDAT`, `SYSTIM`, `SYSDAT`, and `SYSYMD` (year-month-day format).
     - Converts system date to 8-digit format (`SYSDT8`) for comparisons.
     - Sets separator (`SEP`) to `'* '` for report formatting.
     - Initializes page number (`PAGE`) and accumulators (`L2CNT`, `L2VHSH`, `L2AMT`, `L2PAMT`, `L2FAMT`, `L2DISC`, `L2NET`, `L2PPD`, `L2PPA`, `L2PPW`, `L2PPE`) to zero.
   - Validates company number (`ATCONO`) against `APCONT`. If not found (`92`), sets error indicator.
   - At the detail level (`L1`):
     - Initializes detail accumulators (`L1AMT`, `L1PAMT`, `L1FAMT`, `L1DISC`, `L1NET`) to zero.
     - For wire transfer transactions (`WIREDS ≠ *BLANKS`), prints a header (`PRTHDR`) for each entry.

2. **Main Processing Loop (Lines 0109–0125)**:
   - Processes header records (`01`, `L1`) and detail records (`02`, `L1`) from `APTRAN`:
     - For header records (`01`), executes the `HDR` subroutine.
     - For detail records (`02`, non-deleted `N51`), executes the `DET` subroutine.
   - Accumulates totals at `L1` (if no delete, `N51`):
     - Adds detail amounts (`L1AMT`, `L1PAMT`, `L1FAMT`, `L1DISC`, `L1NET`) to `L2` totals (`L2AMT`, `L2PAMT`, `L2FAMT`, `L2DISC`, `L2NET`).
     - For prepaid (`28`), ACH (`24`), wire transfer (`25`), or employee expense (`32`) payments, accumulates amounts into `L2PPD`, `L2PPA`, `L2PPW`, or `L2PPE`.
   - Updates `APSTAT` with error status (`'Y'` if errors exist, `'N'` otherwise) using `STATKY` (company and workstation).

3. **HDR Subroutine (Lines 0127–0144)**:
   - Validates header records:
     - If the header is deleted (`ATHDEL = 'D'`), sets indicator `51` and skips to `ENDHDR`.
     - Increments invoice count (`L2CNT`) and vendor hash total (`L2VHSH`).
     - Validates retention percentage (`ATRTPC ≠ 0`, sets `26`), hold code (`ATHOLD = 'H'`, sets `77`), canceled voucher (`ATCNVO ≠ 0`, sets `27`), prepaid code (`ATPAID = 'P'`, sets `28`; `= 'A'`, sets `24`; `= 'E'`, sets `32`; `= 'U'`, sets `33`), single check (`ATSNGL = 'S'`, sets `29`), and sales order (`ATSORN ≠ 0`, sets `19`).
     - Checks hold description (`ATHLDD`) for blanks (sets `77` if blank).
     - Calls `HEADCK` subroutine for additional header validations.

4. **HEADCK Subroutine (Lines MG02)**:
   - Validates company, vendor, and invoice details:
     - Chains `ATCONO` to `APCONT`. If not found (`99`), sets error `95`.
     - Chains vendor key (`ATCONO`, `ATVEND`) to `APVEND`. If not found (`99`), deleted (`VNDEL = 'D'`), inactive (`VNDEL = 'I'`), or vendor name blank (`VNVNAM = *BLANKS`), sets error `95`.
     - Retrieves terms code (`VNTERM`) from `GSTABL`. If not found (`99`), clears term description (`TRMDSC`).
     - Ensures invoice date (`ATINDT`) is non-zero. If zero, sets error `95`.
     - Compares invoice date (`ATIND8`) to system date minus one year (`SYSDT8 - 10000`). If older, sets warning `96` (MG09).
     - Ensures invoice amount (`ATIAMT`) is non-zero. If zero, sets error `95`.
     - Ensures invoice number (`ATINV#`) is not blank. If blank, sets error `95`.
     - For non-prepaid (`N28`), non-canceled (`N27`), non-wire-transfer (`N25`) transactions with non-zero vendor (`ATVEND ≠ 0`):
       - Checks for duplicate invoices in `APTRNX` (`INVKEY = XXKEY`, `XXENT ≠ ATENT#`, sets `95` if found).
       - Checks `APINVH` for duplicate invoice keys (`INVKEY = AIKEY`, sets `95` if found, MGXX).
       - Checks `APOPNHC` for duplicate invoice (`OCCONO = ATCONO`, `OCVEND = ATVEND`, `OCINVN = ATINV#`, sets `95` if found).
     - Validates A/P G/L (`ATAPGL`) and bank G/L (`ATBKGL`) against `GLMAST`. If not found (`99`), deleted (`GLDEL = 'D'`), or inactive (`GLDEL = 'I'`), sets error `95`.

5. **DET Subroutine (Lines 0146–0169)**:
   - Validates detail records:
     - If the detail is deleted (`ATDDEL = 'D'`), sets `52` and skips to `ENDDTL`.
     - Checks if gallons (`ATGALN ≠ 0`, sets `60`) or job number (`ATJOB# ≠ *BLANKS`, sets `30`) are present.
     - Validates discount:
       - If discount percentage (`ATDSPC ≠ 0`), ensures discount amount (`ATDISC`) is zero (sets `10` if both non-zero).
       - If `ATDSPC ≠ 0`, calculates `ATDISC = ATAMT * (ATDSPC / 100)`.
     - Calculates net amount (`NETAMT = ATAMT - ATDISC`).
     - Accumulates detail amounts (`L1AMT`, `L1PAMT`, `L1FAMT`, `L1DISC`, `L1NET`).
     - For prepaid transactions (`28`), calls `PPDCHK` subroutine.
     - Calls `DETLCK` subroutine for additional detail validations.

6. **PPDCHK Subroutine (Lines 0171–0183)**:
   - Validates prepaid checks:
     - Chains check key (`CKKY21`) to `APCHKT`. If not found (`N90`), adds `ATAMT` to `ACCKAM`. If found (`90`), sets `ACCKAM = ATAMT`.
     - Subtracts discount (`ATDISC`) from `ACCKAM`.
     - Writes or updates `APCHKT` with the prepaid check record (`PPDREC`).
     - Writes the prepaid check detail to `APLIST`.

7. **DETLCK Subroutine (Lines MG02–MGXX)**:
   - Validates detail line fields:
     - Validates gallons/receipt requirements based on vendor (`VNGRRQ`) and G/L (`GLAPCD`):
       - If `VNGRRQ = 'Y'`, requires `ATGALN` and `ATRCPT` (sets `95` if either is zero).
       - If `VNGRRQ = 'N'`, prohibits `ATGALN` and `ATRCPT` (sets `95` if either is non-zero).
       - If `GLAPCD = 'Y'` and `ATAMT > 0`, requires `ATGALN` (sets `95` if zero).
       - If `ATGALN > 0`, requires `GLAPCD = 'Y'` (sets `95` if not).
     - If G/L requires a purchase order (`GLPOCD = 'Y'`, MG08), ensures `ATPONO` is not blank (sets `95` if blank).
     - Validates receipt number (`ATRCPT ≠ 0`):
       - Chains `RCTKEY` to `INFIL1` or `INTZH1`. If not found (`47` and `77`), sets error `95`.
       - Accumulates net quantity (`RNQTY = IHNQTY + IHNQTF - IHAPTQ - IHAPTF`) and A/P quantity (`APQTY = IHAPTQ + IHAPTF`).
       - If `ATGALN > RNQTY`, sets error `95` and `46`.
       - Ensures receipt code (`ATCLCD`) is `'O'` or `'C'` (sets `95` if not).
     - Validates discount:
       - If `ATDISC ≠ 0` or `ATDSPC ≠ 0`, requires a discount G/L (`ACDSGL ≠ 0`, sets `95` if not).
       - If `ATDSD8 ≠ 0` and `ATDSD8 ≤ SYSDT8`, sets warning `96` (MGXX).
       - If terms have no discount (`TBDISC = 0`), prohibits `ATDSPC` or `ATDISC` (sets `95` if non-zero).
       - If terms have a discount (`TBDISC ≠ 0`), ensures `ATDSPC = TBDISC` (sets `95` if not) or requires `ATDISC` and `ATDSPC` to be non-zero (sets `95` if both zero).
     - Validates expense G/L (`ATEXGL`) against `GLMAST`. If not found (`99`), deleted (`GLDEL = 'D'`), or inactive (`GLDEL = 'I'`), sets error `95`.

8. **Output to APLIST (Lines 0195–0340)**:
   - Generates a formatted report:
     - **Header (L2)**: Prints company name (`ACNAME`), page number, date (`SYSDAT`), time (`SYSTIM`), and static text ("ACCOUNTS PAYABLE VOUCHER EDIT").
     - **Detail (01, N51)**: Prints header details (`ATENT#`, `ATVEND`, `ATVNAM`, `ATINV#`, `ATINDT`, `TRMDSC`, `ATAPGL`, `ATDUDT`, `ATDSDT`, `ATBKGL`, `ATSNGL`, `ATRTGL`, `ATRTPC`, `ATCNVO`, `ATHOLD`, `ATHLDD`, `ATSORN`, `ATSSRN`, `ATCAID`, `ATPTYP`).
     - **Detail (02, N51, N52)**: Prints detail line details (`ATPONO`, `ATDDES`, `ATPRAM`, `ATFRAM`, `ATAMT`, `ATDISC`, `NETAMT`, `ATEXGL`, `ATEXCO`, `ATGALN`, `ATRCPT`, `ATCLCD`, `ATJOB#`, `ATCCOD`, `ATCTYP`, `ATJQTY`).
     - **Errors/Warnings**: Prints error (`95`) or warning (`96`) messages with entry number (`ATENT#`).
     - **Totals (L1, N51)**: Prints entry totals (`L1PAMT`, `L1FAMT`, `L1AMT`, `L1DISC`, `L1NET`) and prepaid details (`ATPPCK`, `ATPCKD`).
     - **Totals (L2)**: Prints invoice totals (`L2PAMT`, `L2FAMT`, `L2AMT`, `L2DISC`, `L2NET`), prepaid totals (`L2PPD`, `L2PPA`, `L2PPW`, `L2PPE`), invoice count (`L2CNT`), and vendor hash total (`L2VHSH`).

9. **File Updates**:
   - Writes or updates `APCHKT` with prepaid check records (`PPDREC`).
   - Writes or updates `APSTAT` with error status (`STATAD`, `STATUP`).

10. **Termination**:
    - Processes all records in `APTRAN`, generates the report, and terminates when no more records are found.

---

### Business Rules

1. **Header Validation**:
   - Company number (`ATCONO`) must exist in `APCONT` and not be deleted (`ACDEL ≠ 'D'`).
   - Vendor number (`ATVEND`) must exist in `APVEND`, not be deleted (`VNDEL ≠ 'D'`), not inactive (`VNDEL ≠ 'I'`), and have a non-blank name (`VNVNAM ≠ *BLANKS`).
   - Invoice date (`ATINDT`) must be non-zero and not older than one year prior to the system date (warning only, MG09).
   - Invoice amount (`ATIAMT`) must be non-zero.
   - Invoice number (`ATINV#`) must be non-blank and unique (checked against `APTRNX`, `APINVH`, `APOPNHC` for non-prepaid, non-canceled, non-wire-transfer transactions).
   - A/P G/L (`ATAPGL`) and bank G/L (`ATBKGL`) must exist in `GLMAST`, not be deleted (`GLDEL ≠ 'D'`), and not inactive (`GLDEL ≠ 'I'`).
   - Hold codes (`ATHOLD`) must be `'H'` (hold), `'A'` (ACH), `'W'` (wire transfer), `'E'` (employee expense), or `'U'` (utility auto-payment).
   - Prepaid codes (`ATPAID`) must be `'P'` (prepaid), `'A'` (ACH), `'E'` (employee expense), or `'U'` (utility auto-payment).
   - Single check (`ATSNGL = 'S'`) and retention percentage (`ATRTPC ≠ 0`) are flagged if present.
   - Canceled voucher (`ATCNVO ≠ 0`) is flagged if present.

2. **Detail Validation**:
   - Expense G/L (`ATEXGL`) must exist in `GLMAST`, not be deleted (`GLDEL ≠ 'D'`), and not inactive (`GLDEL ≠ 'I'`).
   - If G/L requires a purchase order (`GLPOCD = 'Y'`), `ATPONO` must be non-blank.
   - Gallons and receipt validation:
     - If vendor requires gallons/receipts (`VNGRRQ = 'Y'`), `ATGALN` and `ATRCPT` must be non-zero.
     - If vendor does not require gallons/receipts (`VNGRRQ = 'N'`), `ATGALN` and `ATRCPT` must be zero.
     - If G/L requires gallons (`GLAPCD = 'Y'`), `ATGALN` must be non-zero for positive amounts (`ATAMT > 0`).
     - If `ATGALN > 0`, G/L must require gallons (`GLAPCD = 'Y'`).
     - Receipt number (`ATRCPT`) must exist in `INFIL1` or `INTZH1`, with sufficient quantity (`ATGALN ≤ RNQTY`) and no prior A/P postings (removed in MG04).
     - Receipt code (`ATCLCD`) must be `'O'` (open) or `'C'` (closed).
   - Discount validation:
     - Discount amount (`ATDISC`) and percentage (`ATDSPC`) cannot both be non-zero.
     - If `ATDISC` or `ATDSPC` is non-zero, a discount G/L (`ACDSGL`) must exist.
     - If terms have no discount (`TBDISC = 0`), `ATDISC` and `ATDSPC` must be zero.
     - If terms have a discount (`TBDISC ≠ 0`), `ATDSPC` must match `TBDISC`, or both `ATDISC` and `ATDSPC` must be non-zero.
     - If discount due date (`ATDSD8`) is non-zero and not later than the system date (`SYSDT8`), a warning (`96`) is issued.

3. **Prepaid Check Validation**:
   - Prepaid check amounts (`ACCKAM`) are updated in `APCHKT` by adding or setting `ATAMT - ATDISC`.
   - Prepaid check details are written to the report.

4. **Error and Warning Handling**:
   - Errors (`95`) are flagged for critical validation failures (e.g., invalid company, vendor, G/L, duplicate invoice).
   - Warnings (`96`) are flagged for non-critical issues (e.g., invoice date older than one year, discount date expired).
   - Errors are written to `APSTAT` (`AXERR = 'Y'`) and reported in `APLIST`.

5. **Reporting**:
   - The report includes headers, detail lines, entry totals, and grand totals for invoices, prepaid amounts, and vendor hash.
   - Errors and warnings are listed with entry numbers for correction.

---

### Tables (Files) Used

The program uses the following files, defined with specific attributes:

1. **APTRAN**:
   - Primary input file (`IP`), 404 bytes, key length 10, contains voucher header and detail records.
   - Fields: `ATHDEL`, `ATCONO`, `ATENT#`, `ATVEND`, `ATCNVO`, `ATAPGL`, `ATIDES`, `ATINDT`, `ATDUDT`, `ATSNGL`, `ATHOLD`, `ATHLDD`, `ATPAID`, `ATPPCK`, `ATVNAM`, `ATVAD1–4`, `ATBKGL`, `ATIAMT`, `ATRTGL`, `ATRTPC`, `ATPCKD`, `ATIND8`, `ATDUD8`, `ATPCK8`, `ATFRTL`, `ATPIVN`, `ATPIIN`, `ATSORN`, `ATSSRN`, `ATCAID`, `ATPTYP`, `ATDSDT`, `ATDSD8`, `ATINV#`, `ATDDEL`, `ATENSQ`, `ATCORD`, `ATEXCO`, `ATEXGL`, `ATDDES`, `ATAMT`, `ATDISC`, `ATDSPC`, `ATITEM`, `ATQTY`, `ATUNMS`, `ATJOB#`, `ATCCOD`, `ATCTYP`, `ATJQTY`, `ATGALN`, `ATRCPT`, `ATCLCD`, `ATPRAM`, `ATFRAM`, `ATPONO`.

2. **APTRNX**:
   - Input file (`IF`), 404 bytes, key length 27, external key, used for duplicate invoice checking.
   - Fields: `XXCO`, `XXENT`, `XXVEND`, `XXINV`.

3. **APOPNHC**:
   - Input file (`IF`), 384 bytes, key length 32, used for duplicate invoice checking.
   - Fields: `OCDEL`, `OCCONO`, `OCVEND`, `OCVONO`, `OCINDS`, `OCINVN`.

4. **APVEND**:
   - Input file (`IF`), 579 bytes, key length 7, used for vendor validation.
   - Fields: `VNDEL`, `VNVNAM`, `VNAD1–4`, `VNGRRQ`, `VNHOLD`, `VNSNGL`, `VNEXGL`, `VNTERM`, `VNCAID`, `VNPRID`, `VNACLS`, `VNACOS`, `VNARTE`, `VNABK#`.

5. **APCONT**:
   - Input file (`IF`), 256 bytes, key length 2, used for company validation.
   - Fields: `ACDEL`, `ACNAME`, `ACAPGL`, `ACCAGL`, `ACDSGL`, `ACNXTE`, `ACJCYN`, `ACRTGL`, `ACPOYN`, `ACEEGL`.

6. **APCHKT**:
   - Update file (`UF`), 80 bytes, key length 21, used for prepaid check validation.
   - Fields: `ACCKAM`, `ATPCKD`, `ATPCK8`, `ATHOLD`.

7. **GLMAST**:
   - Input file (`IF`), 256 bytes, key length 11, used for G/L validation.
   - Fields: `GLDEL`, `GLDESC`, `GLAPCD`, `GLPOCD`.

8. **APINVH**:
   - Input file (`IF`), 64 bytes, key length 32, used for duplicate invoice checking.
   - Fields: `AIKEY`, `AIVONO`.

9. **APSTAT**:
   - Update file (`UF`), 14 bytes, key length 12, used to store error status.
   - Fields: `AXCODE`, `AXCONO`, `AXWSTN`, `AXERR`.

10. **GSTABL**:
    - Input file (`IF`), 256 bytes, key length 12, used for terms validation.
    - Fields: `TBDEL`, `TBDESC`, `TBNETD`, `TBPRXD`, `TBDISC`, `TBADON`, `TBDISD`.

11. **INFIL1**:
    - Input file (`IF`), 448 bytes, key length 9, external key, used for receipt validation.
    - Fields: `IHNQTY`, `IHNQTF`, `IHUNMS`, `IHAPLP`, `IHAPTQ`, `IHAPTF`, `IHAPTD`.

12. **INTZH1**:
    - Input file (`IF`), 592 bytes, key length 9, external key, used for receipt validation.
    - Fields: `IHNQTY`, `IHNQTF`, `IHUNMS`, `IHAPLP`, `IHAPTQ`, `IHAPTF`, `IHAPTD`.

13. **APLIST**:
    - Output printer file (`O`), 164 bytes, used to generate the voucher edit report.
    - Contains headers, detail lines, error/warning messages, and totals.

---

### External Programs Called

- **None**: The `AP110` program does not call any external programs. It operates independently, processing input files, performing validations, and generating the report.

---

### Summary

The `AP110` RPG program validates A/P voucher transactions by processing header and detail records from `APTRAN`. It enforces strict business rules, including company, vendor, G/L, invoice, gallons/receipt, purchase order, and discount validations. Errors (`95`) and warnings (`96`) are flagged and reported in `APLIST`, with error status updated in `APSTAT`. The program supports ACH, wire transfer, employee expense, and utility auto-payment transactions, and includes enhancements for FlexiCapture invoice uploads. It accumulates totals for invoices, prepaid amounts, and vendor hash, producing a comprehensive report for correction. No external programs are called, making it a self-contained validation routine.