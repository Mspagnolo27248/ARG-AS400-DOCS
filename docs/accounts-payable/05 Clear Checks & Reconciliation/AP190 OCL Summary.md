### List of Use Cases Implemented by the AP190/AP195 Program Suite

The `AP190.ocl36`, `AP190.rpg36`, and `AP195.rpg36` programs collectively implement a single primary use case for Accounts Payable (A/P) check reconciliation:

1. **A/P Cancelled Check Reconciliation and Reporting**:
   - **Description**: This use case allows users to enter, validate, and store cancelled check reconciliation data (company number, bank G/L number, check number, clear date, and clear amount) and generate a printed report summarizing the reconciled checks, grouped by company and bank G/L account with totals.
   - **Components**:
     - `AP190.ocl36`: Orchestrates file setup and program execution (calls `AP190` and `AP195`).
     - `AP190.rpg36`: Handles interactive data entry and validation through two screens (`AP190S1` for company, bank G/L, and check number; `AP190S2` for clear date and amount), updating the reconciliation transaction file (`APCRTR`).
     - `AP195.rpg36`: Generates a report listing cancelled checks with totals by bank G/L and company.

### Function Requirement Document for A/P Cancelled Check Reconciliation



# Function Requirement Document: A/P Cancelled Check Reconciliation

## Overview
The A/P Cancelled Check Reconciliation function processes and validates cancelled check data, storing reconciliation records and generating a summary report grouped by company and bank G/L account.

## Inputs
- **Company Number** (`CONO`, 2 bytes): Identifies the company.
- **Bank G/L Number** (`BKGL`, 8 bytes): Specifies the bank general ledger account.
- **Check Number** (`CHK#`, 6 bytes): Unique identifier for the check.
- **Clear Date** (`CLDT`, 6 digits, MMDDYY): Date the check cleared.
- **Clear Amount** (`CLAM`, 11.2 numeric): Amount cleared for the check.

## Outputs
- **Reconciliation Records**: Stored in `APCRTR` file with company, bank G/L, check number, clear date, and clear amount.
- **Printed Report**: Lists cancelled checks with check number, clear date, clear amount, and totals by bank G/L and company, including company name, date, time, and page number.

## Process Steps
1. **Validate Inputs**:
   - Verify `CONO` exists in `APCONT` and is not deleted (`ACDEL ≠ 'D'`).
   - Verify `CONO` + `BKGL` exists in `GLMAST` and is not deleted (`GLDEL ≠ 'D'`).
   - Verify `CHK#` exists in `APCHKR`, is open (`AMCODE = 'O'`), and not deleted (`D`), reconciled (`R`), or voided (`V`).
   - Validate `CLDT` as a valid date (MMDDYY, month 1–12, day 1–31 based on month/leap year).
   - Ensure `CLAM` matches the check amount (`AMCKAM`) from `APCHKR`.

2. **Store Reconciliation Data**:
   - Write or update `APCRTR` with `CONO`, `BKGL`, `CHK#`, `CLDT`, and `CLAM`.
   - Support deletion of existing `APCRTR` records.

3. **Generate Report**:
   - Read `APCRTR` records, group by `CONO` and `BKGL`.
   - Retrieve company name (`ACNAME`) from `APCONT`.
   - Print headers (company name, bank G/L, date, time, page).
   - List check details (`CHK#`, `CLDT`, `CLAM`).
   - Calculate and print totals:
     - Bank G/L total (`L1CLAM`): Sum of `CLAM` for each bank G/L.
     - Company total (`L2CLAM`): Sum of `L1CLAM` for each company.
   - Output after 21 detail lines or at group breaks.

## Business Rules
1. **Validation**:
   - Invalid `CONO`, `BKGL`, or `CHK#` prevents processing.
   - `CLDT` must be a valid date, accounting for leap years (February 28/29, others 30/31 days).
   - `CLAM` must equal `AMCKAM` from `APCHKR`.
2. **Data Integrity**:
   - Only open checks (`AMCODE = 'O'`) are processed.
   - Deleted records in `APCONT` or `GLMAST` are invalid.
3. **Reporting**:
   - Group by company and bank G/L, with subtotals and company totals.
   - Include company name if available; otherwise, proceed without it.
   - Format dates (MMDDYY) and amounts (11.2 decimal) for readability.
4. **Record Management**:
   - Allow updates or deletions in `APCRTR`.
   - Ensure unique keys (`CONO` + `BKGL` + `CHK#`) in `APCRTR`.

## Calculations
- **Date Validation**:
  - Extract month, day, year from `CLDT` (MMDDYY).
  - Validate month (1–12).
  - Validate day: February (28 or 29 for leap years), April/June/September/November (30), others (31).
  - Leap year: If year divisible by 4 (or by 400 for century years), allow 29 days for February; else, 28.
  - Construct 8-digit date (`CLDT8`, YYYYMMDD) using century (`Y2KCEN`) for Y2K compliance.
- **Totals**:
  - `L1CLAM` = Sum of `CLAM` for each bank G/L group.
  - `L2CLAM` = Sum of `L1CLAM` for each company.

## Data Sources
- **APCRTR**: Stores reconciliation data (key: `CONO` + `BKGL` + `CHK#`).
- **APCONT**: Provides company name (`ACNAME`) and deletion flag (`ACDEL`).
- **GLMAST**: Validates bank G/L (`GLDEL`, `GLDESC`).
- **APCHKR**: Provides check details (`AMCODE`, `AMVEN#`, `AMCKAM`, `AMCKDT`, `AMVNNM`).

