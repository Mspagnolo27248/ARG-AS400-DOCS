The `AP200P.rpg36.txt` file is an RPG III program (`AP200P`) called within the `AP200.ocl36.txt` OCL procedure to prompt for and validate input parameters for the Purchase Journal and Cash Disbursements Journal, including dates and accounting periods. Below, I provide a detailed explanation of its **process steps**, **business rules**, **tables/files used**, and **external programs called**, along with its purpose in the context of the `AP200` OCL procedure.

---

### **Purpose in AP200 OCL**
The `AP200P` program serves as the initial step in the Purchase Journal process within `AP200.ocl36.txt`. It prompts the user for key input parameters, such as the Purchase Journal date (`PJDATE`), Cash Disbursements Journal date (`CDDATE`), and accounting periods/years (`KYPD`, `KYPDYY`, `CDPD`, `CDPDYY`). It validates these inputs against system controls and writes validated data to the payment transaction file (`APPYTR`) for further processing. This program ensures that the journal process starts with accurate and valid parameters, particularly for handling prepaid, ACH, wire transfer, or employee expense transactions.

In the `AP200` OCL procedure, `AP200P` is called early in the workflow (`LOAD AP200P`) to set up the necessary parameters before transaction processing and journal generation.

---

### **Process Steps**

1. **Initialization**:
   - **Clear Indicators and Variables**:
     - Clears indicators `81` and `90` (`SETOF 8190`).
     - Initializes `MSG30` (message field) to blanks.
     - Sets zero fields (`Z5`, `Z2`, `Z8`, `Z6`) to 0 for use in output.
   - **Cancel Check**:
     - If function key `KG` (cancel) is pressed, sets `CANCEL` to `'CANCEL'`, sets last record indicator (`LR`), clears `81`, and jumps to `END`.

2. **One-Time Setup (`ONETIM` Subroutine)**:
   - **Check for Prepaid Transactions**:
     - Sets lower limit (`SETLL`) on `APTRAN` to check for records.
     - Reads `APTRAN` until end-of-file (`09`) or a non-deleted record is found (`N08`).
     - Checks `ATPAID` (payment type) for:
       - `'P'` (prepaid, sets `21` and `22`).
       - `'A'` (ACH, sets `21` and `23`, per `JB01`).
       - `'W'` (wire transfer, sets `21` and `24`, per `JB01`).
       - `'E'` (employee expense, sets `21` and `25`, per `JB01`).
     - If no prepaid/ACH/wire/employee transactions exist (`N21`), sets indicator `20`.
   - **Accounting Period Check**:
     - Chains to `GSCONT` to check `GX13GL` (13 accounting periods flag).
     - If `GX13GL = 'Y'` or no prepaid transactions exist (`N20`), sets indicator `19` (prompt for period/year).
     - If prepaid transactions exist (`21`), sets indicator `18` (prompt for Cash Disbursements date).

