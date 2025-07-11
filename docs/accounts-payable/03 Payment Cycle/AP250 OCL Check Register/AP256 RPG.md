The RPG program `AP256.RPG36` is invoked by the OCL procedure `AP256.OCL36`, which is called from the main OCL as part of the Accounts Payable (A/P) Check Register process. This program generates vendor payment detail reports for ACH (Automated Clearing House) vendors, with output spoolflexed to up to four email addresses per vendor. The preprocessing program `AP256A.RPG` creates the `APDTWSC` file, which provides the count of email addresses per vendor. Below is a detailed explanation of the **process steps**, **business rules**, **tables/files used**, and **external programs called** for `AP256.RPG36`, incorporating the context of `AP256A.RPG`.

---

### Process Steps

The `AP256.RPG36` program processes payment records from the `APDTWS` file, retrieves vendor and company information, and generates up to four payment detail reports (`REPORT1` to `REPORT4`) for emailing to ACH vendors. The processing is structured around three subroutines: `ONETIM`, `APVEND`, and `TRANS`, with calculations and output handled at level breaks. The steps are as follows:

1. **Initialization (ONETIM Subroutine)**:
   - Executes once at the start of the program (`N98` condition ensures it runs only if indicator `98` is off).
   - Sets indicator `98` to prevent re-execution.
   - Initializes a zero field (`ZERO9`) to 0 (not used elsewhere in the provided code).

2. **Company-Level Processing (Level 2 Break - L2)**:
   - Executes for each company (`ADCO`, level break `L2`).
   - Chains to the `APCONT` file using the company number (`ADCO`) to retrieve the company name (`ACNAME`).
     - Sets indicator `95` if the record is not found.
   - Clears indicators `61`, `62`, `63`, and `64` (not used elsewhere in the provided code).

3. **Vendor-Level Processing (APVEND Subroutine)**:
   - Executes at level 1 break (`L1`) for each vendor (`ADVEND`).
   - Chains to the `APVEND` file using the company/vendor key (`ADCOVN`, positions 2–8 from `APDTWS`) to retrieve vendor details (e.g., `VNNAME`, `VNADD1`–`VNADD4`).
     - Sets indicator `99` if the vendor record is not found.
   - Chains to the `APDTWSC` file (populated by `AP256A.RPG`) using `ADCOVN` to retrieve the email account count (`ACECNT`, positions 9–10).
     - Sets indicator `98` if the record is not found.
   - Based on the email account count (`ACECNT`):
     - If `ACECNT = 4`, sets indicators `50`, `51`, `52`, `53` (all four reports: `REPORT1`–`REPORT4`).
     - If `ACECNT = 3`, sets indicators `50`, `51`, `52` (three reports).
     - If `ACECNT = 2`, sets indicators `50`, `51` (two reports).
     - If `ACECNT = 1`, sets indicator `50` (one report).
   - Checks the vendor category (`VNCATG`) for `'CRDACT'` (crude account):
     - If true, sets indicator `60` to use crude-specific messages (`MSG,4`–`MSG,6`); otherwise, uses standard messages (`MSG,1`–`MSG,3`).
   - Initializes vendor-level accumulators (`L1INV$`, `L1DIS$`, `L1PYM$`) to zero at each `L1` break.

4. **Process Payment Transactions (TRANS Subroutine)**:
   - Executes for each `APDTWS` record (indicator `01`).
   - Calculates payment amounts:
     - Adds invoice amount (`ADINV$`, positions 54–59, packed) to total printed amount (`TOTPRT`) and level 1 invoice total (`L1INV$`).
     - Calculates payment amount (`ADPYM$`) as `ADINV$` minus discount (`ADDISC`, positions 60–64, packed).
     - Adds discount (`ADDISC`) to level 1 discount total (`L1DIS$`).
     - Adds payment amount (`ADPYM$`) to level 1 payment total (`L1PYM$`).
   - Retrieves email addresses from `APVNFMX`:
     - Constructs a key (`KEY20`, 20 bytes) using `ADCOVN` (company/vendor) and `'ACHE'` (form type).
     - Sets the lower limit (`SETLL`) on `APVNFMX` using `KEY20`.
     - Reads `APVNFMX` records sequentially, looping until end-of-file (`10`) or all email addresses are retrieved:
       - Skips records with mismatched company/vendor (`ADCOVN ≠ AMCOVN`).
       - Skips deleted records (`AMDEL = 'D'`).
       - Skips records not marked for original sending (`AMFMYN ≠ 'Y'`).
       - Skips records with incorrect form type (`AMFMTY ≠ 'ACHE'`).
       - Assigns email addresses (`AMEMLA`, positions 72–131) to `EMALP1`–`EMALP4` based on a counter (`COUNT`):
         - `COUNT = 1`: `EMALP1`
         - `COUNT = 2`: `EMALP2`
         - `COUNT = 3`: `EMALP3`
         - `COUNT = 4`: `EMALP4` (stops after fourth email).

5. **Level 1 Calculations (L1CALC Subroutine)**:
   - Executes at level 1 break (`L1`) for each vendor.
   - Triggers an exception output (`EXCPT`) to write report data to `REPORT1`–`REPORT4` based on indicators `50`–`53`.

