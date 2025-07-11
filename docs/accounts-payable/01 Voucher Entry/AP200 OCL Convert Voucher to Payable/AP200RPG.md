The RPG program `AP200` is an Accounts Payable (A/P) Purchase Register program that processes vendor invoices, vouchers, and related transactions. Below is a detailed analysis of the **process steps**, **business rules**, **tables used**, and **external programs called** based on the provided source code.

---

### Process Steps
The program follows a structured flow to process A/P transactions, including header and detail records, cancellations, intercompany transfers, and updates to various files. The main steps are:

1. **Initialization (ONCE routine)**:
   - Executes once at program start.
   - Converts the journal date (`JRDATE`) to a year-month-day format (`PJYMD`).
   - Handles Y2K logic to determine the century (`JCN`).
   - Initializes variables (e.g., `Z5`, `Z6`, `Z7`, `Z8`, `Z9`, `Z10`, `Z4`, `ONE1`, `ONE3`, `TWO1`, `THREE1`) to zeros or specific values.
   - Sets the `POKEY` to '000' and increments the `ONCE` counter.

2. **Level 2 Processing (L2DET subroutine)**:
   - Initializes page number, captures system time and date.
   - Converts system date (`SYSDAT`) to a year-month-day format (`SYSYMD`).
   - Chains to `APCONT` to retrieve company data and next voucher number (`ACNXVO`).
   - Determines journal ID (`JRNID`) based on wire transfer flag (`WIRE`):
     - 'WT' for wire transfers, 'EE' for employee expenses, or 'PJ' otherwise.
   - Updates `ACJRNL` (journal number) and writes to `APCONT` if not found.
   - Initializes level 2 accumulators (`L2PAMT`, `L2FAMT`, `L2AMT`, `L2DISC`).

3. **Level 1 Processing (L1DET subroutine)**:
   - Initializes level 1 accumulators (`L1PAMT`, `L1FAMT`, `L1AMT`, `L1DISC`, `L1RTAM`, `L1RTDS`).
   - Resets indicators (e.g., 03, 10, 12, 13, 19, 24, 25, 32).
   - Initializes sequence number (`SEQ#`).

4. **Header Record Processing (EACH01 subroutine)**:
   - Processes header records from `APTRAN`.
   - Checks if the voucher is deleted (`ATHDEL = 'D'`):
     - If deleted, updates `FRCFBH` and `FRCINH` to blank `FRAPST` if it was 'Y'.
     - Skips further processing for deleted vouchers.
   - Converts dates (`ATINDT`, `ATDUDT`, `ATPCKD`, `ATDSDT`) to internal format.
   - Validates vendor number (`ATVEND`), canceled voucher (`ATCNVO`), prepaid status (`ATPAID`), single check (`ATSNGL`), and hold codes (`ATHOLD`).
   - Assigns voucher number (`VOUCHR`) from `NXTVO` or retention voucher (`RTVO`) if applicable.
   - Processes prepaid vouchers (check, ACH, wire transfer, employee expense) by updating `APPYTR`.
   - Handles canceled vouchers by calling the `CANCEL` subroutine.
   - Checks for retention (`ATRTPC`) and calculates retention amounts if applicable.

5. **Detail Record Processing (EACH02 subroutine)**:
   - Processes detail records from `APTRAN`.
   - Handles gallons (`ATGALN`) and receipt number (`ATRCPT`) for printing.
   - Calculates discounts based on discount percent (`ATDSPC`) and amount (`ATAMT`).
   - Processes retention amounts (`ATRTAM`, `ATRTDS`) if applicable.
   - Updates payment transaction records (`APPYTR`) for discounts.
   - Accumulates amounts (`L1AMT`, `L1PAMT`, `L1FAMT`, `L1DISC`, `L1RTAM`, `L1RTDS`).
   - Checks for intercompany transfers by calling `INTRCO` if company numbers differ (`ATCONO ≠ ATEXCO`).
   - Updates purchase order files (`POFILEH`, `POFILED`) if `ACPOYN = 'Y'` and `ATPONO` is not blank.

6. **Cancel Voucher Processing (CANCEL subroutine)**:
   - Processes canceled vouchers (`ATCNVO`).
   - Builds a key (`OPKY12`) using company and vendor data.
   - Reads `APOPEN` to find matching records.
   - Updates `APOPENH`, `APOPEND`, or `APOPENV` based on record type (`OPRCTY`).
   - Writes history records to `APHISTH`, `APHISTD`, or `APHISTV`.

