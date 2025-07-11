### **Use Cases Implemented by the AP150-AP156 Call Stack**

The call stack, consisting of `AP150.ocl36.txt`, `AP151.rpg36.txt`, `AP155.rpg36.txt`, and `AP156.ocl36.txt` with `AP156.rpg36.txt`, implements a single primary use case in the Accounts Payable (A/P) system:

1. **Generate and Process Accounts Payable Payments**:
   - This use case encompasses selecting vouchers for payment based on criteria (e.g., due date, payment method), creating payment records, generating a cash requirements report, and producing a NACHA file for ACH payments. The process handles multiple payment methods (checks, ACH, wire transfers, employee expenses, utility auto-pay) and tracks discounts, ensuring accurate payment processing and reporting.

---

### **Function Requirement Document: Generate and Process Accounts Payable Payments**

#### **Function Overview**
The `GenerateAndProcessAPPayments` function automates the selection, processing, reporting, and transmission of Accounts Payable payments. It takes input parameters defining payment criteria and produces payment records, a cash requirements report, and a NACHA file for ACH payments. The function supports checks, ACH, wire transfers, employee expenses, and utility auto-pay, while handling discounts and validations.

#### **Inputs**
- **Company Number (PTCONO)**: Identifies the company for payment processing (7 digits).
- **Bank G/L Number (PTBKGL)**: Bank general ledger number for payment (8 digits).
- **Next Check Number (PTNXCK)**: Starting check number for non-prepaid payments (6 digits).
- **Check Date (PTCKDT)**: Date of payment issuance (6 digits, YYMMDD).
- **Pay-By Date (PTDATE)**: Cutoff date for selecting vouchers (6 digits, YYMMDD).
- **Force Discount Flag (PTFDIS)**: `'D'` to force discounts, otherwise blank.
- **Payment Method (PTHOLD)**: `' '` (check), `A` (ACH), `W` (wire transfer), `E` (employee expense), `U` (utility auto-pay).
- **Vendor Number (PTVEND)**: Optional vendor number for specific vendor payments (5 digits).
- **Voucher Number (PTVO)**: Optional voucher number for specific voucher payments (5 digits).
- **Partial Payment Amount (PTAMT)**: Amount for partial payments (7.2 digits, packed).
- **Discount Amount (PTDISC)**: Specific discount to apply (5.2 digits, packed).
- **Pay or Hold (PTPORH)**: `'P'` to pay held vouchers, `'H'` to hold, otherwise blank.
- **Single Check Flag (PTSNGL)**: `'S'` for single check per voucher, otherwise blank.
- **Make Prepaid Flag (PTMKPP)**: `'P'` to mark as prepaid, otherwise blank.
- **Prepaid Check Number (PTPPCK)**: Check number for prepaid vouchers (6 digits).
- **Prepaid Date (PTPPDT)**: Date for prepaid vouchers (6 digits, YYMMDD).
- **Period (PTPD)**: Accounting period (2 digits).
- **Year (PTPDYY)**: Accounting year (2 digits).

#### **Outputs**
- **Payment Records (APPAY)**: Records with payment details (company, vendor, voucher, amount, discount, check number, etc.).
- **Missed Discount Records (APPYDS)**: Records for vouchers with missed discounts.
- **Check Records (APPYCK)**: Check details (check number, amount, status, etc.).
- **Invoice Detail Records (APDETINV)**: Aggregated invoice details for reporting.
- **Cash Requirements Report (APCSHRQ)**: Report detailing payments, discounts, and totals.
- **NACHA File (ACHFILE)**: NACHA-formatted file for ACH payments (if applicable).

#### **Process Steps**
1. **Validate Inputs**:
   - Ensure company number, bank G/L, and payment method are valid.
   - Validate dates (PTCKDT, PTDATE, PTPPDT) for Y2K compliance (convert to 8-digit format, e.g., CCYYMMDD).

