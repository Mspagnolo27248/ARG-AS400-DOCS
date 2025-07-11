The RPG program `AP250.RPG36` is a core component of the Accounts Payable (A/P) Check Register process, invoked by the OCL procedure `AP250.OCL36`. It handles the generation of the check register, updates master files, and processes payment-related data, including support for ACH, wire transfers, and employee expenses. Below is a detailed breakdown of the **process steps**, **business rules**, **tables/files used**, and **external programs called** as requested.

---

### Process Steps

The `AP250.RPG36` program is structured around a series of subroutines that process input records, update files, and generate output for the check register and related journals. The steps are derived from the RPG source code and are organized by the main processing flow and subroutine execution:

1. **Initialization (L4DET Subroutine)**:
   - Retrieves the current system time and date, formatting them for output (e.g., `TIMDAT`, `DATYMD`, `DATE8`).
   - Initializes counters and accumulators (e.g., `PAGE`, `ZERO6`, `ZERO9`).
   - Sets a separator field (`SEP`) to `'* '`.
   - Retrieves the check date from `APPYTR` (chain to `PTCKDT`, convert to `CDYMD8` with century adjustment using `Y2KCEN` and `Y2KCMP`).
   - Chains to `APCONT` to retrieve company details (e.g., `ACNAME`, `ACDSGL`, `ACCDJR`, `ACCKNO`).
   - Determines the journal ID (`JRNID`) based on whether a wire transfer is indicated (`WIRE = 'WT'` sets `JRNID` to `'WD'`, otherwise `'CD'`; additional logic for ACH, wire transfer, or employee expenses sets `JRNID` to `'AD'`, `'WD'`, or `'ED'` respectively).
   - Sets indicator `50` for wire transfers, ACH, or employee expenses to skip writing to `APCHKR`.

2. **Process Each Check Record (EACH01 Subroutine)**:
   - Processes records from `APPYCK` (check file).
   - Converts the check date (`AXCKDT`) to a formatted date (`AXYMD8`).
   - Copies the check number (`AXCHEK`) to the PNC check number field (`PNCCHK`) for positive pay format.
   - Evaluates the record code (`AXRECD`):
     - `'C'`: Credit/no pay, skip processing (indicator `19`).
     - `'F'`: Full stub, void check, continue to next stub with same check number (indicator `12`).
     - `'V'`: Full stub, void check, use next check number (indicator `13`).
     - `'P'`, `'A'`, `'W'`, `'E'`: Prepaid check, ACH, wire transfer, or employee expense, respectively (indicators `25`, `26`, `27`, `28` set indicator `11`).
   - Updates the A/P check reconciliation file (`APCHKR`):
     - Chains to `APCHKR` using a key constructed from `CONO`, `BKGL`, and `AXCHEK`.
     - If not found (`90` on), initializes fields (`AMCODE`, `AMCKAM`, `AMCLDT`, `AMOCAM`).
     - For voided checks (`13` on), sets `AMCKAM` to 0 and `AMCODE` to `'V'`, updates clear date (`AMCLDT`, `CLDT8`).
     - For non-voided checks, sets `AMCODE` to `'O'` and updates `AMCKAM` with `AXAMT`.
     - Formats the date for PNC positive pay (`PNCDT8`) and sets `PNCCOD` to `'V'` for voided checks or `'I'` otherwise.
   - Updates counters and accumulators:
     - Increments `C3CNT` and adds `AXAMT` to `C3AMT` for computer checks (non-prepaid, non-void).
     - Increments `P3CNT`, `A3CNT`, `W3CNT`, or `E3CNT` and adds `AXAMT` to respective accumulators (`P3AMT`, `A3AMT`, `W3AMT`, `E3AMT`) for prepaid, ACH, wire transfer, or employee expense records.
     - Increments `L3CNT` and adds `AXAMT` to `L3AMT` and `L2PAID` for non-voided checks.
   - Stores the check number (`AXCHEK`) in `L4CHEK` for updating `APCONT`.