6. **Generate Vendor Payment Reports (REPORT1–REPORT4)**:
   - Outputs up to four reports (`REPORT1`, `REPORT2`, `REPORT3`, `REPORT4`) based on indicators `50`, `51`, `52`, `53`, respectively, sent to output queues (`QUSRSYS/APACHOUTQ` or `QUSRSYS/TESTOUTQ`) for spoolflex emailing.
   - Each report includes:
     - **Headers** (level 1 break, `L1`, indicators `50`–`53`):
       - Date (`UDATE`, formatted as `Y`, position 99).
       - Company number (`ADCO`, position 9) and vendor number (`ADVEND`, position 15).
       - Vendor name (`VNNAME`, position 35) and address (`VNADD1`–`VNADD4`, positions 35).
       - Check date (`KYCKDTY`, position 73, from User Data Structure, positions 434–439).
       - Messages (`MSG,1`–`MSG,3` for normal vendors, `MSG,4`–`MSG,6` for crude vendors, position 89).
       - Email address (`EMALP1`–`EMALP4`, position 62, specific to each report).
     - **Detail Lines** (indicator `01`, indicators `50`–`53`, lines 31–33):
       - Check date (`KYCKDTY`, position 16), invoice number (`ADINVN`, position 41).
       - Invoice amount (`ADINV$M`, position 58), discount (`ADDISCM`, position 80), payment amount (`ADPYM$M`, position 116).
     - **Totals** (level 1 break, `L1`, indicators `50`–`53`, line 3):
       - Total invoice amount (`L1INV$M`, position 58), total discount (`L1DIS$M`, position 80), total payment (`L1PYM$M`, position 116).
     - **Continued Message** (if line count exceeds 62):
       - Outputs `'CONTINUED ON NEXT PAGE'` (position 22).

---

### Business Rules

The program enforces the following business rules:

1. **Vendor-Specific Reporting**:
   - Processes payment records for ACH vendors only, as indicated by the `APDTWS` file and `APVNFMX` form type (`AMFMTY = 'ACHE'`).
   - Generates up to four reports per vendor, based on the email account count (`ACECNT`) from `APDTWSC` (populated by `AP256A.RPG`).

2. **Email Address Retrieval**:
   - Retrieves only valid email addresses from `APVNFMX` where:
     - Company/vendor key matches (`ADCOVN = AMCOVN`).
     - Form type is `'ACHE'`.
     - Record is not deleted (`AMDEL ≠ 'D'`).
     - Record is marked for original sending (`AMFMYN = 'Y'`).
   - Limits to a maximum of four email addresses per vendor (`EMALP1`–`EMALP4`).

3. **Crude vs. Normal Vendors**:
   - Identifies crude vendors by checking `VNCATG = 'CRDACT'` in `APVEND`.
   - Uses distinct message sets:
     - Normal vendors (`N60`): `MSG,1`–`MSG,3` (e.g., "THE INVOICES LISTED BELOW WERE PAID BY ARG THROUGH ACH ON MM/DD/YY.").
     - Crude vendors (`60`): `MSG,4`–`MSG,6` (e.g., "THIS EMAIL IS NOTIFICATION THAT A DEPOSIT WILL BE MADE INTO YOUR ACCOUNT." with a reference to `crudestatements@amref.com`).
   - Added on 5-01-24 by Marty Greenberg to differentiate messaging.

4. **Payment Calculations**:
   - Calculates payment amount (`ADPYM$`) as invoice amount (`ADINV$`) minus discount (`ADDISC`).
   - Accumulates totals at the vendor level (`L1INV$`, `L1DIS$`, `L1PYM$`) for reporting.

5. **Report Output**:
   - Generates up to four reports per vendor, each sent to a different email address (`EMALP1`–`EMALP4`) via spoolflex.
   - Includes vendor details (name, address), payment details (invoice, amounts), and totals.
   - Handles pagination by including a continuation message if the report exceeds 62 lines.
   - Added support for four email addresses on 5-13-19 by Marty Greenberg (previously two).

6. **Error Handling**:
   - Skips invalid `APVNFMX` records (deleted, non-ACH, or non-original).
   - Proceeds without error logging if vendor (`APVEND`), control (`APDTWSC`), or company (`APCONT`) records are not found, relying on indicators (`95`, `98`, `99`).

7. **Preprocessing Dependency**:
   - Relies on `AP256A.RPG` to populate `APDTWSC` with the email account count (`ACECNT`) for each vendor, ensuring the correct number of reports is generated.

---

### Tables/Files Used

The program interacts with the following files, as defined in the OCL and RPG source:



# AP256 File Summary

