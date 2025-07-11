The provided document is an **Operation Control Language (OCL)** program, specifically `AP190.ocl36.txt`, used for Accounts Payable (A/P) check reconciliation in an IBM System/3x or AS/400 environment. Below, I’ll break down the **process steps**, identify the **external programs called**, and list the **tables (files)** used.

### Process Steps of the AP190 OCL Program

OCL is a scripting language used to control job execution, file operations, and program invocation on IBM midrange systems. The program `AP190` handles A/P check reconciliation, performing file setup, validation, and editing. Here’s a step-by-step explanation of the process:

1. **Initial File Setup (BLDFILE)**:
   - `// IFF DATAF1-?9?APCR?WS? BLDFILE ?9?APCR?WS?,I,RECORDS,500,80,,,2,16,DFILE,,50`
     - This command checks if the file `?9?APCR?WS?` (a work file for A/P check reconciliation) exists. If not, it creates it using the `BLDFILE` operation.
     - Parameters:
       - `I`: Input mode.
       - `RECORDS,500`: Allocates space for 500 records.
       - `80`: Record length of 80 bytes.
       - `2,16`: Likely specifies key field attributes (e.g., key starts at position 2, length 16).
       - `DFILE,,50`: Indicates a disk file with a block size or extent of 50.
     - **Purpose**: Ensures the work file `?9?APCR?WS?` is available for processing.

2. **Load and Execute AP190 Program**:
   - `// LOAD AP190`
   - `// FILE NAME-APCRTR,LABEL-?9?APCR?WS?,EXTEND-100`
   - `// FILE NAME-APCONT,LABEL-?9?APCONT,DISP-SHR`
   - `// FILE NAME-GLMAST,LABEL-?9?GLMAST,DISP-SHR`
   - `// FILE NAME-APCHKR,LABEL-?9?APCHKR,DISP-SHR`
   - `// RUN`
     - **LOAD AP190**: Loads the `AP190` program (likely an RPG program) into memory.
     - **FILE Definitions**:
       - `APCRTR` (mapped to `?9?APCR?WS?`): The work file for check reconciliation transactions, with an extension of 100 records.
       - `APCONT` (mapped to `?9?APCONT`): A control file, opened in shared mode (`DISP-SHR`).
       - `GLMAST` (mapped to `?9?GLMAST`): General Ledger master file, opened in shared mode.
       - `APCHKR` (mapped to `?9?APCHKR`): A/P check reconciliation file, opened in shared mode.
     - **RUN**: Executes the `AP190` program, which processes the check reconciliation data using these files.
     - **Purpose**: The `AP190` program likely validates or processes check reconciliation data, updating or reading from the specified files.

3. **Conditional Check for File Existence**:
   - `// IF ?F'A,?9?APCR?WS?'?/00000000 GOTO END`
     - This checks if the work file `?9?APCR?WS?` is empty or has no records (condition `?F'A` checks file attributes, and `/00000000` likely indicates zero records).
     - If true, the program jumps to the `END` tag, skipping further processing.
     - **Purpose**: Prevents unnecessary execution if there’s no data to process.

4. **Display Message**:
   - `// * 'A/P CHECK RECONCILIATION EDIT EXECUTING'`
     - Outputs a message to the console or log indicating that the A/P check reconciliation edit process is running.
     - **Purpose**: Provides feedback to the operator about the program’s status.

5. **Load and Execute AP195 Program**:
   - `// LOAD AP195`
   - `// FILE NAME-APCRTR,LABEL-?9?APCR?WS?`
   - `// FILE NAME-APCONT,LABEL-?9?APCONT,DISP-SHR`
   - `// RUN`
     - **LOAD AP195**: Loads the `AP195` program (another RPG program) into memory.
     - **FILE Definitions**:
       - `APCRTR` (mapped to `?9?APCR?WS?`): Reuses the work file from the previous step.
       - `APCONT` (mapped to `?9?APCONT`): Reuses the control file in shared mode.
     - **RUN**: Executes the `AP195` program.
     - **Purpose**: The `AP195` program likely performs additional processing or validation on the check reconciliation data, such as generating reports or finalizing edits.

6. **End of Program**:
   - `// TAG END`
     - Marks the end of the program execution.
     - **Purpose**: Terminates the OCL script.

### External Programs Called

The OCL program invokes the following external programs:
1. **AP190**: The main program for A/P check reconciliation, likely responsible for initial data processing or validation.
2. **AP195**: A secondary program, possibly for further editing, reporting, or finalizing the reconciliation process.

### Tables (Files) Used

The program references the following files (tables):
1. **APCRTR** (labeled `?9?APCR?WS?`):
   - A work file used for check reconciliation transactions.
   - Created or extended as needed.
2. **APCONT** (labeled `?9?APCONT`):
   - A control file, likely containing configuration or reference data for A/P processing.
   - Opened in shared mode (`DISP-SHR`).
3. **GLMAST** (labeled `?9?GLMAST`):
   - General Ledger master file, containing financial account data.
   - Opened in shared mode (`DISP-SHR`).
4. **APCHKR** (labeled `?9?APCHKR`):
   - A/P check reconciliation file, likely storing check details or reconciliation status.
   - Opened in shared mode (`DISP-SHR`).

### Summary

The `AP190` OCL program orchestrates an A/P check reconciliation process by:
1. Creating or verifying the work file `APCRTR`.
2. Running the `AP190` program to process reconciliation data using `APCRTR`, `APCONT`, `GLMAST`, and `APCHKR`.
3. Skipping further execution if no data is present in `APCRTR`.
4. Running the `AP195` program for additional processing or reporting using `APCRTR` and `APCONT`.
5. Displaying status messages for operator feedback.

**External Programs**: `AP190`, `AP195`.  
**Files Used**: `APCRTR`, `APCONT`, `GLMAST`, `APCHKR`.