The provided **AP140.rpg36.txt** is an **RPG/36** program (used on IBM System/36 or AS/400 systems) that handles the **A/P Payment Transaction Entry Screen** for processing accounts payable (A/P) transactions, specifically for employee expense payments. It includes modifications for handling 13 accounting periods, ACH/wire transfer selections, and utility auto-pay. Below, I’ll explain the **process steps**, **business rules**, **tables used**, and **external programs called**, referencing the OCL context from the previous query where `AP140` is the first program executed.

---

### **Process Steps of the AP140 RPG Program**

The `AP140` program is an interactive screen-based application that allows users to enter and manage A/P payment transactions. It validates input, retrieves data from control and master files, and updates transaction records. The steps are structured around subroutines (`ONETIM`, `S1`, `S2`, `S2EDIT`, `DTEDIT`, `ROLLFW`, `ROLLBW`) and screen formats (`AP140S1`, `AP140S2`). Here’s a detailed breakdown:

1. **Program Initialization**:
   - **Indicators and Setup** (lines 0133–0137): Resets indicators (81, 82, 90, 93) and clears message fields (`MSG30`, `MSG230`) to prepare for screen interactions.
   - **Roll Key Handling** (lines 0138–0139): Calls `ROLLFW` (roll forward) or `ROLLBW` (roll backward) subroutines if indicators 18 or 19 are set, allowing navigation through transaction records.
   - **End of Job** (lines 0156–0158): If indicator `KG` (end of job) is set, resets indicators 01 and 02, sets `LR` (last record) and `U1`, and exits.

2. **One-Time Setup (`ONETIM` Subroutine)** (lines 0183–0200):
   - **Check Accounting Periods** (lines 0185–0187): Checks `GSCONT` for `GX13GL` (Y/N for 13 accounting periods). Sets indicators 13, 12 if `GX13GL = 'Y'`.
   - **Check for Existing Transactions** (lines 0189–0193): Chains to `ADPYTR` with key `'00000'`. If not found (indicator 81 on), sets add mode (indicator 17 on, 16 off) and initializes sequence number (`NXTSEQ = 1`). If found, proceeds to update mode.
   - **Set Defaults** (lines 0195–0198, 0685–0694): In update mode, sets company number (`CONO = PTCONO`), sequence number (`NXTSEQ = LSTSEQ + 1`), and clears fields (`VEND`, `VO`, `AMT`, `DISC`, `FDIS`, `PORH`, `SNGL`, `MKPP`, `PPCK`, `PPDT`).

3. **Screen 1 Processing (`S1` Subroutine)** (lines 0202–0233):
   - **Validate Company Number** (lines 0204–0207): Chains to `APCONT` using `CONO`. If not found (indicator 91 on), sets error indicator 90, displays message "INVALID COMPANY #" (MSG,1), and jumps to `ENDS1`.
   - **Retrieve or Set Transaction Data** (lines 0209–0226):
     - If no transaction exists (indicator 95 on), sets defaults: `BKGL = ACEEGL` (employee expense G/L from `APCONT`), `KYHOLD = 'E'` (employee expense), `BTCH = 99`, and zeros for `CKDT`, `DATE`, `KYPD`, `KYPDYY`. Clears `FDISC`.
     - If a transaction exists (indicator 95 off), populates screen fields with `ADPYTR` values (`PTBKGL`, `PTBTCH`, `PTCKDT`, `PTDATE`, `PTFDIS`, `PTPD`, `PTPDYY`, `PTHOLD`).
   - **Protect `KYHOLD`** (lines 0218, 0226): Sets indicator 57 to protect `KYHOLD` in update mode (non-editable) or unprotect it in add mode.
   - **Call `S2EDIT`** (line 0228): Validates screen 2 data (even though screen 1 is displayed).
   - **Display Screen 1** (lines 0229–0231): Sets indicator 82 to display `AP140S2` format, clears error indicators and messages if no errors.

4. **Screen 2 Processing (`S2` Subroutine)** (lines 0235–0252):
   - **Validate Input** (line 0237): Calls `S2EDIT` to validate screen 2 fields.
   - **Error Handling** (lines 0238–0239): If error indicator 90 is on, redisplays screen 2 (indicator 82 on) and jumps to `ENDS2`.
   - **Double Enter Check** (lines 0242–0243): If indicator 89 is on (user pressed Enter twice), redisplays screen 2.
   - **Write Transaction** (lines 0245–0247): If no errors, sets indicator 70, writes to `ADPYTR` (via `EXCPT`), and resets indicator 70.