7. **Level 1 Totals (L1TOT subroutine)**:
   - Accumulates level 1 totals into level 2 totals (`L2PAMT`, `L2FAMT`, `L2AMT`, `L2DISC`).
   - Updates vendor totals (`VN$YTD`, `VNPURC`, `VNCBAL`) in `APVEND`.
   - Writes invoice header (`APINVH`) if vendor is not zero.
   - Writes totals to `APPRINT`.

8. **Intercompany Transfers (INTRCO subroutine)**:
   - Processes intercompany transfers when company numbers differ.
   - Sets debit and credit company codes (`IDRCO`, `ICRCO`) and G/L accounts (`IDRGL`, `ICRGL`).
   - Writes journal entries to `APPJJR` for debit and credit sides.
   - Handles retention amounts separately if applicable.

9. **Output Processing**:
   - Writes records to output files (`APOPENH`, `APOPEND`, `APOPENV`, `APHISTH`, `APHISTD`, `APHISTV`, `APPJJR`, `APPYTR`, `APVEND`, `APINVH`, `POFILEH`, `POFILED`, `FRCINH`, `FRCFBH`).
   - Generates a purchase register report via `APPRINT`.

---

### Business Rules
The program enforces several business rules to ensure accurate A/P processing:

1. **Voucher Number Assignment**:
   - Voucher numbers are assigned from `ACNXVO` in `APCONT` and incremented.
   - Retention vouchers (`RTVO`) are assigned separately if `ATRTPC` is non-zero.
   - Canceled vouchers use the original voucher number (`ATCNVO`).

2. **Prepaid Vouchers**:
   - Prepaid vouchers are flagged with `ATPAID = 'P'` (check), `'A'` (ACH), `'W'` (wire transfer), or `'E'` (employee expense).
   - Payment transactions are written to `APPYTR` for prepaid vouchers.

3. **Hold Vouchers**:
   - Vouchers can be held with `ATHOLD = 'H'` (hold), `'A'` (ACH), `'W'` (wire transfer), `'E'` (employee expense), or `'U'` (utility auto-pay).
   - Hold descriptions (`ATHLDD`) are printed for held vouchers.

4. **Discounts**:
   - Discounts are calculated if `ATDSPC` (discount percent) is non-zero.
   - Discount amount (`ATDISC`) is computed as `ATAMT * (ATDSPC / 100)`.
   - Discounts are accumulated in `L1DISC` and written to `APPYTR`.

5. **Retention**:
   - Retention is processed if `ATRTPC` (retention percent) is non-zero.
   - Retention amount (`ATRTAM`) is calculated as `ATAMT * (ATRTPC / 100)`.
   - Retention vouchers are written to `APOPENH`, `APOPEND`, and `APOPENV` with a hold code (`'H'`) and description ('RETENTION').

6. **Intercompany Transfers**:
   - If `ATCONO ≠ ATEXCO`, intercompany journal entries are written to `APPJJR`.
   - Debit and credit entries use the intercompany G/L account (`ACICGL`).

7. **Purchase Order Integration**:
   - If `ACPOYN = 'Y'` and `ATPONO` is non-blank, updates `POFILEH` (header) and `POFILED` (detail).
   - Updates applied amount (`POAPPU`), received quantity (`PDRCQT`), and A/P voucher amount (`PDAPV$`).

8. **Deleted Vouchers**:
   - If `ATHDEL = 'D'`, blanks `FRAPST` in `FRCINH` or `FRCFBH` if it was 'Y'.
   - Skips further processing for deleted vouchers.

9. **Single Check and Canceled Vouchers**:
   - Single check vouchers are flagged with `ATSNGL = 'S'`.
   - Canceled vouchers are processed by updating `APOPEN` and writing to `APHIST`.

10. **Y2K Date Handling**:
    - Adjusts century for dates based on `Y2KCMP` and `Y2KCEN`.

11. **Journal ID Assignment**:
    - Assigns `JRNID` as 'WT' for wire transfers, 'EE' for employee expenses, or 'PJ' otherwise.

### Business Rules

1. **Voucher Processing**:
   - Processes header (`NS 01`) and detail (`NS 02`) records from `APTRAN`, skipping deleted vouchers (`ATHDEL = 'D'`).
   - Assigns new voucher numbers (`NXTVO`) for non-canceled, non-100% retention vouchers and retention vouchers.
   - Supports multiple payment types: prepaid (`P`), ACH (`A`), wire transfer (`W`), employee expense (`E`), and utility auto-pay (`U`).

2. **Freight Invoice Handling**:
   - Clears `FRAPST` to blank in `FRCINH` or `FRCFBH` if a voucher is deleted and `FRAPST = 'Y'`.
   - Prioritizes `FRCFBH` (freight bill override header) over `FRCINH` (carrier invoice header) when checking freight status.

