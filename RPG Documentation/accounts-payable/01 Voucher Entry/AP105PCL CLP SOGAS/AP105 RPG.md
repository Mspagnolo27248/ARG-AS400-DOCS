Let’s analyze the provided System/36 RPG program `AP105.rpg36`, which is called by the OCL program `AP105.ocl36` via the `STRS36PRC` command in the CLP program `AP105PCL.clp`. This program is part of the A/P SOGAS ACH payment processing system, specifically for voucher entry and editing. Below, I’ll outline the process steps, business rules, tables used, and external programs called.

---

### **Process Steps**
The RPG program `AP105.rpg36` performs the following steps:

1. **File and Data Structure Declarations**:
   - Defines files:
     - `APSOGAS`: Input primary file (127 bytes, disk).
     - `APSGACH`: Input file with keyed access (64 bytes, 7-byte key, disk).
     - `APTRAN`: Update file with keyed access (404 bytes, 10-byte key, disk).
     - `APCONT`: Update file with keyed access (256 bytes, 2-byte key, disk).
     - `APVEND`: Input file with keyed access (579 bytes, 7-byte key, disk).
     - `APDATE`: Input file with keyed access (19 bytes, 10-byte key, disk).
   - Defines data structures:
     - `CKDATE` (6 bytes) with subfields `CKDMM` (month), `CKDDD` (day), `CKDYY` (year).
     - User Data Structure (UDS) for Local Data Area (LDA) with fields:
       - `CANCEL` (positions 100–105)
       - `ERRORS` (position 106)
       - `DESC` (positions 107–119)
       - `CANTAX` (position 120, Canadian tax upload flag)
       - `LDAWT` (positions 198–199, wire transfer flag)
       - `Y2KCEN` (positions 509–510, century, e.g., 19 for 1900s)
       - `Y2KCMP` (positions 511–512, comparison year, e.g., 80)

2. **Initialization**:
   - Initializes variables:
     - `Z5` and `Z3` (5- and 3-digit numerics) to 0.
     - `MSG` and `MSG2` (40-character message fields) to blanks.
     - `PTYP` (process type) to 'SOGAS '.
     - Clears indicators `50`, `51`, `53`, `60`, `61`, `55`, and `91`.
   - If indicator `09` is off:
     - Clears `TERMKY` (12 characters) and sets it to 'APTERM'.
     - Clears `GSCKEY` (12 characters).
     - Sets `POKEY` (11 characters) to '000'.
     - Captures system time (`TIMDAT`, 12 digits) and extracts:
       - `SYTIME` (6-digit time).
       - `SYDATE` (6-digit date).
       - `SYDYMD` (YYYYMMDD format) by multiplying `SYDATE` by 10000.01.
       - `MONTH` (2 digits) and `YEAR` (2 digits) from `SYDATE`.
     - If `CANTAX` = 'Y', sets indicator `54` on.
     - Sets indicator `09` on.

3. **Retrieve Control Data**:
   - Chains to `APCONT` using a hardcoded key `10` (indicator `50` on if not found).
   - Sets `APGL` (A/P GL number) from `ACAPGL`.
   - Sets `BKGL` (bank GL number) from `ACCAGL`.

4. **Prepare Entry Number**:
   - Builds a company entry key `COENT` (7 characters) by combining '10' and `ENT#` (entry number).
   - Sets `CHKAMT` (11.2 digits) to `ANCHAM` (check amount from `APSOGAS`).

5. **Process Dates and Invoice Data**:
   - Builds invoice and due dates:
     - Constructs `YMD6` (6 digits) from `DUDMM` (month), `DUDDD` (day), and `DUDYY` (year) from `APSOGAS`.
     - Converts system date `SYDYMD` to `MMDDYY` format (6 digits) by multiplying by 100.0001.
     - Sets `INDT` (invoice date) to `MMDDYY`.
     - Sets invoice number `INV#` to `ANCHNM` (check name) or `DESC` (description from LDA).
   - Validates invoice date (`INDT`):
     - Converts to `IYMD` (YYYYMMDD) by multiplying by 10000.01.
     - Extracts century `IYY` (2 digits).
     - If `IYY` >= `Y2KCMP` (e.g., 80), sets `ICN` (century) to `Y2KCEN` (e.g., 19); otherwise, adds 1 to `Y2KCEN`.
     - Builds `INDT8` (8-digit date) using `ICN` and `IYMD`.
     - Subtracts 1 year from `INDT8` to get `CMPDT8`.
   - Validates due date (`DUDT`):
     - Converts `YMD6` to `DUDT` (MMDDYY) by multiplying by 100.0001.
     - Extracts century `DYY` (2 digits).
     - If `DYY` >= `Y2KCMP`, sets `DCN` to `Y2KCEN`; otherwise, adds 1 to `Y2KCEN`.
     - Builds `DUDT8` (8-digit date) using `DCN` and `YMD6`.
     - Chains to `APDATE` using `DUDT8` as the key (`DATEKY`).
     - If found (indicator `92` off), replaces `DUDT8` and `DUDT` with `ADNED8` (non-holiday/weekend due date from `APDATE`).

