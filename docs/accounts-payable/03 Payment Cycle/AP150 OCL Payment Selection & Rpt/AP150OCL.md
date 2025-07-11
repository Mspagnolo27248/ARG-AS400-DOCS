The provided document is an **Operation Control Language (OCL)** program, specifically `AP150.ocl36.txt`, used in IBM midrange systems (e.g., AS/400, now IBM i) for managing Accounts Payable (A/P) payment transactions. OCL is a scripting language used to control job execution and manage files in such environments. Below, I’ll explain the process steps of the program, identify the external programs called, and list the tables (files) used.

---

### **Process Steps of the AP150 OCL Program**

The OCL program `AP150` handles the entry, editing, and processing of A/P payment transactions, including creating payment records and generating a cash requirements report. It involves file operations, sorting, and invoking external programs. Here’s a step-by-step breakdown of the process:

1. **Initial Conditional Check for Automation**:
   - The program checks if it’s running in "AUTO" mode (triggered by another process, `AP200`).
     - `IF ?2?/AUTO GOTO AP151`: If `?2?` (a parameter) equals "AUTO," the program jumps to the `AP151` tag, skipping initial file setup.
   - If not in AUTO mode, it proceeds with file creation and setup.

2. **File Creation for Work Files**:
   - If the file `?9?APPT?WS?` (a work file for payment transactions) doesn’t exist, it creates it:
     - `BLDFILE ?9?APPT?WS?,I,RECORDS,500,128,,,2,5,DFILE`
     - Creates a file with 500 records, 128 bytes each, with specific attributes.
   - Sets a local variable at offset 135 with data `'F'` for `?9?APPT?WS?`.
   - If the file `?9?APPO?WS?` exists, sets a local variable at offset 300 to `'Y'`.

3. **Load Initial Program (`AP150`)**:
   - Loads the program `AP150` (likely an RPG or CL program).
   - Opens the following files:
     - `APPYTR` (labeled `?9?APPT?WS?`, shared, extendable by 100 records): Transaction work file.
     - `APCONT` (labeled `?9?APCONT`, shared): A/P control file.
     - `GLMAST` (labeled `?9?GLMAST`, shared): General ledger master file.
     - `APVEND` (labeled `?9?APVEND`, shared): Vendor master file.
     - `APOPEN` (labeled `?9?APOPEN`, shared): Open A/P file.
     - `GSTABL` (labeled `?9?GSTABL`, shared): General system table.
     - `GSCONT` (labeled `?9?GSCONT`, shared): General system control file.
   - Runs the `AP150` program to process payment transactions.

4. **AP151 Tag - Conditional File Deletion and Creation**:
   - Checks if the local variable at offset 300 is `'Y'`:
     - If not `'Y'`, jumps to the `END` tag, terminating the program.
     - If `'Y'`, deletes work files: `APPO?WS?`, `APPY?WS?`, `APPS?WS?`, `APPC?WS?`, `APDT?WS?`, `APDT?WS?C`, `APDS?WS?`.
   - Creates new work files:
     - `?9?APPY?WS?`: 999,000 records, 384 bytes.
     - `?9?APPC?WS?`: 999,000 records, 96 bytes.
     - `?9?APDT?WS?`: 500 records, 256 bytes.
     - `?9?APDS?WS?`: 999,000 records, 384 bytes.

5. **Sort Payment Records**:
   - Loads the `#GSORT` program (a system sort utility).
   - Sorts the input file `?9?APPT?WS?` into output file `?9?AP151S` (999,000 records, retained job file).
   - Sort specifications:
     - Sort by company (bytes 7-8), vendor/voucher (bytes 36-45), sequence number (bytes 2-6), and full record (bytes 1-128).
   - Runs the sort to organize payment transaction records.

6. **Load AP151 Program**:
   - Loads the `AP151` program (likely an RPG program for further transaction processing).
   - Opens files:
     - `APPYTR` (labeled `?9?AP151S`): Sorted transaction file.
     - `APOPEN` (labeled `?9?APOPEN`, shared): Open A/P file.
     - `APPAY` (labeled `?9?APPY?WS?`, extendable): Payment work file.
     - `APPYDS` (labeled `?9?APDS?WS?`, extendable): Payment discount work file.
   - Runs `AP151` to process sorted transactions and update payment records, including handling discounts for late payments (noted in the comment: "ADD CODE TO SAVE INVOICE INFO WHEN A DISCOUNT IS AVAILABLE BUT PAID TOO LATE").

