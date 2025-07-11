The `AP155.rpg36.txt` file is an RPG III program called by the `AP150.ocl36.txt` OCL script, designed to generate a **Cash Requirements Report** for Accounts Payable (A/P) transactions. It processes payment records from the `APPAY` file, updates invoice details in `APDETINV`, and produces a formatted report in `APCSHRQ`. Below, I detail the **process steps**, **business rules**, **tables used**, and **external programs called** based on the provided code.

---

### **Process Steps of the AP155 RPG Program**

The `AP155` program generates a cash requirements report by processing payment records from `APPAY`, aggregating totals, validating checks, and updating invoice details. It handles both prepaid and non-prepaid payments, supports multiple payment methods (checks, ACH, wire transfers, employee expenses, utility auto-pay), and tracks missed discounts. Here’s a step-by-step breakdown:

1. **Initialization (`L6DET` Subroutine)**:
   - Initializes variables: sets page number to zero, clears counters (`C6CNT`, `P6CNT`, `L6CNT`, etc.), and prepares the report header.
   - Retrieves the system date and time, formatting the date as an 8-digit value (`DATE8`) for Y2K compliance.
   - Chains to `APCONT` to retrieve the company name (`ACNAME`) and check if pre-numbered checks are used (`ACPRE#`).
   - Chains to `APPYTR` to get the next check number (`PTNXCK`), check date (`PTCKDT`), pay-by date (`PTDATE`), force discount flag (`PTFDIS`), and payment method (`PTHOLD`).
   - Sets the payment method description (`PAYBY`) based on `PTHOLD`:
     - `' '`: "PAY BY CHECK"
     - `A`: "PAY BY ACH"
     - `W`: "PAY BY WIRE TFR"
     - `E`: "PAY BY PAYROLL"
     - `U`: "PAY BY UTIL AUPY"
   - Writes the report header to `APCSHRQ` with company name, payment method, check date, and other details.

2. **Process APPAY Records**:
   - Reads records from `APPAY` (payment file) sorted by company (`OPCONO`), vendor (`OPVEND`), and sequence number (`OPSEQ#`).
   - For each record:
     - Checks if the record is not deleted (`OPDEL ≠ 'D'`).
     - Identifies payment type:
       - Prepaid payments: `OPPAID = 'P'` (check), `'A'` (ACH), `'W'` (wire transfer), `'E'` (employee expenses), `'U'` (utility auto-pay).
       - Single check: `OPSNGL = 'S'`.
     - Updates `APDETINV` to track invoice details:
       - Constructs a key (`APKY27`) using company (`OPCONO`), vendor (`OPVEND`), and invoice number (`OPINVN`).
       - Chains to `APDETINV` to check for existing records.
       - If found, adds gross amount (`OPGRAM`) and discount (`OPDISC`) to existing totals (`APGRAM`, `APDISC`).
       - If not found, creates a new record with invoice details.

3. **Aggregate Totals**:
   - Accumulates totals for gross amount (`CKGRAM`), discount (`CKDISC`), and payment amount (`CKAMT`) for each check.
   - Tracks invoice count (`COUNT`) per check, with a maximum of 36 invoices to avoid stub overflow.
   - Maintains company-level totals:
     - `C6CNT`, `C6GRAM`, `C6DISC`, `C6LPAM`: Computer checks.
     - `P6CNT`, `P6GRAM`, `P6DISC`, `P6LPAM`: Prepaid payments.
     - `L6CNT`, `L6GRAM`, `L6DISC`, `L6LPAM`: Total checks.

4. **Check Validation (`CHECK` Subroutine)**:
   - Determines the check number (`THISCK`):
     - For prepaid payments (`OPPAID = 'P', 'A', 'W', 'E', 'U'`), uses `OPCKNO`.
     - For non-prepaid, uses the next check number (`NXCK`) and increments it.
   - If the payment amount is zero or negative (`CKAMT ≤ 0`), marks the check as "CREDIT / NO PAY" and processes it in the `NOPAY` subroutine.
   - Checks for missed discounts by chaining to `APPYDS` with a key (`DSKY12`) based on company and vendor/voucher. If found, sets indicator `50` to note "DISCOUNT NOT TAKEN" on the report.
   - Validates the check against `APCHKR`:
     - For non-void checks (`CKAMT > 0`), ensures the check does not exist or is not open (`AMCODE ≠ 'O'`).
     - For void checks (`CKAMT < 0`), ensures the check exists, is open, and the entire amount is voided (`VOIDAM = AMCKAM`).
   - Writes the check record to `APPYCK` with fields like check number, payment amount, and status (`'F'` for full stub, `'V'` for void, `'C'` for credit/no pay).

