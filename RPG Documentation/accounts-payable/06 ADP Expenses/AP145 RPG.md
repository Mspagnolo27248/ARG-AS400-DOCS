The **AP145.rpg36.txt** is an **RPG/36** program (used on IBM System/36 or AS/400 systems) that generates the **Employee Expenses Voucher Selection Spreadsheet and Report** as part of the A/P process. It is the final program called in the OCL script after the second sort (`#GSORT`) and processes sorted payment records from `ADPPAY` to produce reports (`APEEEXP`, `APEEEXPO`) and a disk file (`APEEPY`). Below, I explain the **process steps**, **business rules**, **tables used**, and **external programs called**, referencing the context of the OCL script and prior programs (`AP140`, `AP141`).

---

### **Process Steps of the AP145 RPG Program**

The `AP145` program processes payment records from `ADPPAY` (sorted by company, bank G/L, vendor, prepaid code, check number, and single check code) and `AP145S` (a sorted version of `ADPPAY`) to generate detailed employee expense reports and a summary file. It accumulates totals, validates checks, and handles payment types (checks, ACH, wire transfers, employee expenses, utility auto-pay). The program uses subroutines (`L6DET`, `L4DET`, `CHECK`, `NOPAY`, `EDITCK`) to manage processing.

1. **Program Initialization**:
   - Reads `ADPPAY` as the primary file (`UP`) and `AP145S` as a secondary file (`IR`) with matching records logic.
   - **Level Breaks** (lines 0061–0069):
     - At company level break (`L6`, company change), calls `L6DET` to initialize company-level data and print headers.
     - At vendor level break (`L4`, vendor change), sets indicator 14 and calls `L4DET` to process vendor details.
     - At prepaid code level (`L3`), checks `OPPAID` for `'P'` (check, indicator 25), `'A'` (ACH, 26), `'W'` (wire transfer, 27), `'E'` (employee expense, 28), or `'U'` (utility auto-pay, 29). Sets indicator 11 for prepaid records.
     - At single check level (`L1`, `OPSNGL = 'S'`), sets indicator 10 for single check processing.
   - **Stub Check** (line 0070): If stub is full (`COUNT = 12`, indicator 12) and not prepaid (indicator 11 off), calls `CHECK` to process the check.

2. **Process Each Invoice** (lines 0073–0082):
   - Accumulates totals: `CKGRAM` (gross amount), `CKDISC` (discount), `CKAMT` (payment amount).
   - Calculates negative amount (`NEGAMT = CKAMT * -1`) for reporting.
   - Increments invoice count (`COUNT`).
   - Calls `L4DET` at vendor break (indicator 14 on).
   - Writes detail record to `APEEEXP`/`APEEEXPO` (via `EXCPT`, indicator 80) and updates `ADPPAY` with sequence number (`SEQ#`).
   - Sets overflow indicator (76) if printer overflow occurs (`OF` on).

3. **Check for Full Stub** (lines 0085–0092):
   - If not a single check (`L1` off) and `COUNT = 12` (indicator 12 on), sets full stub condition.
   - For single checks (`L1` on) or non-single checks with full stub, calls `CHECK` to finalize the check.

4. **Company-Level Processing (`L6` Break)** (lines 0095–0109):
   - Sets indicator 86 to print company totals.
   - Writes company total record to `APEEEXP`/`APEEEXPO` (via `EXCPT`).
   - Resets company-level counters (`C6CNT`, `C6GRAM`, `C6DISC`, `C6LPAM`, `P6CNT`, `P6GRAM`, `P6DISC`, `P6LPAM`, `L6CNT`, `L6GRAM`, `L6DISC`, `L6LPAM`).

5. **L6DET Subroutine (Company-Level Processing)** (lines 0112–0126):
   - Initializes page number (`PAGE = 0`) and separator (`SEP = '* '`).
   - Gets current date and time (`TIME`, `DATE`) and converts `DATE` to 8-digit format (`DATE8`).
   - Chains to `APCONT` with `OPCONO` to get company name (`ACNAME`) and pre-numbered check flag (`ACPRE#`).
   - Chains to `ADPYTR` with `'00000'` to get next check number (`PTNXCK`) and payment type (`PTHOLD`).
   - Sets `PAYBY` based on `PTHOLD`:
     - `' '`: "PAY BY CHECK"
     - `'A'`: "PAY BY ACH"
     - `'W'`: "PAY BY WIRE TFR"
     - `'E'`: "PAY BY PAYROLL"
     - `'U'`: "PAY BY UTIL-AUPY"
   - Writes report header to `APEEEXP`/`APEEEXPO` (via `EXCPT`, indicators 76, 77).