2. **Select Vouchers (AP151)**:
   - **Pay by Date**:
     - Select `APOPEN` vouchers where:
       - Company (`OPCONO`) matches `PTCONO`.
       - Bank G/L (`OPBKGL`) matches `PTBKGL`.
       - Payment method (`OPHALT`) matches `PTHOLD` (`' '`, `A`, `W`, `E`, `U`).
       - Not deleted (`OPDEL ≠ 'D'`) or on hold (`OPHALT ≠ 'H'` unless `PTPORH = 'P'`).
       - Due date (`OPDUED`) is on or before `PTDATE` (unless `PTFDIS = 'D'`).
     - For prepaid vouchers (`OPPAID = 'P', 'A', 'W', 'E', 'U'`), ensure payment method matches `PTHOLD`.
   - **Pay by Vendor/Voucher**:
     - Select `APOPEN` vouchers matching `PTVEND` and optionally `PTVO`.
     - Apply same company, bank G/L, and payment method checks.
     - Handle partial payments (`PTAMT`) and specific discounts (`PTDISC`).

3. **Calculate Payments and Discounts**:
   - **Payment Amount**: `OPLPAM = OPGRAM - OPDISC - OPPPTD`.
     - `OPGRAM`: Gross voucher amount.
     - `OPDISC`: Discount (if applicable).
     - `OPPPTD`: Partial paid to date.
   - **Discount Logic**:
     - If `PTFDIS = 'D'`, apply `OPDISC` regardless of discount due date.
     - Otherwise, apply `OPDISC` only if discount due date (`OPDSDT`) is on or after `PTCKDT` and before or on `PTDATE`.
     - If discount is missed and `PTFDIS ≠ 'D'`, set `OPDISC = 0` and write to `APPYDS`.
   - For partial payments, set `OPLPAM = PTAMT` and adjust `OPDISC` to zero if `PTAMT = OPLPAM`.

4. **Create Payment Records**:
   - Write to `APPAY` with fields: `OPDEL` (delete flag), `OPLPAM`, `OPDISC`, `OPCKNO` (check number, `PTNXCK` for non-prepaid, `PTPPCK` for prepaid), `OPPAID` (payment method), `OPSNGL` (`'S'` for single check or one-time vendor), `OPCKDT` (check date), `PTSEQ#` (sequence).
   - For held vouchers (`PTPORH = 'H'`), mark `APPAY` record for deletion (`PYDEL = 'D'`).
   - Write missed discount records to `APPYDS`.

5. **Generate Cash Requirements Report (AP155)**:
   - Aggregate payment totals by company and check:
     - Computer checks (`C6CNT`, `C6GRAM`, `C6DISC`, `C6LPAM`).
     - Prepaid payments (`P6CNT`, `P6GRAM`, `P6DISC`, `P6LPAM`).
     - Total checks (`L6CNT`, `L6GRAM`, `L6DISC`, `L6LPAM`).
   - Update `APDETINV` with aggregated invoice amounts (`APGRAM`, `APDISC`) for same invoice numbers.
   - Validate checks against `APCHKR`:
     - Non-void checks must not exist or be open.
     - Void checks must exist, be open, and fully voided.
   - Write check records to `APPYCK` with status (`'F'` for full stub, `'V'` for void, `'C'` for credit/no pay).
   - Output report to `APCSHRQ` with invoice details, check totals, and error messages (e.g., "CHECK IS ALREADY OPEN").

6. **Create NACHA File for ACH Payments (AP156)**:
   - If `PTHOLD = 'A'` (indicated by LDA position 400 = `'A'`):
     - Clear `ACHFILE`.
     - Write NACHA records for `APPYCK` records with `PYSTAT = 'A'`:
       - **File Header (Type 1)**: ABA numbers, transmission date/time, company names.
       - **Batch Header (Type 5)**: Company details, effective date (`CKYMD`).
       - **Entry Detail (Type 6)**: Vendor bank routing (`VNARTE`), account number (`VNABK#`), amount (`PYCKAM`), transaction code (`22` for checking, `32` for savings).
       - **Batch Control (Type 8)**: Batch entry count, hash, and credit totals.
       - **File Control (Type 9)**: File-level counts and totals.
       - Filler records to pad blocks to multiples of 10.
     - Output report to `REPORT` for logging.