3. **Screen 1 Processing (`S1` Subroutine)**:
   - **Purchase Journal Date Validation**:
     - Moves `PJDATE` to `DATE` and calls `DATCHK` to validate the date format.
     - If invalid (`79`), sets `8190`, displays error, and jumps to `ENDS1`.
     - Converts `PJDATE` to `PJYMD` (YYYYMMDD) by multiplying by 10000.01.
     - Extracts year (`PJYR`) and determines century (`PJCN`):
       - If `PJYR >= Y2KCMP` (80), sets `PJCN` to `Y2KCEN` (19).
       - Otherwise, sets `PJCN` to `Y2KCEN + 1` (20).
     - Combines into `PJYMD8` (century + YYYYMMDD).
   - **Cash Disbursements Date Validation (if Prepaid)**:
     - If prepaid transactions exist (`21`), validates `CDDATE` using `DATCHK`.
     - If invalid (`79`), sets `8190`, displays error, and jumps to `ENDS1`.
     - Converts `CDDATE` to `CDYMD` (YYYYMMDD) and determines century (`CN`), forming `CDDAT8`.
   - **Purchase Journal Period/Year Validation (if 13 Periods)**:
     - If `16` (13 periods), validates `KYPD` (period, 1–13):
       - If `KYPD < 1` or `> 13`, sets `819050`, displays error (`MSG,5`), and jumps to `ENDS1`.
     - Retrieves period end date (`TBPDDT`) from `GSTABL` using `KYPD` and `KYPDYY`:
       - If not found (`10`), sets `819050`, displays error (`MSG,5`), and jumps to `ENDS1`.
     - Converts `TBPDDT` to `HIDATE` (YYYYMMDD) and `HIDAT8` (century + YYYYMMDD).
     - Compares `PJYMD8` to `HIDAT8` (high date); if `PJYMD8 > HIDAT8`, sets `819050`, displays error (`MSG,6`), and jumps to `ENDS1`.
     - Determines low date for previous period (e.g., period 1 uses period 13 of prior year):
       - Retrieves `TBPDDT` for previous period from `GSTABL`.
       - If not found (`10`), sets `819050`, displays error (`MSG,5`), and jumps to `ENDS1`.
     - Compares `PJYMD8` to `LODAT8` (low date); if `PJYMD8 < LODAT8`, sets `819050`, displays error (`MSG,6`), and jumps to `ENDS1`.
     - **Fiscal Year Check**:
       - Chains to `GLCONT` to get last fiscal year closed (`GCLSYR`).
       - If not found (`99`), clears date fields.
       - Compares `PJDATE` (month/year) to fiscal year boundaries; if outside current fiscal year, sets `819051`, displays error (`MSG,7`), and jumps to `ENDS1`.
   - **Cash Disbursements Period/Year Validation (if 13 Periods)**:
     - Similar validation for `CDPD` and `CDPDYY` using `GSTABL` and date comparisons.
     - Displays errors (`MSG,5` or `MSG,6`) if invalid.
   - **Set Output**:
     - Sets `JRDATE` to `PJDATE` and `JRYMD8` to `PJYMD8`.
     - Clears `CANCEL`.
     - Sets indicator `82` (valid input) and clears `81`.

4. **Screen 2 Processing (`S2` Subroutine)**:
   - Displays confirmation screen (`AP200PS2`) with input values.
   - If user enters `YORN = 'Y'`, sets `LR` (last record), clears `81`, and proceeds.
   - Otherwise, clears `PJDATE`, sets `0181`, clears `0282`, and redisplays screen.

5. **Date Check (`DATCHK` Subroutine)**:
   - Validates date format (`MMDDYY`):
     - Breaks down into month (`$MONTH`), day (`$DAY`), year (`$YR`).
     - Checks month (1–12); sets `79` if invalid.
     - Validates day based on month:
       - For February, checks leap year:
         - Non-century years: Divides year by 4 (or multiplies by 0.25).
         - Century years: Combines century and year, divides by 400 (or multiplies by 0.0025).
         - Leap year: Allows up to 29 days; non-leap year: 28 days.
       - Other months: Allows 30 days for April, June, September, November; 31 days for others.
     - Sets `79` if day is invalid.

6. **Write to Output File**:
   - If valid (`LR 21`), writes to `APPYTR`:
     - Includes zero fields (`Z5`, `Z2`, `Z8`, `Z6`), `CDDATE`, `CDDAT8`, `CDPD`, `CDPDYY`, and payment type flag (`' '`, `'A'`, `'W'`, or `'E'` based on `ATPAID`).

7. **End Processing**:
   - Jumps to `END` on cancel or error.
   - Closes files and terminates.

---

### **Business Rules**
1. **Prepaid Transaction Check**:
   - Checks `APTRAN` for prepaid (`'P'`), ACH (`'A'`), wire transfer (`'W'`), or employee expense (`'E'`) transactions.
   - Prompts for Cash Disbursements date (`CDDATE`) only if such transactions exist (`21`).
