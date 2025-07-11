The provided document is an **RPG (Report Program Generator) program** named `AP193` for the IBM System/36 or AS/400, called from the OCL program discussed previously. It serves as an edit program for Accounts Payable (A/P) check reconciliation data uploaded from PNC, validating data in the workfile (`APCRTR`) against control files (`APCONT`, `GLMAST`, `APCHKR`) and producing a report (`LIST`). Below, I’ll explain the process steps, business rules, tables used, and any external programs called.

### Process Steps of the RPG Program (AP193)

The `AP193` program reads records from the A/P check reconciliation workfile (`APCRTR`), validates them against control files (`APCONT`, `GLMAST`, `APCHKR`), accumulates totals, and generates a report (`LIST`) with validation results and errors. Here’s a step-by-step breakdown:

1. **File Definitions**:
   - **Input Files**:
     - `APCRTR` (Primary Input, `IP`, 80 bytes, indexed with 2-byte key at position 16): Workfile containing check reconciliation data from `AP192`.
     - `APCONT` (Input Control, `IC`, 256 bytes, indexed with 2-byte key at position 2): A/P control file with company or configuration data.
     - `GLMAST` (Input Control, `IC`, 256 bytes, indexed with 2-byte key at position 11): General Ledger master file for account validation.
     - `APCHKR` (Input Control, `IC`, 128 bytes, indexed with 2-byte key at position 16): A/P check reconciliation file with historical check data.
   - **Output File**:
     - `LIST` (Output, `O`, 132 bytes, printer file): Generates a report detailing check reconciliation results, errors, and totals.
   - **Data Structures**:
     - `COM` (Error message array, 10 elements, 30 bytes each): Stores error messages (e.g., "INVALID CHECK #", "CHECK NOT FOUND").
     - `SEP` (66-byte array, 2 elements): Likely used for report separators (e.g., lines or spaces).

2. **Input Record Mapping**:
   - **APCRTR** (Primary Input):
     - `ATKEY` (positions 2–17): Key field (likely check number).
     - `ATCO` (positions 2–3): Company code, used for `L3` (company-level) totaling.
     - `ATGL#` (positions 4–11): General Ledger number, used for `L2` (G/L-level) totaling.
     - `ATCHK#` (positions 12–17): Check number, used for validation.
     - `ATCLAM` (positions 23–33): Clear amount (check amount to be reconciled).
     - `ATCLDT` (positions 40–45): Clear date (date the check cleared).
   - **APCONT** (Control):
     - `ACDEL` (position 1): Deletion flag (‘D’ for deleted).
     - `ACNAME` (positions 4–33): Company name.
   - **GLMAST** (Control):
     - `GLDEL` (position 1): Deletion flag (‘D’ for deleted).
     - `GLDESC` (positions 13–37): General Ledger description.
   - **APCHKR** (Control):
     - `AMCODE` (position 1): Check status code (‘D’, ‘O’, ‘R’, ‘V’ for deleted, open, reconciled, voided).
     - `AMVEN#` (positions 18–22): Vendor number.
     - `AMCKAM` (positions 23–33): Check amount.
     - `AMCKDT` (positions 34–39): Check date.
     - `AMVNNM` (positions 46–75): Vendor name.
   - **UDS** (User Data Structure):
     - `Y2KCEN` (positions 509–510): Century for Y2K handling.
     - `Y2KCMP` (positions 511–512): Company code for Y2K.

3. **Initialization (N09 Block)**:
   - If indicator `09` is off (`N09`), the program:
     - Initializes `L2CLAM` and `L3CLAM` (G/L and company clear amount totals) to zero (`Z-ADD*ZEROS`).
     - Captures system time (`TIME`) and moves it to `TIMDAT` (12 bytes), then extracts `SYSTIM` (time, 6 bytes) and `SYSDAT` (date, 6 bytes).
     - Sets `SEP` to `'* '` (separator for report).
     - Initializes `PAGE` to zero for report pagination.
     - Sets `GLKEY` to `'C'` (11 bytes, likely a default G/L key).
     - Sets indicator `09` on to prevent re-execution.
   - This block runs once at program start to set up variables and report parameters.

