The provided RPG program, `AP190.rpg36`, is an RPG II or RPG/400 program designed for Accounts Payable (A/P) cancelled check entry and reconciliation, executed from the `AP190.ocl36` OCL program. Below, I’ll explain the **process steps**, **business rules**, **tables (files) used**, and **external programs called**.

### Process Steps of the AP190 RPG Program

The program facilitates data entry and validation for cancelled checks through two display formats (`AP190S1` and `AP190S2`) on a workstation screen, performing checks against various files and updating the A/P check reconciliation work file (`APCRTR`). Here’s a detailed breakdown of the process steps:

1. **Program Initialization**:
   - **Header (H) and File (F) Specifications**:
     - Defines the program (`AP190`) and files: `SCREEN` (workstation display), `APCRTR` (update file), `APCONT` (control file), `GLMAST` (general ledger master), and `APCHKR` (check reconciliation file).
     - `SCREEN` uses `KINFSR ROLLKY` for handling roll keys (page up/down) and `KINFDS INFDS` for status information.
   - **Indicator Setup**:
     - Lines 0072–0074: Initializes indicators (30–34, 81–82, 90–91) to off, ensuring a clean state.
     - Line 0075: Clears the message field (`MSG60`) to blanks.
   - **Purpose**: Prepares the program environment for processing user input and file operations.

2. **Handle Roll Keys**:
   - **Subroutine ROLLKY** (Lines 0357–0363):
     - Checks workstation status codes (`STATUS`) to detect roll forward (up, code 01122, sets indicator 18) or roll backward (down, code 01123, sets indicator 19).
   - **Subroutine ROLLFW (Roll Forward)** (Lines 0365–0380):
     - Reads the next record from `APCRTR` using `SCKEY` (search key, company + bank G/L + check number).
     - If a record is found (`N60`), moves the key (`ATKEY`) to `SCKEY`, calls `S1` to display, and sets indicators.
     - If end-of-file (`60`), displays message “END OF FILE HAS BEEN REACHED” and clears fields.
   - **Subroutine ROLLBW (Roll Backward)** (Lines 0382–0394):
     - Reads the previous record from `APCRTR` using `SCKEY`.
     - Similar logic to `ROLLFW`, but displays “BEGIN OF FILE HAS BEEN REACHED” if no prior record exists.
   - **Purpose**: Allows users to navigate through `APCRTR` records using roll keys.

3. **Process Function Keys**:
   - **KA (Rekey, No Add/Update)** (Lines 0080–0085):
     - Calls `CLEAR` subroutine to reset fields, sets indicators (32, 81 on; 01, 02, 09 off), and jumps to `END`.
   - **KD (Delete Record)** (Lines 0087–0092):
     - Calls `DELETE` subroutine to remove a record from `APCRTR`, sets indicators, and jumps to `END`.
   - **KG (End of Job)** (Lines 0094–0099):
     - Sets the Last Record (`LR`) indicator to terminate the program, clears indicators, and jumps to `END`.
   - **Purpose**: Handles user commands for rekeying, deleting, or ending the job.

