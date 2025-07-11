The provided document is an **OCL (Operation Control Language) program** used on IBM System/36 or AS/400 systems to manage Accounts Payable (A/P) check reconciliation uploads and edits from PNC. Below, I’ll explain the process steps of the program, identify the external programs called, and list the tables (files) used.

### Process Steps of the OCL Program

The OCL script outlines a sequence of operations for processing A/P check reconciliation data. Here’s a step-by-step breakdown of the program’s logic:

1. **Initial Setup and Conditional File Deletion**:
   - The program checks if a file named `?9?APCR?WS?` exists in `DATAF1`.
   - If the file exists (`IFF DATAF1-?9?APCR?WS?`), it is deleted (`DELETE ?9?APCR?WS?,F1`).
   - This ensures that any previous version of the working file is removed before proceeding, preventing data conflicts.

2. **Conditional File Creation**:
   - If the file `?9?APCR?WS?` does not exist in `DATAF1` or after deletion, the program creates a new file (`BLDFILE ?9?APCR?WS?,I,RECORDS,500,80,,,2,16,DFILE,,50`).
   - The `BLDFILE` command specifies:
     - File name: `?9?APCR?WS?`
     - Type: Indexed file (`I`)
     - Initial record count: 500 records
     - Record length: 80 bytes
     - Other parameters: Likely related to file attributes like key length (2 bytes) and key position (16th byte).
     - File is created in `DFILE` with a block size of 50.

3. **Conditional Branching**:
   - The program checks if the file `?9?APCR?WS?` has a specific condition (`?F'A,?9?APCR?WS?'?/00000000`).
   - If the condition is met (likely checking if the file is empty or has no records), the program branches to the `SKIP` tag, bypassing the execution of `AP192`.

4. **Execution of AP192**:
   - If the condition in step 3 is not met (i.e., the file exists and has data), the program proceeds to load and run the `AP192` program.
   - Files used by `AP192`:
     - `APCHKUP` (labeled `?9?APCHKUP`, disposition `SHR` for shared access): Likely the input file containing check reconciliation data uploaded from PNC.
     - `APCRTR` (labeled `?9?APCR?WS?`, disposition `SHR`, extendable by 100 records): The working file for check reconciliation transactions.
   - The `RUN` command executes `AP192`, which presumably processes the uploaded check data and updates the `APCRTR` file.

5. **Execution of AP193**:
   - After `AP192` completes (or if the program branches to `SKIP`), the program loads and runs the `AP193` program.
   - Files used by `AP193`:
     - `APCRTR` (labeled `?9?APCR?WS?`, disposition `SHR`): The same working file used in `AP192`, containing processed check reconciliation data.
     - `APCONT` (labeled `?9?APCONT`, disposition `SHR`): Likely a control file for A/P processing, containing configuration or summary data.
     - `GLMAST` (labeled `?9?GLMAST`, disposition `SHR`): General Ledger master file, used for updating financial records.
     - `APCHKR` (labeled `?9?APCHKR`, disposition `SHR`): A file likely used for storing reconciled check data or audit trails.
   - The `RUN` command executes `AP193`, which likely finalizes the reconciliation process, updates the General Ledger, and stores results in the appropriate files.

6. **Program Flow Control**:
   - The `TAG SKIP` label serves as a branching point. If the file `?9?APCR?WS?` is empty or meets the condition in step 3, the program skips `AP192` and proceeds directly to `AP193`.
   - This ensures that `AP193` runs regardless of whether `AP192` was executed, providing a consistent final processing step.

### External Programs Called

The OCL program explicitly calls the following external programs:
1. **AP192**: Handles the initial processing of A/P check reconciliation data, likely validating and uploading data from `APCHKUP` to `APCRTR`.
2. **AP193**: Performs final processing, such as updating the General Ledger (`GLMAST`), reconciling checks in `APCHKR`, and using control data from `APCONT`.

### Tables (Files) Used

The program interacts with the following files (referred to as tables in some contexts):
1. **APCHKUP** (`?9?APCHKUP`): Input file containing check reconciliation data uploaded from PNC. Used by `AP192`.
2. **APCRTR** (`?9?APCR?WS?`): Working file for check reconciliation transactions, created or deleted as needed. Used by both `AP192` and `AP193`.
3. **APCONT** (`?9?APCONT`): Control file for A/P processing, likely containing configuration or summary data. Used by `AP193`.
4. **GLMAST** (`?9?GLMAST`): General Ledger master file, used for financial updates. Used by `AP193`.
5. **APCHKR** (`?9?APCHKR`): File for reconciled check data or audit trails. Used by `AP193`.

### Summary

The OCL program orchestrates an A/P check reconciliation process by:
- Managing the lifecycle of a working file (`APCRTR`) through deletion and creation.
- Conditionally running `AP192` to process uploaded check data if the working file has valid data.
- Always running `AP193` to finalize reconciliation, update the General Ledger, and store results.
- The program uses shared files (`SHR` disposition) to allow concurrent access and extends the `APCRTR` file as needed.

**External Programs**: `AP192`, `AP193`  
**Files Used**: `APCHKUP`, `APCRTR`, `APCONT`, `GLMAST`, `APCHKR`

If you need further clarification or details about specific file formats or program logic, let me know!