7. **Return Outputs**:
   - Return updated files (`APPAY`, `APPYDS`, `APPYCK`, `APDETINV`), report (`APCSHRQ`), and NACHA file (`ACHFILE`).

#### **Business Rules**
1. **Payment Selection**:
   - Vouchers must match company, bank G/L, and payment method.
   - Held vouchers (`OPHALT = 'H'`) require `PTPORH = 'P'` to be paid.
   - Prepaid vouchers (`OPPAID = 'P', 'A', 'W', 'E', 'U'`) must match `PTHOLD`.

2. **Discount Handling**:
   - Discounts applied if `PTFDIS = 'D'` or discount due date is valid.
   - Missed discounts (past due, no force discount) are recorded in `APPYDS`.

3. **Check Number Assignment**:
   - Non-prepaid payments use `PTNXCK`, incremented per check.
   - Prepaid payments use `PTPPCK` and `PTPPDT`.

4. **Single Check and One-Time Vendors**:
   - One-time vendors (`OPVEND = 0`) or `PTSNGL = 'S'` require single checks (`OPSNGL = 'S'`).

5. **Check Validation**:
   - Non-void checks must not exist or be open in `APCHKR`.
   - Void checks must exist, be open, and fully voided.
   - Zero/negative amounts are marked "CREDIT / NO PAY".

6. **Stub Limits**:
   - Maximum 36 invoices per check to prevent stub overflow.

7. **ACH Processing**:
   - Only executed if `PTHOLD = 'A'`.
   - Requires valid vendor ACH details (`VNARTE`, `VNABK#`, `VNACOS`).
   - NACHA file adheres to standard format with proper record types and block padding.

8. **Invoice Aggregation**:
   - Multiple invoices with the same number for a vendor are aggregated in `APDETINV`.

#### **Calculations**
- **Payment Amount**: `OPLPAM = OPGRAM - OPDISC - OPPPTD`.
  - For partial payments: `OPLPAM = PTAMT`, `OPDISC = 0` if `PTAMT = OPLPAM`.
- **Discount**: `OPDISC = OPDISC` if valid (force discount or within date range), else `0`.
- **Check Totals**: Sum `OPGRAM`, `OPDISC`, `OPLPAM` per check and company.
- **NACHA Hash**: Sum of vendor routing numbers (`VNARTE`) for batch (`L2HASH`) and file (`LRHASH`).
- **NACHA Block Count**: `LRBLOK = CEIL(RECCNT / 10)`.

#### **Assumptions**
- Input data is pre-validated and sorted (e.g., `APPYTR` sorted by `#GSORT` in `AP150`).
- Files (`APOPEN`, `APVEND`, `APCONT`, `APCHKR`) are populated with valid data.
- ACH payments require vendor ACH details in `APVEND`.

#### **Error Handling**
- Skips invalid records (deleted, mismatched company/bank G/L, or payment method).
- Reports errors in `APCSHRQ` (e.g., check already open, invalid void).
- Logs missed discounts in `APPYDS`.



### **Tables Used in the AP150-AP156 Call Stack and Data Flow**

Below is a detailed explanation of the tables (files) used in the Accounts Payable (A/P) payment processing call stack (`AP150.ocl36.txt`, `AP151.rpg36.txt`, `AP155.rpg36.txt`, `AP156.ocl36.txt`, `AP156.rpg36.txt`), their purposes, and a summary of the data flow.

---

### **Tables (Files) Used and Their Purpose**



## Tables Used in the AP150-AP156 Call Stack