5. **Handle Credit/No Pay and Full Stubs (`NOPAY` Subroutine)**:
   - For checks with zero or negative amounts (`CKAMT ≤ 0`) or full stubs (36 invoices), marks related `APPYCK` records as "CREDIT / NO PAY" (`AXRECD = 'C'`, `AXCHEK = 0`).
   - Adjusts counters (`C6CNT`, `L6CNT`) if a full stub was previously written with the same check number.

6. **Report Output**:
   - Writes detail lines to `APCSHRQ` for each invoice, including sequence number (`OPSEQ#`), invoice number (`OPINVN`), description (`OPINDS`), gross amount (`OPGRAM`), discount (`OPDISC`), paid-to-date (`OPPPTD`), payment amount (`OPLPAM`), due date (`OPDUED`), vendor (`OPVEND`), and voucher (`OPVONO`).
   - Writes check totals (`CKGRAM`, `CKDISC`, `CKAMT`) with annotations for prepaid payments, full stubs, or void checks.
   - Writes company totals (`C6CNT`, `C6GRAM`, `C6DISC`, `C6LPAM`, etc.) at the end of each company group.
   - Includes error messages for invalid checks (e.g., "CHECK IS ALREADY OPEN", "WHOLE CHECK AMOUNT MUST BE VOIDED").

7. **End of Processing**:
   - At the end of each company (`L6`), writes company totals and resets counters.
   - Continues processing until all `APPAY` records are read, then terminates.

---

### **Business Rules in the AP155 RPG Program**

The program enforces the following business rules:

1. **Payment Method Handling**:
   - Supports payment methods: checks (`' '`), ACH (`A`), wire transfers (`W`), employee expenses (`E`), and utility auto-pay (`U`).
   - Prepaid payments (`OPPAID = 'P', 'A', 'W', 'E', 'U'`) use the provided check number (`OPCKNO`) and date (`OPCKDT`).
   - Non-prepaid payments increment the next check number (`NXCK`) from `APPYTR`.

2. **Single Check Processing**:
   - Vouchers marked as single check (`OPSNGL = 'S'`) are processed individually, ensuring separate checks for one-time vendors or specific vouchers.

3. **Invoice Aggregation**:
   - For multiple invoices with the same invoice number for a vendor, aggregates gross (`APGRAM`) and discount (`APDISC`) amounts in `APDETINV` to avoid duplicate entries (per modifications `JB03` and `MG04`).

4. **Check Validation**:
   - Non-void checks must not already exist in `APCHKR` or must not be open (`AMCODE ≠ 'O'`).
   - Void checks must exist, be open, and have the entire amount voided.
   - Zero or negative payment amounts (`CKAMT ≤ 0`) are marked as "CREDIT / NO PAY" and not paid.

5. **Stub Limits**:
   - A maximum of 36 invoices per check is enforced to prevent stub overflow. If exceeded, the check is marked as a full stub (`'F'` or `'V'`), and processing continues with a new check number.

6. **Missed Discount Reporting**:
   - If a record exists in `APPYDS` for a voucher, indicates a missed discount on the report ("DISCOUNT NOT TAKEN").

7. **Company and Vendor Validation**:
   - Chains to `APCONT` to validate company number and retrieve company name.
   - Chains to `APVEND` or `APOPEN` to retrieve vendor name (`VNNAME`) and sort abbreviation (`VNSORT`) for reporting.

8. **Report Formatting**:
   - The report includes headers with company name, payment method, bank G/L, next check number, and dates.
   - Detail lines include invoice details, and totals are provided for computer checks, prepaid payments, and overall checks per company.