6. **Validate Vendor**:
   - Chains to `APSGACH` using `ANOWNR` (owner number from `APSOGAS`) (indicator `99` on if not found).
   - If not found, jumps to `SKIP1` tag.
   - If found, builds `VNKEY` (7 characters) using '10' and `AGVEND` (vendor number from `APSGACH`).
   - Chains to `APVEND` using `VNKEY` (indicator `53` on if not found).
   - If not found, jumps to `SKIP1` tag.
   - If found, moves vendor data to fields:
     - `VNAM` (vendor name), `VAD1–VAD4` (address lines).
   - If `VNHOLD` = 'H', sets indicator `55` on (vendor on hold).

7. **Add Transaction Header**:
   - Calls subroutine `HDRADD`:
     - Chains to `APTRAN` using `KEYENT` (entry key) (indicator `95` on if found).
     - Sets indicator `70` on, writes header record (`EXCPTHEADER`), and sets `70` off.
   - Increments `NXLINE` (next line number, 3 digits) by 1.
   - Writes detail record (`EXCPTDETAIL`).
   - Increments `ENT#` (entry number).

8. **Check and Update Entry Number**:
   - If `ENT#` = 0 (indicator `60` on):
     - Builds `KEYENT` (10 characters) from `COENT` or '000' if not set.
     - Chains to `APTRAN` using `KEYENT` (indicator `51` on if found).
     - If found, writes `RELAPC` exception to release `APCONT` and jumps to `ENDS1`.
   - If `ENT#` ≠ 0:
     - Sets `RECSTS` to 'ADDNEW'.
     - Sets `ENT#` to `ACNXTE` (next entry number from `APCONT`).
     - Increments `ACNXTE` and `ENT#` unless `ENT#` ≥ 99999 (indicator `61` on), then resets `ACNXTE` to 1.
     - Updates `COENT` with `ENT#`.
     - Writes `APCONT` update (`EXCPT` with indicator `79`) and clears `79`.

9. **Initialize Fields**:
   - Clears or initializes fields for the next record:
     - `CNVO`, `PPCK`, `PCKD`, `IAMT`, `RTGL`, `RTPC`, `FRTL`, `SORN`, `SSRN`, `SVDSPC`, `POSQ`, `PRAM`, `FRAM`, `SVLNGL`, `GALN`, `RCPT`, `JQTY`, `AMT`, `DISC` (numeric fields to 0).
     - `SNGL`, `HOLD`, `HLDD`, `RTGLNM`, `DDES`, `EXGLNM`, `JOB#`, `CTYP`, `ITEM`, `CAID` (character fields to blanks).
     - `CLCD` to 'C' (open/closed status).
     - `DSPC` to 0 if `SVDSPC` = 0.
     - Sets `EXGL` (expense GL) to 12010008 (if `CANTAX` ≠ 'Y') or 12010009 (if `CANTAX` = 'Y').

10. **Output Records**:
    - Writes to `APTRAN`:
      - **Header Record** (`EXCPTHEADER`):
        - Record type 'A', company '10', `ENT#`, `Z3`, `VNVEND`, `CNVO`, `APGL`, `ANCHNM`, `INV#`, `DUDT` (twice), `SNGL`, hold status ('A' or 'H' based on `VNHOLD`), hold description, `PPCK`, `VNAM`, `VAD1–VAD4`, `BKGL`, `CHKAMT`, `RTPC`, `ATRTGL`, `DUDT8` (twice), `FRTL`, `SORN`, `SSRN`, `CAID`, `VNTERM`, `PTYP`, `INV#`.
      - **Detail Record** (`EXCPTDETAIL`):
        - `DDEL`, company '10', `ENT#`, `NXLINE`, `VNVEND`, expense company '10', `EXGL` (12010008 or 12010009), `INV#`, `CHKAMT`, `DISC`, `DSPC`, `ITEM`, `LNQTY`, `JOB#`, `CTYP`, `JQTY`, `PONO`, `GALN`, `RCPT`, `CLCD` ('C'), `POSQ`, `CHKAMT`, `FRAM`.
    - Updates `APCONT`:
      - Updates `ACNXTE` (next entry number) if indicator `79` is on.
      - Releases `APCONT` via `RELAPC` exception.

