The provided `AP100.rpg36.txt` is an RPG III program for IBM midrange systems (e.g., AS/400 or iSeries), called by the OCL program `AP100.ocl36.txt`. It handles the entry and validation of Accounts Payable (A/P) voucher transactions, including header and detail records, with support for various payment types (e.g., wire transfers, ACH, employee expenses) and validations for inventory, purchase orders, and vendor data. Below is an explanation of the process steps, business rules, tables used, and external programs called.

---

### Process Steps of the RPG Program (AP100)

The RPG program `AP100` manages the interactive entry and editing of A/P voucher transactions through multiple screen formats (`AP100S1`, `AP100S2`, `AP100S3`, `AP100S5`). It validates input, retrieves data from files, and updates transaction records. Here are the key process steps:

1. **Initialization**:
   - Clears variables and sets indicators to their initial state (e.g., `Z-ADD *ZERO`, `MOVE *BLANKS`, `SETOF`).
   - Sets the process type (`PTYP`) to `'NORMAL'` if blank (modification `JK01`).
   - Retrieves the current system date and time (`TIME`), formats it into various date fields (`SYTIME`, `SYDATE`, `SYDYMD`, `MONTH`, `YEAR`), and calculates century for date handling (`Y2KCEN`).
   - Checks for wire transfer (`LDAWT = 'WT'`) or employee expense (`LDAWT = 'EE'`) flags, setting indicators `22` or `23` accordingly.
   - Initializes keys for file access (e.g., `TERMKY` for terms, `GSCKEY` for carrier ID).

2. **Screen Processing**:
   - The program uses multiple screen formats:
     - **AP100S1**: Collects company number (`CONO`), vendor number (`VEND`), entry number (`ENT#`), canceled voucher number (`CNVO`), and purchase order number (`PONO`).
     - **AP100S2**: Displays and collects header information (e.g., vendor details, invoice amount, invoice number, dates, hold codes).
     - **AP100S3**: Handles detail line entry (e.g., expense G/L, amount, discount, purchase order, gallons, receipt number).
     - **AP100S5**: Allows entry of a vendor number (`VEND#`) for lookup or navigation.
   - The main loop (`DO` at line 0446) processes user input based on the screen format triggered (indicators `01`, `02`, `03`, `05` for `S1`, `S2`, `S3`, `S5` subroutines).

3. **S1 Subroutine (AP100S1 Screen)**:
   - Validates the company number (`CONO`) against `APCONT`. If invalid or deleted (`ACDEL = 'D'`), displays error message (`COM,1`) and exits.
   - Checks if vendor (`VEND`), entry number (`ENT#`), canceled voucher (`CNVO`), or purchase order (`PONO`) are blank or zero. If so, displays error (`COM,31`) and exits.
   - Searches for vendor information (`VNSRCH` subroutine) if vendor is valid. If not found, displays error (`COM,32`).
   - Retrieves the next entry number (`ACNXTE`) from `APCONT` for new entries (`ADDNEW` mode) or validates existing entry numbers (`ENT#`) against `APTRAN`.
   - Checks if job cost (`ACJCYN = 'Y'`) or purchase order (`ACPOYN = 'Y'`) processing is active, setting indicators `65` and `67`.
   - Updates `APCONT` with the next entry number and writes the record.

4. **S2 Subroutine (AP100S2 Screen)**:
   - Retrieves or clears header information (vendor name, address, invoice amount, dates, hold codes, etc.).
   - Validates vendor number (`VEND`) against `APVEND`. If invalid, deleted (`VNDEL = 'D'`), or inactive (`VNDEL = 'I'`), displays errors (`COM,2` or `COM,67`).
   - Applies vendor defaults (e.g., hold code `VNHOLD`, single check `VNSNGL`, expense G/L `VNEXGL`, terms `VNTERM`).
   - For employee vendors (`VNPRID ≠ 0`), sets hold code to `'E'` and uses employee expense G/L (`ACEEGL`).
   - Validates hold codes (`H`, `A`, `W`, `E`, `U`) and prepay codes (`P`, `A`, `W`, `E`), setting appropriate hold descriptions (`HLDD`).
   - Checks invoice number (`INV#`) for duplicates in `APTRANX`, `APOPEN`, `APOPNHC`, or `APHSTHC`. If found, requires override codes (`COM2,2`, `COM2,3`).
   - Calculates due date (`DUDT`) and discount due date (`DSDT`) using terms from `GSTABL` (subroutines `@DTE1`, `@DTE2`, `TMDATP`).
   - Ensures invoice amount (`IAMT`) is non-zero and matches detail totals (`CLCTOT` subroutine).
   - For freight invoices (`FRTL ≠ 0`), calls `AP1011` to populate detail lines with calculated freight amounts.

