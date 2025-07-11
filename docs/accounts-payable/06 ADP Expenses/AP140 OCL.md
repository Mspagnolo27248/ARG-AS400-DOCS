The provided document is an **OCL (Operation Control Language)** script used on IBM midrange systems (like AS/400 or IBM i) to control the execution of RPG programs and manage file operations for an **Accounts Payable (A/P) Employee Expenses Report and Spreadsheet** process. Below, I’ll explain the process steps, list the external programs called, and identify the tables (files) used.

---

### **Process Steps of the OCL Program**

The OCL script orchestrates a sequence of file operations, program executions, and sorts to generate an employee expenses report and spreadsheet. Here’s a step-by-step breakdown of the process:

1. **Initialization and File Setup**:
   - **GSY2K and SCPROCP**: These are likely system or environment setup commands or parameters, possibly related to the operating system or job control. `SCPROCP ,,,,,,,,?9?` suggests a parameter `?9?` (likely a job or library identifier) is being passed.
   - **SWITCH 00000000**: Initializes job control switches to off (all zeros), which are used later for conditional branching.
   - **GSDELETE**: Deletes temporary work files (`ADPY?WS?`, `ADPS?WS?`, `ADPC?WS?`, `ADPO?WS?`, `ADPT?WS?`) to ensure a clean slate. The `?WS?` and `?9?` are placeholders for dynamic values (e.g., work library or job-specific identifiers).
   - **IFF DATAF1-?9?ADPT?WS? BLDFILE**: Conditionally builds a temporary file `?9?ADPT?WS?` with 500 records, 128 bytes each, if it doesn’t exist. The `DFILE` parameter indicates it’s a disk file.
   - **CLRPFM FILE(?9?APEEPY)**: Clears the physical file `APEEPY`, which likely stores the final employee expense data.

2. **Load and Run AP140**:
   - **LOAD AP140**: Loads the RPG program `AP140`.
   - **File Definitions**:
     - `ADPYTR` (labeled `?9?ADPT?WS?`, shared access, extended by 100 records): Transaction file for A/P data.
     - `APCONT` (labeled `?9?APCONT`, shared): A/P control file.
     - `GLMAST` (labeled `?9?GLMAST`, shared): General Ledger master file.
     - `GSTABL` (labeled `?9?GSTABL`, shared): General system table file.
     - `GSCONT` (labeled `?9?GSCONT`, shared): General system control file.
   - **RUN**: Executes `AP140`, which likely processes A/P transactions, retrieves control data, and prepares initial data for the expense report.

3. **Conditional Branching (SWITCH1)**:
   - **IF SWITCH1-1 GOTO END**: Checks if switch 1 is set to 1. If true, the program jumps to the `END` tag, terminating the process. This suggests `AP140` may set this switch to indicate an error or completion condition.

4. **Tag AP141 and File Preparation**:
   - **TAG AP141**: Marks a program section for branching.
   - **GSDELETE**: Deletes temporary files again to ensure no residual data.
   - **BLDFILE**:
     - Builds `?9?ADPY?WS?` (999,000 records, 226 bytes) for A/P payment data.
     - Builds `?9?ADPC?WS?` (999,000 records, 96 bytes) for check-related data.

