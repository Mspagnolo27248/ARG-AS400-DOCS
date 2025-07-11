The RPG program `AP251.RPG36` is invoked by the OCL procedure `AP251.OCL36` as part of the Accounts Payable (A/P) Check Register process, specifically to update the commission table (`APTORCY`) with payment information from the payment file (`APPAY`). Below is a detailed explanation of the **process steps**, **business rules**, **tables/files used**, and **external programs called**, as requested.

---

### Process Steps

The `AP251.RPG36` program is a straightforward RPG program designed to update the commission table (`APTORCY`) based on payment records from the `APPAY` file. The processing is structured around two subroutines: `ONETIM` for initialization and `EACH01` for processing each payment record. The steps are as follows:

1. **Initialization (ONETIM Subroutine)**:
   - Executes once at the start of the program (`N09` condition ensures it runs only if indicator `09` is off).
   - Retrieves the current system time and date using the `TIME` operation, storing them in `TIMDAT` (12 digits).
   - Splits `TIMDAT` into `TIME` (6 digits, time) and `DATE` (6 digits, date).
   - Converts the date to a year-month-day format (`DATYMD`) by multiplying `DATE` by 10000.01.
   - Formats the date into an 8-digit format (`DATE8`) by prefixing the century (`20`) and appending `DATYMD`.
   - Sets indicator `09` to prevent re-execution of the subroutine.

2. **Process Each Payment Record (EACH01 Subroutine)**:
   - Executes for each record in the `APPAY` file (triggered by record indicator `01`).
   - Constructs a key (`KEY27`, 27 bytes) for chaining to the `APTORCY` file:
     - Copies the company number (`CONO`) to `KEY27`.
     - Copies the vendor number (`VEND`) to a temporary key (`KEY25`).
     - Copies the invoice description (`OPIN20`, positions 35–54) to a temporary key (`KEY20`).
     - Combines `KEY20` into `KEY25` and then into `KEY27` to form the full key.
   - Chains to the `APTORCY` file using `KEY27` to locate the corresponding commission record (sets indicator `99` if not found).
   - If the record is found (`N99`):
     - Converts the check date (`AXCKDT`) to a year-month-day format (`AXYMD`) by multiplying by 10000.01.
     - Formats the check date into an 8-digit format (`AXYMD8`) by prefixing the century (`20`) and appending `AXYMD`.
     - Sets the amount field (`AMT92`) to the gross amount (`OPGRAM`) from the `APPAY` record.
     - Writes an exception record to `APTORCY` via the `UPDATE` output specification.

3. **Update Commission Table (APTORCY)**:
   - For each matching record in `APTORCY`, updates the following fields:
     - `OPINV#` (A/P invoice number, positions 207–226 from `APPAY`) to `ATINV` (positions 84–103).
     - `AMT92` (gross amount from `OPGRAM`) to `ATAPMT` (positions 104–108, packed).
     - Sets `ATSTAT` (position 109) to `'P'` to indicate the payment status.
     - `ATCHK#` (check number, positions 91–96 from `APPAY` as `OPCKNO`) to `ATCHK#` (positions 110–115).

---

### Business Rules

The program enforces the following business rules, which govern the updating of the commission table:

1. **Key Matching for Commission Records**:
   - The program matches `APPAY` records to `APTORCY` records using a composite key (`KEY27`) built from:
     - Company number (`CONO`, positions 2–3 in `APPAY`, `ATCO` in `APTORCY`).
     - Vendor number (`VEND`, positions 4–8 in `APPAY`, `ATVEND` in `APTORCY`).
     - Invoice description (`OPIN20`, positions 35–54 in `APPAY`, corresponding to `ATINV` in `APTORCY`).
   - If no matching record is found in `APTORCY` (indicator `99` on), the program skips the update for that payment record.

2. **Payment Status Update**:
   - Updates the commission table only when a matching record is found.
   - Sets the payment status (`ATSTAT`) to `'P'` (paid) for matched records.
   - Records the gross amount (`OPGRAM`) as the payment amount (`ATAPMT`) and the check number (`OPCKNO`) in `ATCHK#`.

3. **Date Handling**:
   - Uses the check date (`AXCKDT`, positions 434–439 in the User Data Structure) and formats it into an 8-digit year-month-day format (`AXYMD8`) with a century prefix (`20`).
   - Includes Y2K compliance fields (`Y2KCEN = 19`, `Y2KCMP = 80`) in the User Data Structure (UDS), though they are not explicitly used in date calculations in this program (likely inherited from a standard template).

4. **No Output Reports**:
   - The program does not generate printed reports or additional output files; it solely updates the `APTORCY` file.

5. **Single-Pass Processing**:
   - Processes each `APPAY` record once, with no accumulation or totaling across records.
   - Updates are performed immediately for each matched record via exception output (`EXCPTUPDATE`).

---

### Tables/Files Used

The program interacts with two files, as defined in the OCL and RPG source:

| **Logical Name** | **Label**         | **Usage**                                                                 | **Disposition** | **Record Length** |
|------------------|-------------------|---------------------------------------------------------------------------|-----------------|-------------------|
| APPAY            | ?9?APPY?WS?       | Input: Payment data (company, vendor, invoice, gross amount, check number). | Input (IP)      | 384               |
| APTORCY          | ?9?APTORCY        | Update: Commission table (vendor, invoice, payment amount, status).        | Update (UF)     | 211               |

**Key Fields in APPAY**:
- `CONO` (Company Number, positions 2–3)
- `VEND` (Vendor Number, positions 4–8)
- `OPIN20` (Invoice Description, positions 35–54)
- `OPGRAM` (Gross Amount, packed, positions 18–23)
- `OPCKNO` (Check Number for Prepay, positions 91–96)
- `AXCKDT` (Check Date, positions 434–439 in UDS)

**Key Fields in APTORCY**:
- `ATCO` (Company Number, positions 2–3)
- `ATVEND` (Vendor Number, positions 59–63)
- `ATINV` (Invoice Number, positions 84–103)
- `ATAPMT` (A/P Payment Amount, packed, positions 104–108)
- `ATSTAT` (Payment Status, position 109)
- `ATCHK#` (Check Number, positions 110–115)

---

### External Programs Called

The `AP251.RPG36` program does not explicitly call any external programs. All processing is handled within the program through its subroutines (`ONETIM` and `EACH01`). The program is self-contained, relying solely on file operations and internal logic to update the commission table.

---

### Notes
- **File Dispositions**:
  - `APPAY` is the primary input file (`IP`), read sequentially to process payment records.
  - `APTORCY` is an update file (`UF`) with an external key (`EXTK`), allowing updates to specific records based on the constructed key (`KEY27`).
- **Modifications**:
  - `JB01`: Added support for prepaid vouchers (`OPPAID`, position 97).
  - `JB02`: Added sales order number (`OPSORN`, positions 281–286) and modified invoice handling to use `OPIN20` instead of `HANDTK` for key construction.
- **Y2K Compliance**: The program includes fields for century handling (`Y2KCEN`, `Y2KCMP`) in the User Data Structure, but they are not used in this specific logic, suggesting a standard template inclusion.
- **Error Handling**: If a matching `APTORCY` record is not found (`99` on), the program skips the update without logging errors or generating output, ensuring silent continuation.
- **Purpose**: The program is narrowly focused on updating the commission table with payment details, likely used to track commissions owed or paid to vendors based on invoice payments.

This summary provides a comprehensive overview of the `AP251.RPG36` program's functionality, business logic, file interactions, and lack of external program calls. Let me know if you need further clarification or additional analysis!