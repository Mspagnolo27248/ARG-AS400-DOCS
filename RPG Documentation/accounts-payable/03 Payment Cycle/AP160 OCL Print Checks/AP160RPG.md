The RPG program `AP160.rpg36.txt` is designed to handle the business process of printing Accounts Payable (A/P) checks for vendor payments in an IBM midrange environment (e.g., AS/400 or IBM i). Below, I’ll explain the process steps and business rules from a business perspective, focusing on the purpose, functionality, and rules governing the check printing process, rather than the technical RPG details.

---

### Process Steps of the AP160 Program

The `AP160` program orchestrates the generation and printing of A/P checks, ensuring payments to vendors are accurately processed, formatted, and recorded. Here’s a step-by-step breakdown of the process from a business viewpoint:

1. **Retrieve Payment and Vendor Information**:
   - The program starts by accessing payment data from the payment file (`APPAY`), which contains details about invoices selected for payment, such as vendor numbers, invoice numbers, gross amounts, discounts, and payment amounts.
   - It cross-references this with:
     - The vendor master file (`APVEND`) for vendor details like name and address.
     - The open payables file (`APOPEN`) for additional invoice-related data.
     - The control file (`APCONT`) for company-specific information, such as the company name for check printing.
     - The transaction file (`APPYTR`) for check-related data, like the check date.
     - The check file (`APPYCK`) for check numbers and statuses.

2. **Validate Payment Type and Check Eligibility**:
   - For each payment record, the program checks the payment type (stored in the `AXRECD` field of `APPYCK`):
     - **Prepaid (`P`), ACH (`A`), Wire Transfer (`W`), Employee Expense (`E`), Credit/No Pay (`C`)**: These payments are not printed as physical checks. They are either already paid (e.g., prepaid checks) or processed via electronic methods (e.g., ACH, wire transfer), so the program skips printing.
     - **Full Stub/Void Check (`F`, `V`)**: These indicate voided checks, which are printed with a "VOID" label. For `F`, the same check number is reused for the next stub; for `V`, the next check number is used.
     - **Normal Payments**: Payments not flagged as `P`, `A`, `W`, `E`, `C`, `F`, or `V` are processed for check printing.
   - This step ensures only valid, non-electronic payments result in printed checks, aligning with payment method policies.

3. **Calculate Check Totals**:
   - For each invoice, the program aggregates:
     - **Gross Amount**: The total invoice amount before discounts.
     - **Discount Amount**: Any applicable vendor discounts (e.g., early payment discounts).
     - **Net Payment Amount**: The actual amount to be paid (gross minus discount).
   - These amounts are accumulated to calculate the total check amount (`CKAMT`), ensuring the check reflects all invoices paid to a vendor in a single transaction.

4. **Assign and Validate Check Numbers**:
   - The program retrieves a check number from the check file (`APPYCK`) using a sequence number (`SEQ#`).
   - If the check number is marked as void (`V`), the program increments the sequence number to assign a new check number, ensuring only valid check numbers are used for printing.

5. **Format Check Information**:
   - The program prepares the check for printing by pulling together:
     - **Company Information**: The company name from `APCONT` is printed at the top of the check (except for company code `01`, which skips this per a business rule).
     - **Vendor Information**: Vendor name and address (up to four address lines) from `APVEND` or `APOPEN` are formatted for the check payee section.
     - **Payment Details**: Invoice number, invoice date, description, gross amount, discount, and payment amount are included for each invoice.
     - **Check Date**: Retrieved from `APPYTR` and formatted for the check.
     - **Check Amount**: The total payment amount is printed in both numeric and written form (e.g., "ONE HUNDRED TWENTY-THREE THOUSAND FOUR HUNDRED FIFTY-SIX AND 78/100 DOLLARS").
   - For amounts exceeding $999,999.99, special handling ensures proper formatting, and for zero or invalid amounts, the program adjusts to avoid errors.

6. **Convert Numeric Amount to Words**:
   - The program converts the check amount’s dollar portion into words for the check’s written amount line (e.g., 123456.78 becomes "ONE HUNDRED TWENTY-THREE THOUSAND FOUR HUNDRED FIFTY-SIX").
   - The cents are appended as a fraction (e.g., "78/100"), and the word "DOLLARS" is added.
   - The conversion eliminates excess spaces to fit within a standard check’s 10 CPI (characters per inch) layout, ensuring readability and compliance with check printing standards.

