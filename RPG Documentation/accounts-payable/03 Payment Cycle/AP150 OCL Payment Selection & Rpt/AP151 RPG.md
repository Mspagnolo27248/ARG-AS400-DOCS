The `AP151.rpg36.txt` file is an RPG III program called by the `AP150.ocl36.txt` OCL script, specifically invoked after sorting payment transactions in the `APPYTR` file. Its primary purpose is to create payment records in the `APPAY` file and handle discount tracking in the `APPYDS` file for Accounts Payable (A/P) transactions. Below, I’ll detail the **process steps**, **business rules**, **tables used**, and **called programs** based on the provided code.

---

### **Process Steps of the AP151 RPG Program**

The `AP151` program processes sorted payment transaction records from `APPYTR` to generate payment records in `APPAY`, applying logic for payment selection, discount handling, and prepaid vouchers. It operates in two main modes based on the record type in `APPYTR`: paying by date (`EACH01`) or paying by vendor/voucher (`EACH02`). Here’s a step-by-step breakdown:

1. **Read APPYTR Records**:
   - The program reads records from the `APPYTR` file, which contains payment transaction data sorted by company, vendor/voucher, and sequence number.
   - Records are processed in two formats:
     - **Format 01**: Header records (pay by date, no specific vendor/voucher).
     - **Format 02**: Detail records (pay by specific vendor/voucher).

2. **Process Header Records (`EACH01` Subroutine)**:
   - Converts check date (`PTCKDT`) and pay-by date (`PTDATE`) to 8-digit formats (`CKYMD8`, `PTDAT8`) with century handling for Y2K compliance.
   - For each header record:
     - If the record is not marked for deletion (`PTDEL ≠ 'D'`), processes all applicable vouchers in `APOPEN`.
     - Filters `APOPEN` records based on:
       - Matching company number (`OPCONO = PTCONO`).
       - Matching bank G/L number (`OPBKGL = PTBKGL`).
       - Payment method (`PTHOLD` must match `OPHALT`: `' '` for checks, `A` for ACH, `W` for wire transfer, `E` for employee expenses, `U` for utility auto-pay).
       - Not deleted (`OPDEL ≠ 'D'`) or on hold (`OPHALT ≠ 'H'`).
     - For prepaid vouchers (`OPPAID = 'P'`, `'A'`, `'W'`, `'E'`, or `'U'`), ensures the payment method matches and processes them directly.
     - For non-prepaid vouchers, checks discount eligibility:
       - If force discount (`PTFDIS = 'D'`), applies the discount (`OPDISC`).
       - Otherwise, compares the discount due date (`OPDSD8`) and due date (`DTYMD8`) against the check date (`CKYMD8`) and pay-by date (`PTDAT8`).
       - If the discount due date is valid (on or after check date and before or on pay-by date), applies the discount.
       - If the discount due date is missed, records the voucher in `APPYDS` for tracking missed discounts and sets the discount to zero.
     - Calculates the payment amount (`OPLPAM = OPGRAM - OPDISC - OPPPTD`).
     - For one-time vendors (`OPVEND = 0`), sets the single check flag (`OPSNGL = 'S'`).
     - Writes or updates the `APPAY` record with payment details.

3. **Process Detail Records (`EACH02` Subroutine)**:
   - For detail records (specific vendor/voucher):
     - If marked for deletion (`PTDEL = 'D'`), skips processing.
     - Filters `APOPEN` records based on:
       - Matching company number (`OPCONO = PTCONO`).
       - Matching vendor number (`OPVEND = PTVEND`).
       - Matching voucher number (`OPVONO = PTVO`) if paying a specific voucher.
       - Matching bank G/L number (`OPBKGL = PTBKGL`) if paying the whole vendor.
       - Payment method (`PTHOLD` matches `OPHALT`).
       - Not deleted (`OPDEL ≠ 'D'`).
     - Handles hold status (`PTPORH = 'H'` to skip, `'P'` to pay).
     - For prepaid vouchers, ensures the payment method matches (`PTMKPP` and `OPPAID`).
     - Applies discounts based on `PTDISC` if provided, or sets to zero if the header discount is zero or the due date is past.
     - Calculates the payment amount, adjusting for partial payments (`PTAMT`) if specified.
     - Writes or updates the `APPAY` record, marking it for deletion if held (`PYDEL = 'D'`).

4. **Discount Tracking**:
   - If a discount is missed (check date past discount due date and no force discount), writes a record to `APPYDS` to track the missed discount (`EXCPTMISDIS`).

5. **Output to APPAY and APPYDS**:
   - Writes payment records to `APPAY` with fields like payment amount (`OPLPAM`), discount (`OPDISC`), check number (`OPCKNO`), check date (`OPCKDT`), and payment method (`OPPAID`).
   - Writes missed discount records to `APPYDS` with similar fields for tracking purposes.

6. **Loop and Termination**:
   - Continues processing `APPYTR` records until all are read.
   - Terminates when no more records are available in `APPYTR`.

---

### **Business Rules in the AP151 RPG Program**

