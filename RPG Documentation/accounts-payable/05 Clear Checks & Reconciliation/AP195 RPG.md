The provided RPG program, `AP195.rpg36`, is an RPG II or RPG/400 program called from the `AP190.ocl36` OCL program as part of the Accounts Payable (A/P) check reconciliation process. It generates a report listing cancelled checks, including totals by bank G/L account and company. Below, I’ll explain the **process steps**, **business rules**, **tables (files) used**, and **external programs called**.

### Process Steps of the AP195 RPG Program

The program reads the A/P check reconciliation transaction file (`APCRTR`), retrieves company information from the control file (`APCONT`), and produces a printed report on the `LIST` file with details of cancelled checks, subtotals by bank G/L account, and company totals. Here’s a detailed breakdown of the process steps:

1. **Program Initialization**:
   - **Header (H) and File (F) Specifications** (Lines 0002–0011):
     - Defines the program (`AP195`) and files:
       - `APCRTR`: Input primary file (IP) for check reconciliation transactions.
       - `APCONT`: Input control file (IC) for company data.
       - `LIST`: Output file (O) for the printer, producing the report.
     - `APCRTR` is keyed by a 16-byte field (company + bank G/L + check number).
     - `APCONT` is keyed by a 2-byte company number.
   - **Data Structures** (Lines 0014–0022):
     - Defines input fields for `APCRTR`: `ATCONOL2` (company #), `ATBKGLL1` (bank G/L #), `ATCHK#` (check number), `ATCLAM` (clear amount), `ATCLDT` (clear date).
     - Defines `ACNAME` (company name) from `APCONT`.
   - **Purpose**: Sets up the environment for reading transaction data and generating the report.

2. **Report Initialization** (Lines 0025–0032):
   - **Level 2 (L2) Processing** (Company-level):
     - Executes at the start of a new company (`L2` indicator).
     - Retrieves the current time and date (`TIME` to `TIMDAT`, split into `SYSTIM` and `SYSDAT`).
     - Initializes the separator line (`SEP`) to `'* '`.
     - Resets the page number (`PAGE`) to zero.
     - Chains to `APCONT` using `ATCONO` (company #). If not found (`92`), `ACNAME` is not updated.
     - Resets the company total clear amount (`L2CLAM`) to zero.
   - **Purpose**: Prepares headers and totals for each company in the report.

3. **Bank G/L Level Processing** (Lines 0038–0040):
   - **Level 1 (L1) Processing** (Bank G/L-level):
     - Executes at the start of a new bank G/L number (`L1` indicator).
     - Resets the bank G/L total clear amount (`L1CLAM`) to zero.
   - **Purpose**: Initializes subtotals for each bank G/L account within a company.

4. **Detail Processing and Accumulation** (Lines 0042–0044):
   - For each `APCRTR` record:
     - Adds the clear amount (`ATCLAM`) to the bank G/L total (`L1CLAM`).
     - At the `L1` break (change in bank G/L), adds `L1CLAM` to the company total (`L2CLAM`).
   - **Purpose**: Accumulates totals for reporting at both bank G/L and company levels.

5. **Report Output** (Lines 0047–0082):
   - **Header Output** (Lines 0047–0072):
     - At `L1` break or overflow (`OFNL1`):
       - Outputs company name (`ACNAME`) if found (`N92`).
       - Prints page number (`PAGE`), system date (`SYSDAT`), and time (`SYSTIM`).
       - Prints report title (“A/P CANCELLED CHECKS EDIT”) and bank G/L number (`ATBKGL`).
       - Outputs column headers: “CHECK #”, “CLEAR DATE”, “CLEAR AMOUNT”.
       - Prints separator lines (`SEP`).
   - **Detail Lines** (Lines 0073–0076):
     - For each `APCRTR` record (`01` indicator):
       - Prints check number (`ATCHK#`), clear date (`ATCLDT`), and clear amount (`ATCLAM`).
   - **Total Lines** (Lines 0077–0082):
     - At `L1` break (after 21 lines, `T 21 L1`): Prints bank G/L total (`L1CLAM`) with label “BANK G/L # TOTAL”.
     - At `L2` break (after 21 lines, `T 21 L2`): Prints company total (`L2CLAM`) with label “COMPANY TOTAL”.
   - **Purpose**: Generates a formatted report with check details and totals.

### Business Rules

The program enforces the following business rules for the A/P cancelled checks edit report:
1. **Data Source**:
   - Reads all records from `APCRTR` sequentially, grouped by company (`ATCONOL2`) and bank G/L number (`ATBKGLL1`).
2. **Company Validation**:
   - Attempts to retrieve company name (`ACNAME`) from `APCONT` using `ATCONO`. If not found, the report omits the company name but continues processing.
3. **Report Structure**:
   - Organizes the report by company (`L2`) and bank G/L number (`L1`), with subtotals for each bank G/L and company.
   - Includes headers with company name, bank G/L number, date, time, and page number.
   - Lists check number, clear date, and clear amount for each transaction.
   - Prints totals after 21 detail lines or at level breaks (`L1`, `L2`).
4. **Formatting**:
   - Uses a separator line (`SEP = '* '`) to visually distinguish sections.
   - Formats dates (`ATCLDT`, `SYSDAT`) and amounts (`ATCLAM`, `L1CLAM`, `L2CLAM`) for readability (e.g., `Z` for zero suppression, `M` for monetary format, `Y` for date format).
5. **Accumulation**:
   - Accumulates clear amounts (`ATCLAM`) into bank G/L totals (`L1CLAM`) and company totals (`L2CLAM`) for accurate reporting.

### Tables (Files) Used

The program uses the following files:
1. **APCRTR**:
   - Input primary file (IP) for check reconciliation transactions.
   - Keyed by a 16-byte field (company + bank G/L + check number).
   - Fields: `ATCONOL2` (company #), `ATBKGLL1` (bank G/L #), `ATCHK#` (check number), `ATCLAM` (clear amount), `ATCLDT` (clear date).
   - Record length: 80 bytes.
2. **APCONT**:
   - Input control file (IC) for company data.
   - Keyed by company number (2 bytes).
   - Field: `ACNAME` (company name).
   - Record length: 256 bytes.
3. **LIST**:
   - Output file (O) for the printed report.
   - Record length: 132 bytes (standard printer width).

### External Programs Called

The program does **not** call any external programs. All processing is handled within `AP195` using its RPG logic.

### Summary

The `AP195` RPG program generates a report for A/P cancelled checks:
- **Process**: Reads `APCRTR` for check reconciliation data, retrieves company names from `APCONT`, accumulates totals by bank G/L and company, and outputs a formatted report to `LIST` with headers, detail lines, and totals.
- **Business Rules**: Groups data by company and bank G/L, validates company numbers, formats output for readability, and provides totals after 21 lines or level breaks.
- **Files Used**: `APCRTR` (input), `APCONT` (input), `LIST` (output).
- **External Programs**: None.

This program complements the `AP190` program (data entry and validation) by producing a final edit report, as orchestrated by the `AP190.ocl36` OCL script.