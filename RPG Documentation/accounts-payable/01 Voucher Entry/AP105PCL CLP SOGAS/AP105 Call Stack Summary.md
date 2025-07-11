### Use Cases Implemented by the Program Call Stack

The call stack consists of the following programs:
- **AP105PCL.clp** (IBM i CLP): Entry point, sets up file overrides and calls other programs.
- **AP105P.rpg** (IBM i RPG): Validates A/P SOGAS ACH payment data and generates error reports.
- **AP105.ocl36** (System/36 OCL): Manages file setup and calls System/36 programs.
- **AP105.rpg36** (System/36 RPG): Processes A/P voucher entry and creates transaction records.
- **AP105A** (System/36 program, not provided): Copies data to a history file.

Based on the analysis of these programs, the primary use case implemented by this call stack is:

1. **Validate and Process A/P SOGAS ACH Payment Vouchers**:
   - This use case involves validating ACH payment data from an upload file, ensuring owners and vendors exist, adjusting due dates to avoid holidays/weekends, creating A/P voucher transactions, applying hold status based on vendor settings, and archiving processed data to a history file.

No additional distinct use cases are evident, as the programs work together to achieve this single, cohesive process.

---

### Function Requirements Document: Process A/P SOGAS ACH Payment Vouchers

**Document Title**: Process A/P SOGAS ACH Payment Vouchers  
**Artifact ID**: 7a8b9c0d-e29b-41d4-a716-446655440001  
**Content Type**: text/markdown



# Function Requirements: Process A/P SOGAS ACH Payment Vouchers

## Purpose
To validate A/P SOGAS ACH payment data, create voucher transactions, adjust due dates, apply vendor hold status, and archive processed data without requiring interactive screen input.

## Inputs
- **File Group Prefix (FGRP)**: 1-character string to prefix file names (e.g., environment identifier).
- **Payment Data (APSOGAS)**: Records containing:
  - Owner Number (ANOWNR, 7 chars)
  - Check Amount (ANCHAM, packed decimal)
  - Check Name (ANCHNM, 7 chars)
  - Due Date (ANDUDT, 10 chars, MMDDYY format)
  - Description (DESC, 13 chars, from LDA)
- **Control Flags (LDA)**:
  - Canadian Tax Flag (CANTAX, 'Y' or blank)
  - Upload Type (UPTYPE, 'T' or blank)
  - Century Data (Y2KCEN, Y2KCMP for date calculations)
- **Reference Data**:
  - APSGACH: Owner-to-vendor mappings (AGOWNR, AGVEND).
  - APVEND: Vendor details (VNVEND, VNHOLD, VNTERM, etc.).
  - APCONT: Control data (ACAPGL, ACCAGL, ACNXTE).
  - APDATE: Non-holiday/weekend due dates (ADDUD8, ADNED8).

## Outputs
- **Transaction File (APTRAN)**: Header and detail records for A/P vouchers.
- **History File (APSOGSH)**: Archived payment data.
- **Error Report (LIST132)**: Validation errors (owner/vendor not found, counts).
- **Updated Control File (APCONT)**: Updated next entry number (ACNXTE).
- **Local Data Area (LDA)**: Updated with error flag (ERRORS = 'Y' or blank).

## Process Steps (Pseudocode Summary)
1. **Initialize**:
   - Construct file names: APSOGAS = FGRP + 'APSOGAS', APSGACH = FGRP + 'APSGACH', etc.
   - Override files to use FGRP-prefixed names.
   - Initialize counters (COUNT, ERRCNT = 0).
   - Set system date/time (SYDYMD = YYYYMMDD).
   - If CANTAX = 'Y', set CanadianTaxFlag = true.
   - If UPTYPE = 'T', set UploadTypeFlag = true.
   - If !CanadianTaxFlag && !UploadTypeFlag, set ERRORS = 'Y', exit.

2. **Validate Payment Data**:
   - For each APSOGAS record:
     - Chain APSGACH with ANOWNR:
       - If not found, increment ERRCNT, set ERRORS = 'Y', write error to LIST132, skip.
     - Chain APVEND with AGVEND from APSGACH:
       - If not found, increment ERRCNT, set ERRORS = 'Y', write error to LIST132, skip.
     - If VNHOLD = 'H', set HoldFlag = true.

3. **Adjust Due Date**:
   - Convert ANDUDT to DUDT8 (YYYYMMDD).
   - Chain APDATE with DUDT8:
     - If found, replace DUDT8 with ADNED8 (non-holiday/weekend date).

4. **Create Voucher**:
   - Retrieve ACAPGL (A/P GL), ACCAGL (Bank GL), ACNXTE (next entry) from APCONT.
   - If ENT# = 0, set ENT# = ACNXTE, increment ACNXTE.
   - If ENT# ≥ 99999, reset ACNXTE = 1.
   - Write APTRAN header:
     - Fields: 'A', '10', ENT#, VNVEND, ANCHNM, DUDT8, HoldFlag ? 'H' : 'A', etc.
   - Write APTRAN detail:
     - Fields: '10', ENT#, NXLINE, VNVEND, EXGL (CANTAX ? 12010009 : 12010008), ANCHAM, etc.
   - Increment NXLINE, ENT#.

5. **Archive Data**:
   - Copy APSOGAS record to APSOGSH with user ID and workstation ID.

6. **Clean Up**:
   - Update APCONT with new ACNXTE.
   - Delete file overrides.
   - Clear LDA (positions 1–512).
   - If ERRORS = 'Y', exit; else continue.

## Business Requirements
- **Dynamic File Naming**: Use FGRP to prefix file names for environment flexibility.
- **Validation**:
  - ANOWNR must exist in APSGACH.
  - AGVEND must exist in APVEND.
  - Log errors to LIST132 if validations fail.
- **Due Date Adjustment**: Replace due date with non-holiday/weekend date from APDATE.
- **Hold Status**: If VNHOLD = 'H', mark voucher as 'H' with 'VENDOR ON HOLD'; else 'A' with 'ON HOLD FOR ACH'.
- **GL Assignment**: Use 12010009 for Canadian tax (CANTAX = 'Y'); else 12010008.
- **Entry Number Management**: Ensure unique ENT# from ACNXTE, reset at 99999.
- **Archiving**: Copy processed APSOGAS records to APSOGSH with audit data (user, workstation).
- **Error Handling**: Set ERRORS = 'Y' in LDA for invalid CANTAX/UPTYPE or validation failures, skip further processing.
- **Y2K Compliance**: Adjust dates using Y2KCEN/Y2KCMP for century correctness.

## Constraints
- No interactive screen input; process all data programmatically.
- Files are shared (DISP-SHR) for concurrent access.
- APTRAN is extended by 100 records.
- All files use first member (*FIRST).
- Century handling assumes Y2KCEN = 19, Y2KCMP = 80.

## Assumptions
- Input files (APSOGAS, APSGACH, APVEND, APCONT, APDATE) are populated and accessible.
- APSOGSH is available for archiving.
- LDA is initialized with valid CANTAX, UPTYPE, Y2KCEN, Y2KCMP values.