4. **Indicator and Variable Reset**:
   - Indicators `30`, `31`, `32`, `33`, `34`, `81`, `90`, `91`, `92`, `93`, `94`, `95`, `96` are turned off (`SETOF`).
   - Error message fields (`MSG`, `MSG1`–`MSG6`) are cleared to blanks.
   - `COUNT` (error counter) is reset to zero.

5. **Validation Logic**:
   - **Company Validation**:
     - The program chains `ATCO` (company code from `APCRTR`) to `APCONT` (`CHAINAPCONT`, indicator `30`).
     - If no record is found or `ACDEL` = ‘D’ (deleted), indicator `30` is set, and `CONONM` (company name) is cleared; otherwise, `CONONM` is set to `ACNAME`.
   - **General Ledger Validation**:
     - Constructs `GLKEY` by combining `ATCO` and `ATGL#` into `GLKY10` (10 bytes) and moving it to `GLKEY`.
     - Chains `GLKEY` to `GLMAST` (`CHAINGLMAST`, indicator `31`).
     - If no record is found or `GLDEL` = ‘D’ (deleted), indicator `31` is set, and `BKGLNM` (G/L description) is cleared; otherwise, `BKGLNM` is set to `GLDESC`.
   - **Check Validation**:
     - Chains `ATKEY` (check number) to `APCHKR` (`CHAINAPCHKR`, indicator `32`).
     - If no record is found, sets indicator `90`, logs error “CHECK # NOT FOUND” (`COM,3`/`COM,4` to `MSG1`), increments `COUNT`, and branches to `AROUND` (skips further checks).
     - Checks `AMCODE` in `APCHKR` for:
       - ‘D’ (Deleted): Sets indicator `91`, logs “CHECK WAS PREVIOUSLY DELETED” (`COM,3`/`COM,5` to `MSG2`), increments `COUNT`.
       - ‘R’ (Reconciled): Sets indicator `92`, logs “CHECK IS ALREADY RECONCILED” (`COM,3`/`COM,6` to `MSG3`), increments `COUNT`.
       - ‘V’ (Voided): Sets indicator `93`, logs “CHECK WAS PREVIOUSLY VOIDED” (`COM,3`/`COM,7` to `MSG4`), increments `COUNT`.
       - ‘O’ (Open): Sets indicator `94`, logs “CHECK IS NOT OPEN” (`COM,3`/`COM,8` to `MSG5`), increments `COUNT`.
     - Compares `ATCLAM` (clear amount from `APCRTR`) to `AMCKAM` (check amount from `APCHKR`) (`COMP`, indicator `34`).
     - If amounts don’t match, sets indicator `95`, logs “CLEAR AMOUNT DOES NOT MATCH” (`COM,10` to `MSG6`), increments `COUNT`.
   - If any validation fails, the program branches to `AROUND` to skip further processing for the record.

6. **Accumulation**:
   - If validations pass (no branch to `AROUND`), adds `ATCLAM` to `L2CLAM` (G/L total) and `L3CLAM` (company total).

7. **Report Generation**:
   - The program writes to the `LIST` printer file:
     - **Header (Level `L3`, Company-Level)**:
       - Prints company name (`ACNAME`), page number (`PAGE`), system date (`SYSDAT`), bank info, and report title (“A/P CANCELLED CHECKS EDIT FROM PNC UPLOAD”).
       - Includes system time (`SYSTIM`) and separators (`SEP`).
       - Column headers: “CHECK #”, “CLEAR DATE”, “CLEAR AMOUNT”.
     - **Detail Lines (Level `01`)**:
       - Prints check number (`ATCHK#`), clear date (`ATCLDT`), and clear amount (`ATCLAM`).
       - If errors exist (indicators `90`–`95`), prints corresponding error messages (`MSG1`–`MSG6`) at position 110.
     - **Totals (Level `L2` and `L3`)**:
       - At `L2` (G/L break), prints “BANK G/L # TOTAL” with `L2CLAM`.
       - At `L3` (company break), prints “COMPANY TOTAL” with `L3CLAM`.

