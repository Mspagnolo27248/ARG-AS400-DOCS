### List of Use Cases Implemented by the AP160 Program

Based on the provided OCL (`AP160.ocl36.txt`) and RPG (`AP160.rpg36.txt`) programs, the primary use case implemented is:

1. **Print Accounts Payable Checks for Vendor Payments**:
   - This use case involves processing selected vendor payments, generating physical checks (and copies) with detailed invoice information, and handling special cases like void checks or non-printed payments (e.g., ACH, wire transfers). The program retrieves data from various files, validates payment types, calculates totals, formats check output, and sends it to designated printers.

No additional distinct use cases are explicitly implemented in the provided code, as the program's focus is solely on check printing for Accounts Payable.

---

### Functional Requirements Document



# Functional Requirements: Accounts Payable Check Printing

## Overview
The Accounts Payable (A/P) Check Printing function automates the generation and printing of checks for vendor payments, ensuring accurate payment processing, proper formatting, and compliance with business rules. It processes input data from payment, vendor, and control files to produce checks and copies, handling various payment types and exceptions.

## Inputs
- **Payment Data**: Invoice details (vendor number, invoice number, gross amount, discount, payment amount, invoice date, sequence number).
- **Vendor Data**: Vendor name, address (up to four lines), and sort code.
- **Control Data**: Company name and code.
- **Transaction Data**: Check date.
- **Check Data**: Check number and payment type (e.g., normal, prepaid, ACH, wire transfer, void).
- **Environment Parameter**: Indicator for production (`G`) or test (`Z`) environment to determine output queues.

## Process Steps
1. **Retrieve Input Data**: Collect payment, vendor, control, transaction, and check data from respective input sources.
2. **Validate Payment Type**: Identify payment type to determine if a physical check is required:
   - Skip printing for prepaid (`P`), ACH (`A`), wire transfer (`W`), employee expense (`E`), or credit/no pay (`C`) payments.
   - Process normal payments or void checks (`F`, `V`) for printing.
3. **Calculate Check Totals**: Aggregate gross amount, discount, and net payment amount for each vendor’s invoices.
4. **Assign Check Number**: Use a valid check number, incrementing for voided checks (`V`) to avoid duplicates.
5. **Format Check Output**:
   - Include company name (except for company code `01`), vendor name, address, check date, and check number.
   - List invoice details (date, number, description, gross amount, discount, payment amount).
   - Convert net payment amount to words (e.g., "ONE HUNDRED DOLLARS AND 50/100") for the check’s written line.
   - Mark void checks with "* VOID * VOID * VOID *".
6. **Print Checks and Copies**:
   - Send primary check to the production (`APCHECKS`) or test (`TESTOUTQ`) queue.
   - Send check copy to the production (`APCHKCPY`) or test (`TESTOUTQ`) queue.
7. **Create Temporary File (if needed)**: Generate a temporary file for check processing data if specified.
8. **Reset Accumulators**: Clear totals after each check to prepare for the next vendor.

## Business Rules
1. **Selective Printing**: Print checks only for normal payments or void checks (`F`, `V`); skip prepaid, ACH, wire transfer, employee expense, or credit/no pay payments.
2. **Void Check Handling**:
   - Print "VOID" on checks marked `F` (reuse check number) or `V` (use next check number).
3. **Company Name Exclusion**: Omit company name on checks for company code `01`.
4. **Amount Formatting**: Convert amounts to words, removing excess spaces to fit 10 CPI check layout; handle amounts over $999,999.99 correctly.
5. **Vendor Data Fallback**: Use vendor details from open payables if missing from vendor master; blank fields if unavailable.
6. **Check Number Integrity**: Skip voided check numbers to ensure valid, sequential issuance.
7. **Output Separation**: Print checks and copies to separate queues for record-keeping.
8. **Invoice Aggregation**: Combine multiple invoices for a vendor into one check with summarized totals.
9. **Date Consistency**: Use transaction file’s check date for all checks in a batch.
10. **Printer Configuration**: Apply 6 lines per inch, 12 characters per inch, and standard quality for check printing.

## Outputs
- **Primary Check**: Formatted check with vendor details, invoice data, totals, and written amount, sent to the designated output queue.
- **Check Copy**: Duplicate of the check for record-keeping, sent to a separate output queue.
- **Temporary File (optional)**: Temporary data file for processing, if required.

## Constraints
- Must handle large payment amounts without formatting errors.
- Must align with standard check printing formats (10 CPI).
- Must support production and test environments via output queue configuration.