| **Logical Name** | **Label**         | **Usage**                                                                 | **Disposition** | **Record Length** |
|------------------|-------------------|---------------------------------------------------------------------------|-----------------|-------------------|
| APDTWS           | ?9?APDT?WS?       | Input: ACH payment data (company, vendor, invoice, amounts, check date).  | Input (IP)      | 256               |
| APDTWSC          | ?9?APDT?WS?C      | Input: ACH vendor control (email account count, populated by AP256A).     | Input (IF)      | 10                |
| APVEND           | ?9?APVEND         | Input: Vendor master (name, address, category, ACH details).              | Input (IF)      | 579               |
| APVNFMX          | ?9?APVNFMX        | Input: Vendor email/fax details (email addresses, form type).             | Input (IF)      | 266               |
| APCONT           | ?9?APCONT         | Input: A/P control (company name).                                        | Input (IF)      | 256               |
| REPORT1          | N/A               | Output: Payment detail report for first email address.                    | Output (O)      | 132               |
| REPORT2          | N/A               | Output: Payment detail report for second email address.                   | Output (O)      | 132               |
| REPORT3          | N/A               | Output: Payment detail report for third email address.                    | Output (O)      | 132               |
| REPORT4          | N/A               | Output: Payment detail report for fourth email address.                   | Output (O)      | 132               |

## Key Fields
- **APDTWS**:
  - `ADCO` (company number, positions 2–3, level 2 break)
  - `ADVEND` (vendor number, positions 4–8, level 1 break)
  - `ADCOVN` (company/vendor key, positions 2–8)
  - `ADINVN` (invoice number, positions 9–28)
  - `ADINV$` (invoice amount, packed, positions 54–59)
  - `ADDISC` (discount, packed, positions 60–64)
  - `ADLPAM` (payment amount, packed, positions 65–70)
  - `ADDATE` (date, positions 71–76)
  - `ADVCH#` (check number, positions 77–81)
- **APDTWSC**:
  - `ACCO` (company number, positions 2–3)
  - `ACVEND` (vendor number, positions 4–8)
  - `ACECNT` (email account count, positions 9–10)
- **APVEND**:
  - `VNCO` (company number, positions 2–3)
  - `VNVEND` (vendor number, positions 4–8)
  - `VNNAME` (vendor name, positions 9–38)
  - `VNADD1`–`VNADD4` (address lines, positions 39–158)
  - `VNCATG` (vendor category, positions 495–500)
- **APVNFMX**:
  - `AMCONO` (company number, positions 2–3)
  - `AMCVEN` (vendor number, positions 4–8)
  - `AMCOVN` (company/vendor key, positions 2–8)
  - `AMFMTY` (form type, positions 9–12, must be `'ACHE'`)
  - `AMEMLA` (email address, positions 72–131)
  - `AMFMYN` (send original flag, position 152)
- **APCONT**:
  - `ACCONO` (company number, positions 2–3)
  - `ACNAME` (company name, positions 4–33)
- **User Data Structure (UDS)**:
  - `KYCKDT` (check date, positions 434–439)



---

### External Programs Called

The `AP256.RPG36` program does not explicitly call any external programs within its code. However, it relies on the preprocessing program `AP256A.RPG`, which is executed by the OCL procedure `AP256A` to populate the `APDTWSC` file with the email account count (`ACECNT`) for each vendor. The `AP256A.RPG` program is critical for ensuring that `AP256.RPG36` generates the correct number of reports based on the number of email addresses.

---

### Notes
- **OCL Context**:
  - The OCL procedure `AP256A` creates the `APDTWSC` file with a capacity of 50 records and a record length of 10 bytes, using the `BLDFILE` command.
  - The OCL procedure `AP256` loads `AP256.RPG36` and configures output queues (`QUSRSYS/APACHOUTQ` or `QUSRSYS/TESTOUTQ`) for the report files (`REPORT1`–`REPORT4`) to support spoolflex emailing.
- **Modifications**:
  - **5-13-19 (Marty Greenberg)**: Increased email accounts from two to four, adding support for `REPORT3` and `REPORT4`.
  - **5-01-24 (Marty Greenberg)**: Updated messages to differentiate between crude (`CRDACT`) and normal vendors, with specific email instructions for crude vendors (e.g., referencing `crudestatements@amref.com`).
- **Report Output**:
  - Reports are formatted for spoolflex emailing, with each report (`REPORT1`–`REPORT4`) sent to a different email address (`EMALP1`–`EMALP4`).
  - Includes vendor details, payment details (invoice number, amounts, discounts), and totals, with pagination handling for reports exceeding 62 lines.
- **Preprocessing Dependency**:
  - The `APDTWSC` file, populated by `AP256A.RPG`, provides the email account count (`ACECNT`), which determines how many reports (`REPORT1`–`REPORT4`) are generated for each vendor.
- **Error Handling**:
  - Skips invalid `APVNFMX` records (deleted, non-ACH, or non-original) and continues processing without logging errors.
  - Handles missing `APVEND`, `APDTWSC`, or `APCONT` records gracefully using indicators (`95`, `98`, `99`).
- **Business Context**:
  - Designed for ACH vendors, ensuring payment notifications are emailed efficiently with tailored messaging for crude vendors.
  - Supports up to four email addresses per vendor, reflecting modernized ACH notification processes.

This summary provides a comprehensive overview of the `AP256.RPG36` program's functionality, business logic, file interactions, and dependency on `AP256A.RPG`. Let me know if you need further clarification or additional analysis!