8. **Program Flow**:
   - The RPG cycle reads each `APCRTR` record, validates it against `APCONT`, `GLMAST`, and `APCHKR`, logs errors, accumulates totals, and writes report lines.
   - The program continues until all `APCRTR` records are processed, producing a report with headers, detail lines, error messages, and totals.

### Business Rules

1. **Validation of Company and G/L**:
   - Each record’s company code (`ATCO`) must exist in `APCONT` and not be deleted (`ACDEL ≠ ‘D’`).
   - The G/L number (`ATGL#`) combined with `ATCO` must exist in `GLMAST` and not be deleted (`GLDEL ≠ ‘D’`).

2. **Check Validation**:
   - The check number (`ATKEY`) must exist in `APCHKR`.
   - The check must not be deleted (`AMCODE ≠ ‘D’`), reconciled (`AMCODE ≠ ‘R’`), voided (`AMCODE ≠ ‘V’`), or non-open (`AMCODE ≠ ‘O’`).
   - The clear amount (`ATCLAM`) must match the check amount (`AMCKAM`) in `APCHKR`.

3. **Error Handling**:
   - Errors are logged with predefined messages (e.g., “CHECK # NOT FOUND”, “CLEAR AMOUNT DOES NOT MATCH”).
   - Each error increments a counter (`COUNT`) and is printed in the report.

4. **Totaling**:
   - Clear amounts are accumulated at the G/L level (`L2CLAM`) and company level (`L3CLAM`) for valid records.
   - Totals are printed at G/L and company breaks in the report.

5. **Report Formatting**:
   - The report includes headers with company, date, time, and bank info.
   - Detail lines show check number, clear date, clear amount, and any errors.
   - Totals are printed for each G/L account and company.

### Tables (Files) Used

1. **APCRTR**:
   - Primary input file, 80 bytes, indexed.
   - Contains check reconciliation data from `AP192` (check number, G/L number, company code, clear amount, clear date).
2. **APCONT**:
   - Input control file, 256 bytes, indexed.
   - Stores company data (name, deletion flag).
3. **GLMAST**:
   - Input control file, 256 bytes, indexed.
   - Stores General Ledger data (description, deletion flag).
4. **APCHKR**:
   - Input control file, 128 bytes, indexed.
   - Stores historical check data (status code, vendor number, check amount, check date, vendor name).
5. **LIST**:
   - Output printer file, 132 bytes.
   - Generates the reconciliation edit report.

### External Programs Called

The `AP193` RPG program does not explicitly call any external programs. It is a self-contained program that processes input from `APCRTR`, validates against `APCONT`, `GLMAST`, and `APCHKR`, and produces a report via `LIST`. It is called by the OCL program as part of the A/P reconciliation workflow.

### Summary

The `AP193` RPG program validates A/P check reconciliation data from `APCRTR` against control files (`APCONT`, `GLMAST`, `APCHKR`), logs errors, accumulates totals, and generates a report (`LIST`). It:
- Validates company, G/L, and check data, ensuring records are not deleted, reconciled, or voided, and amounts match.
- Logs errors with predefined messages and counts errors.
- Accumulates clear amount totals at G/L and company levels.
- Produces a formatted report with headers, detail lines, error messages, and totals.

**Business Rules**:
- Validate company, G/L, and check data for existence and status.
- Ensure clear amounts match historical check amounts.
- Log and report errors for invalid records.
- Accumulate and report totals by G/L and company.

**Tables Used**: `APCRTR` (input), `APCONT` (control), `GLMAST` (control), `APCHKR` (control), `LIST` (output).  
**External Programs Called**: None.

If you need further details, such as specific field formats or deeper analysis of the report structure, let me know!