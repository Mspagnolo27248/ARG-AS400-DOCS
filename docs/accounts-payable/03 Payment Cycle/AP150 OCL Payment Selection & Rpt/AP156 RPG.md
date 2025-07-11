Below, I provide a detailed analysis of the **AP156.ocl36.txt** OCL script and the **AP156.rpg36.txt** RPG program, which are part of the Accounts Payable (A/P) payment processing system. The OCL script invokes the `AP156` RPG program to create a National Automated Clearing House Association (NACHA) file for ACH payments to PNC Bank. I will outline the **process steps**, **business rules**, **tables used**, and **external programs called** for both the OCL and RPG components.

---

### **AP156.ocl36.txt (OCL Script)**

#### **Process Steps**
The `AP156.ocl36.txt` OCL script is responsible for initiating the creation of a NACHA file for ACH payments. The steps are as follows:

1. **Conditional Execution Check**:
   - Checks if the payment run is for ACH payments by examining position 400 of the Local Data Area (LDA). The value must be `'A'`, set by the `AP155` program if ACH payments (`OPPAID = 'A'`) are detected in `APPAY`.
   - If the LDA position 400 does not contain `'A'`, the script jumps to the `END` tag, skipping execution of the `AP156` program.

2. **Clear ACH File**:
   - Executes the `CLRPFM` command to clear the physical file `ACHFIL` (labeled `?9?ACHFIL`), ensuring no residual data from previous runs.

3. **Load and Run AP156 Program**:
   - Loads the `AP156` RPG program using the `LOAD AP156` command.
   - Specifies input and output files:
     - `APPYCK`: Check file, labeled `?9?APPC?WS?`, shared access (`DISP-SHR`).
     - `APCONT`: A/P control file, labeled `?9?APCONT`, shared access.
     - `APVEND`: Vendor file, labeled `?9?APVEND`, shared access.
     - `ACHFILE`: Output NACHA file, labeled `?9?ACHFIL`, shared access.
   - Executes the program using the `RUN` command.

4. **Termination**:
   - If the ACH condition is not met, the script terminates at the `END` tag without running `AP156`.

#### **Business Rules**
1. **ACH Payment Requirement**:
   - The script only proceeds if the payment run includes ACH payments (`LDA position 400 = 'A'`).
   - If no ACH payments are present, the script skips execution to avoid unnecessary processing.

2. **File Preparation**:
   - The `ACHFIL` file must be cleared before processing to ensure a clean slate for the NACHA file output.

3. **Shared File Access**:
   - All files (`APPYCK`, `APCONT`, `APVEND`, `ACHFILE`) are opened with shared access (`DISP-SHR`) to allow concurrent access by other programs or processes.

#### **Tables (Files) Used**
1. **APPYCK** (`?9?APPC?WS?`):
   - Check file containing payment records.
2. **APCONT** (`?9?APCONT`):
   - A/P control file with company details.
3. **APVEND** (`?9?APVEND`):
   - Vendor file with vendor details, including ACH information.
4. **ACHFILE** (`?9?ACHFIL`):
   - Output file for the NACHA-formatted ACH payment data.

#### **External Programs Called**
- **AP156**: The RPG program loaded and executed to create the NACHA file.

---

### **AP156.rpg36.txt (RPG Program)**

#### **Process Steps**
The `AP156` RPG program generates a NACHA-formatted file (`ACHFILE`) for ACH payments to PNC Bank, processing records from `APPYCK` and retrieving additional data from `APCONT` and `APVEND`. The program produces a structured file with specific record types (1, 5, 6, 8, 9) as required by NACHA standards. Here are the steps:

1. **Initialization (`ONETIM` Subroutine)**:
   - Executes once (`ONCE = 1`) to set up the environment:
     - Retrieves system date and time (`SYTMDT`) and formats the date for NACHA records.
     - Initializes counters: `BATCH#` (batch number), `TRACE#` (trace number), `LRCNT` (entry count), `LRHASH` (hash total), `LRDR` (debit total), `LRCR` (credit total), `RECCNT` (record count).
     - Writes the **File Header Record (Type 1)** to `ACHFILE` with fields like priority code, ABA numbers, transmission date/time, and company names.

2. **Process APPYCK Records**:
   - Reads `APPYCK` records (check file) sorted by company (`PYCONO`) and vendor (`PYVEND`).
   - For each record:
     - Validates that the record is not a detail record (`NS 01`) and has a valid status (`PYSTAT = 'A'` for ACH payments).
     - Chains to `APCONT` to retrieve company details (e.g., `ACNAME`, `ACBKGL`) using `PYCONO`.
     - Chains to `APVEND` to retrieve vendor ACH details (e.g., `VNARTE`, `VNABK#`, `VNACOS`, `PYNAME`) using `PYVEND`.

3. **Write Batch Header (`L2DET` Subroutine)**:
   - On the first record for a new company (`L1, N84`), writes a **Batch Header Record (Type 5)** to `ACHFILE`.
   - Includes fields like service class code (`200` for credits), company name, tax ID, and effective entry date (`CKYMD` from `PYCKDT`).
   - Initializes batch counters (`L2CNT`, `L2HASH`, `L2DR`, `L2CR`).