7. **Generate Cash Requirements Report**:
   - Loads `#GSORT` again to sort payment records for reporting.
   - Sorts the input file `?9?APPY?WS?` into output file `?9?APPS?WS?` (999,000 records).
   - Sort specifications:
     - Sort by company (bytes 2-3), bank G/L number (bytes 153-160), vendor (bytes 4-8), prepaid code (byte 97), check number (bytes 91-96), and single check code (byte 152).
   - Runs the sort to prepare data for the cash requirements report.

8. **Set Wire Transfer Indicator**:
   - Checks parameter `?3?`:
     - If `?3?` equals `'WT'`, sets a local variable at offset 198 to `'WT*** WIRE TRANSFER ***'`.
     - Otherwise, sets it to a blank string.

9. **Load AP155 Program**:
   - Loads the `AP155` program (likely for generating the cash requirements report or final payment processing).
   - Opens files:
     - `APPAY` (labeled `?9?APPY?WS?`, shared): Payment work file.
     - `AP155S` (labeled `?9?APPS?WS?`): Sorted payment file.
     - `APCONT` (labeled `?9?APCONT`, shared): A/P control file.
     - `APPYTR` (labeled `?9?APPT?WS?`): Transaction work file.
     - `APVEND` (labeled `?9?APVEND`, shared): Vendor master file.
     - `APOPEN` (labeled `?9?APOPEN`, shared): Open A/P file.
     - `APCHKR` (labeled `?9?APCHKR`, shared): Check register file.
     - `APPYCK` (labeled `?9?APPC?WS?`, extendable): Check work file.
     - `APDETINV` (labeled `?9?APDT?WS?`, extendable): Invoice detail work file.
     - `APPYDS` (labeled `?9?APDS?WS?`, extendable): Payment discount work file.
   - Overrides printer file `APCSHRQ` to output queue `QUSRSYS/APEDIT` or `QUSRSYS/TESTOUTQ` based on parameter `?9?`.
   - Runs `AP155` to generate the cash requirements report or finalize payments.

10. **AP156 Program and Cleanup**:
    - Invokes `AP156` with parameter `?9?`.
    - If in AUTO mode (`?2?/AUTO`), clears all local variables (`LOCAL BLANK-*ALL`).
    - Proceeds to the `END` tag, terminating the program.

---

### **External Programs Called**

The OCL program invokes the following external programs:
1. **AP150**: Initial program for A/P payment transaction entry and editing.
2. **#GSORT**: System sort utility, called twice for sorting payment and report data.
3. **AP151**: Program for processing sorted payment transactions and handling discounts.
4. **AP155**: Program for generating the cash requirements report or finalizing payments.
5. **AP156**: Final program, likely for cleanup or additional processing.

---

### **Tables (Files) Used**

The program references the following files (tables), with their labels and descriptions:
1. **APPYTR** (`?9?APPT?WS?`, `?9?AP151S`): Payment transaction work file (input and sorted output).
2. **APCONT** (`?9?APCONT`): A/P control file, shared.
3. **GLMAST** (`?9?GLMAST`): General ledger master file, shared.
4. **APVEND** (`?9?APVEND`): Vendor master file, shared.
5. **APOPEN** (`?9?APOPEN`): Open A/P file, shared.
6. **GSTABL** (`?9?GSTABL`): General system table, shared.
7. **GSCONT** (`?9?GSCONT`): General system control file, shared.
8. **APPAY** (`?9?APPY?WS?`): Payment work file, extendable.
9. **AP155S** (`?9?APPS?WS?`): Sorted payment file for reporting.
10. **APCHKR** (`?9?APCHKR`): Check register file, shared.
11. **APPYCK** (`?9?APPC?WS?`): Check work file, extendable.
12. **APDETINV** (`?9?APDT?WS?`): Invoice detail work file, extendable.
13. **APPYDS** (`?9?APDS?WS?`): Payment discount work file, extendable.
14. **APPO?WS?**: Temporary work file (checked for existence).
15. **APCSHRQ**: Printer file for the cash requirements report.

---

### **Summary**

The `AP150` OCL program orchestrates the A/P payment transaction process by:
- Creating and managing work files for transactions, payments, and discounts.
- Sorting data for processing and reporting.
- Invoking RPG programs (`AP150`, `AP151`, `AP155`, `AP156`) for specific tasks.
- Handling special cases like wire transfers and late payment discounts.
- Generating a cash requirements report.

It interacts with multiple files (both permanent and temporary) and uses the `#GSORT` utility for data organization. The program is designed to run either standalone or in AUTO mode as part of a larger process (`AP200`).