The provided document is an OCL (Operation Control Language) program, `AP160.ocl36.txt`, used on IBM midrange systems like the AS/400 (now IBM i) to manage Accounts Payable (A/P) check printing. Below, I’ll explain the process steps of the program, list any external programs called, and identify tables or files created, destroyed, or read from.

---

### Process Steps of the AP160 OCL Program

The OCL program automates the printing of A/P checks, interacting with various files and printer configurations. Here’s a step-by-step breakdown of the process:

1. **Initialization and Conditional Execution Check**:
   - The program checks if the parameter `?2?` is set to `AUTO`. If true, it jumps to the `AP160` tag, indicating it was called automatically by another procedure (likely `AP200`).
   - If not running in `AUTO` mode, the program proceeds with manual execution.

2. **User Interaction (Pause for Confirmation)**:
   - A commented-out `PAUSE` statement (not currently active) would prompt the user to either cancel the process (by pressing ATTN, 2, ENTER) or continue (by pressing 0, ENTER).
   - The `ATTR INQUIRY-YES,CANCEL-NO` suggests an inquiry screen for user confirmation, but it’s not explicitly active in the provided code.

3. **Program Load and File Definitions (TAG AP160)**:
   - The program reaches the `AP160` tag and loads the `AP160` program (likely an RPG or CL program responsible for the core logic).
   - It defines several files with specific labels and attributes:
     - `APPAY` (labeled `?9?APPY?WS?`): Likely the payment file.
     - `AP160S` (labeled `?9?APPS?WS?`): Possibly a selection or summary file.
     - `APCONT` (labeled `?9?APCONT`, `DISP-SHR`): A control file, shared access.
     - `APPYTR` (labeled `?9?APPT?WS?`): A transaction file.
     - `APVEND` (labeled `?9?APVEND`, `DISP-SHR`): Vendor master file, shared access.
     - `APOPEN` (labeled `?9?APOPEN`, `DISP-SHR`): Open payables file, shared access.
     - `APPYCK` (labeled `?9?APPC?WS?`): Check file.
   - The `?9?` and `?WS?` are placeholders for environment-specific prefixes or suffixes (e.g., library or workspace identifiers).

4. **Printer File Overrides**:
   - The program configures printer output for two files: `APCHECK` (main check output) and `CHECKCPY` (check copy output).
   - Conditional overrides are applied based on the value of `?9?`:
     - If `?9?` is `G` (likely production environment):
       - `APCHECK` is directed to the `QUSRSYS/APCHECKS` output queue.
       - `CHECKCPY` is directed to the `QUSRSYS/APCHKCPY` output queue.
       - Both use device type `*SCS`, 6 lines per inch (LPI), 12 characters per inch (CPI), standard print quality, form type `APCK`, and are saved after printing.
     - If `?9?` is `Z` (likely test environment):
       - Both `APCHECK` and `CHECKCPY` are directed to the `QUSRSYS/TESTOUTQ` output queue with the same settings.
   - These overrides ensure checks are printed to the correct printer based on the environment.

5. **Program Execution**:
   - The `RUN` command executes the `AP160` program, which processes the defined files and generates check output.
   - The RPG program (AP160) likely reads from the vendor, open payables, and payment files, performs calculations (e.g., payment amounts), and writes check data to the `APPYCK` file and printer files.

6. **Temporary File Creation (Conditional)**:
   - If a file named `?9?APPO?WS?` exists in the `DATAF1` parameter, the program builds a temporary file with the same name using the `BLDFILE` command.
   - The file is created with:
     - `S` (sequential organization).
     - `RECORDS` (populated with records).
     - Parameters `1,10` (possibly record length or block size).
   - This temporary file is likely used to store intermediate check printing data, as indicated by the comment `--TEMPORARY FILE TO ALLOW CHECK P`.

7. **Completion**:
   - After the `AP160` program completes, the OCL script ends. Any printed checks are sent to the configured output queues, and temporary files (if created) may persist until explicitly cleared.

---

### List of External Programs Called

- **AP160**: The main program loaded and executed by the OCL script. It’s likely an RPG or CL program that handles the core logic for printing A/P checks.
- **AP200** (implied): The comment suggests `AP200` may call `AP160` in `AUTO` mode, but it’s not directly called within this script.

No other external programs are explicitly called in the provided OCL code.

---

### Tables or Files Created, Destroyed, or Read From

The OCL program interacts with several files, either by reading, writing, or creating them. Below is a categorized list based on the code and typical A/P check printing processes:

#### Files Read From:
- **APCONT** (`?9?APCONT`, `DISP-SHR`): Control file, likely containing configuration or control data for A/P processing (e.g., bank account details).
- **APVEND** (`?9?APVEND`, `DISP-SHR`): Vendor master file, containing vendor details (e.g., name, address, payment terms).
- **APOPEN** (`?9?APOPEN`, `DISP-SHR`): Open payables file, listing outstanding invoices to be paid.

#### Files Read From or Written To:
- **APPAY** (`?9?APPY?WS?`): Payment file, likely containing selected payments to be processed into checks.
- **AP160S** (`?9?APPS?WS?`): Selection or summary file, possibly used to store temporary data during processing.
- **APPYTR** (`?9?APPT?WS?`): Transaction file, likely used to log payment transactions.
- **APPYCK** (`?9?APPC?WS?`): Check file, where check details (e.g., check number, amounts) are written.

#### Files Created:
- **`?9?APPO?WS?`** (conditional): A temporary file is created if `BLDFILE` is triggered for `?9?APPO?WS?`. This file is used for check processing and is marked as temporary.

#### Files Destroyed:
- No files are destroyed in the provided code. Temporary files like `?9?APPO?WS?` may be cleared by other processes outside this script, but this isn’t explicitly shown.

#### Printer Output Files:
- **APCHECK**: Printer file for main checks, directed to an output queue (`APCHECKS` or `TESTOUTQ`).
- **CHECKCPY**: Printer file for check copies, directed to an output queue (`APCHKCPY` or `TESTOUTQ`).

---

### Notes and Assumptions
- **Placeholders**: The `?9?` and `?WS?` are variables resolved at runtime, representing library names, typically representing names, company codes, or workspace identifiers. Without specific values, I’ve described their general purpose.
- **Commented Code**: The commented-out code (e.g., `PAUSE`, initial display) suggests optional user interaction, which may be enabled in certain conditions or older versions.
- **RPG Logic**: The actual check printing logic (e.g., calculations, formatting) is in `AP160`, which isn’t provided. My analysis focuses on the OCL orchestration.
- **Temporary File Scope**: The `BLDFILE` creates a temporary file, but its lifecycle (e.g., deletion) isn’t shown here. It’s likely cleared by a cleanup job.

If you have the `AP160` RPG source or additional context (e.g., file schemas, specific `?9?` values), I can provide deeper insights into the data flow or logic. Let me know if you’d like me to analyze related content or search for more information!