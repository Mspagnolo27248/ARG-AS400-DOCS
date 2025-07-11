Let’s analyze the provided AS400 IBM i CLP (Control Language Program) `AP105PCL.clp` to extract the process steps, business rules, programs called, and tables used.

---

### **Process Steps**
The program performs the following sequential steps:

1. **Declare Variables**:
   - Defines a parameter `&P$FGRP` (1-character input parameter).
   - Declares character variables:
     - `&APSOGAS` (10 characters)
     - `&APSOGASH` (10 characters)
     - `&APSGACH` (10 characters)
     - `&APVEND` (10 characters)
     - `&PARM9` (10 characters)

2. **Construct File Names**:
   - Concatenates the input parameter `&P$FGRP` with fixed strings to create file names:
     - `&APSOGAS` = `&P$FGRP` + 'APSOGAS'
     - `&APSOGASH` = `&P$FGRP` + 'APSOGASH'
     - `&APSGACH` = `&P$FGRP` + 'APSGACH'
     - `&APVEND` = `&P$FGRP` + 'APVEND'
   - Constructs a parameter string `&PARM9` by concatenating ',,,,,,,,' with `&P$FGRP`.

3. **Override Database Files**:
   - Overrides the logical files to point to the physical files in the library list (`*LIBL`):
     - `APSOGAS` to `&APSOGAS` (first member)
     - `APSOGASH` to `&APSOGASH` (first member)
     - `APSGACH` to `&APSGACH` (first member)
     - `APVEND` to `&APVEND` (first member)

4. **Call Program**:
   - Calls the program `AP105P`, passing the parameter `&P$FGRP`.

5. **Check Local Data Area (LDA)**:
   - Checks if position 106 of the LDA (Local Data Area) contains 'Y'.
   - If true:
     - Clears the LDA (positions 1 to 512) to blanks.
     - Returns (exits the program).

6. **Start S/36 Procedure**:
   - If the LDA condition is not met, calls the S/36 procedure `AP105` with the parameter `&PARM9` (',,,,,,,,' + `&P$FGRP`).
   - Note: A commented-out line suggests a hardcoded parameter `',,,,,,,,G'` was previously used.

7. **Clean Up**:
   - Deletes all file overrides (`DLTOVR FILE(*ALL)`).
   - Clears the LDA (positions 1 to 512) to blanks.

8. **End Program**:
   - Ends the program with `ENDPGM`.

---

### **Business Rules**
The program includes the following business rules:

1. **Dynamic File Naming**:
   - File names are dynamically constructed by prefixing the input parameter `&P$FGRP` to fixed file suffixes (`APSOGAS`, `APSOGASH`, `APSGACH`, `APVEND`). This implies the program supports multiple file groups or environments based on the input parameter.

2. **Conditional Logic Based on LDA**:
   - If position 106 of the LDA contains 'Y', the program clears the LDA and exits without invoking the S/36 procedure. This suggests that `AP105P` may set this flag to indicate a condition (e.g., error or completion) that skips further processing.

3. **File Overrides**:
   - The program ensures that file operations use the correct files by overriding them to the dynamically named files in the library list, ensuring flexibility across different file groups or environments.

4. **S/36 Compatibility**:
   - The use of `STRS36PRC` indicates compatibility with System/36 procedures, suggesting this program may be part of a legacy system or migration to IBM i.

---

### **Programs Called**
The program invokes the following:

1. **AP105P**:
   - A program called via `CALL PGM(AP105P)` with the parameter `&P$FGRP`.
   - Likely an IBM i program that processes A/P SOGAS ACH payments, possibly setting the LDA flag at position 106.

2. **AP105**:
   - A System/36 procedure called via `STRS36PRC PRC(AP105)` with the parameter `&PARM9` (',,,,,,,,' + `&P$FGRP`).
   - This procedure likely performs additional processing for A/P SOGAS ACH payments in a System/36 environment.

---

### **Tables Used**
The program references the following files (tables):

1. **APSOGAS**:
   - Overridden to `&APSOGAS` (constructed as `&P$FGRP` + 'APSOGAS').
   - Likely contains A/P SOGAS ACH payment data.

2. **APSOGASH**:
   - Overridden to `&APSOGASH` (constructed as `&P$FGRP` + 'APSOGASH').
   - Likely a header file or summary data for A/P SOGAS ACH payments.

3. **APSGACH**:
   - Overridden to `&APSGACH` (constructed as `&P$FGRP` + 'APSGACH').
   - Likely contains ACH-specific data for A/P SOGAS payments.

4. **APVEND**:
   - Overridden to `&APVEND` (constructed as `&P$FGRP` + 'APVEND').
   - Likely a vendor master file containing vendor details for A/P payments.

These files are accessed from the library list (`*LIBL`) and use the first member (`*FIRST`).

---

### **Summary**
- **Process Steps**: Declare variables, construct file names, override files, call `AP105P`, check LDA, call S/36 procedure `AP105` (if LDA condition not met), clean up overrides, clear LDA, and end.
- **Business Rules**: Dynamic file naming based on `&P$FGRP`, conditional exit based on LDA position 106, file overrides for flexibility, and S/36 compatibility.
- **Programs Called**: `AP105P` (IBM i program), `AP105` (System/36 procedure).
- **Tables Used**: `APSOGAS`, `APSOGASH`, `APSGACH`, `APVEND`.

This program serves as a wrapper to check and process A/P SOGAS ACH payment files, bridging IBM i and System/36 environments.