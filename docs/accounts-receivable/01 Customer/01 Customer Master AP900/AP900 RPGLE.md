The `AR900.rpgle.txt` is an RPGLE (RPG IV) program called by the `AR900.ocl36.txt` OCL program, designed for maintaining the Accounts Receivable (AR) customer master file. It handles adding, updating, and reactivating customer records, with additional features like history tracking and validation. Below, I’ll explain the process steps, business rules, tables used, and external programs called.

---

### Process Steps of the RPGLE Program (AR900)

The program is a screen-based application for customer master file maintenance, operating in two modes: **Entry Mode** (add new customers) and **Update Mode** (modify existing customers). It uses multiple display file formats (`FMT01`, `FMT02`, `FMT03`, `FMT05`) to interact with users and updates files like `ARCUST` (customer master) and `ARCUSP` (supplemental customer data). Here’s a step-by-step breakdown of the process:

1. **Program Initialization (`ONETIM` Subroutine)**:
   - Initializes variables (e.g., `CO`, `CUSN`, `ZERO5`, `ZERO6`, etc.) to zero or blanks.
   - Sets up key fields for table lookups (`SLKEY`, `TRMKEY`, `CUCLKY`, `GRUPKY`).
   - Captures system date (`UDATE`) and time (`SYSTIM`) for history file updates, formatting the date as `SYSCYM` (e.g., `20YYMMDD`).
   - Sets the initial screen format (`FMT01`) and indicator `81` for display.

2. **Main Processing Loop**:
   - The program enters a loop (`fmtagn`) that displays and processes screen formats until terminated (e.g., via `KG` for End of Job).
   - Depending on indicators (`*IN81`, `*IN82`, `*IN83`, `*IN85`), it displays one of the following:
     - `FMT01`: Initial customer lookup screen.
     - `FMT02`: Customer details entry/update screen.
     - `FMT03`: Supplemental customer data screen.
     - `FMT05`: Additional maintenance screen (e.g., for ship-to products).
   - Clears error messages (`MSG1`, `MSG2`) and indicators before each screen display.

3. **Screen Processing**:
   - **S1 (FMT01 - Customer Lookup)**:
     - Validates the company number (`CO`) against `ARCONT`. If invalid, displays error message `MSG(1)` ("INVALID COMPANY NUMBER ENTERED").
     - Retrieves aging limits (`ACLMT1`, `ACLMT2`, `ACLMT3`, `ACLMT4`) from `ARCONT` for display.
     - Chains to `ARCUST` and `ARCUSP` using `ARKEY` (company + customer number). If not found, displays errors (`MSG(2)`, `MSG(22)`).
     - Checks if the customer is deleted/inactive (`ARDEL = 'D'` or `'I'`). If so, sets indicator `94` and displays `MSG(6)` ("THIS CUSTOMER WAS PREVIOUSLY DELETED").
     - Retrieves billing instructions (`BCINST`) from `BICONT` to set the screen header (`S4HEAD`).
     - Calls `GETCUS` and `GETSUP` to populate customer and supplemental data for display.
     - Sets indicator `82` to display `FMT02`.

   - **S2 (FMT02 - Customer Details)**:
     - Handles both Entry (`08`) and Update (`07`) modes.
     - In Entry Mode, validates that the customer number (`CUSN`) is not zero (`MSG(19)`) and doesn’t exist in `ARCUST` (`MSG(7)`).
     - Performs validations via `SC2EDT` subroutine (see Business Rules).
     - If validations pass, chains to `ARCUSP` to check for existing supplemental records. If found, displays errors (`MSG(26)`, `MSG(27)`).
     - Sets indicator `83` to display `FMT03`.

   - **S3 (FMT03 - Supplemental Data)**:
     - Validates date fields (`STDATE`, `FSDATE`, `ICDATE`) for start date, last financial statement date, and insurance certification expiration date.
     - Adds century to dates if not provided, using `Y2KCEN` (e.g., `19` or `20`) based on `Y2KCMP`.
     - Calls `@DTEDT` to validate date formats (e.g., checks for valid months, days, leap years). If invalid, sets indicator `79` and displays errors (`MSG(44)`, `MSG(45)`, `MSG(46)`).
     - Calls `BI907AC` to maintain ship-to products in `ARCUPR` (replacing previous `S4` logic).
     - Sets indicator `85` to display `FMT05`.

   - **S5 (FMT05 - Final Processing)**:
     - Updates or adds records to `ARCUST` and `ARCUSP` based on mode (`07` or `08`).
     - Updates `ARCLGR` (group ledger) to synchronize the credit problem flag (`CLPB`) across grouped accounts.
     - Writes history records to `ARCUSHS` (customer history) and `ARCUSPH` (supplemental history) with system date, time, user ID, and workstation ID.
     - Clears fields via `CLEAR` subroutine and redisplays `FMT02`.