4. **Write Entry Detail (`EACH` Subroutine)**:
   - For each `APPYCK` record:
     - Determines the transaction code (`TRNCDE`): `'22'` for checking accounts (`VNACOS = 'C'`) or `'32'` for savings accounts.
     - Sets the payment amount (`AMOUNT = PYCKAM`).
     - Updates counters: increments `TRACE#`, `L2CNT`, `LRCNT`, `RECCNT`, and adds `VNARTE` to `L2HASH` and `LRHASH`, and `AMOUNT` to `L2CR` and `LRCR`.
     - Writes an **Entry Detail Record (Type 6)** to `ACHFILE` with vendor bank routing code (`VNARTE`), account number (`VNABK#`), amount, vendor ID, and name.

5. **Write Batch Control (`L2TOT` Subroutine)**:
   - At the end of each company (`L2, 84`), writes a **Batch Control Record (Type 8)** to `ACHFILE`.
   - Includes batch entry count (`L2CNT`), hash total (`L2HASH`), credit total (`L2CR`), and batch number (`BATCH#`).

6. **Write File Control and Filler (`LRTOT` Subroutine)**:
   - At the end of processing (`LR, 10`), writes a **File Control Record (Type 9)** to `ACHFILE` with batch count (`LRBCNT`), block count (`LRBLOK`), entry count (`LRCNT`), hash total (`LRHASH`), and credit total (`LRCR`).
   - Calculates the number of blocks (`LRBLOK = RECCNT / 10`, rounded up) and fills remaining block space with filler records containing `'999999999999999999999999'`.

7. **Report Output**:
   - Outputs a report to `REPORT` (printer file) for logging or verification, though specific details are not defined in the code.

8. **Termination**:
   - Completes after processing all `APPYCK` records and writing the necessary NACHA records.

#### **Business Rules**
1. **ACH Payment Validation**:
   - Only processes `APPYCK` records with `PYSTAT = 'A'` (ACH payments), as confirmed by the OCL script’s LDA check.

2. **NACHA Record Structure**:
   - Adheres to NACHA file format standards:
     - **Type 1 (File Header)**: Includes fixed ABA numbers (`043000096`, `1222318612`), transmission date/time, and company names.
     - **Type 5 (Batch Header)**: Uses service class code `200` (credits only), company tax ID (`1222318612`), and effective entry date.
     - **Type 6 (Entry Detail)**: Uses transaction codes (`22` for checking, `32` for savings), vendor bank details, and payment amount.
     - **Type 8 (Batch Control)**: Summarizes batch entries and totals.
     - **Type 9 (File Control)**: Summarizes file-level counts and totals.
   - Filler records pad blocks to multiples of 10.

3. **Vendor ACH Details**:
   - Requires valid ACH data in `APVEND` (`VNARTE`, `VNABK#`, `VNACOS`).
   - Transaction code is set based on account type (`VNACOS = 'C'` for checking, otherwise savings).

4. **Counter and Hash Management**:
   - Maintains accurate counts for batches (`L2CNT`), entries (`LRCNT`), blocks (`LRBLOK`), and hash totals (`L2HASH`, `LRHASH`) for NACHA compliance.
   - Credit totals (`L2CR`, `LRCR`) reflect payment amounts.

5. **Company and Vendor Validation**:
   - Chains to `APCONT` to ensure valid company number (`PYCONO`).
   - Chains to `APVEND` to ensure valid vendor number (`PYVEND`) and retrieve ACH-specific fields.

#### **Tables (Files) Used**
1. **APPYCK** (`IP`, Input Primary, 96 bytes):
   - Check file containing ACH payment records.
   - Fields: `PYSTAT` (status), `PYCONO` (company), `PYBKGL` (bank G/L), `PYCHK#` (check number), `PYVEND` (vendor), `PYCKAM` (check amount), `PYCKDT` (check date), `PYNAME` (vendor name), `PYSEQ#` (sequence), `PYCNTR` (count).

2. **APCONT** (`IC`, Input with Chain, 256 bytes):
   - A/P control file for company details.
   - Fields: `ACNAME` (company name), `ACCONO` (company number), `ACBKGL` (bank G/L), `ACPRE#` (pre-numbered checks flag).

3. **APVEND** (`IC`, Input with Chain, 579 bytes):
   - Vendor file for ACH payment details.
   - Fields: `VNNAME` (vendor name), `VNARTE` (ACH bank routing code), `VNABK#` (ACH bank account number), `VNACOS` (checking/savings), `VNVEND` (vendor number).

4. **ACHFILE** (`O`, Output, 94 bytes):
   - Output file for NACHA-formatted ACH payment data.
   - Record types: 1 (file header), 5 (batch header), 6 (entry detail), 8 (batch control), 9 (file control), filler.

5. **REPORT** (`O`, Output, 132 bytes):
   - Printer file for logging or reporting (details not specified in code).

#### **External Programs Called**
- The `AP156` RPG program does not call any external programs. It is a standalone program invoked by the `AP156.ocl36.txt` script, processing input files and producing output files internally.

---

### **Summary**
The `AP156.ocl36.txt` OCL script conditionally invokes the `AP156` RPG program to create a NACHA file for ACH payments if `LDA position 400 = 'A'`. It clears the `ACHFIL` file and loads `AP156` with access to `APPYCK`, `APCONT`, `APVEND`, and `ACHFILE`. The `AP156` RPG program processes `APPYCK` records to generate a NACHA-compliant file with file header, batch header, entry detail, batch control, and file control records, adhering to strict formatting rules. It uses vendor ACH details from `APVEND` and company data from `APCONT`, maintaining counters and hash totals for compliance. The program interacts with five files and does not call external programs, completing the ACH payment file creation process for PNC Bank.