5. **S3 Subroutine (AP100S3 Screen)**:
   - Validates detail line fields (e.g., expense G/L `EXGL`, amount `AMT`, discount `DISC`, purchase order `PONO`, gallons `GALN`, receipt `RCPT`).
   - Ensures expense G/L (`EXGL`) matches company (`LNCO`) and is valid in `GLMAST`. If deleted (`GLDEL = 'D'`) or inactive (`GLDEL = 'I'`), displays error (`COM,7`).
   - Applies vendor default expense G/L (`VNEXGL`) if blank.
   - Validates discounts: Ensures `DISC` and `DSPC` (discount percentage) are not both non-zero (`COM,35`) and that a discount G/L (`ACDSGL`) exists (`COM,36`).
   - For sales orders (`SORN ≠ 0`), prohibits receipt numbers or gallons (`COM,57`).
   - Validates gallons/receipt requirements based on `VNGRRQ` and `GLAPCD`:
     - If `VNGRRQ = 'Y'`, requires gallons (`GALN`) and receipt (`RCPT`) (`COM,71`, `COM,72`).
     - If `VNGRRQ = 'N'`, prohibits gallons and receipt (`COM,73`, `COM,74`).
     - Ensures G/L requires gallons (`GLAPCD = 'Y'`) if vendor requires them, and vice versa (`COM,75`, `COM,76`).
   - If `GLPOCD = 'Y'` (PO required for G/L), ensures `PONO` is entered (`COM,78`).
   - Validates receipt numbers (`RCPT`) against `INFIL1` or `INTZH1`, checking quantities and prior postings (`COM,40`, `COM,42`, `COM,43`).
   - Ensures receipt code (`CLCD`) is `'O'` or `'C'` (`COM,44`).
   - Calculates freight amounts (`FRAM`) for detail lines if total freight (`FRTL`) is non-zero, ensuring detail amounts sum to invoice total (`IAMT`).
   - Updates or adds detail records to `APTRAN`.

6. **S5 Subroutine (AP100S5 Screen)**:
   - Handles vendor lookup by number (`VEND#`) or name search (`ABBR`).
   - If `VEND#` is entered, sets `VEND` and clears `ABBR`, then returns to `S1`.
   - If blank, performs a name search (`NRLFWD` subroutine) to display vendor options.

7. **Roll Forward/Backward Handling**:
   - **ROLFWD**: Navigates to the next detail line in `APTRANX`, populating fields (`DETGET`) and validating (`S3EDIT`).
   - **ROLLBK**: Navigates to the previous detail line or header, populating fields (`DETGET` or `HDRGET`) and validating (`S3EDIT` or `S2EDIT`).

8. **Date Handling**:
   - Subroutines `@DTE1` and `@DTE2` convert between Gregorian and Julian dates for due date and discount date calculations.
   - Ensures due dates (`DUDT`) and discount due dates (`DSDT`) are valid and not on holidays/weekends (`APDATE` file, modification `MG17`).

9. **Freight Invoice Validation**:
   - Checks `FRCFBH` and `FRCINH` for freight invoices:
     - For `NORMAL` mode, ensures `FRAPST` (A/P status) is blank (`COM,60`).
     - For `PAPER` mode, ensures `FRINTY = 'P'` (`COM,62`).
     - For `ARGLMS` mode, ensures `FRINTY = 'O'` or `'S'` (`COM,61`).

10. **File Updates**:
    - Adds or updates header records in `APTRAN` (e.g., `EADD 70`, `E 70N95`).
    - Adds or updates detail records in `APTRAN` (e.g., `EADD 71`, `E 71N96`).
    - Updates `APCONT` with the next entry number (`ACNXTE`).
    - Updates one-time vendor information in `APTRAN` if needed.
    - Writes to `APOPENH` for open invoices.

11. **Error Handling**:
    - Displays error messages from the `COM` and `COM2` arrays for various validation failures (e.g., invalid vendor, duplicate invoice, missing fields).
    - Sets indicator `90` to highlight errors and prevent progression until corrected.

12. **Termination**:
    - Ends the program when the user presses the end-of-job key (`KG`), setting the last record indicator (`LR`) and exiting.

---

### Business Rules

1. **Company and Vendor Validation**:
   - Company number (`CONO`) must exist in `APCONT` and not be deleted (`ACDEL ≠ 'D'`).
   - Vendor number (`VEND`) must exist in `APVEND`, not be deleted (`VNDEL ≠ 'D'`) or inactive (`VNDEL ≠ 'I'`).

2. **Invoice Number Uniqueness**:
   - Invoice numbers (`INV#`) must be unique within the batch (`APTRANX`) and not exist in open (`APOPEN`, `APOPNHC`) or paid (`APHSTHC`) invoices unless overridden (`MG10`).

3. **Hold and Prepay Codes**:
   - Hold codes (`HOLD`) must be `'H'` (hold), `'A'` (ACH), `'W'` (wire transfer), `'E'` (employee expense), or `'U'` (utility auto-payment).
   - Prepay codes (`PAID`) must be `'P'`, `'A'`, `'W'`, or `'E'`, with a corresponding check number (`PPCK`) if applicable.