6. **L4DET Subroutine (Vendor-Level Processing)** (lines 0128–0144):
   - Constructs vendor key (`VNKEY`) from `OPCONO` and `OPVEND`.
   - Chains to `APVEND` to get vendor name (`VNNAME`) and sort abbreviation (`VNSORT`).
   - If not found (indicator 94 on), chains to `APOPEN` with `OPKEY` (constructed from `VNKEY`, `OPVONO`, and `'3001'`) to get `VNNAME` and `VNSORT`.
   - If still not found, clears `VNNAME` and `VNSORT`.
   - Writes vendor detail to `APEEEXP` (via `EXCPT`, indicator 74) and handles overflow.

7. **CHECK Subroutine (Check Processing)** (lines 0146–0194):
   - Resets indicators 19 (credit/no pay) for non-prepaid and non-full stub cases.
   - Sets check number (`THISCK`):
     - For prepaid (`OPPAID = 'P', 'A', 'W', 'E', 'U'`), uses `OPCKNO`.
     - For credit/no pay (`CKAMT = 0`), sets `THISCK = 0`.
     - Otherwise, uses `PTNXCK` (next check number).
   - If credit/no pay or full stub without pre-numbered checks, calls `NOPAY`.
   - Calls `EDITCK` to validate check.
   - Writes check record to `ADPYCK` (via `EXCPT`, indicator 81).
   - Increments `NXCK` (next check number) unless credit/no pay or full stub.
   - Updates counters:
     - Non-prepaid, non-credit (`C6CNT`, `C6GRAM`, `C6DISC`, `C6LPAM`).
     - Prepaid (`P6CNT`, `P6GRAM`, `P6DISC`, `P6LPAM`).
     - Non-credit (`L6CNT`, `L6GRAM`, `L6DISC`, `L6LPAM`).
   - Resets `CKGRAM`, `CKDISC`, `CKAMT`, `NEGAMT`, `COUNT`, and sets vendor break (indicator 14).

8. **NOPAY Subroutine (Credit/No Pay Processing)** (lines 0196–0217):
   - Handles negative or zero-amount checks by marking related `ADPYCK` records as credit/no pay (`AXRECD = 'C'`).
   - Reads backward through `ADPYCK` starting from `SEQ#` (`CRSEQ#`).
   - For full stub records (`AXRECD = 'F'` or `'V'`), updates `NXCK`, decrements counters (`C6CNT`, `L6CNT`), sets `AXRECD = 'C'`, and clears `AXCHEK`.
   - Writes updated `ADPYCK` record (via `EXCPTNOPAYX`).

9. **EDITCK Subroutine (Check Validation)** (lines 0219–0248):
   - Validates check amount (`CKAMT`):
     - If `CKAMT = 0`, sets indicators 20 and 21 (credit/no pay).
   - Constructs check key (`ATKEY`) from `OPCONO`, `OPBKGL`, and `THISCK`.
   - Chains to `APCHKR` to check if the check exists:
     - For non-void checks (`CKAMT ≠ 0` and not found), ensures `AMCODE ≠ 'O'` (open). If open, sets error indicator 23.
     - For void checks (`CKAMT = 0` and found), ensures `AMCODE = 'O'` and `VOIDAM = AMCKAM`. If not, sets error 23.

10. **File Output** (lines 0251–0470):
    - **ADPPAY**: Updates `SEQ#` for each record.
    - **ADPYCK**:
      - Adds records with `AXRECD` set to `' '` (normal), `'C'` (credit/no pay), `'P'` (prepaid check), `'A'` (ACH), `'W'` (wire), `'E'` (employee expense), `'U'` (utility auto-pay), `'F'` (full stub), or `'V'` (full stub/void).
      - Includes `OPCONO`, `OPBKGL`, `THISCK`, `OPVEND`, `CKAMT`, `PTCKDT` or `OPCKDT`, `VNNAME`, `SEQ#`, `COUNT`.
    - **APEEPY**: Writes summary records with ADP payroll ID (`VNPRID`) and negative amount (`NEGAMT`).
    - **APEEEXP/APEEEXPO**:
      - Prints headers with company name, payment type (`PAYBY`), date, time, and column labels.
      - Prints detail lines with sequence number, invoice number, description, gross amount, discount, partial paid to date, payment amount, due date, vendor, and voucher number.
      - Prints check totals, prepaid indicators, and error messages (e.g., "CHECK IS ALREADY OPEN").
      - Prints company totals with employee count and aggregates.

---

### **Business Rules**

1. **Payment Type Handling**:
   - Processes payments based on `OPPAID`/`PTHOLD`: `'P'` (check), `'A'` (ACH), `'W'` (wire transfer), `'E'` (employee expense), `'U'` (utility auto-pay).
   - Labels payment types in reports (e.g., "PAY BY CHECK", "PAY BY UTIL-AUPY").