5. **Screen 2 Edit (`S2EDIT` Subroutine)** (lines 0254–0366):
   - **Validate Bank G/L Number** (lines 0256–0266):
     - Compares `SVBKGL` to `BKGL`. If different, updates `SVBKGL`.
     - Chains to `GLMAST` using `GLKEY` (constructed from `CONO`, `BKGL`, and `'C'`). If not found or marked deleted/inactive (`GLDEL = 'D'` or `'I'`), sets error indicator 90 and displays "INVALID BANK G/L #".
   - **Validate Batch Number** (lines 0268–0271): If `BTCH = 0`, sets error indicator 90 and displays "CHECK # CANNOT BE ZERO".
   - **Validate Check Date** (lines 0273–0285):
     - Calls `DTEDIT` to validate `CKDT`. If invalid (indicator 79 on), sets error 90 and displays "INVALID CHECK DATE".
     - Converts `CKDT` to 8-digit format (`CKDT8`) with century handling.
   - **Validate Pay-By Date** (lines 0287–0299):
     - Calls `DTEDIT` to validate `DATE`. If invalid, sets error 90 and displays "INVALID DATE TO PAY BY".
     - Converts `DATE` to 8-digit format (`DATE8`) with century handling.
   - **Validate Force Discount** (lines 0301–0305): If `FDISC` is not blank or `'D'`, sets error 90 and displays "FORCE DISCOUNTS MUST BE 'D'".
   - **Validate Period/Year for 13 Periods** (lines 0307–0362, if indicator 12 on):
     - Checks if `KYPD` is between 1 and 13. If not, sets error 81/90/55 and displays "INVALID PERIOD/YEAR".
     - Chains to `GSTABL` to get period end date (`TBPDDT`) for `KYPD`/`KYPDYY`. If not found, sets error.
     - Validates `CKDT` against period end date (`HIDATE`) and prior period’s end date (`LODATE`). If outside range, sets error and displays "DATE INVALID FOR PD/YR KEYED".
   - **Validate Voucher Payment Type (`KYHOLD`)** (lines JB01, MG03):
     - Ensures `KYHOLD` is `' '`, `'A'`, `'W'`, `'E'`, or `'U'`. If invalid, sets error 56/90 and displays "VOUCHER TO PAY MUST BE ' ',A,W OR E" (or includes `'U'` for utility auto-pay).

6. **Date Edit (`DTEDIT` Subroutine)** (lines 0368–0480):
   - Validates dates (`CKDT`, `DATE`) in MMDDYY format:
     - Breaks down into month, day, year (`$MONTH`, `$DAY`, `$YR`).
     - Validates month (1–12).
     - For February, checks leap year (divisible by 4 or 400 for century years) and ensures day ≤ 29 (leap year) or ≤ 28 (non-leap year).
     - For other months, checks day ≤ 30 (for April, June, September, November) or ≤ 31 (other months).
     - Sets indicator 79 if invalid.

7. **Roll Key Navigation (`ROLLKY`, `ROLLFW`, `ROLLBW` Subroutines)** (lines 0737–0771):
   - **ROLLKY**: Detects roll keys (status codes 01122 for forward, 01123 for backward), sets update mode (indicator 16 on, 17 off).
   - **ROLLFW**: Chains to `ADPYTR` by `SEQ#`, reads next record, and updates `SEQ#` if found.
   - **ROLLBW**: Chains to `ADPYTR`, reads previous record, handles edge case for sequence 0, and updates `SEQ#`.

8. **File Output** (lines 0774–0799):
   - **Update/Delete (`E 70N95`)**: Writes updated `ADPYTR` record with fields like `CONO`, `BKGL`, `BTCH`, `CKDT`, `DATE`, `FDISC`, `KYPDYY`, `KYPD`, `CKDT8`, `DATE8`, `KYHOLD`.
   - **Add (`EADD 70 95`)**: Writes new `ADPYTR` record with sequence number (`Z5`) and same fields.
   - **Delete (`EDEL`)**: Marks record for deletion.
   - **Screen Output** (lines 0844–0860):
     - `AP140S1`: Displays `CONO` and `MSG30`.
     - `AP140S2`: Displays `CONO`, `ACNAME`, `BKGL`, `GLDESC`, `BTCH`, `CKDT`, `DATE`, `FDISC`, `KYPD`, `KYPDYY`, `MSG30`, `KYHOLD`.

