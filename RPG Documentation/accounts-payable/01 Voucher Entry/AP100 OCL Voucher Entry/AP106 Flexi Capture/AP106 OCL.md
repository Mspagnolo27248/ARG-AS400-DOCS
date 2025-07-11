### **AP106 OCL Program**
This is an Operation Control Language (OCL) file (AP106.ocl36.txt) used on the IBM i (AS/400) system to handle Accounts Payable (A/P) voucher entry uploads for FlexiCapture import files. Below is a detailed explanation of the **process steps**, **external programs called**, and **tables/files used** in this program, based on the provided OCL code.

---

### **Overview**
The OCL file automates the process of uploading A/P voucher entries from a FlexiCapture import file into the IBM i system. It ensures that the upload process is synchronized with other system operations (e.g., Profound Genie) by introducing pauses and checks. The program processes the uploaded data, saves it to a history table, and prepares it for further processing by other programs (e.g., AP107, AP110). It also interacts with PC-based batch files for file uploads and manages various database files on the IBM i system.

---

### **Process Steps**

1. **Initial Checks for A/P Voucher Entry File (APTR?WS?)**
   - The program checks if there are records in the A/P voucher entry file (`?9?APTR?WS?`) using conditional statements (`IFF`).
   - If records exist, it displays messages indicating:
     - "There are records in the A/P voucher entry file."
     - "You must post the batch before you can run the import from FlexiCapture on this workstation."
   - A `PAUSE` is introduced to ensure the FlexiCapture upload process completes before further processing (to address synchronization issues with Profound Genie).

2. **Clear and Rebuild A/P Voucher Entry Files**
   - If the A/P voucher entry file exists (`DATAF1-?9?APTR?WS?`), the program:
     - Clears the file using `CLRPFM ?9?APTR?WS?`.
     - Rebuilds the file with `BLDFILE ?9?APTR?WS?,I,RECORDS,500,404,,,2,10`, specifying 500 records, a record length of 404 bytes, and other parameters.
   - If the index file exists (`DATAF1-?9?APTX?WS?`), it builds an index using `BLDINDEX ?9?APTX?WS?,2,2,?9?APTR?WS?,+,,,12,5,385,20`.

3. **Set Program Variable**
   - The variable `P20` is set to `'APTR?WS?'` to reference the A/P voucher entry file.

4. **Start PC Organizer and Run FlexiCapture Upload**
   - The program calls `STRPCOCLP` to start the PC Organizer, which prepares the system for PC-based commands.
   - It executes a PC batch file (`STRPCCMD`) located at `C:\Program Files (x86)\FLEXICAPTURE_UPLOAD\APINVUPD.BAT` to upload the journal entry file. This batch file is executed regardless of the `?9?/G` condition (same command in both `IF` and `ELSE` branches).
   - A `PAUSE` is displayed with the message "TYPE 0, ENTER TO CONTINUE AFTER UPLOAD PROCESS IS COMPLETE," ensuring the user waits for the upload to finish.

5. **Run AP107 Program**
   - The program loads `AP107` and opens two files:
     - `APINVUP` (the uploaded journal entry file).
     - `APFLEXH` (the history table for FlexiCapture uploads).
   - The `AP107` program processes the uploaded data and saves it to the history table (`APFLEXH`).

6. **Run AP106 Program**
   - The program loads itself (`AP106`) and opens several files:
     - `APINVUP` (the uploaded journal entry file).
     - `APTRAN` (A/P transaction file, labeled `?9?APTR?WS?`).
     - `APCONT` (A/P control file).
     - `APVEND` (A/P vendor file).
     - `GSTABL` (general system table).
     - `APDATE` (A/P date file).
   - The `AP106` program processes the uploaded data into the A/P transaction file and validates it against control, vendor, and other reference files.

7. **Clear and Rebuild A/P Check Temporary File**
   - The program deletes the A/P check temporary file (`APCT?WS?`) using `GSDELETE APCT?WS?,,,,,,,,?9?`.
   - It rebuilds the file with `BLDFILE ?9?APCT?WS?,I,RECORDS,500,80,,,2,21`, specifying 500 records and a record length of 80 bytes.

8. **Run AP110 Program**
   - The program loads `AP110` and opens multiple files:
     - `APTRAN` (A/P transaction file).
     - `APCONT` (A/P control file).
     - `APCHKR` (A/P check register file).
     - `APCHKT` (A/P check temporary file, labeled `?9?APCT?WS?`).
     - `APTRNX` (A/P transaction index file, labeled `?9?APTX?WS?`).
     - `GLMAST` (general ledger master file).
     - `APOPNHC` (A/P open history control file).
     - `GSTABL` (general system table).
     - `APINVH` (A/P invoice history file).
     - `APSTAT` (A/P status file).
     - `APVEND` (A/P vendor file).
     - `INFIL1` (information file 1).
     - `INTZH1` (internal history file 1).
   - It overrides the printer file `APLIST` to output to either `QUSRSYS/APEDIT` or `QUSRSYS/TESTOUTQ` based on the `?9?/G` condition.
   - The `AP110` program processes the A/P transactions, generates checks, and updates related files (e.g., general ledger, invoice history).

