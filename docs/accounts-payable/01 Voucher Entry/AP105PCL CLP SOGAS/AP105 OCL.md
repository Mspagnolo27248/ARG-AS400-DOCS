Let’s analyze the provided System/36 OCL (Operation Control Language) program `AP105.ocl36.txt`, which is called by the CLP program `AP105PCL.clp` via the `STRS36PRC` command. This program is part of the A/P SOGAS ACH payment processing system and runs in a System/36 environment. Below, I’ll outline the process steps, business rules, tables used, and external programs called.

---

### **Process Steps**
The OCL program `AP105.ocl36` performs the following steps:

1. **Clear and Build Temporary Files**:
   - Checks if the file `?9?APTR?WS?` exists:
     - If it exists, clears the physical file using `CLRPFM ?9?APTR?WS?`.
   - If it does not exist, builds a new file `?9?APTR?WS?` with:
     - Type: Indexed (`I`)
     - Initial records: 500
     - Record length: 404 bytes
     - Reserved records: 2
     - Key length: 10 bytes
   - If the index file `?9?APTX?WS?` does not exist, builds an index for `?9?APTR?WS?` with:
     - Key length: 2 bytes
     - Alternate key at position 12, length 5
     - Alternate key at position 385, length 20

2. **Load and Run Program AP105**:
   - Loads the program `AP105`.
   - Declares the following files with shared access (`DISP-SHR`) and dynamic labeling based on the `?9?` parameter (likely the `&P$FGRP` from `AP105PCL.clp`):
     - `APSOGAS` labeled as `?9?APSOGAS`
     - `APSGACH` labeled as `?9?APSGACH`
     - `APTRAN` labeled as `?9?APTR?WS?` with an extension of 100 records
     - `APCONT` labeled as `?9?APCONT`
     - `APVEND` labeled as `?9?APVEND`
     - `GSTABL` labeled as `?9?GSTABL`
     - `APDATE` labeled as `?9?APDATE`
   - Executes the `AP105` program (`RUN`).

3. **Create and Copy to History Table**:
   - Sets local data:
     - At offset 400, stores the user ID (`?USER?`).
     - At offset 410, stores the workstation ID (`?WS?`).
   - Loads the program `AP105A`.
   - Declares the following files with shared access (`DISP-SHR`):
     - `APSOGAS` labeled as `?9?APSOGAS`
     - `APSOGSH` labeled as `?9?APSOGSH` (likely a history file)
   - Executes the `AP105A` program (`RUN`).

4. **Clear Local Data**:
   - Clears all local data (`LOCAL BLANK-*ALL`).

---

### **Business Rules**
The program enforces the following business rules:

1. **Dynamic File Labeling**:
   - File names are dynamically constructed using the `?9?` parameter (substituted with `&P$FGRP` from `AP105PCL.clp`), allowing the program to work with different file groups or environments.

2. **Temporary File Management**:
   - Ensures the temporary file `?9?APTR?WS?` is cleared if it exists or created if it does not, with a predefined structure (500 records, 404 bytes, 10-byte key).
   - Builds an index file `?9?APTX?WS?` for efficient access to `?9?APTR?WS?` if needed.

3. **File Sharing**:
   - All files are opened with shared access (`DISP-SHR`), allowing concurrent access by other processes or users.

4. **History Tracking**:
   - The program copies data from `APSOGAS` to a history file `APSOGSH` using `AP105A `_`, indicating that processed payment data is archived for historical or audit purposes.

5. **User and Workstation Tracking**:
   - Stores the user ID and workstation ID in local data (offsets 400 and 410) for auditing or logging purposes during the history copy process.

6. **File Extension**:
   - Extends the `APTRAN` file by 100 records, ensuring sufficient space for transaction data during processing.

---

### **Tables Used**
The program references the following files (tables):

1. **APSOGAS**:
   - Labeled as `?9?APSOGAS` (e.g., `&P$FGRP` + 'APSOGAS').
   - Contains A/P SOGAS ACH payment data (likely source data for voucher entry/editing).

2. **APSGACH**:
   - Labeled as `?9?APSGACH`.
   - Contains ACH-related data, likely for owner or vendor validation.

3. **APTRAN**:
   - Labeled as `?9?APTR?WS?`.
   - Temporary transaction file, cleared or built with 500 records, 404 bytes, and a 10-byte key. Extended by 100 records during processing.

4. **APTX**:
   - Labeled as `?9?APTX?WS?`.
   - Index file for `APTRAN`, with keys at positions 12 (5 bytes) and 385 (20 bytes).

5. **APCONT**:
   - Labeled as `?9?APCONT`.
   - Likely contains control or configuration data for A/P processing.

6. **APVEND**:
   - Labeled as `?9?APVEND`.
   - Vendor master file containing vendor details.

7. **GSTABL**:
   - Labeled as `?9?GSTABL`.
   - Likely a general system table containing configuration or reference data.

8. **APDATE**:
   - Labeled as `?9?APDATE`.
   - Likely contains date-related data for A/P processing (e.g., payment or due dates).

9. **APSOGSH**:
   - Labeled as `?9?APSOGSH`.
   - History file for archiving processed `APSOGAS` data.

---

### **External Programs Called**
The program invokes the following external programs:

1. **AP105**:
   - Loaded and executed with `LOAD AP105` and `RUN`.
   - Processes A/P voucher entry and editing for SOGAS ACH payment creation, using the files `APSOGAS`, `APSGACH`, `APTRAN`, `APCONT`, `APVEND`, `GSTABL`, and `APDATE`.

2. **AP105A**:
   - Loaded and executed with `LOAD AP105A` and `RUN`.
   - Copies data from `APSOGAS` to the history file `APSOGSH`, likely for audit or record-keeping purposes.

---

### **Summary**
- **Process Steps**: Clear or build temporary files (`APTRAN`, `APTX`), load and run `AP105` for voucher entry/editing, set user/workstation data, load and run `AP105A` to copy to history file, and clear local data.
- **Business Rules**: Dynamic file naming with `?9?` parameter, shared file access, temporary file management, history tracking, and user/workstation logging.
- **Tables Used**: `APSOGAS` (payment data), `APSGACH` (ACH data), `APTRAN` (temporary transactions), `APTX` (index for `APTRAN`), `APCONT` (control data), `APVEND` (vendor data), `GSTABL` (system table), `APDATE` (date data), `APSOGSH` (history file).
- **External Programs Called**: `AP105` (voucher entry/editing), `AP105A` (history copy).

This OCL program manages the file setup and execution of A/P voucher processing and archiving for SOGAS ACH payments in a System/36 environment, integrating with the IBM i system via dynamic file naming.