2. **Check Number Assignment**:
   - Uses `OPCKNO` for prepaid checks, `PTNXCK` for non-prepaid, or 0 for credit/no pay.
   - Increments `PTNXCK` for new checks unless full stub or credit/no pay.
3. **Stub Limits**:
   - Limits stubs to 12 invoices (`COUNT = 12` triggers full stub).
   - Marks full stubs as `'F'` or `'V'` (void) in `ADPYCK`.
4. **Credit/No Pay**:
   - Negative or zero-amount checks (`CKAMT = 0`) are marked as credit/no pay (`AXRECD = 'C'`) with `AXCHEK = 0`.
   - Adjusts prior full stub records in `ADPYCK` to credit/no pay.
5. **Check Validation**:
   - Non-void checks must not exist in `APCHKR` or must not be open (`AMCODE ≠ 'O'`).
   - Void checks must exist, be open (`AMCODE = 'O'`), and have matching amounts.
6. **Vendor Information**:
   - Retrieves `VNNAME` and `VNSORT` from `APVEND` or `APOPEN` if not found.
   - Uses ADP payroll ID (`VNPRID`) for `APEEPY` output.
7. **Totals and Aggregates**:
   - Tracks company-level (`L6CNT`, `L6GRAM`, `L6DISC`, `L6LPAM`), prepaid (`P6CNT`, `P6GRAM`, `P6DISC`, `P6LPAM`), and check-level (`C6CNT`, `C6GRAM`, `C6DISC`, `C6LPAM`) totals.
   - Resets check-level totals (`CKGRAM`, `CKDISC`, `CKAMT`) after each check.
8. **Report Formatting**:
   - Prints detailed reports with invoice details, check totals, and company summaries.
   - Handles overflow and page breaks.
   - Outputs summary data to `APEEPY` for payroll integration.

---

### **Tables (Files) Used**

The program uses the following files:
1. **ADPPAY**: A/P payment file (226 bytes, update mode, `UP`, indexed by 16 keys). Primary input with payment data (e.g., `OPCONO`, `OPVEND`, `OPVONO`, `OPGRAM`, `OPDISC`, `OPLPAM`, `OPPAID`).
2. **AP145S**: Sorted A/P payment file (3 bytes, input with relations, `IR`). Used for matching records (extension of `ADPPAY`).
3. **APCONT**: A/P control file (256 bytes, input, `IC`, indexed by 2 keys). Contains company data (e.g., `ACNAME`, `ACPRE#`).
4. **ADPYTR**: A/P transaction file (128 bytes, input, `IC`, indexed by 5 keys). Provides next check number (`PTNXCK`) and payment type (`PTHOLD`).
5. **APVEND**: Vendor master file (579 bytes, input, `IC`, indexed by 7 keys). Contains vendor details (e.g., `VNNAME`, `VNSORT`, `VNPRID`).
6. **APOPEN**: A/P open items file (384 bytes, input, `IC`, indexed by 16 keys). Provides vendor name and sort data if not in `APVEND`.
7. **APCHKR**: Check register file (128 bytes, input, `IC`, indexed by 16 keys). Validates check status (`AMCODE`, `AMCKAM`).
8. **ADPYCK**: Check file (96 bytes, update mode, `UC`, indexed by 9 keys). Stores check records (e.g., `AXRECD`, `AXCHEK`).
9. **APEEEXP**: Printer file (142 bytes, output, `O`). Primary employee expense report.
10. **APEEEXPO**: Printer file (142 bytes, output, `O`). Secondary report (paperless).
11. **APEEPY**: Employee expense disk file (74 bytes, output, `O`). Summary for payroll integration.

---

### **External Programs Called**

The `AP145` program does **not** explicitly call any external programs. It is self-contained, relying on internal subroutines (`L6DET`, `L4DET`, `CHECK`, `NOPAY`, `EDITCK`) for processing.

---

### **Summary**

**AP145** generates the final employee expense report and spreadsheet by:
- Processing sorted `ADPPAY` records to accumulate invoice and check totals.
- Validating checks against `APCHKR` and handling credit/no pay cases.
- Retrieving vendor and company data from `APVEND`, `APOPEN`, and `APCONT`.
- Writing check records to `ADPYCK` and summary data to `APEEPY`.
- Printing detailed reports (`APEEEXP`, `APEEEXPO`) with headers, invoice details, check totals, and company summaries.
- Supporting payment types (checks, ACH, wire, employee expense, utility auto-pay).
- Using 11 files and no external programs.

The output (`APEEEXP`, `APEEEXPO`, `APEEPY`) completes the A/P employee expense process defined in the OCL script.