11. **Loop Control**:
    - Jumps to `SKIP1` tag on validation failures (e.g., `APSGACH` or `APVEND` not found).
    - Continues processing `APSOGAS` records until the end of the file.

---

### **Business Rules**
The program enforces the following business rules:

1. **Dynamic File Access**:
   - Uses files dynamically labeled with a prefix (e.g., `?9?` from `AP105.ocl36`, derived from `&P$FGRP` in `AP105PCL.clp`), supporting multiple file groups or environments.

2. **Vendor and Owner Validation**:
   - Validates that `ANOWNR` from `APSOGAS` exists in `APSGACH` (indicator `99`).
   - Validates that `AGVEND` from `APSGACH` exists in `APVEND` (indicator `53`).
   - Skips processing if either validation fails.

3. **Hold Status Handling**:
   - If `VNHOLD` = 'H' in `APVEND`, sets indicator `55` and marks the transaction header with 'H' and 'VENDOR ON HOLD' in the hold description; otherwise, uses 'A' and 'ON HOLD FOR ACH'.

4. **Due Date Adjustment**:
   - Replaces the due date (`DUDT8`) with a non-holiday/non-weekend date from `APDATE` (`ADNED8`) if found, ensuring payments are scheduled on valid business days.

5. **Invoice Description**:
   - Copies the invoice description from `DESC` (LDA) or `ANCHNM` (check name) to `INV#` for new records.

6. **Entry Number Management**:
   - Manages entry numbers (`ENT#`, `ACNXTE`) to ensure unique transaction identifiers, resetting to 1 if `ENT#` reaches 99999.
   - Updates `APCONT` with the next entry number (`ACNXTE`).

7. **Canadian Tax Handling**:
   - If `CANTAX` = 'Y', uses expense GL 12010009; otherwise, uses 12010008.

8. **Transaction Structure**:
   - Creates header and detail records in `APTRAN` for each valid `APSOGAS` record, populating fields like vendor details, GL accounts, amounts, and dates.
   - Ensures detail records are linked to headers via `ENT#` and `NXLINE`.

9. **Y2K Date Handling**:
   - Adjusts dates for century compliance using `Y2KCEN` and `Y2KCMP` to determine whether to use the current or next century.

---

### **Tables Used**
The program uses the following files (tables):

1. **APSOGAS** (Input Primary, 127 bytes):
   - Fields:
     - `ANOWNR` (1–7): Owner number.
     - `ANNAME` (8–42): Name.
     - `ANCHNM` (43–49): Check name.
     - `ANDATE` (50–59): Date (subfields: `CKDMM` 50–51, `CKDDD` 53–54, `CKDYY` 58–59).
     - `ANCHAM` (60–65, packed): Check amount.
     - `ANSTDT` (66–75): Start date.
     - `ANENTE` (76–85): Entry date.
     - `ANDUDT` (86–95): Due date (subfields: `DUDMM` 86–87, `DUDDD` 89–90, `DUDYY` 92–93).
   - Source of payment data for voucher creation.

2. **APSGACH** (Input, Keyed, 64 bytes, 7-byte key):
   - Fields:
     - `AGOWNR` (1–7): Owner number.
     - `AGVEND` (8–12): Vendor number.
   - Used to validate `ANOWNR` and retrieve `AGVEND`.