4. **Gallons and Receipt Validation**:
   - If the vendor requires gallons/receipts (`VNGRRQ = 'Y'`), both `GALN` and `RCPT` must be entered.
   - If `VNGRRQ = 'N'`, gallons and receipts are prohibited.
   - If the G/L account requires gallons (`GLAPCD = 'Y'`), `GALN` must be entered and match the sign of the amount (`AMT`).
   - Receipt numbers (`RCPT`) must exist in `INFIL1` or `INTZH1`, with valid quantities and no prior A/P postings.

5. **Purchase Order Requirements**:
   - If the G/L account requires a purchase order (`GLPOCD = 'Y'`), a valid `PONO` must be entered.

6. **Discount Validation**:
   - Discount amount (`DISC`) and percentage (`DSPC`) cannot both be non-zero.
   - A discount G/L (`ACDSGL`) must exist if discounts are used.
   - Discount due date (`DSDT`) must be entered if discounts are applied, or an error is raised (`MG20`).

7. **Freight Invoices**:
   - Freight invoices must be processed in the correct mode (`NORMAL`, `PAPER`, `ARGLMS`) based on invoice type (`FRINTY`) in `FRCFBH` or `FRCINH`.
   - Freight amounts (`FRTL`) are allocated to detail lines, with calculations ensuring detail totals match the invoice total.

8. **Sales Order Restrictions**:
   - If a sales order number (`SORN`) is entered, receipt numbers and gallons are not allowed.

9. **Date Handling**:
   - Due dates and discount due dates are calculated based on vendor terms (`VNTERM`) from `GSTABL`.
   - Dates are adjusted to avoid holidays and weekends (`APDATE`).

10. **Amount Validation**:
    - Invoice amount (`IAMT`) must be non-zero and match the sum of detail line amounts (`AMT` + `FRAM`).

---

### Tables (Files) Used

The program uses the following files, defined with specific attributes (e.g., `UC` for update/create, `IF` for input, `ID` for indexed):

1. **SCREEN**: Workstation file for interactive display (1200 bytes, `WORKSTN`).
2. **APTRAN**: A/P transaction file (404 bytes, update/create, key length 10).
3. **APTRANX**: A/P transaction index file (404 bytes, input, key length 10).
4. **APTRNX**: A/P transaction index file (404 bytes, input, external key, 27 bytes).
5. **APOPEN**: A/P open items file (384 bytes, input, key length 16).
6. **APOPENH**: A/P open items history file (384 bytes, update/create, key length 16).
7. **APOPNHC**: A/P open items history control file (384 bytes, input, key length 32).
8. **APCONT**: A/P control file (256 bytes, update/create, key length 2).
9. **APVEND**: A/P vendor file (579 bytes, input, key length 7).
10. **APVENDX**: A/P vendor index file (579 bytes, input, external key, 17 bytes).
11. **GLMAST**: General ledger master file (256 bytes, input, key length 11).
12. **GSTABL**: General system table (256 bytes, input, key length 12).
13. **APINVH**: A/P invoice history file (64 bytes, input, key length 32).
14. **APHSTHC**: A/P history control file (384 bytes, input, key length 32).
15. **INFIL1**: Inventory file (448 bytes, input, external key, 9 bytes).
16. **INTZH1**: Inventory transaction holding file (592 bytes, input, external key, 9 bytes).
17. **SA5FIUD**: Sales file (detail, 1024 bytes, input, external key, 14 bytes).
18. **SA5FIUM**: Sales file (misc, 1024 bytes, input, key length 14).
19. **SA5MOUD**: Sales file (detail, 1024 bytes, input, external key, 14 bytes).
20. **SA5MOUM**: Sales file (misc, 1024 bytes, input, key length 14).
21. **FRCINH**: Freight invoice history file (206 bytes, input, key length 33).
22. **FRCFBH**: Freight feedback history file (206 bytes, input, key length 39).
23. **APDATE**: A/P date file (19 bytes, input, key length 10).

---

### External Programs Called

1. **AP1011**: Called to populate freight detail lines with calculated percentage amounts when a sales order (`SORN`) and freight total (`FRTL`) are present. It uses the `SALES` data structure as a parameter.

---

### Summary

The `AP100` RPG program is a critical component of the A/P voucher entry system, handling interactive input through multiple screens (`AP100S1`, `AP100S2`, `AP100S3`, `AP100S5`). It validates company, vendor, invoice, and detail line data, ensuring compliance with business rules such as unique invoice numbers, valid hold/prepay codes, gallons/receipt requirements, and freight invoice processing. The program integrates with numerous files for transaction, vendor, inventory, and financial data, and calls `AP1011` for freight detail calculations. It enforces strict validations to maintain data integrity and supports various payment types and processing modes (e.g., `NORMAL`, `PAPER`, `ARGLMS`).