7. **Print Checks and Copies**:
   - The program generates output for two printer files:
     - **APCHECK**: The primary check output, sent to the designated output queue (e.g., `APCHECKS` for production or `TESTOUTQ` for testing, as set in the OCL).
     - **CHECKCPY**: A copy of the check, sent to a separate output queue (e.g., `APCHKCPY` or `TESTOUTQ`).
   - Each check includes:
     - Header: Company name (if applicable), vendor name, check date, and check number.
     - Detail Lines: Invoice details (date, number, description, gross amount, discount, payment amount).
     - Totals: Summarized gross amount, discount, and net payment amount.
     - Written Amount: The check amount in words, with an asterisk (*) at the end for security.
   - Void checks are marked with "* VOID * VOID * VOID *" across the check to prevent misuse.
   - The program ensures proper alignment and formatting for both the check and its copy, adhering to printer settings (6 LPI, 12 CPI, standard quality).

8. **Handle Void Checks**:
   - For voided checks (`F` or `V`), the program prints the check with a "VOID" label and may either reuse the check number (`F`) or assign a new one (`V`), depending on the business need to track voided transactions.

9. **Clear Accumulators**:
   - After printing a check, the program resets the accumulated gross amount, discount, and payment totals to zero, preparing for the next vendor’s check.

10. **Completion**:
    - The program continues processing until all selected payments are handled, generating checks and copies as needed. The output is sent to the configured printers, and the process ends.

---

### Business Rules

The program enforces several business rules to ensure accurate, compliant, and efficient check printing:

1. **Selective Check Printing**:
   - Checks are only printed for payments not flagged as prepaid (`P`), ACH (`A`), wire transfer (`W`), employee expense (`E`), or credit/no pay (`C`). This ensures physical checks are issued only for standard payment methods, while electronic or prepaid payments are processed separately.

2. **Void Check Handling**:
   - Void checks (`F` or `V`) are printed with clear "VOID" markings to prevent fraudulent use.
   - For `F` (full stub, same check number), the check number is reused for the next stub, useful for multi-part check forms.
   - For `V` (void, next check number), a new check number is assigned, ensuring accurate tracking of voided checks.

3. **Company Name Exclusion**:
   - The company name is printed at the top of checks unless the company code is `01` (e.g., A.R.G.), likely due to specific branding or legal requirements for that company.

4. **Amount Formatting**:
   - The check amount is printed in both numeric and written forms, with the written form adhering to a compact format to fit standard check layouts (10 CPI).
   - Amounts exceeding $999,999.99 are handled to prevent formatting errors, ensuring reliability for large payments.
   - Zero or invalid amounts are processed to avoid printing blank or incorrect checks.

5. **Vendor Information Validation**:
   - If vendor details (e.g., name, address) are missing from the vendor file (`APVEND`), the program falls back to the open payables file (`APOPEN`) or blanks the fields, ensuring checks are still printed without errors.

6. **Check Number Management**:
   - The program ensures valid check numbers by skipping voided ones (`V`), preventing duplicate or invalid check numbers in the payment process.

7. **Invoice Detail Inclusion**:
   - Each check includes detailed invoice information (date, number, description, amounts) to provide transparency to vendors and support accounting reconciliation.

8. **Printer Output Separation**:
   - Checks and their copies are sent to separate output queues, allowing businesses to maintain distinct records for primary checks and copies for auditing or filing purposes.

9. **Payment Aggregation**:
   - Multiple invoices for a single vendor are aggregated into one check, with totals for gross amount, discount, and net payment, streamlining payment processing and reducing check issuance costs.

10. **Date and Check Number Consistency**:
    - The check date is sourced from the transaction file (`APPYTR`), ensuring consistency with the payment batch.
    - Check numbers are tracked via the check file (`APPYCK`), maintaining sequential and accurate check issuance.

---

### Summary

From a business perspective, the `AP160` program automates the critical process of printing A/P checks, ensuring payments to vendors are accurately calculated, formatted, and printed according to strict business rules. It handles vendor payments by aggregating invoices, validating payment types, assigning check numbers, and producing both primary checks and copies for record-keeping. The program supports various payment scenarios (e.g., void checks, electronic payments) and enforces rules like excluding certain company names or formatting amounts for compliance. By integrating data from multiple files (payment, vendor, open payables, etc.), it ensures accuracy and transparency, making it a vital component of the A/P workflow.

If you need further clarification, additional analysis (e.g., specific file layouts), or related information, let me know!