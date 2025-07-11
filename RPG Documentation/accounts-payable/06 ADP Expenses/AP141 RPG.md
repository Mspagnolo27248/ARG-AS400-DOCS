The **AP141.rpg36.txt** is an **RPG/36** program (used on IBM System/36 or AS/400 systems) that processes A/P payment transactions from the `ADPYTR` file (created by `AP140`) to generate payment records in the `ADPPAY` file, matching them against open payables in the `APOPEN` file. It is called in the OCL script after the first sort (`#GSORT`) and is part of the A/P Employee Expenses Report and Spreadsheet process. Below, I explain the **process steps**, **business rules**, **tables used**, and **external programs called**, referencing the context of the OCL script and prior programs (`AP140`).

---

### **Process Steps of the AP141 RPG Program**

The `AP141` program reads transaction records from `ADPYTR`, matches them with open payables in `APOPEN`, and creates or updates payment records in `ADPPAY`. It handles two types of transaction records (distinguished by `NS 01` and `NS 02`) and supports payment methods like checks, ACH, wire transfers, employee expenses, and utility auto-pay. The program is structured around two main subroutines: `EACH01` (for pay-by-date records) and `EACH02` (for vendor-specific records).

1. **Program Initialization**:
   - The program reads `ADPYTR` records sequentially (defined as input primary file, `IP`) and processes them based on their type (`NS 01` or `NS 02`).
   - **Record Type Check** (lines 0067–0069):
     - For `NS 01` records (pay by date), calls `EACH01` subroutine.
     - For `NS 02` records (pay by vendor/voucher), checks if `PTDEL = 'D'` (deleted). If deleted, calls `EACH02`. Otherwise, processes normally.

2. **EACH01 Subroutine (Pay by Date)** (lines 0072–0162):
   - **Date Conversion** (lines 0074–0092):
     - Converts `PTCKDT` (check date) and `PTDATE` (pay-by date) to 8-digit format (`CKYMD8`, `PTDAT8`) with century handling using `Y2KCEN` and `Y2KCMP`.
   - **Check Pay-By Date** (line 0094): If `PTDATE ≠ 0`, proceeds to match open payables; otherwise, skips to `END01`.
   - **Set Up `APOPEN` Read** (lines 0099–0101):
     - If `PTFDIS = 'D'`, sets force discount flag (indicator 10).
     - Sets lower limit (`OPLIM`) with `PTCONO` for `APOPEN` read.
   - **Read `APOPEN` Loop** (lines 0103–0160, `AGN01` tag):
     - Reads `APOPEN` records, skipping detail records (indicator 06), deleted records (`OPDEL = 'D'`), or halted records (`OPHALT = 'H'`).
     - Validates:
       - Company number (`OPCONO = PTCONO`).
       - Bank G/L number (`OPBKGL = PTBKGL`).
       - Payment type (`PTHOLD` vs. `OPHALT`):
         - If `PTHOLD = ' '`, selects records where `OPHALT ≠ 'A', 'W', 'E', 'U'` (checks).
         - If `PTHOLD = 'A'`, selects only `OPHALT = 'A'` (ACH).
         - If `PTHOLD = 'W'`, selects only `OPHALT = 'W'` (wire transfer).
         - If `PTHOLD = 'E'`, selects only `OPHALT = 'E'` (employee expense).
         - If `PTHOLD = 'U'`, selects only `OPHALT = 'U'` (utility auto-pay).
     - Converts due date (`OPDUED`) to 8-digit format (`DTYMD8`).
     - For prepaid vouchers (`OPPAID = 'P'` and `PTHOLD = ' '`), clears `OPCKNO` and `OPCKDT`.
     - Skips vouchers with due date after `PTDATE` (`DTYMD8 > PTDAT8`).
     - For ACH, wire, employee expense, or utility auto-pay (`PTHOLD = 'A', 'W', 'E', 'U'`), sets `OPPAID`, `OPCKNO`, and `OPCKDT` accordingly.
   - **Calculate Payment Amount** (lines 0142–0149):
     - If voucher is past due (`DTYMD8 > CKYMD8`) or force discount is off, sets `OPDISC = 0`.
     - If partially paid (`OPPPTD ≠ 0`), sets `OPDISC = 0`.
     - Calculates last payment amount (`OPLPAM = OPGRAM - OPDISC - OPPPTD`).
   - **Handle One-Time Vendor** (lines 0151–0152): If `OPVEND = 0`, sets `OPSNGL = 'S'` (single check).
   - **Write `ADPPAY`** (lines 0154–0159):
     - Chains to `ADPPAY` with `OPKEY`. If not found (indicator 89 on), adds new record; otherwise, updates existing record.
     - Writes record with `OPREC`, `OPDISC`, `OPCKNO`, `OPPAID`, `OPSNGL`, `OPBKGL`, `OPLPAM`, `OPCKDT`, `PTSEQ#`.