---

### **Business Rules**

The program enforces the following business rules:
1. **Company Validation**: Company number (`CONO`) must exist in `APCONT`. Invalid company triggers "INVALID COMPANY #".
2. **Bank G/L Validation**: Bank G/L number (`BKGL`) must exist in `GLMAST` and not be deleted (`GLDEL ≠ 'D'`) or inactive (`GLDEL ≠ 'I'`). Invalid G/L triggers "INVALID BANK G/L #".
3. **Batch Number**: Batch number (`BTCH`) cannot be zero ("CHECK # CANNOT BE ZERO").
4. **Date Validation**:
   - Check date (`CKDT`) and pay-by date (`DATE`) must be valid (MMDDYY format, valid month/day, leap year handling).
   - For 13 accounting periods, `CKDT` must fall within the period defined by `KYPD`/`KYPDYY` in `GSTABL`.
5. **Force Discount**: `FDISC` must be blank or `'D'` ("FORCE DISCOUNTS MUST BE 'D'").
6. **Period/Year**: If 13 periods are enabled (`GX13GL = 'Y'`), `KYPD` must be 1–13, and `CKDT` must align with period start/end dates in `GSTABL`.
7. **Voucher Payment Type (`KYHOLD`)**: Must be `' '` (checks), `'A'` (ACH), `'W'` (wire transfer), `'E'` (employee expense), or `'U'` (utility auto-pay). Invalid values trigger "VOUCHER TO PAY MUST BE ' ',A,W OR E".
8. **Sequence Number**: Managed automatically (`NXTSEQ = LSTSEQ + 1` in update mode, or 1 in add mode).
9. **Navigation**: Supports rolling forward/backward through transactions using sequence numbers.
10. **Error Handling**: Displays specific error messages for invalid inputs and prevents record updates until errors are resolved.

---

### **Tables (Files) Used**

The program uses the following files:
1. **SCREEN**: Workstation file (300 bytes) for interactive screen display (`AP140S1`, `AP140S2`).
2. **ADPYTR**: A/P transaction file (128 bytes, update mode, indexed by 5 keys). Stores payment transactions (e.g., `PTCONO`, `PTBKGL`, `PTBTCH`, `PTCKDT`, `PTDATE`, `PTHOLD`).
3. **APCONT**: A/P control file (256 bytes, input mode, indexed by 2 keys). Contains company data (e.g., `ACNAME`, `ACEEGL`, `ACBKGL`).
4. **GLMAST**: General Ledger master file (256 bytes, input mode, indexed by 11 keys). Contains G/L account details (e.g., `GLDESC`, `GLDEL`).
5. **GSCONT**: General system control file (512 bytes, input mode, indexed by 2 keys). Stores system settings (e.g., `GX13GL` for 13 periods).
6. **GSTABL**: General system table file (256 bytes, input mode, indexed by 12 keys). Stores period end dates (`TBPDDT`).

---

### **External Programs Called**

The `AP140` program does **not** explicitly call any external programs. It is self-contained, relying on internal subroutines (`ONETIM`, `S1`, `S2`, `S2EDIT`, `DTEDIT`, `ROLLKY`, `ROLLFW`, `ROLLBW`) for processing.

---

### **Summary**

**AP140** is an interactive RPG/36 program that facilitates A/P payment transaction entry for employee expenses. It:
- Displays two screens (`AP140S1`, `AP140S2`) for entering company, bank G/L, batch, check date, pay-by date, and payment type.
- Validates inputs against `APCONT`, `GLMAST`, `GSCONT`, and `GSTABL`, ensuring valid company, G/L accounts, dates, and periods.
- Supports add/update/delete modes for `ADPYTR` records, with navigation via roll keys.
- Handles special cases like 13 accounting periods and payment types (`' '`, `'A'`, `'W'`, `'E'`, `'U'`).
- Uses five files (`SCREEN`, `ADPYTR`, `APCONT`, `GLMAST`, `GSCONT`, `GSTABL`) and no external programs.

The program’s output in `ADPYTR` feeds into the subsequent sort and `AP141` processing in the OCL script, as described in the main process flow.