### List of Use Cases Implemented by the Program

The call stack consists of the OCL program (`AP192.ocl36.txt`), the RPG program `AP192` (`AP192.rpg36.txt`), and the RPG program `AP193` (`AP193.rpg36.txt`). Together, they implement a single primary use case for Accounts Payable (A/P) check reconciliation from PNC uploads:

1. **Process and Validate A/P Check Reconciliation Data from PNC**:
   - **Description**: This use case involves uploading check reconciliation data from PNC, transforming it into a structured workfile, validating it against company, General Ledger, and historical check data, and producing a report with validation results and totals.
   - **Components**:
     - The OCL program orchestrates the workflow by managing file creation/deletion and invoking `AP192` and `AP193`.
     - `AP192` transforms PNC upload data (`APCHKUP`) into a workfile (`APCRTR`) with formatted records.
     - `AP193` validates the workfile data against control files (`APCONT`, `GLMAST`, `APCHKR`) and generates a report (`LIST`) with errors and totals.

### Function Requirement Document



# A/P Check Reconciliation Function Requirements

## Overview
The A/P Check Reconciliation function processes and validates check reconciliation data uploaded from PNC, transforming it into a structured format, validating it against company, General Ledger, and historical check data, and producing a report with validation results and totals.

## Inputs
- **PNC Upload File (`APCHKUP`)**:
  - Fields: Check number (6 bytes), amount (11 bytes), year (4 bytes), two-digit year (2 bytes), month (2 bytes), day (2 bytes).
  - Format: Fixed-length records (25 bytes).
- **Control Files**:
  - `APCONT`: Company data (company code, name, deletion flag).
  - `GLMAST`: General Ledger data (G/L number, description, deletion flag).
  - `APCHKR`: Historical check data (check number, status code, vendor number, check amount, check date, vendor name).

## Outputs
- **Workfile (`APCRTR`)**:
  - Fields: Company code, G/L number (default: 11000001), check number, clear amount, clear date (year, two-digit year, month, day, repeated month/day).
  - Format: Fixed-length records (80 bytes, indexed).
- **Report (`LIST`)**:
  - Content: Company and G/L headers, check details (check number, clear date, clear amount), error messages, G/L and company totals.
  - Format: Printer file (132 bytes).

## Process Steps
1. **File Management**:
   - Delete existing `APCRTR` workfile if present.
   - Create new `APCRTR` workfile (500 records, 80 bytes, indexed, 2-byte key at position 16) if it doesn’t exist.

2. **Data Transformation** (via `AP192`):
   - Read `APCHKUP` records.
   - Map fields: check number, amount, year, two-digit year, month, day.
   - Add default G/L number (11000001) and transaction code (‘10’).
   - Write formatted records to `APCRTR`.

3. **Validation and Reporting** (via `AP193`):
   - Read `APCRTR` records.
   - Validate:
     - Company code exists in `APCONT` and is not deleted.
     - G/L number exists in `GLMAST` and is not deleted.
     - Check number exists in `APCHKR` and is not deleted, reconciled, voided, or non-open.
     - Clear amount matches `APCHKR` check amount.
   - Log errors for invalid records (e.g., “CHECK # NOT FOUND”, “CLEAR AMOUNT DOES NOT MATCH”).
   - Accumulate clear amounts for valid records by G/L (`L2CLAM`) and company (`L3CLAM`).
   - Generate report (`LIST`) with headers, check details, errors, and totals.

## Business Rules
1. **Data Validation**:
   - Company code must exist in `APCONT` and not be marked deleted (`ACDEL ≠ ‘D’`).
   - G/L number must exist in `GLMAST` and not be marked deleted (`GLDEL ≠ ‘D’`).
   - Check number must exist in `APCHKR` with status code ‘O’ (open), not ‘D’ (deleted), ‘R’ (reconciled), or ‘V’ (voided).
   - Clear amount in `APCRTR` must match check amount in `APCHKR`.

2. **Error Handling**:
   - Log errors for invalid company, G/L, check status, or amount mismatch.
   - Increment error counter for each validation failure.
   - Include error messages in the report.

3. **Calculations**:
   - Initialize G/L and company totals to zero.
   - Add clear amount to G/L total (`L2CLAM`) and company total (`L3CLAM`) for valid records.
   - Report totals at G/L and company levels.

4. **Report Formatting**:
   - Include company name, G/L number, date, time, and bank info in headers.
   - List check number, clear date, clear amount, and errors for each record.
   - Print G/L and company totals at respective breaks.

## Assumptions
- Input file `APCHKUP` is correctly formatted with valid data.
- Control files (`APCONT`, `GLMAST`, `APCHKR`) are up-to-date and accessible.
- Output workfile `APCRTR` is used by downstream processes (not covered in this function).