3. **APTRAN** (Update, Keyed, 404 bytes, 10-byte key):
   - Header Fields:
     - `ATHDEL` (1): Header delete flag.
     - `ATENSQ` (9–11): Entry sequence number.
     - `ATVEND` (12–16): Vendor number.
     - `ATAPGL` (24–31): A/P GL number.
     - `ATIDES` (42–66): Invoice description.
     - `ATINDT` (67–72): Invoice date.
     - `ATDUDT` (73–78): Due date.
     - `ATSNGL` (79): Single check flag.
     - `ATHOLD` (80): Hold flag.
     - `ATHLDD` (81–105): Hold description.
     - `ATPAID` (106): Prepaid flag.
     - `ATPPCK` (107–112): Prepaid check number.
     - `ATVNAM` (113–142): Vendor name.
     - `ATVAD1–ATVAD4` (143–262): Address lines.
     - `ATBKGL` (263–270): Bank GL number.
     - `ATIAMT` (271–281): Invoice amount.
     - `ATRTGL` (282–289): Retention GL.
     - `ATRTPC` (290–295): Retention percentage.
     - `ATPCKD` (296–301): Prepaid check date.
     - `ATFRTL` (326–332): Freight to allocate.
     - `ATSORN` (348–353): Sales order number.
     - `ATSSRN` (354–356): Sales SRN number.
     - `ATCAID` (357–362): Carrier ID.
     - `ATTERM` (363–364): Vendor payment terms.
     - `ATPTYP` (365–370): Process type.
     - `ATDSDT` (371–376): Discount due date.
     - `ATINV#` (385–404): Vendor invoice number.
   - Detail Fields:
     - `ATDDEL` (1): Detail delete flag.
     - `ATENSQ` (9–11): Entry sequence number.
     - `ATVEND` (12–16): Vendor number.
     - `ATEXCO` (18–19): Expense company.
     - `ATEXGL` (20–27): Expense GL number.
     - `ATDDES` (28–52): Detail description.
     - `ATAMT` (53–58, packed): Detail line amount.
     - `ATDISC` (59–64, packed): Discount.
     - `ATDSPC` (65–69): Discount percentage.
     - `ATITEM` (79–91): Inventory item number.
     - `ATQTY` (92–97, packed): Inventory quantity.
     - `ATJOB#` (100–105): Job number.
     - `ATCCOD` (108–113): Job cost code.
     - `ATCTYP` (114–115): Job cost type.
     - `ATJQTY` (116–119, packed): Job cost quantity.
     - `ATPONO` (120–125): Purchase order number.
     - `ATGALN` (126–129, packed): Gallons.
     - `ATRCPT` (130–136): Receipt number.
     - `ATCLCD` (137): Open/closed status.
     - `ATPOSQ` (138–140): PO line sequence.
     - `ATPRAM` (141–146, packed): Product amount.
     - `ATFRAM` (147–150, packed): Freight amount.
   - Stores transaction header and detail records.

4. **APCONT** (Update, Keyed, 256 bytes, 2-byte key):
   - Fields:
     - `ACDEL` (1): Delete flag.
     - `ACAPGL` (34–41): A/P GL number.
     - `ACCAGL` (42–49): Cash GL number.
     - `ACDSGL` (50–57): Discounts GL number.
     - `ACNXTE` (76–80): Next entry number.
     - `ACJCYN` (87): Job cost active flag.
     - `ACRTGL` (88–95): Retention GL number.
     - `ACPOYN` (96): PO active flag.
     - `ACEEGL` (97–104): Employee expense GL number.
   - Stores control data and manages entry numbers.

5. **APVEND** (Input, Keyed, 579 bytes, 7-byte key):
   - Fields:
     - `VNDEL` (1): Record code.
     - `VNCO` (2–3): Company number.
     - `VNVEND` (4–8): Vendor number.
     - `VNVNAM` (9–38): Vendor name.
     - `VNAD1–VNAD4` (39–158): Address lines.
     - `VNHOLD` (240): Hold invoices flag.
     - `VNSNGL` (241): Single check flag.
     - `VNEXGL` (254–261): Expense GL and sub-account.
     - `VNTERM` (262–263): AP terms code.
     - `VNCAID` (294–299): Carrier ID.
     - `VNPRID` (384–387, packed): ADP payroll ID.
     - `VNACLS` (388–390): ACH class.
     - `VNACOS` (391): ACH checking or savings.
     - `VNARTE` (392–400): ACH bank routing code.
     - `VNABK#` (401–417): ACH bank account number.
   - Used to validate and retrieve vendor details.

6. **APDATE** (Input, Keyed, 19 bytes, 10-byte key):
   - Fields:
     - `ADDEL` (1): Delete flag.
     - `ADCO` (2–3): Company.
     - `ADDUD8` (4–11): Calculated due date.
     - `ADNED8` (12–19): New due date (non-holiday/weekend).
   - Used to adjust due dates to valid business days.

---

### **External Programs Called**
- **None**: The program does not explicitly call any external programs via `CALL` or other mechanisms. It relies on subroutines (`HDRADD`) and file operations.

---

### **Summary**
- **Process Steps**: Declare files and data structures, initialize variables, retrieve control data, process dates and invoice numbers, validate vendors, adjust due dates, add header and detail records to `APTRAN`, manage entry numbers, and update `APCONT`.
- **Business Rules**: Validates owners and vendors, adjusts due dates for holidays/weekends, applies hold status for vendors, manages entry numbers, supports Canadian tax GL accounts, and ensures Y2K-compliant dates.
- **Tables Used**: `APSOGAS` (payment data), `APSGACH` (owner validation), `APTRAN` (transaction headers/details), `APCONT` (control data), `APVEND` (vendor data), `APDATE` (due date adjustments).
- **External Programs Called**: None.

This RPG program processes `APSOGAS` records to create A/P vouchers in `APTRAN`, validates data, adjusts dates, and handles vendor hold status, preparing transactions for ACH payment export.