5. **First Sort (#GSORT for AP141)**:
   - **LOAD #GSORT**: Loads the system sort utility.
   - **File Definitions**:
     - Input: `?9?ADPT?WS?` (from `AP140` output).
     - Output: `?9?ADP151S` (999,000 records, retained as a job file).
   - **Sort Specifications**:
     - `HSORTR 17A 3X 128 N`: Sorts in reverse order, 17-character key, no sequence checking.
     - `I C 1 1NECD`: Includes records where position 1 is not equal to a specific condition (likely a deletion flag).
     - Sort keys:
       - `FNC 7 8 COMPANY`: Sorts by company code (positions 7–8).
       - `FNC 36 45 VENDOR/VOUCHER`: Sorts by vendor/voucher number (positions 36–45).
       - `FNC 2 6 SEQ#`: Sorts by sequence number (positions 2–6).
       - `FDC 1 128 RECORDS`: Includes entire record (positions 1–128).
   - **RUN**: Executes the sort, producing a sorted file `?9?ADP151S`.

6. **Load and Run AP141**:
   - **LOAD AP141**: Loads the RPG program `AP141`.
   - **File Definitions**:
     - `ADPYTR` (labeled `?9?ADP151S`): Sorted transaction file from the previous step.
     - `APOPEN` (labeled `?9?APOPEN`, shared): A/P open items file.
     - `ADPPAY` (labeled `?9?ADPY?WS?`, extended by 100 records): A/P payment file.
   - **RUN**: Executes `AP141`, which likely processes sorted transactions, matches them with open items, and prepares payment data.

7. **Second Sort (#GSORT for AP145)**:
   - **LOAD #GSORT**: Loads the sort utility again.
   - **File Definitions**:
     - Input: `?9?ADPY?WS?` (from `AP141` output).
     - Output: `?9?ADPS?WS?` (999,000 records).
   - **Sort Specifications**:
     - `HSORTA 23A 3X N`: Sorts in ascending order, 23-character key, no sequence checking.
     - `I C 1 1NECD`: Includes records based on position 1 condition.
     - Sort keys:
       - `FNC 2 3 COMPANY`: Sorts by company code (positions 2–3).
       - `FNC 153 160 BANK G/L #`: Sorts by bank general ledger number (positions 153–160).
       - `FNC 4 8 VENDOR`: Sorts by vendor code (positions 4–8).
       - `FNC 97 97 PREPAID CODE`: Sorts by prepaid code (position 97).
       - `FNC 91 96 CHECK #`: Sorts by check number (positions 91–96).
       - `FNC 152 152 SINGLE CHECK CODE`: Sorts by single check code (position 152).
   - **RUN**: Executes the sort, producing a sorted payment file `?9?ADPS?WS?`.

8. **Conditional Label for Employee Expense**:
   - **IF ?3?/EE LOCAL OFFSET-198,DATA-'EE** EMPLOYEE EXPENSE**'**: If parameter `?3?` equals `EE`, sets a data field at offset 198 to indicate an employee expense report.
   - **ELSE LOCAL OFFSET-198,DATA-' '**: Otherwise, clears the field.

9. **Load and Run AP145**:
   - **LOAD AP145**: Loads the RPG program `AP145`.
   - **File Definitions**:
     - `ADPPAY` (labeled `?9?ADPY?WS?`, shared): Payment file.
     - `AP145S` (labeled `?9?ADPS?WS?`): Sorted payment file.
     - `APCONT` (labeled `?9?APCONT`, shared): A/P control file.
     - `ADPYTR` (labeled `?9?ADPT?WS?`): Transaction file.
     - `APVEND` (labeled `?9?APVEND`, shared): Vendor master file.
     - `APOPEN` (labeled `?9?APOPEN`, shared): A/P open items file.
     - `APCHKR` (labeled `?9?APCHKR`, shared): Check register file.
     - `ADPYCK` (labeled `?9?ADPC?WS?`, extended by 100 records): Check file.
     - `APEEPY` (labeled `?9?APEEPY`, shared): Employee expense file.
   - **Printer Overrides**:
     - If parameter `?9?` equals `G`, overrides printer files `APEEEXP` and `APEEEXPO` to output queue `QUSRSYS/APPOST` (production) or `QUSRSYS/TESTOUTQ` (test).
   - **RUN**: Executes `AP145`, which generates the final employee expense report and spreadsheet, using sorted payment data, vendor information, and check details.

10. **Cleanup and Termination**:
    - **GSDELETE**: Deletes temporary files again.
    - **IFF ?9?/G GOTO END**: If parameter `?9?` equals `G`, skips to the `END` tag.
    - **TAG END**: Marks the end of the process.
    - **SWITCH 00000000**: Resets switches to off.

---

### **External Programs Called**

The OCL script explicitly loads and runs the following external programs:
1. **AP140**: Processes initial A/P transactions and prepares data.
2. **AP141**: Processes sorted transactions, matches with open items, and prepares payment data.
3. **AP145**: Generates the final employee expense report and spreadsheet.
4. **#GSORT** (system sort utility, called twice): Performs sorting of transaction and payment data.

---

### **Tables (Files) Used**

The OCL script references the following files (tables):
1. **ADPYTR (?9?ADPT?WS?)**: A/P transaction file (temporary, used across programs).
2. **APCONT (?9?APCONT)**: A/P control file (shared, contains control settings).
3. **GLMAST (?9?GLMAST)**: General Ledger master file (shared, used in `AP140`).
4. **GSTABL (?9?GSTABL)**: General system table file (shared, used in `AP140`).
5. **GSCONT (?9?GSCONT)**: General system control file (shared, used in `AP140`).
6. **ADP151S (?9?ADP151S)**: Sorted transaction file (temporary, output from first `#GSORT`).
7. **ADPPAY (?9?ADPY?WS?)**: A/P payment file (temporary, used in `AP141` and `AP145`).
8. **APOPEN (?9?APOPEN)**: A/P open items file (shared, used in `AP141` and `AP145`).
9. **AP145S (?9?ADPS?WS?)**: Sorted payment file (temporary, output from second `#GSORT`).
10. **APVEND (?9?APVEND)**: Vendor master file (shared, used in `AP145`).
11. **APCHKR (?9?APCHKR)**: Check register file (shared, used in `AP145`).
12. **ADPYCK (?9?ADPC?WS?)**: Check file (temporary, used in `AP145`).
13. **APEEPY (?9?APEEPY)**: Employee expense file (shared, cleared initially and used in `AP145`).

---

### **Summary**

The OCL script automates the generation of an A/P employee expenses report and spreadsheet by:
1. Initializing and clearing temporary files.
2. Running `AP140` to process A/P transactions.
3. Sorting data with `#GSORT` for `AP141`, which prepares payment data.
4. Sorting payment data again with `#GSORT` for `AP145`.
5. Running `AP145` to produce the final report and spreadsheet, with conditional printer output.
6. Cleaning up temporary files and terminating.

The process uses three RPG programs (`AP140`, `AP141`, `AP145`) and the system sort utility (`#GSORT`), along with 13 files for data storage and processing. Parameters like `?9?`, `?3?`, and switches control the flow and environment (e.g., production vs. test output).