2. **13 Accounting Periods**:
   - If `GX13GL = 'Y'` in `GSCONT`, validates period/year (`KYPD`, `KYPDYY`, `CDPD`, `CDPDYY`) against `GSTABL` period end dates.
   - Ensures dates fall within valid period boundaries (high and low dates).
3. **Date Validation**:
   - Validates `PJDATE` and `CDDATE` for correct format and fiscal year.
   - Checks leap years for February dates, handling century calculations for Y2K compliance.
4. **Fiscal Year Check**:
   - Ensures `PJDATE` is within the current fiscal year based on `GCLSYR` and first fiscal month (`GCFFMO`).
5. **User Confirmation**:
   - Requires user confirmation (`YORN = 'Y'`) to proceed with validated inputs.
6. **Error Handling**:
   - Displays error messages for invalid dates, periods, or fiscal year mismatches.
   - Cancels processing if user presses `KG` or inputs are invalid.
7. **ACH/Wire Support** (`JB01`, 05/01/13):
   - Supports ACH (`'A'`), wire transfer (`'W'`), and employee expense (`'E'`) payment types in addition to prepaid (`'P'`).

---

### **Tables/Files Used**
- **Input**:
  - `APTRAN` (Input with Delete, `ID`):
    - A/P transaction file (404 bytes).
    - Fields: `ATPAID` (payment type: `'P'`, `'A'`, `'W'`, `'E'`).
    - Keys: Positions 9 and 10 (company and other keys).
  - `GSCONT` (Chained Input, `IC`):
    - System control file (512 bytes).
    - Fields: `GX13GL` (13 accounting periods flag).
  - `GLCONT` (Input, `IF`):
    - General ledger control file (256 bytes).
    - Fields: `GCDEL` (delete flag), `GCCO` (company number), `GCNAME` (company name), `GCADR1–3` (address), `GCFFMO` (first fiscal month), `GCNXGJ` (next general journal), `GCICGL` (intercompany G/L), `GCLSYR` (last fiscal year closed), `GCRETC` (retained earnings current year), `GCYTDP` (YTD profit), `GCRETP` (retained earnings prior years), `GCINLN` (total income line), `GCCONS` (consolidated company code), `GCCOLM` (consolidated column), `GCBSGL` (balance sheet rounding G/L), `GCISGL` (income statement rounding G/L), `GCMINV` (month inventory cost flag), `GCLSY4` (last fiscal year closed, century), `GCLMCC` (last month closed for costing), `GCCOUM` (costing unit of measure).
  - `GSTABL` (Chained Input, `IC`):
    - General ledger table file (256 bytes).
    - Fields: `TBPDDT` (period end date).
    - Key: Positions 1–12 (constructed from period and year).
  - `SCREEN` (Workstation, `CP`):
    - Display file for user prompts (`AP200PS1`, `AP200PS2`).
    - Fields: `PJDATE` (Purchase Journal date), `CDDATE` (Cash Disbursements date), `KYPD` (PJ period), `KYPDYY` (PJ year), `CDPD` (CD period), `CDPDYY` (CD year), `YORN` (confirmation), `MSG30` (error message).
- **Output**:
  - `APPYTR` (Output, `O`):
    - Payment transaction file (128 bytes).
    - Fields: Zero fields (`Z5`, `Z2`, `Z8`, `Z6`), `CDDATE`, `CDDAT8`, `CDPD`, `CDPDYY`, payment type flag (`' '`, `'A'`, `'W'`, `'E'`).
  - `SCREEN` (Workstation, `O`):
    - Outputs prompts and error messages (`AP200PS1` for input, `AP200PS2` for confirmation).

---

### **External Programs Called**
- **None**:
  - The `AP200P` program does not call external programs. It relies on internal subroutines (`ONETIM`, `S1`, `S2`, `DATCHK`) for processing.

---