3. **EACH02 Subroutine (Pay by Vendor/Voucher)** (lines 0164–0281):
   - **Validate Input** (lines 0168–0171):
     - Sets force discount flag if `PTFDIS = 'D'` (indicator 10).
     - Checks if paying whole vendor (`PTVO = 0`, indicator 12).
     - Checks if partial payment (`PTAMT ≠ 0`, indicator 14).
     - Checks if vendor/voucher is on hold (`PTPORH = 'H'`, indicator 15).
   - **Set Up `APOPEN` Read** (lines 0172–0178):
     - Constructs `OPLIM` with `PTCONO`, `PTVEND`, and `PTVO` for `APOPEN` read.
   - **Read `APOPEN` Loop** (lines 0180–0269, `AGN02` tag):
     - Reads `APOPEN` records, skipping detail records (indicator 06), deleted records (`OPDEL = 'D'`), or mismatched company (`OPCONO ≠ PTCONO`) or vendor (`OPVEND ≠ PTVEND`).
     - For whole vendor (`PTVO = 0`), ensures `OPBKGL = PTBKGL`. For specific voucher, ensures `OPVONO = PTVO`.
     - Validates payment type (`PTHOLD` vs. `OPHALT`) as in `EACH01`.
     - For prepaid vouchers (`PTMKPP ≠ ' '`), ensures `OPPAID` matches `PTHOLD` (e.g., `'A'` for ACH).
     - For held vouchers (`OPHALT = 'H'`), requires `PTPORH = 'P'` to pay.
     - Converts dates and sets `OPPAID`, `OPCKNO`, `OPCKDT` for ACH, wire, employee expense, or utility auto-pay.
   - **Calculate Payment Amount** (lines 0242–0258):
     - Applies override discount (`PTDISC`) if provided (`OPDISC = PTDISC`).
     - Adjusts `OPDISC` for past due or partially paid vouchers.
     - Calculates `OPLPAM = OPGRAM - OPDISC - OPPPTD`.
     - For partial payments (`PTAMT ≠ 0`), adjusts `OPLPAM` and `PTAMT` accordingly.
   - **Handle Single Check and One-Time Vendor** (lines 0260–0265):
     - Sets `OPSNGL = PTSNGL` if provided, or `'S'` for one-time vendors (`PTVEND = 0`).
   - **Write/Delete `ADPPAY`** (lines 0267–0279):
     - Chains to `ADPPAY` with `OPKEY`.
     - If on hold (`PTPORH = 'H'`), marks `ADPPAY` record for deletion (`PYDEL = 'D'`).
     - Otherwise, adds or updates `ADPPAY` record with fields as in `EACH01`.

4. **File Output** (lines 0284–0303):
   - **Add (`EADD 80 89`)**: Writes new `ADPPAY` record with `OPREC`, `OPDISC`, `OPCKNO`, `OPPAID`, `OPSNGL`, `OPBKGL`, `OPLPAM`, `OPCKDT`, `PTSEQ#`.
   - **Update/Delete (`E 80N89`)**: Updates existing `ADPPAY` record, setting `PYDEL` to `'D'` for deletion or `' '` for update.

---

### **Business Rules**

