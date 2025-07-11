Let’s analyze the provided RPG program `AP105P.rpg` to extract the process steps, business rules, tables used, and external programs called. This program is called by the CLP program `AP105PCL.clp` and is part of the A/P SOGAS ACH payment processing system on IBM i.

---

### **Process Steps**
The RPG program `AP105P.rpg` performs the following steps:

1. **File Declarations**:
   - Defines the following files:
     - `AP105PSC`: Workstation file (screen display file).
     - `APSOGAS`: Input primary file (127 bytes, disk).
     - `APSGACH`: Input file with keyed access (64 bytes, 7-byte key, disk).
     - `APVEND`: Input file with keyed access (579 bytes, 7-byte key, disk).
     - `LIST132`: Output printer file (132 bytes).
   - Defines the User Data Structure (UDS) for Local Data Area (LDA) fields:
     - `CANCEL` (positions 100–105)
     - `ERRORS` (position 106)
     - `DESC` (positions 107–119)
     - `CANTAX` (position 120, Canadian tax upload flag)
     - `UPTYPE` (position 121, positive/negative dollar values flag)

2. **Parameter Input**:
   - Accepts a single 1-character parameter `@FGRP` via `*ENTRY PLIST`.

3. **Initialization (Indicator 09 Off)**:
   - If indicator `09` is off (likely controlled by file status or program logic):
     - Initializes counters:
       - `COUNT` (5-digit numeric) to 0.
       - `ERRCNT` (5-digit numeric, error count) to 0.
       - `DUP` (5-digit numeric, duplicate count) to 0.
       - `DATES` (5-digit numeric) to 0.
       - `ERRORS` (LDA position 106) to blanks.
     - Captures system time and date:
       - Stores current time/date in `TIMDAT` (12 digits).
       - Extracts time to `SYTIME` (6 digits).
       - Extracts date to `SYDATE` (6 digits).
       - Converts `SYDATE` to `SYDYMD` (YYYYMMDD format) by multiplying by 10000.01.
       - Extracts `MONTH` (2 digits) and `YEAR` (2 digits) from `SYDATE`.
     - Sets indicators:
       - Sets indicator `09` and `51` on.
       - If `CANTAX` = 'Y', sets indicator `54` on.
       - If `UPTYPE` = 'T', sets indicator `55` on.
       - If both `54` and `55` are off, sets indicator `56` on and sets `ERRORS` to 'Y', then turns off indicators `50` and `51`.

4. **Increment Counter**:
   - Increments `COUNT` by 1 for each record processed.

5. **Validation of `APSOGAS` Record**:
   - Uses `ANOWNR` (owner number, positions 1–7) from `APSOGAS` to chain (lookup) in `APSGACH`.
   - If the chain fails (indicator `99` on, record not found):
     - Increments `ERRCNT`.
     - Sets indicator `50` on, `51` off.
     - Jumps to `SKIP` tag (bypassing further processing for this record).
   - If the chain succeeds (indicator `99` off):
     - Builds a 7-character key `VNKEY` by combining a hardcoded '10' with `AGVEND` (vendor number) from `APSGACH`.
     - Chains to `APVEND` using `VNKEY`.
     - If the chain to `APVEND` fails (indicator `53` on, vendor not found):
       - Sets indicator `50` on, `51` off.
       - Sets `ERRORS` to 'Y'.
       - Increments `ERRCNT`.
       - Writes an exception output (`EXCPT`) to `LIST132`.
     - Jumps to `SKIP` tag.

6. **Clear and Display Screen**:
   - If indicator `50` is off, clears indicator `56`.
   - Executes `EXFMT` (write/read) for `SCREEN1` in the `AP105PSC` display file.

7. **Output to Printer**:
   - Writes to `LIST132` (printer file) with fields:
     - `ANOWNR` (positions 1–10)
     - `ERRCNT` (positions 11–17)
     - `ANNAME` (positions 18–55)

---

### **Business Rules**
The program enforces the following business rules:

1. **Owner Validation**:
   - Each `APSOGAS` record’s `ANOWNR` must exist in `APSGACH`. If not, an error is logged (`ERRCNT` incremented, `ERRORS` set to 'Y'), and processing skips to the next record.

2. **Vendor Validation**:
   - The vendor number (`AGVEND`) from `APSGACH` must exist in `APVEND`. If not, an error is logged (`ERRCNT` incremented, `ERRORS` set to 'Y'), and an exception report is generated.

