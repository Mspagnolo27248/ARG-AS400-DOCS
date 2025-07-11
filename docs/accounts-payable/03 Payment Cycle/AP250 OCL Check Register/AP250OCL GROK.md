The RPG OCL program `AP250.OCL36` is an Accounts Payable (A/P) Check Register process that handles check printing, journal updates, and file maintenance. Below is a detailed breakdown of the **process steps**, **external programs called**, and **tables/files used** as requested.

---

### Process Steps

The OCL program follows a structured sequence of steps, primarily focused on generating an A/P check register, updating related files, and producing a cash disbursements journal. The steps are derived from the OCL code and are listed in execution order:

1. **Initialization**:
   - Calls the procedure `STRPCOCLP` to set up the processing environment.
   - Conditionally sets a local variable at offset 198 to indicate a wire transfer (`WT*** WIRE TRANSFER ***`) if parameter `?3?` is `/WT`, otherwise clears it.

2. **Check for Auto-Run and Workstation Lock**:
   - If parameter `?2?` is `/AUTO`, skips to the `AP250` tag to proceed with processing.
   - If the workstation is locked for check posting (`DATAF1-?9?APPO?WS?`), displays a warning message indicating that checks cannot be posted until printed via Option 11 of `APMENU`. The job pauses, allowing cancellation (press 0, Enter to cancel), and jumps to the `END` tag if cancelled.

3. **Prompt User to Continue or Cancel**:
   - Displays a message: `'A/P CHECK REGISTER, JOURNAL, AND UPDATE FILES'` and pauses, allowing the user to cancel (press ATTN, 2, Enter) or continue (press 0, Enter).

4. **File Preparation**:
   - Deletes the temporary file `APCD?WS?` if it exists.
   - Creates a new file `?9?APCD?WS?` with a sequential organization, 999,000 records, and a record length of 128 bytes.

5. **Main Check Register Processing (AP250)**:
   - Displays the message: `'CHECK REGISTER, UPDATE FILES EXECUTING'`.
   - Loads and runs the program `AP250` with multiple file assignments:
     - `APPYCK` → `?9?APPC?WS?`
     - `APPAY` → `?9?APPY?WS?`
     - `AP250S` → `?9?APPS?WS?`
     - `APPYTR` → `?9?APPT?WS?`
     - `APCONT`, `APVEND`, `APVEND2`, `APCHKR`, `APPYDS`, `APOPEN`, `APOPENH`, `APOPEND`, `APOPENV`, `APHISTH`, `APHISTD`, `APHISTV`, `FRCINH`, `FRCFBH` → Shared files with respective labels.
     - `APCDJR` → `?9?APCD?WS?` with an extension of 100 records.
     - `APDSMS` → Shared file.
   - Overrides printer files `APPNCF` and `APPRINT` to specific output queues (`QUSRSYS/PRTPNC`, `QUSRSYS/APPOST`, or `QUSRSYS/TESTOUTQ`) if `?9?` is `/G`.

6. **Update Commission Table (AP251)**:
   - Loads and runs the program `AP251` to update the commission table with payment data.
   - Uses files:
     - `APPAY` → `?9?APPY?WS?` (shared).
     - `APTORCY` → `?9?APTORCY` (shared).

7. **Sort Cash Disbursements Journal Data**:
   - Displays the message: `'CASH DISBURSMENTS JOURNAL EXECUTING'`.
   - Loads and runs the program `#GSORT` to sort the `APCD?WS?` file into `?9?APCS?WS?`.
   - Sorting parameters:
     - Sort key: 30 bytes ascending, starting at position 3.
     - Include records where position 1 is not equal to 'C' or 'D'.
     - Fields included in sort:
       - Company (positions 2–3)
       - C,D (position 12)
       - AP,CASH,DISC (positions 106–115)
       - G/L # (positions 13–20)
       - SEQ # (positions 97–105).

8. **Generate Cash Disbursements Journal (AP255)**:
   - Loads and runs the program `AP255` to process the sorted data.
   - Uses files:
     - `APCDJR` → `?9?APCD?WS?`
     - `AP255S` → `?9?APCS?WS?`
     - `APCONT` → Shared.
     - `TEMGEN` → Shared.
   - Overrides printer file `APPRINT` to output queues (`QUSRSYS/APPOST` or `QUSRSYS/TESTOUTQ`) if `?9?` is `/G`.

9. **Optional Wire Transfer Processing**:
   - Checks if the file `?9?APDT?WS?` has a non-zero record count (`/000000`).
   - If records exist:
     - Creates a new file `?9?APDT?WS?C` with indexed organization, 500 records, and a record length of 10 bytes.
     - Loads and runs `AP256A` with files:
       - `APDTWS` → `?9?APDT?WS?`
       - `APDTWSC` → `?9?APDT?WS?C` (retained as temporary, 50 records).
       - `APVNFMX` → Shared.
     - Loads and runs `AP256` with files:
       - `APDTWS` → `?9?APDT?WS?`
       - `APDTWSC` → `?9?APDT?WS?C` (shared).
       - `APVEND`, `APCONT`, `APVNFMX` → Shared.
     - Overrides printer files `REPORT1`, `REPORT2`, `REPORT3`, `REPORT4` to output queues (`QUSRSYS/APACHOUTQ` or `QUSRSYS/TESTOUTQ`) if `?9?` is `/G`.