The program enforces several business rules to ensure accurate payment processing and discount handling:

1. **Payment Method Matching**:
   - The payment method in `APPYTR` (`PTHOLD`) must match the voucher's hold code in `APOPEN` (`OPHALT`):
     - `' '` (checks), `A` (ACH), `W` (wire transfer), `E` (employee expenses), `U` (utility auto-pay).
     - Mismatches skip the voucher.

2. **Prepaid Voucher Handling**:
   - Prepaid vouchers (`OPPAID = 'P'`, `'A'`, `'W'`, `'E'`, or `'U'`) are paid only if the payment method matches `PTHOLD`.
   - Prepaid check number (`OPCKNO`) and date (`OPCKDT`) are set to the transaction’s check date (`PTCKDT`).

3. **Discount Eligibility**:
   - If force discount is set (`PTFDIS = 'D'`), the discount (`OPDISC`) is applied regardless of the discount due date.
   - Otherwise, discounts are applied only if:
     - The discount due date (`OPDSD8`) is on or after the check date (`CKYMD8`) and before or on the pay-by date (`PTDAT8`).
     - If the discount due date is missed, the discount is set to zero, and a record is written to `APPYDS`.
   - If the voucher is partially paid (`OPPPTD > 0`) or past due, the discount is set to zero unless forced.

4. **Due Date Validation**:
   - Vouchers are selected for payment only if their due date (`DTYMD8`) is on or before the pay-by date (`PTDAT8`), unless force discount is applied.

5. **Partial Payments**:
   - For detail records with a partial payment amount (`PTAMT > 0`), the payment amount (`OPLPAM`) is set to `PTAMT`, and the discount is adjusted accordingly.
   - If the partial payment amount equals the remaining amount (`OPLPAM`), the discount is set to zero.

6. **One-Time Vendors**:
   - For one-time vendors (`OPVEND = 0`), the single check flag (`OPSNGL`) is set to `'S'`.

7. **Hold Status**:
   - Vouchers on hold (`OPHALT = 'H'`) are skipped unless explicitly marked to pay (`PTPORH = 'P'`).
   - Detail records with `PTPORH = 'H'` are marked for deletion in `APPAY` (`PYDEL = 'D'`).

8. **Company and Bank G/L Matching**:
   - The company number (`OPCONO`) and bank G/L number (`OPBKGL`) must match the transaction’s values (`PTCONO`, `PTBKGL`).

9. **Missed Discount Tracking**:
   - If a discount is available but cannot be taken (check date past discount due date), a record is written to `APPYDS` to track the missed discount.

---

### **Tables (Files) Used**

The program interacts with the following files:

1. **APPYTR** (`IP`, Input Primary, 128 bytes):
   - Payment transaction file containing sorted header and detail records.
   - Fields include `PTCONO` (company), `PTVEND` (vendor), `PTVO` (voucher), `PTAMT` (payment amount), `PTDISC` (discount), `PTFDIS` (force discount), `PTHOLD` (payment method), etc.

2. **APOPEN** (`ID`, Input with Disk, 384 bytes):
   - Open A/P file containing voucher details.
   - Fields include `OPCONO` (company), `OPVEND` (vendor), `OPVONO` (voucher), `OPGRAM` (gross amount), `OPDISC` (discount), `OPDSDT` (discount due date), `OPDUED` (due date), `OPHALT` (hold code), `OPPAID` (prepaid code), etc.

3. **APPAY** (`UC`, Update/Create, 384 bytes):
   - Output file for payment records.
   - Fields include `PYDEL` (delete flag), `OPLPAM` (payment amount), `OPDISC` (discount), `OPCKNO` (check number), `OPPAID` (payment method), `OPSNGL` (single check), `PTSEQ#` (sequence number), etc.

4. **APPYDS** (`O`, Output, 384 bytes):
   - Output file for tracking missed discounts.
   - Fields mirror `APPAY`, including `OPREC`, `OPDISC`, `OPCKNO`, `OPPAID`, `OPSNGL`, `OPBKGL`, `OPLPAM`, `PTSEQ#`.

---

### **Called Programs**

The `AP151` program does not explicitly call any external programs. It is a standalone RPG program that processes input from `APPYTR` and `APOPEN`, writing output to `APPAY` and `APPYDS`. The OCL script (`AP150.ocl36.txt`) invokes `AP151` after sorting `APPYTR` with `#GSORT`, but no additional programs are called within `AP151` itself.

---

### **Summary**

The `AP151` RPG program processes sorted payment transactions from `APPYTR` to create payment records in `APPAY` and track missed discounts in `APPYDS`. It handles two types of transactions: pay-by-date (header) and pay-by-vendor/voucher (detail). The program enforces strict business rules for payment method matching, discount eligibility, due date validation, and prepaid voucher handling. It supports multiple payment methods (checks, ACH, wire transfers, employee expenses, utility auto-pay) and ensures accurate payment calculations while tracking missed discounts for reporting. The program interacts with four files (`APPYTR`, `APOPEN`, `APPAY`, `APPYDS`) and does not call external programs.