3. **Error Handling**:
   - Errors are tracked via `ERRCNT` and the `ERRORS` flag in the LDA (position 106). If errors occur, the program may signal to the calling CLP (`AP105PCL`) to skip further processing (e.g., S/36 procedure `AP105`).

4. **Canadian Tax and Upload Type Checks**:
   - If `CANTAX` = 'Y', sets indicator `54` to flag Canadian tax-related processing.
   - If `UPTYPE` = 'T', sets indicator `55` to flag specific dollar value handling (positive/negative).
   - If neither condition is met (`54` and `55` off), sets `ERRORS` to 'Y' and indicator `56` on, indicating a validation failure.

5. **Dynamic File Processing**:
   - The program processes records from `APSOGAS` and validates against `APSGACH` and `APVEND`, ensuring data integrity for A/P SOGAS ACH payments.

6. **Reporting**:
   - Outputs errors or processing results to a printer file (`LIST132`) for reporting, including owner number, error count, and name.

---

### **Tables Used**
The program uses the following files (tables):

1. **APSOGAS** (Input Primary, 127 bytes):
   - Fields:
     - `ANOWNR` (positions 1–7): Owner number.
     - `ANNAME` (positions 8–42): Name.
     - `ANCHNM` (positions 43–49): Check name.
     - `ANDATE` (positions 50–59): Date.
     - `ANCHAM` (positions 60–65, packed): Check amount.
     - `ANSTDT` (positions 66–75): Start date.
     - `ANENTE` (positions 76–85): Entry date.
     - `ANDUDT` (positions 86–95): Due date.
   - Used as the primary input file for payment data.

2. **APSGACH** (Input, Keyed, 64 bytes, 7-byte key):
   - Fields:
     - `AGOWNR` (positions 1–7): Owner number.
     - `AGVEND` (positions 8–12): Vendor number.
   - Used to validate `ANOWNR` from `APSOGAS`.

3. **APVEND** (Input, Keyed, 579 bytes, 7-byte key):
   - Fields:
     - `VNDEL` (position 1): Record code.
     - `VNCO` (positions 2–3): Company number.
     - `VNVEND` (positions 4–8): Vendor number.
     - `VNVNAM` (positions 9–38): Vendor name.
     - `VNAD1` to `VNAD4` (positions 39–158): Address lines.
     - `VNHOLD` (position 240): Hold invoices flag.
     - `VNSNGL` (position 241): Single check flag.
     - `VNEXGL` (positions 254–261): Expense GL and sub-account.
     - `VNTERM` (positions 262–263): AP terms code.
     - `VNCAID` (positions 294–299): Carrier ID.
     - `VNPRID` (positions 384–387, packed): ADP payroll ID.
     - `VNACLS` (positions 388–390): ACH class.
     - `VNACOS` (position 391): ACH checking or savings.
     - `VNARTE` (positions 392–400): ACH bank routing code.
     - `VNABK#` (positions 401–417): ACH bank account number.
   - Used to validate vendors associated with `AGVEND` from `APSGACH`.

4. **LIST132** (Output, Printer, 132 bytes):
   - Used to generate a report with `ANOWNR`, `ERRCNT`, and `ANNAME`.

5. **AP105PSC** (Workstation, Display File):
   - Contains `SCREEN1` for interactive display (written and read via `EXFMT`).

---

### **External Programs Called**
- **None**: The program does not explicitly call any external programs via `CALL` or other mechanisms. It interacts with the display file `AP105PSC` and printer file `LIST132` but does not invoke other RPG or CL programs.

---

### **Summary**
- **Process Steps**: Declare files, initialize counters and LDA, process `APSOGAS` records, validate against `APSGACH` and `APVEND`, handle errors, display screen, and output to printer.
- **Business Rules**: Validates owner and vendor numbers, tracks errors in LDA, handles Canadian tax and upload type flags, and generates error reports.
- **Tables Used**: `APSOGAS` (payment data), `APSGACH` (owner validation), `APVEND` (vendor details), `LIST132` (printer output), `AP105PSC` (display file).
- **External Programs Called**: None.

This RPG program validates A/P SOGAS ACH payment records, ensuring owners and vendors exist in the respective files, and reports errors via a printer file and interactive screen.