10. **Cleanup**:
    - Deletes temporary files: `APPT?WS?`, `APPY?WS?`, `APPS?WS?`, `APPC?WS?`, `APDS?WS?`, `APPO?WS?`, `APCD?WS?`, `APCS?WS?`, `APDT?WS?`, `APDT?WS?C`.
    - If in auto mode (`?2?` is `/AUTO`), clears all local variables.

11. **End Processing**:
    - Jumps to the `END` tag to terminate the job.

---

### External Programs Called

The OCL program invokes the following external programs:

1. **STRPCOCLP**: Initializes the processing environment (likely a system or custom procedure).
2. **AP250**: Main program for generating the A/P check register and updating files.
3. **AP251**: Updates the commission table with payment data.
4. **#GSORT**: Sorts the cash disbursements journal data.
5. **AP255**: Generates the cash disbursements journal.
6. **AP256A**: Processes wire transfer data (first phase).
7. **AP256**: Processes wire transfer data (second phase).

---

### Tables/Files Used

The program interacts with multiple files, some temporary (workstation-specific) and others permanent (shared). The files are listed below with their logical names, labels, and usage context:

| **Logical Name** | **Label**            | **Usage**                                                                 | **Disposition** |
|------------------|----------------------|---------------------------------------------------------------------------|-----------------|
| APPYCK           | ?9?APPC?WS?          | Check-related data (temporary).                                           | Exclusive       |
| APPAY            | ?9?APPY?WS?          | Payment data (temporary, used in AP250 and AP251).                        | Exclusive       |
| AP250S           | ?9?APPS?WS?          | Supporting data for AP250 (temporary).                                    | Exclusive       |
| APPYTR           | ?9?APPT?WS?          | Transaction data (temporary).                                             | Exclusive       |
| APCONT           | ?9?APCONT            | A/P control file (used in AP250, AP255, AP256).                           | Shared          |
| APVEND           | ?9?APVEND            | Vendor master file (used in AP250, AP256).                                | Shared          |
| APVEND2          | ?9?APVEND            | Secondary vendor file (used in AP250).                                    | Shared          |
| APCHKR           | ?9?APCHKR            | Check register file (used in AP250).                                      | Shared          |
| APPYDS           | ?9?APDS?WS?          | Payment distribution file (temporary, shared in AP250).                   | Shared          |
| APOPEN           | ?9?APOPEN            | Open A/P file (used in AP250).                                            | Shared          |
| APOPENH          | ?9?APOPNH            | Open A/P header file (used in AP250).                                     | Shared          |
| APOPEND          | ?9?APOPND            | Open A/P detail file (used in AP250).                                     | Shared          |
| APOPENV          | ?9?APOPNV            | Open A/P vendor file (used in AP250).                                     | Shared          |
| APHISTH          | ?9?APHSTH            | A/P history header file (used in AP250).                                  | Shared          |
| APHISTD          | ?9?APHSTD            | A/P history detail file (used in AP250).                                  | Shared          |
| APHISTV          | ?9?APHSTV            | A/P history vendor file (used in AP250).                                  | Shared          |
| FRCINH           | ?9?FRCINH            | Financial control invoice header (used in AP250).                         | Shared          |
| FRCFBH           | ?9?FRCFBH            | Financial control freight bill header (used in AP250).                    | Shared          |
| APCDJR           | ?9?APCD?WS?          | Cash disbursements journal file (temporary, used in AP250, AP255).        | Exclusive       |
| APDSMS           | ?9?APDSMS            | Distribution master file (used in AP250).                                 | Shared          |
| APTORCY          | ?9?APTORCY           | Commission table file (used in AP251).                                    | Shared          |
| AP255S           | ?9?APCS?WS?          | Sorted cash disbursements data (temporary, used in AP255).                | Exclusive       |
| TEMGEN           | ?9?TEMGEN            | Temporary general ledger file (used in AP255).                            | Shared          |
| APDTWS           | ?9?APDT?WS?          | Wire transfer data (temporary, used in AP256A, AP256).                    | Exclusive       |
| APDTWSC          | ?9?APDT?WS?C         | Wire transfer control data (temporary, used in AP256A, AP256).            | Exclusive/Shared|
| APVNFMX          | ?9?APVNFMX           | Vendor file matrix (used in AP256A, AP256).                               | Shared          |

---

### Notes
- **Temporary Files**: Files with `?WS?` in their labels are workstation-specific and deleted at the end of the job.
- **Shared Files**: Permanent files (e.g., `APCONT`, `APVEND`) are accessed in shared mode, indicating they are used across multiple processes.
- **Printer Overrides**: The program conditionally redirects output to different queues based on the `?9?` parameter (`/G` for production, otherwise test queues).
- **Wire Transfer Processing**: The `AP256A` and `AP256` steps are conditional, only executed if wire transfer data exists in `?9?APDT?WS?`.

This summary provides a comprehensive overview of the `AP250.OCL36` program's functionality, external dependencies, and data interactions. Let me know if you need further clarification or analysis!