---

### **Tables (Files) Used**

The program interacts with the following files:

1. **APPAY** (`UP`, Update Primary, 384 bytes):
   - Payment file containing records to be reported.
   - Fields: `OPDEL` (delete flag), `OPCONO` (company), `OPVEND` (vendor), `OPVONO` (voucher), `OPGRAM` (gross amount), `OPDISC` (discount), `OPPPTD` (partial paid), `OPINVN` (invoice number), `OPLPAM` (payment amount), `OPPAID` (prepaid code), `OPSNGL` (single check), `OPCKNO` (check number), `OPCKDT` (check date).

2. **AP155S** (`IR`, Input Random, 3 bytes):
   - Input file for report sequencing (likely a control or sort file).

3. **APCONT** (`IC`, Input with Chain, 256 bytes):
   - A/P control file for company details.
   - Fields: `ACNAME` (company name), `ACPRE#` (pre-numbered checks flag).

4. **APPYTR** (`IC`, Input with Chain, 128 bytes):
   - Payment transaction file for header information.
   - Fields: `PTBKGL` (bank G/L), `PTNXCK` (next check number), `PTCKDT` (check date), `PTDATE` (pay-by date), `PTFDIS` (force discount), `PTHOLD` (payment method).

5. **APVEND** (`IC`, Input with Chain, 579 bytes):
   - Vendor file for vendor details.
   - Fields: `VNNAME` (vendor name), `VNSORT` (sort abbreviation).

6. **APOPEN** (`IC`, Input with Chain, 384 bytes):
   - Open A/P file for voucher details.
   - Fields: `VNNAME` (vendor name), `VNSORT` (sort abbreviation).

7. **APCHKR** (`IC`, Input with Chain, 128 bytes):
   - Check register file for validating check status.
   - Fields: `AMCODE` (status: D, O, R, V), `AMCKAM` (check amount).

8. **APPYCK** (`UC`, Update/Create, 96 bytes):
   - Check file for recording check details.
   - Fields: `AXRECD` (record code: ' ', F, V, C, P, A, W, E, U), `AXCHEK` (check number), `CKAMT` (payment amount), `PTCKDT` (check date).

9. **APDETINV** (`UF`, Update with File, 256 bytes):
   - Invoice detail file for tracking invoice totals.
   - Fields: `APDEL` (delete flag), `APCONO` (company), `APVEND` (vendor), `APINVN` (invoice number), `APGRAM` (gross amount), `APDISC` (discount), `OPPPTD` (partial paid), `OPINDS` (description), `OPDUED` (due date), `OPVONO` (voucher).

10. **APPYDS** (`IF`, Input with File, 384 bytes):
    - Missed discount file for identifying discounts not taken.
    - Fields: `DSDEL` (delete flag), `DSCONO` (company), `DSVEND` (vendor), `DSVONO` (voucher), `DSDISC` (discount), `DSDSDT` (discount due date).

11. **APCSHRQ** (`O`, Output, 142 bytes):
    - Printer file for the cash requirements report.
    - Outputs headers, detail lines, check totals, and company totals.

---

### **External Programs Called**

The `AP155` program does not explicitly call any external programs. It is a standalone RPG program invoked by the `AP150.ocl36.txt` OCL script after `AP151` and sorting operations. All processing is handled internally through subroutines (`L6DET`, `L4DET`, `CHECK`, `NOPAY`, `EDITCK`).

---

### **Summary**

The `AP155` RPG program generates a Cash Requirements Report by processing `APPAY` records, updating `APDETINV` for invoice tracking, and validating checks against `APCHKR`. It supports multiple payment methods (checks, ACH, wire transfers, employee expenses, utility auto-pay) and enforces rules for check validation, stub limits (36 invoices), and missed discount reporting. The program interacts with 11 files (`APPAY`, `AP155S`, `APCONT`, `APPYTR`, `APVEND`, `APOPEN`, `APCHKR`, `APPYCK`, `APDETINV`, `APPYDS`, `APCSHRQ`) and produces a detailed report in `APCSHRQ` with invoice details, check totals, and company summaries. No external programs are called, as all logic is self-contained.