The following tables are used across the AP150-AP156 call stack to process Accounts Payable payments, generate reports, and create NACHA files for ACH payments. Each table's purpose, key fields, and usage in the programs are described below.

### 1. APPYTR (Payment Transaction File)
- **Purpose**: Stores sorted payment transaction data used as input for payment selection criteria. It contains header records (pay by date) and detail records (pay by vendor/voucher).
- **File Usage**:
  - **AP151**: Primary input file, read to determine which vouchers to select from `APOPEN` based on company, vendor, voucher, payment method, and dates.
  - **AP155**: Chained to retrieve next check number, check date, pay-by date, and payment method for the cash requirements report.
- **Key Fields**:
  - `PTCONO` (Company Number, 7 digits)
  - `PTVEND` (Vendor Number, 5 digits)
  - `PTVO` (Voucher Number, 5 digits)
  - `PTAMT` (Partial Payment Amount, 7.2 digits, packed)
  - `PTDISC` (Discount Amount, 5.2 digits, packed)
  - `PTBKGL` (Bank G/L Number, 8 digits)
  - `PTNXCK` (Next Check Number, 6 digits)
  - `PTCKDT` (Check Date, 6 digits, YYMMDD)
  - `PTDATE` (Pay-By Date, 6 digits, YYMMDD)
  - `PTFDIS` (Force Discount, `'D'` or blank)
  - `PTHOLD` (Payment Method, `' '`, `A`, `W`, `E`, `U`)
  - `PTPORH` (Pay or Hold, `'P'`, `'H'`, or blank)
  - `PTMKPP` (Make Prepaid, `'P'` or blank)
  - `PTSEQ#` (Sequence Number)
- **Record Length**: 128 bytes
- **Access**: Input Primary (AP151), Input with Chain (AP155)

### 2. APOPEN (Open A/P File)
- **Purpose**: Contains open voucher details used to identify eligible vouchers for payment based on selection criteria.
- **File Usage**:
  - **AP151**: Chained to select vouchers matching company, bank G/L, payment method, and due date criteria.
  - **AP155**: Chained to retrieve vendor name and sort abbreviation for reporting if not found in `APVEND`.
- **Key Fields**:
  - `OPCONO` (Company Number, 7 digits)
  - `OPVEND` (Vendor Number, 5 digits)
  - `OPVONO` (Voucher Number, 5 digits)
  - `OPGRAM` (Gross Amount, 7.2 digits, packed)
  - `OPDISC` (Discount Amount, 5.2 digits, packed)
  - `OPPPTD` (Partial Paid to Date, 5.2 digits, packed)
  - `OPINVN` (Invoice Number, 20 bytes)
  - `OPINDS` (Invoice Description, 25 bytes)
  - `OPDSDT` (Discount Due Date, 6 digits, YYMMDD)
  - `OPDUED` (Due Date, 6 digits, YYMMDD)
  - `OPHALT` (Hold Code, `'H'` or payment method)
  - `OPPAID` (Prepaid Code, `'P'`, `'A'`, `'W'`, `'E'`, `'U'`)
  - `OPCKNO` (Prepaid Check Number, 6 digits)
  - `OPSNGL` (Single Check, `'S'` or blank)
  - `OPBKGL` (Bank G/L Number, 8 digits)
- **Record Length**: 384 bytes
- **Access**: Input with Disk (AP151), Input with Chain (AP155)

### 3. APPAY (Payment File)
- **Purpose**: Stores generated payment records, including payment amounts, discounts, and check details for processed vouchers.
- **File Usage**:
  - **AP151**: Output file where payment records are written or updated with calculated payment amounts and check details.
  - **AP155**: Primary input file, read to generate the cash requirements report; updated with sequence numbers.