3. **Discounts and Retentions**:
   - Calculates discounts if `ATDSPC ≠ 0` and `ATDISC = 0` (`ATDISC = ATAMT * (ATDSPC / 100)`).
   - For retentions (`ATRTPC ≠ 0`), computes retention amount (`ATRTAM`) and adjusts `ATAMT`. For 100% retention, moves `ATAMT` to `ATRTAM` and zeros `ATAMT`.

4. **Intercompany Transfers**:
   - Generates journal entries (`APPJJR`) for intercompany transactions (`ATCONO ≠ ATEXCO`) using intercompany G/L accounts (`ACICGL`).

5. **Cancellation**:
   - Marks canceled vouchers (`ATCNVO ≠ *ZEROS`) as deleted (`'D'`) in `APOPENH`, `APOPEND`, `APOPENV` and writes history records (`APHISTH`, `APHISTD`, `APHISTV`).

6. **Vendor and Invoice Updates**:
   - Updates vendor balances (`VN$YTD`, `VNPURC`, `VNCBAL`) with voucher and retention amounts.
   - Records invoice details in `APINVH` for non-one-time vendors.

7. **Journal Entries**:
   - Generates `APPJJR` entries for A/P, expense, and intercompany accounts, including retention and non-retention transactions.

8. **Purchase Order Updates**:
   - Disabled (skipped via `GOTO SKIP`), but intended to update `POFILEH` (`POAPPU`) and `POFILED` (`PDRCQT`, `PDAPV$`, `PDRCDT`, ` PDCOMP`).

9. **Reporting**:
   - Produces a detailed Purchase Register with company, voucher, and line item details, including special fields like sales order, carrier ID, and process type.

---

### Tables Used
The program interacts with the following files (tables):

| File Name  | Type | Description                              | Usage                                      |
|------------|------|------------------------------------------|--------------------------------------------|
| `APTRAN`   | Input | A/P Transaction File                     | Reads header and detail records            |
| `APCONT`   | Update| A/P Control File                         | Retrieves company data, updates `ACJRNL`, `ACNXVO` |
| `APVEND`   | Update| A/P Vendor File                          | Updates vendor totals (`VN$YTD`, `VNPURC`, `VNCBAL`) |
| `APOPEN`   | Input | A/P Open File                            | Reads open vouchers for cancellation       |
| `APOPENH`  | Update| A/P Open Header File                     | Writes/updates header records              |
| `APOPEND`  | Update| A/P Open Detail File                     | Writes detail records                      |
| `APOPENV`  | Update| A/P Open Vendor File                     | Writes vendor records                      |
| `APINVH`   | Input | A/P Invoice Header File                  | Writes invoice header records              |
| `POFILEH`  | Update| Purchase Order Header File               | Updates applied amounts (`POAPPU`)         |
| `POFILED`  | Update| Purchase Order Detail File               | Updates received qty (`PDRCQT`), voucher amounts |
| `APHISTH`  | Output| A/P History Header File                  | Writes history header records              |
| `APHISTD`  | Output| A/P History Detail File                  | Writes history detail records              |
| `APHISTV`  | Output| A/P History Vendor File                  | Writes history vendor records              |
| `APPJJR`   | Output| A/P Journal File                         | Writes journal entries                     |
| `APPYTR`   | Update| A/P Payment Transaction File             | Writes/updates payment transactions        |
| `FRCINH`   | Update| Freight Invoice Header File              | Updates `FRAPST` for deleted vouchers      |
| `FRCFBH`   | Update| Freight Bill Override Header File        | Updates `FRAPST` for deleted vouchers      |
| `APPRINT`  | Output| A/P Purchase Register Report             | Generates the purchase register report     |

---

### External Programs Called
The program does not explicitly call external programs using the `CALL` opcode. All processing is handled within the program through subroutines and file operations. The subroutines used are:

- `L2DET`: Level 2 detail processing.
- `L1DET`: Level 1 detail processing.
- `EACH01`: Header record processing.
- `EACH02`: Detail record processing.
- `CANCEL`: Cancel voucher processing.
- `L1TOT`: Level 1 totals processing.
- `INTRCO`: Intercompany transfer processing.

No external programs are invoked, as the program is self-contained for A/P processing.

---

### Summary
The `AP200` RPG program is a comprehensive A/P Purchase Register system that processes vendor invoices, assigns voucher numbers, handles prepaid and held vouchers, calculates discounts and retentions, and supports intercompany transfers and purchase order integration. It uses 16 files for input, update, and output operations and enforces strict business rules for data integrity. No external programs are called, and all logic is managed via internal subroutines.