3. **Process Payment Records (EACH02 Subroutine)**:
   - Processes records from `APPAY` (payment file).
   - Accumulates discount taken (`OPDISC`) into `OPDSTK` and `CDDISC`.
   - Updates vendor totals for `APVEND`:
     - Adds `OPGRAM` to `L2GRAM` (gross amount).
     - Adds `OPDISC` to `L2DISC` (discount).
     - Adds `OPPPTD` to `L2PPTD` (partial paid to date).
     - Adds `OPLPAM` to `L2AMT` (payment amount).
   - Calculates the A/P reduction (`OPPAY = OPDISC + OPLPAM`) and open amount (`OPOPEN = OPGRAM - OPPPTD - OPPAY`).
   - If `OPOPEN = 0`, marks the record as fully paid (indicator `20`).
   - Updates `OPPPTD` by adding `OPLPAM`.
   - Processes `APOPEN` records:
     - Sets the lower limit (`SETLL`) on `APOPEN` using `OPKEY`.
     - Reads `APOPEN` records, comparing `OPKY1` with `COVNVO` to ensure matching vouchers.
     - Chains to `APOPENH`, `APOPEND`, or `APOPENV` based on record type (indicators `04`, `05`, `06`).
   - Updates freight-related files (`FRCFBH`, `FRCINH`):
     - Constructs a key (`FRCKEY`, `FRCK39`) using `CONO`, `OPCAID`, `OPINVN`, and `OPSORN`.
     - Chains to `FRCFBH` (freight bill header) first; if found and `FRAPST = 'Y'`, writes an exception record (`APFBST`).
     - If not found, chains to `FRCINH` (invoice header) and writes an exception record (`APINST`).
   - Updates the discount missed table (`APPYDS`):
     - Chains to `APPYDS` using `OPKY1`.
     - Writes an exception record to `APPYDS` (indicator `70`).

4. **Vendor Totals Update (L2TOT Subroutine)**:
   - Chains to `APVEND` using a key (`VNKEY`) constructed from `CONO` and `VEND`.
   - If found (`92` off):
     - Updates `VNLPAY` with `L2AMT` (last payment amount).
     - Updates `VNLPDT` and `VNLPD8` with `AXCKDT` (last payment date).
     - Adds `L2DISC` to `VNDMTD` and `VNDYTD` (month-to-date and year-to-date discounts).
     - Adds `L2AMT` and `L2DISC` to `VNPAY` (month-to-date payments).
     - Subtracts `L2AMT` and `L2DISC` from `VNCBAL` (current balance).
     - Adds `L2PAID` to `VNTYDP` (year-to-date paid).

5. **Final Totals and Updates (L4TOT Subroutine)**:
   - Checks if `L4CHEK` is non-zero and compares it with `ACCKNO` (next check number).
   - If equal, increments `ACCKNO` and `ACCDJR` (next cash disbursements journal number).
   - Updates `APCONT` with the new `ACCKNO` and `ACCDJR`.

6. **Output Generation**:
   - Writes to `APCHKR` (check reconciliation) for non-prepaid, non-voided records (`01`, `N90`, `N50`, `N12`, `N19`):
     - Updates or adds records with fields like `AMCODE`, `AMCKAM`, `AMCLDT`, `CLDT8`, `AMOCAM`.
   - Writes to `APOPENH`, `APOPEND`, `APOPENV` (header, detail, one-time vendor) for fully paid records (`70`, `04`/`05`/`06`, `20`):
     - Marks records as deleted (`'D'`) and updates fields like `OPPPTD`, `OPCKNO`, `AXYMD`, `CDYMD`, `OPLPAM`, `OPDSTK`.
   - Writes to `APHISTH`, `APHISTD`, `APHISTV` (history files) for fully paid records:
     - Adds records with payment details, including `OPDISC`, `OPLPAM`, `AXYMD`, `CKDT8`, `OPCAID`, `OPINVN`.
   - Writes to `APPYDS` (discount missed table) for header records (`70`, `04`, `N97`):
     - Includes fields like `DSDEL`, `DSREC1`, `DSREC2`, `OPCKNO`, `OPPAID`, `OPLPAM`, `OPDISC`, `DSINVN`, `VNNAME`.
   - Writes to `FRCINH` and `FRCFBH` (freight-related files) for applicable vouchers:
     - Updates with payment status (`'P'`), vendor, check number, and amount.
   - Writes to `APCDJR` (cash disbursements journal) for payment records (`02`, `N19`):
     - Outputs cash (`'C'`), discount (`'D'`), and A/P (`'AP'`) entries with fields like `CONO`, `JRNID`, `BKGL`, `AXCHEK`, `OPLPAM`, `CDDISC`, `OPPAY`.
   - Writes to `APPNCF` (PNC positive pay file) for non-prepaid, non-voided records (`01`, `N19`, `N12`, `N28`):
     - Outputs bank account, check number, date, amount, vendor name, and status code.
   - Writes to `APPRINT` (printer file) for the check register report:
     - Outputs headers with company name, bank G/L, wire transfer indicator, date, time, and journal ID.
     - Outputs detail lines with check number, vendor number, name, date, and amount.
     - Outputs totals for computer checks, prepaid checks, ACH payments, wire transfers, employee expenses, and overall totals.