- **Key Fields**:
  - `OPDEL` (Delete Flag, `'D'` or blank)
  - `OPCONO` (Company Number, 7 digits)
  - `OPVEND` (Vendor Number, 5 digits)
  - `OPVONO` (Voucher Number, 5 digits)
  - `OPGRAM` (Gross Amount, 7.2 digits, packed)
  - `OPDISC` (Discount Amount, 5.2 digits, packed)
  - `OPPPTD` (Partial Paid to Date, 5.2 digits, packed)
  - `OPLPAM` (Payment Amount, 6.2 digits, packed)
  - `OPPAID` (Payment Method, `'P'`, `'A'`, `'W'`, `'E'`, `'U'`)
  - `OPSNGL` (Single Check, `'S'` or blank)
  - `OPCKNO` (Check Number, 6 digits)
  - `OPCKDT` (Check Date, 6 digits, YYMMDD)
  - `OPSEQ#` (Sequence Number, 5 digits)
  - `OPINVN` (Invoice Number, 20 bytes)
  - `OPINDS` (Invoice Description, 25 bytes)
  - `OPDUED` (Due Date, 6 digits, YYMMDD)
- **Record Length**: 384 bytes
- **Access**: Update/Create (AP151), Update Primary (AP155)

### 4. APPYDS (Missed Discount File)
- **Purpose**: Tracks vouchers where discounts were available but not taken due to missed discount due dates.
- **File Usage**:
  - **AP151**: Output file where missed discount records are written.
  - **AP155**: Chained to check for missed discounts and annotate the cash requirements report ("DISCOUNT NOT TAKEN").
- **Key Fields**:
  - `DSDEL` (Delete Flag, `'D'` or blank)
  - `DSCONO` (Company Number, 7 digits)
  - `DSVEND` (Vendor Number, 5 digits)
  - `DSVONO` (Voucher Number, 5 digits)
  - `DSGRAM` (Gross Amount, 7.2 digits, packed)
  - `DSDISC` (Discount Amount, 5.2 digits, packed)
  - `DSPPTD` (Partial Paid to Date, 5.2 digits, packed)
  - `DSLPAM` (Last Payment Amount, 6.2 digits, packed)
  - `DSDSDT` (Discount Due Date, 6 digits, YYMMDD)
  - `DSDUED` (Due Date, 6 digits, YYMMDD)
  - `DSCKNO` (Check Number, 6 digits)
  - `DSPAID` (Payment Method, `'P'`, `'A'`, `'W'`, `'E'`, `'U'`)
  - `DSSNGL` (Single Check, `'S'` or blank)
  - `DSBKGL` (Bank G/L Number, 8 digits)
- **Record Length**: 384 bytes
- **Access**: Output (AP151), Input with File (AP155)

### 5. APPYCK (Check File)
- **Purpose**: Stores check details, including check number, amount, and status (e.g., full stub, void, credit/no pay).
- **File Usage**:
  - **AP155**: Output file where check records are written or updated with status and totals.
  - **AP156**: Primary input file, read to generate NACHA file for ACH payments.
- **Key Fields**:
  - `AXRECD` (Record Code, `' '`, `'F'`, `'V'`, `'C'`, `'P'`, `'A'`, `'W'`, `'E'`, `'U'`)
  - `PYCONO` (Company Number, 7 digits)
  - `PYVEND` (Vendor Number, 5 digits)
  - `PYBKGL` (Bank G/L Number, 8 digits)
  - `PYCHK#` (Check Number, 6 digits)
  - `PYCKAM` (Check Amount, 11.2 digits, packed)
  - `PYCKDT` (Check Date, 6 digits, YYMMDD)
  - `PYNAME` (Vendor Name, 22 bytes)
  - `PYSEQ#` (Sequence Number, 9 digits)
  - `PYCNTR` (Invoice Count, 9 digits)
- **Record Length**: 96 bytes
- **Access**: Update/Create (AP155), Input Primary (AP156)

### 6. APDETINV (Invoice Detail File)
- **Purpose**: Tracks aggregated invoice details for vendors, combining amounts for invoices with the same number.
- **File Usage**:
  - **AP155**: Updated with aggregated gross and discount amounts for reporting.