The program enforces the following business rules:
1. **Record Selection**:
   - Skips deleted records (`OPDEL = 'D'`), detail records, or halted records (`OPHALT = 'H'`) unless explicitly set to pay (`PTPORH = 'P'`).
   - Matches company number (`OPCONO = PTCONO`) and bank G/L number (`OPBKGL = PTBKGL`).
   - Matches payment type (`PTHOLD` vs. `OPHALT`):
     - `' '` (checks): Selects non-ACH/wire/employee/utility vouchers.
     - `'A'` (ACH), `'W'` (wire), `'E'` (employee expense), `'U'` (utility): Selects matching `OPHALT`.
2. **Pay by Date (`EACH01`)**:
   - Processes vouchers with due date (`OPDUED`) ≤ pay-by date (`PTDATE`).
   - Clears check number/date (`OPCKNO`, `OPCKDT`) for prepaid vouchers if paying by check.
   - Sets `OPPAID`, `OPCKNO`, `OPCKDT` for ACH/wire/employee/utility payments.
3. **Pay by Vendor/Voucher (`EACH02`)**:
   - Matches vendor (`OPVEND = PTVEND`) and, if specified, voucher (`OPVONO = PTVO`).
   - For whole vendor (`PTVO = 0`), ensures bank G/L match.
   - Allows partial payments (`PTAMT`) and override discounts (`PTDISC`).
   - Deletes `ADPPAY` records for held vendors/vouchers (`PTPORH = 'H'`).
4. **Discount Handling**:
   - Applies force discount (`PTFDIS = 'D'`) unless voucher is past due or partially paid.
   - Uses override discount (`PTDISC`) if provided.
   - Sets discount to 0 for past due or partially paid vouchers.
5. **Payment Amount**:
   - Calculates payment amount (`OPLPAM = OPGRAM - OPDISC - OPPPTD`).
   - Adjusts for partial payments, ensuring `PTAMT` does not exceed remaining amount.
6. **Single Check and One-Time Vendor**:
   - Sets `OPSNGL = 'S'` for one-time vendors (`OPVEND = 0`) or if specified (`PTSNGL ≠ ' '`).
7. **Prepaid Vouchers**:
   - Allows prepayment only if `OPPAID` matches `PTHOLD` (e.g., `'A'` for ACH).
   - Sets `OPPAID`, `OPCKNO`, `OPCKDT` for prepaid records.

---

### **Tables (Files) Used**

The program uses the following files:
1. **ADPYTR**: A/P transaction file (128 bytes, input primary, `IP`). Contains transaction data (e.g., `PTCONO`, `PTBKGL`, `PTCKDT`, `PTDATE`, `PTHOLD`, `PTVEND`, `PTVO`).
2. **APOPEN**: A/P open items file (384 bytes, input with database relations, `ID`, indexed by 16 keys). Contains open payables (e.g., `OPCONO`, `OPVEND`, `OPVONO`, `OPGRAM`, `OPDISC`, `OPPAID`).
3. **ADPPAY**: A/P payment file (226 bytes, update mode, `UC`, indexed by 16 keys). Stores payment records (e.g., `PYDEL`, `OPREC`, `OPDISC`, `OPCKNO`, `OPPAID`, `OPSNGL`, `OPLPAM`).

---

### **External Programs Called**

The `AP141` program does **not** explicitly call any external programs. It is self-contained, relying on internal subroutines (`EACH01`, `EACH02`) for Rosettacut 4 times: [System](#), [System](#), [System](#), [System](#) for processing.

---

### **Summary**

**AP141** processes `ADPYTR` transactions to create or update `ADPPAY` records by matching against `APOPEN`. It:
- Handles two transaction types: pay by date (`EACH01`) and pay by vendor/voucher (`EACH02`).
- Validates company, bank G/L, and payment type (`PTHOLD` vs. `OPHALT`).
- Manages discounts, partial payments, and prepaid vouchers.
- Supports checks, ACH, wire transfers, employee expenses, and utility auto-pay.
- Uses three files (`ADPYTR`, `APOPEN`, `ADPPAY`) and no external programs.
- Outputs `ADPPAY` records for the next sort and `AP145` processing in the OCL script.