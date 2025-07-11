The provided document is an Operation Control Language (OCL) program for an IBM System/3x or AS/400 system, specifically for managing customer master data (add/update) in an Accounts Receivable (AR) system. Below, I’ll explain the process steps of the OCL program, list the external programs called, and identify the tables (files) used.

---

### Process Steps of the RPG .OCL Program

OCL is used to define job control and file operations for RPG (Report Program Generator) programs on IBM midrange systems. The program `AR900.ocl36.txt` sets up the environment and files needed to execute the RPG program `AR900` and its associated subprograms. Here’s a step-by-step breakdown of the process:

1. **Program and Environment Setup**:
   - The OCL starts with comments and directives to configure the environment:
     - `SCPROCP ,,,,,,,,?9?`: Likely a system procedure call or parameter placeholder (`?9?` is a variable substituted at runtime).
     - `GSY2K`: Possibly a Year 2000 compliance flag or system identifier.
     - `LOCAL OFFSET-480,DATA-'?9?'`, `LOCAL OFFSET-410,DATA-'?WS?'`, `LOCAL OFFSET-400,DATA-'?USER?'`: These set local variables or parameters, such as a company code (`?9?`), workstation ID (`?WS?`), or user ID (`?USER?`), which are passed to the program at runtime.

2. **Index Build for Credit Details File**:
   - The `IFF` (If) statement checks if the program `AR300` is active and if the `CRDETX` file contains a specific data field (`?9?CRDETX`):
     - `BLDINDEX ?9?CRDETX,2,8,+ ?9?ARDETL,,DUPKEY,,71,8,10,7`: This builds an index for the `CRDETX` file (credit details) to optimize access. The parameters specify:
       - Starting position (2), length (8) for the key.
       - `?9?ARDETL` as the output file.
       - `DUPKEY` allows duplicate keys.
       - Additional key fields at positions 71 (length 8), 10 (length 7).

3. **Load the Main Program**:
   - `LOAD AR900`: Loads the RPG program `AR900`, which handles the core logic for adding or updating customer master records.

4. **File Definitions**:
   - The OCL defines multiple files (tables) used by `AR900` and its subprograms, all opened in shared mode (`DISP-SHR`):
     - `ARCUST`: Customer master file.
     - `ARCUSP`: Customer profile or secondary customer file.
     - `ARCUPR`: Customer pricing or rate file.
     - `CRDETX`: Credit details file.
     - `ARCONT`: AR control file (likely for configuration settings).
     - `BICONT`: Billing control file.
     - `GSPROD`: General system product file.
     - `GSTABL`: General system table file.
     - `BISLTX`: Billing sales tax file.
     - `ARCLGR`: AR customer ledger file.
     - `ARCUST2`: Alternate customer master file (same label as `ARCUST`).
     - `ARCUSHS`: Customer ship-to history file.
     - `ARCUSPH`: Customer phone history file.
     - `ARCUPHS`: Customer pricing history file.

5. **File Definitions for Called Programs**:
   - Additional files are defined for specific subprograms called by `AR900`:
     - **For `AR9009`** (likely a utility or data fix program):
       - `SA5FIXD`, `SA5FIXM`: Fixed data files.
       - `SA5BCXD`, `SA5BCXM`: Billing credit files.
       - `SA5DBXD`, `SA5DBXM`: Debit files.
       - `SA5COXD`, `SA5COXM`: Customer order files.
     - **For `AR9006`** (possibly a pricing update program):
       - `PFARCUPR`, `PTARCUPR`: Pricing files (same label as `ARCUPR` but with different prefixes).
     - **For `AR915P`** (possibly a customer file maintenance program):
       - `ARCUFM`, `ARCUFMX`: Customer file maintenance files.
     - **For `BI907`** (likely a billing or shipping program):
       - `SHIPTO`: Ship-to address file.
       - `ARCUP3`: Another customer pricing file.

6. **Execute the Program**:
   - `RUN`: Executes the `AR900` program, which processes customer master add/update operations using the defined files and calls subprograms as needed.

7. **Post-Processing**:
   - The OCL ends with `ARFX39 ,,,,,,,,?9?`, which might be a cleanup procedure or a call to another program for final processing (e.g., logging or file closing).

---

### External Programs Called

The OCL references the following external programs:
1. **AR900**: The main RPG program for customer master add/update.
2. **AR9009**: A utility program, possibly for data fixes or validations.
3. **AR9006**: A program likely related to customer pricing updates.
4. **AR915P**: A program for customer file maintenance.
5. **BI907**: A billing or shipping-related program.
6. **AR300**: Referenced in the `IFF` condition, possibly for checking system status.
7. **ARFX39**: Likely a cleanup or final processing program.

---

### Tables (Files) Used

The OCL defines the following files (tables) used by the program and its subprograms:
1. **ARCUST**: Customer master file.
2. **ARCUSP**: Customer profile file.
3. **ARCUPR**: Customer pricing file.
4. **CRDETX**: Credit details file.
5. **ARCONT**: AR control file.
6. **BICONT**: Billing control file.
7. **GSPROD**: General system product file.
8. **GSTABL**: General system table file.
9. **BISLTX**: Billing sales tax file.
10. **ARCLGR**: AR customer ledger file.
11. **ARCUST2**: Alternate customer master file.
12. **ARCUSHS**: Customer ship-to history file.
13. **ARCUSPH**: Customer phone history file.
14. **ARCUPHS**: Customer pricing history file.
15. **SA5FIXD**: Fixed data file (for `AR9009`).
16. **SA5FIXM**: Fixed master file (for `AR9009`).
17. **SA5BCXD**: Billing credit fixed data (for `AR9009`).
18. **SA5BCXM**: Billing credit fixed master file (for `AR9009`).
19. **SA5DBXD**: Debit fixed data file (for `AR9009`).
20. **SA5DBXM**: Debit master file (for `AR9009`).
21. **SA5COXD**: Customer order fixed data (for `AR9009`).
22. **SA5COXM**: Customer order master file (for `AR9009`).
23. **PFARCUPR**: Pricing file (for `AR9006`).
24. **PTARCUPR**: Pricing file (for `AR9006`).
25. **ARCUFM**: Customer file maintenance file (for `AR915P`).
26. **ARCUFMX**: Customer file maintenance index file (for `AR915P`).
27. **SHIPTO**: Ship-to address file (for `BI907`).
28. **ARCUP3**: Customer pricing file (for `BI907`).
29. **ARDETL**: Output file for index build (referenced in `BLDINDEX`).

---

### Summary

The `AR900.ocl36.txt` OCL program orchestrates the execution of the `AR900` RPG program to add or update customer master records in an Accounts Receivable system. It:
- Sets up the environment and parameters.
- Builds an index for the credit details file if conditions are met.
- Loads the main program and defines shared files for customer, billing, and system data.
- Calls subprograms (`AR9009`, `AR9006`, `AR915P`, `BI907`) for specific tasks like data fixes, pricing updates, file maintenance, and billing/shipping.
- Executes the program and performs post-processing.

The program interacts with 29 files (tables) and calls 7 external programs to manage customer-related data comprehensively.