- **Key Fields**:
  - `APDEL` (Delete Flag, `'D'` or blank)
  - `APCONO` (Company Number, 7 digits)
  - `APVEND` (Vendor Number, 5 digits)
  - `APINVN` (Invoice Number, 20 bytes)
  - `APGRAM` (Gross Amount, 6.2 digits, packed)
  - `APDISC` (Discount Amount, 5.2 digits, packed)
  - `OPPPTD` (Partial Paid to Date, 5.2 digits, packed)
  - `OPINDS` (Invoice Description, 25 bytes)
  - `OPDUED` (Due Date, 6 digits, YYMMDD)
  - `OPVONO` (Voucher Number, 5 digits)
- **Record Length**: 256 bytes
- **Access**: Update with File (AP155)

### 7. APCONT (A/P Control File)
- **Purpose**: Stores company-level control data, including company name, bank G/L, and check numbering details.
- **File Usage**:
  - **AP155**: Chained to retrieve company name and pre-numbered check flag for the report.
  - **AP156**: Chained to retrieve company name and bank G/L for NACHA file headers.
- **Key Fields**:
  - `ACDEL` (Delete Flag, `'D'` or blank)
  - `ACCONO` (Company Number, 7 digits)
  - `ACNAME` (Company Name, 30 bytes)
  - `ACBKGL` (Bank G/L Number, 8 digits)
  - `ACPRE#` (Pre-Numbered Checks, `'Y'` or blank)
- **Record Length**: 256 bytes
- **Access**: Input with Chain (AP155, AP156)

### 8. APVEND (Vendor File)
- **Purpose**: Contains vendor details, including name, address, and ACH payment information.
- **File Usage**:
  - **AP155**: Chained to retrieve vendor name and sort abbreviation for the report.
  - **AP156**: Chained to retrieve ACH-specific fields (routing code, account number, account type) for NACHA file.
- **Key Fields**:
  - `VNDEL` (Delete Flag, `'D'` or blank)
  - `VNCO` (Company Number, 7 digits)
  - `VNVEND` (Vendor Number, 5 digits)
  - `VNNAME` (Vendor Name, 30 bytes)
  - `VNSORT` (Alpha Sort Abbreviation, 10 bytes)
  - `VNARTE` (ACH Bank Routing Code, 9 digits)
  - `VNABK#` (ACH Bank Account Number, 17 bytes)
  - `VNACOS` (ACH Account Type, `'C'` for checking, else savings)
- **Record Length**: 579 bytes
- **Access**: Input with Chain (AP155, AP156)

### 9. APCHKR (Check Register File)
- **Purpose**: Validates check status to ensure checks are not already open or incorrectly voided.
- **File Usage**:
  - **AP155**: Chained to validate check numbers and statuses for the report.
- **Key Fields**:
  - `AMCODE` (Status Code, `'D'`, `'O'`, `'R'`, `'V'`)
  - `AMCKAM` (Check Amount, 11.2 digits, packed)
- **Record Length**: 128 bytes
- **Access**: Input with Chain (AP155)

### 10. APCSHRQ (Cash Requirements Report File)
- **Purpose**: Printer file for outputting the cash requirements report with payment details, check totals, and company summaries.
- **File Usage**:
  - **AP155**: Output file for writing report headers, invoice details, check totals, and error messages.
- **Key Fields**:
  - Report fields include company name, vendor name, invoice number, gross amount, discount, payment amount, check number, and totals.
- **Record Length**: 142 bytes
- **Access**: Output (AP155)

### 11. ACHFILE (NACHA File)
- **Purpose**: Stores NACHA-formatted records for ACH payments to PNC Bank.
- **File Usage**:
  - **AP156**: Output file for writing file header, batch header, entry detail, batch control, and file control records.