---

### Business Rules

The program enforces several business rules, primarily related to payment processing, file updates, and reporting:

1. **Check Record Processing**:
   - Skips processing for credit/no-pay records (`AXRECD = 'C'`).
   - Handles voided checks differently based on `AXRECD` (`'F'` for same check number, `'V'` for next check number).
   - Supports multiple payment types: prepaid checks (`'P'`), ACH (`'A'`), wire transfers (`'W'`), and employee expenses (`'E'`).

2. **Date Handling**:
   - Adjusts check dates for century (`Y2KCEN`, `Y2KCMP`) to handle Y2K compliance (e.g., `PTCKYY >= Y2KCMP` uses `Y2KCEN`, otherwise adds 1 to century).
   - Formats dates for PNC positive pay (`PNCDT8`) in MMDDYYYY format.

3. **Journal and Check Number Management**:
   - Assigns journal ID (`JRNID`) as `'CD'` (check), `'WD'` (wire transfer), `'AD'` (ACH), or `'ED'` (employee expenses).
   - Increments `ACCKNO` (next check number) and `ACCDJR` (next journal number) only if a valid check number is processed.

4. **Vendor and Payment Updates**:
   - Accumulates gross amount, discount, and payment amounts for each vendor (`L2GRAM`, `L2DISC`, `L2AMT`, `L2PAID`).
   - Updates vendor balances (`VNCBAL`), payments (`VNPAY`, `VNTYDP`), and discounts (`VNDMTD`, `VNDYTD`) in `APVEND`.
   - Marks fully paid vouchers as deleted (`'D'`) in `APOPENH`, `APOPEND`, `APOPENV`.

5. **Freight Invoice Processing**:
   - Checks `FRCFBH` (freight bill header) before `FRCINH` (invoice header) for vouchers with a carrier ID (`OPCAID`).
   - Updates payment status (`FRAPST = 'P'`) in the appropriate freight file.

6. **Discount Missed Tracking**:
   - Writes to `APPYDS` for header records to track missed discounts, including invoice and vendor details.

7. **Check Reconciliation**:
   - Updates `APCHKR` only for non-wire transfer, non-ACH, non-employee expense records (`N50`).
   - Sets `AMCODE` to `'O'` (open) or `'V'` (voided) and updates amounts and dates accordingly.

8. **Reporting**:
   - Generates a detailed check register (`APPRINT`) with headers, detail lines, and totals by payment type.
   - Produces a PNC positive pay file (`APPNCF`) for bank reconciliation, including void status.

---

### Tables/Files Used

The program interacts with multiple files for input, update, and output. The files are listed below with their logical names, labels, usage, and disposition:

| **Logical Name** | **Label**       | **Usage**                                                                 | **Disposition** | **Record Length** |
|------------------|-----------------|---------------------------------------------------------------------------|-----------------|-------------------|
| APPYCK           | ?9?APPC?WS?     | Input: Check data (check number, amount, date, vendor).                   | Input (IP)      | 96                |
| APPAY            | ?9?APPY?WS?     | Input/Update: Payment data (voucher, gross amount, discount, check number).| Input/Update (IS)| 384               |
| AP250S           | ?9?APPS?WS?     | Input: Supporting data (used for array `SEP`).                            | Input (IR)      | 3                 |
| APPYTR           | ?9?APPT?WS?     | Input: Transaction data (check date, year/period).                        | Input (IC)      | 128               |
| APCONT           | ?9?APCONT       | Update: A/P control (company name, next check/journal numbers).           | Update (UC)     | 256               |
| APVEND           | ?9?APVEND       | Update: Vendor master (name, balance, payments, discounts).               | Update (UC)     | 579               |
| APVEND2          | ?9?APVEND       | Input: Vendor master (address, name overflow).                            | Input (IC)      | 579               |
| APCHKR           | ?9?APCHKR       | Update: Check reconciliation (check amount, clear date, status).          | Update (UC)     | 128               |
| APOPEN           | ?9?APOPEN       | Input: Open A/P file (voucher data).                                      | Input (ID)      | 384               |
| APOPENH          | ?9?APOPNH       | Update: Open A/P header (voucher header data).                            | Update (UC)     | 384               |
| APOPEND          | ?9?APOPND       | Update: Open A/P detail (voucher detail data).                            | Update (UC)     | 384               |
| APOPENV          | ?9?APOPNV       | Update: Open A/P one-time vendor data.                                    | Update (UC)     | 384               |
| FRCINH           | ?9?FRCINH       | Update: Freight invoice header (carrier, payment status).                 | Update (UF)     | 206               |
| FRCFBH           | ?9?FRCFBH       | Update: Freight bill header (similar to FRCINH).                          | Update (UF)     | 206               |
| APPYDS           | ?9?APDS?WS?     | Input: Discount missed table (voucher, discount data).                    | Input (IF)      | 384               |
| APHISTH          | ?9?APHSTH       | Output: A/P history header (payment history).                             | Output (O)      | 384               |
| APHISTD          | ?9?APHSTD       | Output: A/P history detail (payment details).                             | Output (O)      | 384               |
| APHISTV          | ?9?APHSTV       | Output: A/P history one-time vendor (payment history).                    | Output (O)      | 384               |
| APCDJR           | ?9?APCD?WS?     | Output: Cash disbursements journal (cash, discount, A/P entries).         | Output (O)      | 128               |
| APPRINT          | ?9?APPRINT      | Output: Check register report (printer file).                             | Output (O)      | 132               |
| APPNCF           | ?9?APPNCF       | Output: PNC positive pay file (bank reconciliation).                      | Output (O)      | 155               |
| APDSMS           | ?9?APDSMS       | Output: Discount missed table (missed discount records).                  | Output (O)      | 384               |

---

### External Programs Called

The `AP250.RPG36` program does not explicitly call any external programs. All processing is handled within the program through its subroutines (`L4DET`, `EACH01`, `EACH02`, `L2TOT`, `L4TOT`). The program is self-contained and relies on file operations and internal logic to complete its tasks.

---

### Notes
- **File Dispositions**:
  - `IP` (Primary Input): `APPYCK` for check records.
  - `IS` (Secondary Input/Update): `APPAY` for payment processing.
  - `IR`/`IC` (Input): Temporary or control files (`AP250S`, `APPYTR`, `APVEND2`).
  - `UC`/`UF` (Update): Master files (`APCONT`, `APVEND`, `APCHKR`, `APOPEN*`, `FRC*`).
  - `ID` (Input for Deletion): `APOPEN` for reading vouchers.
  - `O` (Output): History, journal, and report files (`APHIST*`, `APCDJR`, `APPRINT`, `APPNCF`, `APDSMS`).
- **Modifications**:
  - `JB01` (2012): Added support for ACH (`A`), wire transfers (`W`), and employee expenses (`E`) in `PTHOLD` and `AXRECD`.
  - `JB02` (2018): Added `FRCFBH` processing before `FRCINH` and increased `APPAY` record length to 384 to match `APOPEN`.
  - `MG03` (2022): Added `APPYDS` and `APDSMS` for tracking missed discounts.
- **Y2K Compliance**: The program handles century adjustments for dates to ensure correct processing of check and payment dates.
- **Output Formats**:
  - The check register (`APPRINT`) includes detailed headers and totals by payment type.
  - The PNC positive pay file (`APPNCF`) is formatted for bank reconciliation, including void status.

This summary provides a comprehensive overview of the `AP250.RPG36` program's functionality, business logic, file interactions, and lack of external program calls. Let me know if you need further clarification or additional analysis!