4. **Screen 1 Processing (AP190S1, Subroutine S1)** (Lines 0114–0184):
   - **Input Validation**:
     - Reads user input: company number (`CONO`), bank G/L number (`BKGL`), and check number (`CHK#`).
     - **Company Validation** (Lines 0116–0123):
       - Chains to `APCONT` using `CONO`. If not found (`30`) or deleted (`ACDEL = 'D'`), sets error indicators (81, 90), displays “INVALID COMPANY #”, and exits.
       - Moves company name (`ACNAME`) to `CONONM` for display.
     - **Bank G/L Validation** (Lines 0125–0135):
       - Constructs `GLKEY` from `CONO` and `BKGL`, chains to `GLMAST`. If not found (`31`) or deleted (`GLDEL = 'D'`), sets error indicators, displays “INVALID BANK G/L #”, and exits.
       - Moves G/L description (`GLDESC`) to `BKGLNM`.
     - **Check Validation** (Lines 0137–0172):
       - Chains to `APCHKR` using `SCKEY` (company + bank G/L + check #). If not found (`32`), sets error indicators, displays “CHECK # NOT FOUND” or other status messages based on `AMCODE`:
         - `D`: “CHECK WAS PREVIOUSLY DELETED”.
         - `R`: “CHECK IS ALREADY RECONCILED”.
         - `V`: “CHECK WAS PREVIOUSLY VOIDED”.
         - `O`: Valid open check; moves vendor number (`AMVEN#`), check amount (`AMCKAM`), check date (`AMCKDT`), and vendor name (`AMVNNM`) to variables.
       - Chains to `APCRTR` to check for existing reconciliation record (`SCKEY`). If found (`N92`), retrieves clear date (`ATCLDT`) and amount (`ATCLAM`); if not, sets indicator 34 for new record.
   - **Display**:
     - Sets indicator 82 to display `AP190S2` (next screen) if no errors (81 off).
     - Outputs `AP190S1` with company, bank G/L, check number, and error message (if any).
   - **Purpose**: Validates user-entered company, bank G/L, and check number, retrieving associated data for display.

5. **Screen 2 Processing (AP190S2, Subroutine S2)** (Lines 0186–0217):
   - **Input Validation**:
     - Reads clear date (`CLDT`, `CLYY`) and clear amount (`CLAM`) from the user.
     - **Date Validation** (Lines 0188–0205):
       - Tests `DPCLDT` (clear date) for valid numeric format using `TESTB`. If invalid (`99`), restores saved date (`SVCLDT`) and proceeds.
       - Calls subroutine `DTEDIT` to validate `CLDT` (MMDDYY format):
         - Breaks down date into month (`$MONTH`), day (`$DAY`), and year (`$YR`).
         - Validates month (1–12) and day based on month and leap year rules (e.g., February 28/29, other months 30/31).
         - Handles Y2K compliance by determining century (`Y2KCEN`) and constructing an 8-digit date (`CLDT8`).
         - Sets indicator 79 if the date is invalid, displaying “CLEAR DATE IS INVALID”.
     - **Amount Validation** (Lines 0207–0210):
       - Compares clear amount (`CLAM`) to check amount (`AMCKAM`). If mismatched (`34`), sets error indicators (82, 90), displays “CLEAR AMOUNT DOES NOT MATCH”, and exits.
   - **Record Update**:
     - If no errors, writes or updates `APCRTR` with `OUTREC` (clear amount, date, and key).
     - Calls `CLEAR` to reset fields and sets indicator 32 for the next entry.
   - **Display**:
     - Outputs `AP190S2` with company, bank G/L, check number, vendor details, check date/amount, clear date/amount, and error message (if any).
   - **Purpose**: Validates and stores the clear date and amount, updating the reconciliation file.

6. **Clear Subroutine** (Lines 0334–0347):
   - Resets fields: `CHK#`, `CLDT`, `CLAM`, `VEN#`, `CKAM`, `CKDT`, `VEN#NM`, and `CLDT8` to zeros or blanks.
   - Saves `CLDT` to `SVCLDT` for recovery.
   - **Purpose**: Clears variables for the next entry.

7. **Delete Subroutine** (Lines 0349–0355):
   - Chains to `APCRTR` using `SCKEY`. If found (`N92`), writes a delete record (`DELREC`).
   - Calls `CLEAR` to reset fields.
   - **Purpose**: Deletes a reconciliation record from `APCRTR`.

8. **Program Termination** (Line 0110):
   - Jumps to `END` tag after processing, resetting roll key indicators (18, 19).
   - **Purpose**: Completes the program cycle.

### Business Rules

The program enforces the following business rules for A/P check reconciliation:
1. **Company Validation**:
   - The company number (`CONO`) must exist in `APCONT` and not be marked as deleted (`ACDEL ≠ 'D'`).
2. **Bank G/L Validation**:
   - The bank G/L number (`BKGL`) combined with `CONO` must exist in `GLMAST` and not be deleted (`GLDEL ≠ 'D'`).
3. **Check Validation**:
   - The check number (`CHK#`) must exist in `APCHKR` and have a valid status (`AMCODE = 'O'` for open checks).
   - Checks with `AMCODE` of `D` (deleted), `R` (reconciled), or `V` (voided) are invalid for processing.
4. **Clear Date Validation**:
   - The clear date (`CLDT`) must be a valid date in MMDDYY format, with proper month (1–12) and day (28–31, accounting for leap years).
   - Handles Y2K compliance by determining the century based on `Y2KCMP` and `Y2KCEN`.
5. **Clear Amount Validation**:
   - The clear amount (`CLAM`) must match the check amount (`AMCKAM`) from `APCHKR`.
6. **Record Management**:
   - Existing `APCRTR` records can be updated or deleted; new records are added if none exist.
   - Duplicate key processing is supported by redefining `CLDT` as alphanumeric (`DPCLDT`) for validation.
7. **User Interface**:
   - Errors are displayed on the screen with specific messages (e.g., “INVALID COMPANY #”, “CLEAR DATE IS INVALID”).
   - Roll keys allow navigation through existing reconciliation records.
   - Function keys (`KA`, `KD`, `KG`) control rekeying, deletion, or program termination.

### Tables (Files) Used

The program uses the following files:
1. **SCREEN**:
   - Workstation file for display formats `AP190S1` (input company, bank G/L, check number) and `AP190S2` (input clear date, amount, and display vendor/check details).
   - Record length: 500 bytes.
2. **APCRTR**:
   - Update file (UF) for check reconciliation transactions.
   - Keyed by `ATKEY` (16 bytes, company + bank G/L + check number).
   - Fields: `ATCLAM` (clear amount), `ATCLDT` (clear date).
   - Record length: 80 bytes.
3. **APCONT**:
   - Input file (IC) for company control data.
   - Keyed by company number (2 bytes).
   - Fields: `ACDEL` (delete flag), `ACNAME` (company name).
   - Record length: 256 bytes.
4. **GLMAST**:
   - Input file (IC) for general ledger master data.
   - Keyed by `GLKEY` (11 bytes, company + bank G/L).
   - Fields: `GLDEL` (delete flag), `GLDESC` (description).
   - Record length: 256 bytes.
5. **APCHKR**:
   - Input file (IC) for check reconciliation data.
   - Keyed by check number (16 bytes).
   - Fields: `AMCODE` (status: D, O, R, V), `AMVEN#` (vendor number), `AMCKAM` (check amount), `AMCKDT` (check date), `AMVNNM` (vendor name).
   - Record length: 128 bytes.

### External Programs Called

The program does **not** explicitly call any external programs. All processing is handled within `AP190` through its subroutines (`S1`, `S2`, `DTEDIT`, `CLEAR`, `DELETE`, `ROLLKY`, `ROLLFW`, `ROLLBW`).

### Summary

The `AP190` RPG program is a data entry and validation tool for A/P check reconciliation:
- **Process**: Displays two screens (`AP190S1` for inputting company, bank G/L, and check number; `AP190S2` for clear date and amount), validates inputs against `APCONT`, `GLMAST`, and `APCHKR`, and updates `APCRTR`. Supports navigation (roll keys), deletion, and rekeying.
- **Business Rules**: Ensures valid company, bank G/L, and check data; enforces date and amount accuracy; manages record updates/deletions; and provides error feedback.
- **Files Used**: `SCREEN`, `APCRTR`, `APCONT`, `GLMAST`, `APCHKR`.
- **External Programs**: None.

This program integrates with the `AP190.ocl36` OCL script, which sets up files and calls `AP190` and `AP195` for a complete reconciliation workflow.