- **Key Fields**:
  - Record Type Codes (`'1'`, `'5'`, `'6'`, `'8'`, `'9'`, filler)
  - File Header: ABA numbers, transmission date/time
  - Batch Header: Company name, tax ID, effective date
  - Entry Detail: Vendor routing code, account number, amount
  - Batch Control: Entry count, hash, credit total
  - File Control: Batch count, block count, entry count
- **Record Length**: 94 bytes
- **Access**: Output (AP156)

### 12. REPORT (Printer File)
- **Purpose**: Logs or verifies NACHA file creation details.
- **File Usage**:
  - **AP156**: Output file for logging (specific content not defined in code).
- **Record Length**: 132 bytes
- **Access**: Output (AP156)

### 13. AP155S (Report Sequencing File)
- **Purpose**: Control or sort file used for sequencing the cash requirements report.
- **File Usage**:
  - **AP155**: Input file for report sequencing.
- **Key Fields**: Not detailed in code (likely control flags or sort keys).
- **Record Length**: 3 bytes
- **Access**: Input Random (AP155)

---

## Data Flow Summary

The data flow through the AP150-AP156 call stack is a sequential process that transforms input transaction data into payment records, a report, and an ACH payment file. Below is a summary:

1. **Input Preparation (AP150.ocl36.txt)**:
   - **Input**: User-provided parameters (company, bank G/L, check date, etc.) and `APPYTR` (payment transactions).
   - **Process**: Sorts `APPYTR` by company, vendor, voucher, and sequence using `#GSORT`. Clears `APPAY`, `APPYDS`, and `APPYCK` files to prepare for new data.
   - **Output**: Sorted `APPYTR` file.

2. **Payment Record Creation (AP151.rpg36.txt)**:
   - **Input**: Sorted `APPYTR`, `APOPEN` (open vouchers).
   - **Process**: Reads `APPYTR` to select vouchers from `APOPEN` based on criteria (company, bank G/L, payment method, due date). Calculates payment amounts (`OPLPAM = OPGRAM - OPDISC - OPPPTD`) and applies discounts. Writes payment records to `APPAY` and missed discount records to `APPYDS`.
   - **Output**: Populated `APPAY` and `APPYDS` files.

3. **Cash Requirements Report Generation (AP155.rpg36.txt)**:
   - **Input**: `APPAY`, `APPYTR`, `APCONT`, `APVEND`, `APCHKR`, `APPYDS`, `AP155S`.
   - **Process**: Reads `APPAY` to aggregate payment totals (gross, discount, payment amount) by check and company. Updates `APDETINV` with aggregated invoice details. Validates checks against `APCHKR` and writes check records to `APPYCK`. Outputs report to `APCSHRQ` with invoice details, check totals, and error messages.
   - **Output**: Updated `APPAY`, `APDETINV`, `APPYCK`, and `APCSHRQ` report. Sets LDA position 400 to `'A'` if ACH payments are detected.

4. **NACHA File Creation (AP156.ocl36.txt, AP156.rpg36.txt)**:
   - **Input**: `APPYCK`, `APCONT`, `APVEND`, LDA (position 400 = `'A'`).
   - **Process**: If ACH payments are present, clears `ACHFILE` and reads `APPYCK` to generate NACHA records (file header, batch header, entry detail, batch control, file control). Uses `APVEND` for ACH details and `APCONT` for company data. Outputs to `ACHFILE` and logs to `REPORT`.
   - **Output**: Populated `ACHFILE` and `REPORT`.

**Overall Flow**:
- **Input**: `APPYTR` (transaction criteria), `APOPEN` (vouchers), `APCONT` (company data), `APVEND` (vendor data), `APCHKR` (check validation).
- **Transformation**: Sort transactions (`AP150`), select and process payments (`AP151`), generate report and validate checks (`AP155`), create NACHA file for ACH (`AP156`).
- **Output**: `APPAY` (payments), `APPYDS` (missed discounts), `APPYCK` (checks), `APDETINV` (invoices), `APCSHRQ` (report), `ACHFILE` (NACHA file), `REPORT` (log).