### **Role in AP200 OCL**
In the `AP200` OCL procedure, `AP200P` is called early (`LOAD AP200P`) to:
- Prompt the user for Purchase Journal and Cash Disbursements Journal dates and periods.
- Validate inputs against system controls (`GSCONT`, `GLCONT`, `GSTABL`) for date format, period validity, and fiscal year compliance.
- Write validated parameters to `APPYTR` for use in subsequent steps (e.g., `AP200`, `AP205`, `AP210`).
- Handle special cases like prepaid, ACH, wire transfer, or employee expense transactions.
- Ensure the process aborts gracefully on user cancellation or invalid inputs.

This step is critical for initializing the Purchase Journal process with accurate parameters, ensuring downstream processing (sorting, posting, and reporting) uses valid dates and periods.

---

### **Key Notes**
- **Revisions**:
  - `JB01` (05/01/13): Added support for ACH (`'A'`), wire transfer (`'W'`), and employee expense (`'E'`) payment types.
- **Date Handling**:
  - Robust date validation with leap year checks and Y2K compliance (19xx vs. 20xx).
- **Period Validation**:
  - Supports 13 accounting periods if enabled (`GX13GL = 'Y'`), with strict period/year checks.
- **User Interaction**:
  - Uses two screens (`AP200PS1` for input, `AP200PS2` for confirmation) to ensure user validation.
- **Error Messages**:
  - Provides clear feedback for invalid inputs (e.g., invalid date, period, or fiscal year).

If you need further details on specific fields, logic, or integration with other OCL steps, let me know!

---


# AP200P Process Summary

## Purpose
`AP200P` is an RPG III program called in the `AP200` OCL procedure to prompt for and validate Purchase Journal and Cash Disbursements Journal parameters, including dates and accounting periods, and write them to `APPYTR`.

## Process Steps
1. **Initialization**: Clears indicators, messages, and zero fields. Checks for user cancel (`KG`).
2. **One-Time Setup (`ONETIM`)**:
   - Checks `APTRAN` for prepaid (`'P'`), ACH (`'A'`), wire transfer (`'W'`), or employee expense (`'E'`) transactions.
   - Sets flags for prompting Cash Disbursements date (`21`) or period/year (`19`, `18`).
3. **Screen 1 (`S1`)**:
   - Validates `PJDATE` and `CDDATE` using `DATCHK`.
   - Converts dates to YYYYMMDD and handles Y2K (19xx/20xx).
   - Validates periods (`KYPD`, `CDPD`) against `GSTABL` for 13 accounting periods.
   - Checks fiscal year against `GLCONT`.
   - Displays errors if invalid.
4. **Screen 2 (`S2`)**:
   - Prompts for confirmation (`YORN = 'Y'`).
   - Clears `PJDATE` if not confirmed.
5. **Date Check (`DATCHK`)**:
   - Validates date format, month, day, and leap year.
6. **Write to `APPYTR`**:
   - Outputs validated dates, periods, and payment type to `APPYTR`.
7. **End**: Closes files or jumps to `END` on cancel/error.

## Business Rules
- Prompts for `CDDATE` only if prepaid/ACH/wire/employee transactions exist.
- Validates periods (1–13) if `GX13GL = 'Y'` in `GSCONT`.
- Ensures dates are within period boundaries and current fiscal year.
- Supports ACH/wire/employee payments (`JB01`).
- Displays error messages for invalid inputs.
- Requires user confirmation to proceed.

## Tables/Files
- **Input**:
  - `APTRAN`: Transaction file (`ATPAID`).
  - `GSCONT`: System control (`GX13GL`).
  - `GLCONT`: G/L control (`GCLSYR`, `GCFFMO`).
  - `GSTABL`: G/L table (`TBPDDT`).
  - `SCREEN`: Workstation input (`PJDATE`, `CDDATE`, `KYPD`, `KYPDYY`, `CDPD`, `CDPDYY`, `YORN`).
- **Output**:
  - `APPYTR`: Payment transaction file (dates, periods, payment type).
  - `SCREEN`: Workstation output (prompts, errors).

## External Programs
- None (uses subroutines: `ONETIM`, `S1`, `S2`, `DATCHK`).

## Role in AP200 OCL
Initializes Purchase Journal by prompting and validating dates/periods, writing to `APPYTR` for downstream processing.