4. **Command Key Processing**:
   - **KA (Rekey)**: Clears fields (`CLEAR`) and redisplays `FMT01` or `FMT02` based on mode.
   - **KD (Delete/Reactivate)**:
     - **Delete**: Checks if the customer has outstanding invoices (`BAL ≠ 0`). If so, displays `MSG(9)` and `MSG(10)` ("THIS CUSTOMER HAS OUTSTANDING INVOICES"). If not, marks `ARCUST` and `ARCUSP` as inactive (`'I'`) instead of deleting (per revision note).
     - **Reactivate**: Clears the delete flag (`ARDEL`, `CSDEL`) and sets status to active (`'A'`), displaying `MSG(12)` ("PREVIOUS CUSTOMER WAS REACTIVATED").
   - **KG (End of Job)**: Sets `LR` (Last Record) indicator, closes files, and exits.
   - **KJ (Entry Mode)**: Clears fields, sets `08` (Entry Mode), and displays `FMT02`.
   - **KK (Update Mode)**: Clears fields, sets `07` (Update Mode), and displays `FMT02`.

5. **End of Program**:
   - Closes all files, sets `*INLR = *ON`, and returns.

---

### Business Rules

The program enforces the following business rules:
1. **Customer Number Validation**:
   - Must not be zero in Entry Mode (`MSG(19)`).
   - Must not already exist in `ARCUST` for new records (`MSG(7)`).
   - Must exist in `ARCUST` and `ARCUSP` for updates.

2. **Mandatory Fields**:
   - Sort name (`LSTN`) must not be blank (`MSG(21)`).
   - Federal EIN (`FEIN`) is required (`MSG(55)`).
   - State (`STAT`) must not be blank (`MSG(29)`).

3. **Code Validations** (checked against `GSTABL`):
   - Salesman number (`SLS#`): Must exist in `GSTABL` (`MSG(13)`).
   - Terms code (`TERM`): Must exist in `GSTABL` (`MSG(14)`).
   - Customer class (`CCLASS`): Must exist in `GSTABL` under `ARCUCL` table type (`MSG(47)`).