9. **Clear Uploaded File**
   - The program clears the `APINVUP` file using `CLRPFM ?9?APINVUP` to prepare for the next upload.

10. **End of Program**
    - The program jumps to the `END` tag, clears local variables (`LOCAL BLANK-*ALL`), and terminates.

---

### **External Programs Called**

1. **STRPCOCLP**
   - This program starts the PC Organizer, enabling the execution of PC-based commands (e.g., batch files) from the IBM i system.

2. **AP107**
   - Processes the uploaded journal entry file (`APINVUP`) and saves it to the FlexiCapture history table (`APFLEXH`).

3. **AP106**
   - The OCL file itself is loaded as a program to process the uploaded data into the A/P transaction file (`APTRAN`) and validate it against control, vendor, and other files.

4. **AP110**
   - Processes A/P transactions, generates checks, and updates related files (e.g., general ledger, invoice history).

5. **APINVUP.BAT**
   - A PC batch file (`C:\Program Files (x86)\FLEXICAPTURE_UPLOAD\APINVUP.BAT`) executed via `STRPCCMD` to upload the journal entry file from the PC to the IBM i system.

---

### **Tables/Files Used**

The program interacts with the following database files on the IBM i system, identified by their labels and purposes:

| **File Name** | **Label**            | **Purpose**                                                                 | **Used in Program** |
|---------------|----------------------|-----------------------------------------------------------------------------|---------------------|
| APINVUP       | ?9?APINVUP          | Stores the uploaded journal entry file from FlexiCapture.                   | AP107, AP106        |
| APFLEXH       | ?9?APFLEXH          | History table for FlexiCapture uploads (saves uploaded file details).       | AP107               |
| APTRAN        | ?9?APTR?WS?         | A/P transaction file (stores voucher entries).                              | AP106, AP110        |
| APCONT        | ?9?APCONT           | A/P control file (contains control data for A/P processing).                | AP106, AP110        |
| APVEND        | ?9?APVEND           | A/P vendor file (contains vendor information).                              | AP106, AP110        |
| GSTABL        | ?9?GSTABL           | General system table (contains system-wide configuration data).              | AP106, AP110        |
| APDATE        | ?9?APDATE           | A/P date file (stores date-related data for A/P processing).                | AP106               |
| APCHKR        | ?9?APCHKR           | A/P check register file (tracks issued checks).                             | AP110               |
| APCHKT        | ?9?APCT?WS?         | A/P check temporary file (temporary storage for check data).                | AP110               |
| APTRNX        | ?9?APTX?WS?         | A/P transaction index file (index for A/P transaction file).                | AP110               |
| GLMAST        | ?9?GLMAST           | General ledger master file (stores G/L account information).                | AP110               |
| APOPNHC       | ?9?APOPNHC          | A/P open history control file (tracks open A/P history).                    | AP110               |
| APINVH        | ?9?APINVH           | A/P invoice history file (stores historical invoice data).                  | AP110               |
| APSTAT        | ?9?APSTAT           | A/P status file (tracks status of A/P transactions).                        | AP110               |
| INFIL1        | ?9?INFIL1           | Information file 1 (general-purpose data file).                             | AP110               |
| INTZH1        | ?9?INTZH1           | Internal history file 1 (internal history data).                            | AP110               |

**Notes on File Labels:**
- `?9?` is a library prefix (likely a system or company-specific library).
- `?WS?` is a workstation identifier, making file names unique per workstation.
- `DISP-SHR` indicates shared access, allowing multiple processes to read the file.
- `EXTEND-100` specifies additional record capacity for files like `APTRAN` and `APCHKT`.

---

### **Additional Notes**

- **Synchronization with Profound Genie**: The `PAUSE` statements address a timing issue where Profound Genie processes the FlexiCapture upload before it completes. This ensures proper sequencing.
- **FlexiCapture Integration**: The program relies on FlexiCapture to generate the `APINVUP` file, which is uploaded via a PC batch file (`APINVUP.BAT`).
- **Variable Substitution**: Variables like `?9?`, `?WS?`, `?USER?`, and `?20?` are dynamically replaced at runtime with system-specific values (e.g., library, workstation ID, user ID).
- **Printer Overrides**: The `OVRPRTF` commands redirect the `APLIST` report to different output queues (`APEDIT` or `TESTOUTQ`) based on the `?9?/G` condition, likely for testing or production environments.

---

### **Summary**

- **Process**: The OCL file checks for existing A/P records, clears and rebuilds necessary files, uploads a journal entry file from FlexiCapture, processes it through programs `AP107`, `AP106`, and `AP110`, and updates various A/P and G/L files. It includes pauses to ensure synchronization with external processes like Profound Genie.
- **External Programs**: `STRPCOCLP`, `AP107`, `AP106`, `AP110`, and the PC batch file `APINVUP.BAT`.
- **Files Used**: 16 database files, including `APINVUP`, `APFLEXH`, `APTRAN`, `APCONT`, `APVEND`, and others, for storing and processing A/P transactions, vendor data, checks, and G/L information.

Let me know if you need further clarification or details about specific sections!