4. **Valid Codes**:
   - Statement code (`STMT`): Must be `Y`, `N`, or blank (`MSG(16)`).
   - Finance charge code (`FINC`): Must be `Y`, `N`, or blank (`MSG(17)`).
   - Separate freight code (`SFRT`): Must be `Y`, `N`, or blank (`MSG(30)`).
   - Freight code (`FRCD`): Must be `C` (Collect), `P` (Prepaid), `A` (Prepaid & Add), or blank (`MSG(35)`).
   - EFT participant (`EFT`): Must be `Y`, `N`, or blank (`MSG(32)`).
   - EDI participant (`EDI`): Must be `Y`, `N`, or blank (`MSG(50)`).
   - Credit problem code (`CLPB`): Must be `Y`, `N`, or blank (`MSG(51)`).
   - Wire transfer code (`WIRE`): Must be `Y`, `N`, or blank (`MSG(52)`).
   - Duplicate order match type (`DUPC`): Must be `A` (match on customer/ship-to/product), `B` (match on customer/ship-to/product/PO#), or blank (`MSG(53)`).
   - Product move code (`PRMV`): Must be `Y`, `N`, or blank (`MSG(50)`).

5. **EFT Validation**:
   - If `AREFT = 'Y'`, bank routing code (`CSARTE`) and account number (`CSABK#`) must not be blank (`indicator 40`).

6. **Date Validation**:
   - Validates `STDATE` (start date), `FSDATE` (financial statement date), and `ICDATE` (insurance certification date) using `@DTEDT`.
   - Ensures valid months (1-12), days (1-31 or 1-28/29 for February), and leap year logic.
   - Adds century (`Y2KCEN`) to two-digit years based on `Y2KCMP`.

7. **Group Credit Synchronization**:
   - Updates `CLPB` (credit problem flag) in `ARCUST2` for all customers in the same group (`ARCLGR`) when modified.

8. **Deletion Rules**:
   - Customers with outstanding invoices (`BAL ≠ 0`) cannot be deleted (`MSG(9)`, `MSG(10)`).
   - Deletion marks records as inactive (`'I'`) instead of physical deletion (`JB11` revision).

9. **Freight Handling**:
   - Supports `C` (freight collect), `P` (prepaid), `A` (prepaid & add) (`JB09`, `JB10`).
   - Special case for freight collect with a $100 service fee (`CYY`) when shipping is arranged by ARG but billed by the carrier.

---

### Tables (Files) Used

The program uses the following files:
1. **AR900D** (`WORKSTN`): Display file for user interaction, externally described with `PROFOUNDUI` handler (`JB11`).
2. **ARCUST** (`UF A`, 384 bytes, key starts at position 2): Customer master file for add/update.
3. **ARCUSP** (`UF A`, 1344 bytes, key starts at position 2): Supplemental customer data file.
4. **GSTABL** (`IF`, 256 bytes, key starts at position 2): General system table for validating salesman, terms, and customer class codes.
5. **ARCONT** (`IF`, 256 bytes, key starts at position 2): AR control file for aging limits.
6. **BICONT** (`IF`, 256 bytes, key starts at position 2): Billing control file for invoicing style.
7. **ARCLGR** (`IF`, 240 bytes, key starts at position 2): Customer group ledger file for credit synchronization.
8. **ARCUST2** (`UF`, 384 bytes, key starts at position 2): Alternate customer master file for group updates.
9. **ARCUSHS** (`O A`, 411 bytes, key starts at position 2): Customer history file.
10. **ARCUSPH** (`O A`, 1371 bytes, key starts at position 2): Supplemental customer history file.

---

### External Programs Called

The program calls the following external programs:
1. **BI907AC** (`JB08`): Maintains ship-to products in `ARCUPR` (replaces `S4` logic).
   - Parameters: Company (`@CPCO`), Customer (`@CPCUS`), Ship-to (`@CPSHP`), Mode (`@CPMI = 'MNT'`), File Group (`@CPFGR`).

Note: The following calls are commented out (`DC01`, `JK01`), indicating they are no longer used but were part of earlier versions:
- **AR915P**: For customer form type contacts maintenance (`ARCUFM`).
- **GB730P**: For history inquiries on `ARCUST` and `ARCUSP`.

---

### Summary

The `AR900` RPGLE program is a comprehensive customer master maintenance application that:
- Manages customer data (`ARCUST`, `ARCUSP`) through interactive screens (`FMT01`–`FMT05`).
- Supports adding, updating, and reactivating customers with strict validations (e.g., mandatory fields, valid codes, date checks).
- Synchronizes credit problem flags across grouped accounts (`ARCLGR`).
- Maintains history files (`ARCUSHS`, `ARCUSPH`) for auditing.
- Integrates with `BI907AC` for ship-to product maintenance.
- Uses 10 files and calls 1 active external program (`BI907AC`).

The program enforces robust business rules to ensure data integrity, including validations against `GSTABL`, restrictions on deletions, and special freight handling logic.