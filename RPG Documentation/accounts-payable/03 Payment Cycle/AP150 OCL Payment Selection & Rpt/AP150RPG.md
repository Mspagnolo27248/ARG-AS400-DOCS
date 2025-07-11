The `AP150.rpg36.txt` file is an RPG III program (used on IBM midrange systems like the AS/400, now IBM i) that handles **Accounts Payable (A/P) Payment Transaction Entry and Edit**. It is the first program called by the `AP150.ocl36.txt` OCL script, as specified in the `LOAD AP150` statement. This program provides a user interface for entering and editing payment transactions, validating inputs, and managing payment selections for checks, ACH, wire transfers, employee expenses, or utility auto-pay. Below, I’ll explain the **process steps** and **business rules** embedded in the program, based on the provided code.

---

### **Process Steps of the AP150 RPG Program**

The `AP150` program is a workstation-based application that uses a display file (`SCREEN`) to interact with users for entering and editing A/P payment transactions. It processes data through a series of screens (`AP150S1` to `AP150S5`) and performs validations, file updates, and navigation logic. Here’s a step-by-step breakdown of the process:

1. **Program Initialization (`ONETIM` Subroutine)**:
   - Initializes variables, sets system date and time, and determines the operating mode (add or update).
   - Checks if 13 accounting periods are used (`GX13GL` in `GSCONT` file).
   - If no records exist in `APPYTR` (payment transaction file), sets the program to **add mode** (`indicator 17`) and initializes `NXTSEQ` (next sequence number) to 1.
   - If records exist, sets the program to **update mode** (`indicator 16`), retrieves the company number (`PTCONO`), and increments `NXTSEQ` based on the last sequence number (`LSTSEQ`).

2. **Screen Processing**:
   The program uses five screen formats (`AP150S1` to `AP150S5`) to collect and validate data. Each screen corresponds to a specific step in the payment transaction entry process:
   - **S1 (Screen 1)**: Collects company number (`CONO`).
     - Validates the company number against `APCONT` (A/P control file).
     - If valid, retrieves company details (e.g., cash G/L account `ACCAGL`, next check number `ACCKNO`) and populates screen fields.
     - If invalid, displays error message "INVALID COMPANY #".
   - **S2 (Screen 2)**: Collects bank G/L number (`BKGL`), check date (`CKDT`), payment date (`DATE`), force discount flag (`FDISC`), accounting period/year (`KYPD`, `KYPDYY`), and payment method (`KYHOLD`).
     - Validates inputs (e.g., bank G/L against `GLMAST`, check number not zero, valid dates, valid `KYHOLD` codes: `' '`, `A`, `W`, `E`, `U`).
     - If checks were previously printed (`CHKPRT='Y'`), validates the CAPTCHA code (`APCODE`) to ensure authorized access.
     - Performs date and period validations, especially for 13 accounting periods.
   - **S3 (Screen 3)**: Collects sequence number (`SEQ#`) for a transaction.
     - Validates `SEQ#` against `APPYTR` and ensures it’s not marked for deletion (`PTDEL='D'`).
     - Retrieves transaction details if valid; otherwise, displays an error.
   - **S4 (Screen 4)**: Collects vendor (`VEND`), voucher (`VO`), payment amount (`AMT`), discount amount (`DISC`), and other flags (`FDIS`, `PORH`, `SNGL`, `MKPP`, `PPCK`, `PPDT`).
     - Performs extensive validations (e.g., vendor exists in `APVEND`, voucher exists in `APOPEN`, payment amount doesn’t exceed gross amount, correct payment method).
     - Updates or adds the transaction to `APPYTR`.
   - **S5 (Screen 5)**: Allows the user to start over by deleting all transactions.
     - Requires confirmation with the code `"START OVER"`.
     - Deletes all records in `APPYTR` and resets the program to initial state.

3. **Key Navigation and Modes**:
   - The program supports **add mode** (`indicator 17`) for new transactions and **update mode** (`indicator 16`) for editing existing ones.
   - Function keys control navigation:
     - **KA F2**: Rekey without adding/updating.
     - **KA F4**: Clear selections and start over.
     - **KD F3**: Delete a transaction.
     - **KG F3**: End the job.
     - **KJ F11**: Switch to add mode.
     - **KK F12**: Switch to update mode.
     - **KL**: Allow payment from a different bank G/L number than originally assigned.
     - **Roll Keys (18/19)**: Navigate forward/backward through transactions.
   - The program updates the `LSTSEQ` (last sequence number) and increments `NXTSEQ` for new transactions.

4. **Data Validation and Editing**:
   - The `S2EDIT`, `S4EDIT`, and `DTEDIT` subroutines perform detailed validations:
     - **S2EDIT**: Validates bank G/L, check number, dates, force discount code, and accounting period/year.
     - **S4EDIT**: Validates vendor, voucher, payment amount, discount amount, and payment codes (`FDIS`, `PORH`, `SNGL`, `MKPP`).
     - **DTEDIT**: Validates date formats, including leap year checks and month/day ranges.
   - Errors trigger appropriate indicators (e.g., `90` for general errors) and display messages from the `MSG` array.

5. **File Updates**:
   - Transactions are written to or updated in the `APPYTR` file (payment transaction file) using the `PUTPT` subroutine.
   - Deletions are marked with `PTDEL='D'` in `APPYTR`.
   - The program updates the CAPTCHA code in `GSCONT` if needed (`CODEUP` subroutine).

6. **Error Handling and User Feedback**:
   - Error messages are displayed on the screen (`MSG30`, `MSGC1`, `MSGC2`) based on validation failures (e.g., "INVALID VENDOR #", "CAN’T PAY MORE THAN ->").
   - The program uses indicators (e.g., `81` to `90`) to control screen display and error states.

7. **Cleanup and Termination**:
   - The `CLEAR` subroutine resets input fields for new entries.
   - The `STOVER` subroutine handles the "start over" request.
   - The program terminates when the user presses `KG` (F3) or completes the transaction entry process.

---

### **Business Rules in the AP150 RPG Program**

The program enforces several business rules to ensure accurate and secure A/P transaction processing. These rules are derived from the validation logic and comments in the code:

1. **Company Validation**:
   - The company number (`CONO`) must exist in the `APCONT` file. If not, an error ("INVALID COMPANY #") is displayed, and the user is prompted to correct it.

2. **Bank G/L and Check Number**:
   - The bank G/L number (`BKGL`) must exist in the `GLMAST` file ("INVALID BANK G/L #").
   - The next check number (`NXCK`) cannot be zero ("CHECK # CANNOT BE ZERO").
   - If a different bank G/L is used (`KL` key), the program allows overriding the original bank G/L assigned to a voucher, with a warning ("PRESS F12 TO PAY VOUCHER").

3. **Date Validations**:
   - The check date (`CKDT`) and payment date (`DATE`) must be valid dates, checked via the `DTEDIT` subroutine.
     - Invalid dates trigger "INVALID CHECK DATE" or "INVALID DATE TO PAY BY".
     - Dates must align with the accounting period/year (`KYPD`, `KYPDYY`) if 13 accounting periods are used (`GX13GL='Y'`).
     - Dates are compared against period end dates in `GSTABL` to ensure they fall within valid ranges ("DATE INVALID FOR PD/YR KEYED").
   - If the check date is not today’s date, a warning is displayed ("DATE NOT TODAY - F3 IF OK").

4. **Payment Method Selection**:
   - The `KYHOLD` field specifies the payment method: `' '` (checks), `A` (ACH), `W` (wire transfer), `E` (employee expenses), or `U` (utility auto-pay).
   - The voucher’s hold code (`OPHALT` in `APOPEN`) must match `KYHOLD`. Mismatches trigger the error "CAN’T PAY THIS VOUCHER NOW".
   - For example, a voucher marked for ACH (`OPHALT='A'`) cannot be paid with a check (`KYHOLD=' '`).

5. **Voucher and Vendor Validation**:
   - The vendor number (`VEND`) must exist in `APVEND` ("INVALID VENDOR #").
   - The voucher number (`VO`) must exist in `APOPEN` and match the vendor/company (`OPCOVN`) ("VOUCHER IS NOT OPEN").
   - For one-time vendors (`VEND=0`), a voucher number must be provided ("A VOUCHER MUST BE KEYED").
   - The voucher must not be on hold unless explicitly allowed.

6. **Payment and Discount Amounts**:
   - The payment amount (`AMT`) must not exceed the remaining gross amount (`OPGRAM - OPPPTD`) in `APOPEN` ("CAN’T PAY MORE THAN ->").
   - The discount amount (`DISC`) must not exceed the gross amount ("INVALID DISCOUNT AMOUNT").
   - The force discount code (`FDIS`) must be `' '` or `'D'` ("FORCE DISCOUNTS MUST BE 'D'").

7. **Pay or Hold and Single Check Codes**:
   - The pay/hold code (`PORH`) must be `'P'` (pay) or `'H'` (hold) ("PAY/HOLD MUST BE 'P'/'H'").
   - The single check code (`SNGL`) must be `' '` or `'S'` ("SINGLE CHECK MUST BE 'S'").

8. **Prepaid Check Validation**:
   - If the make prepaid code (`MKPP`) is specified, it must match the payment method:
     - `'P'` for checks, `'A'` for ACH, `'W'` for wire transfers, `'E'` for employee expenses, or `'U'` for utility auto-pay.
     - Errors trigger specific messages (e.g., "MAKE PREPAID MUST BE 'A'").
   - If `MKPP` is set, the prepaid check number (`PPCK`) and date (`PPDT`) must be provided and valid ("PREPAID CHECK # IS MISSING", "INVALID PPD CHECK DATE").
   - If `MKPP` is not set, `PPCK` and `PPDT` must be zero ("PREPAID CHECK # MUST BE ZERO", "PPD CHECK DATE MUST BE ZERO").

9. **Sequence Number Management**:
   - The sequence number (`SEQ#`) must exist in `APPYTR` and not be marked for deletion ("INVALID SEQUENCE #", "SEQ # WAS PREVIOUSLY DELETED").
   - New transactions increment the sequence number (`NXTSEQ`).

10. **Start Over Functionality**:
    - Users can delete all transactions by entering `"START OVER"` in the `STOVER` field ("REQUEST TO START OVER WAS IGNORED" if incorrect).
    - This clears `APPYTR` and resets the program to initial state.

11. **Security and CAPTCHA**:
    - If checks were previously printed (`CHKPRT='Y'`), a CAPTCHA code (`APCODE`) must be validated against `GSCONT` to proceed ("OVERRIDE CODES DO NOT MATCH").
    - A new CAPTCHA code is generated and updated in `GSCONT` after validation (`CODEUP` subroutine).

12. **Accounting Period Handling**:
    - If 13 accounting periods are used (`GX13GL='Y'`), the period (`KYPD`) must be between 1 and 13, and the period/year must align with period end dates in `GSTABL` ("INVALID PERIOD/YEAR").
    - The check date must fall within the specified period’s date range.

13. **Transaction Deletion**:
    - Transactions can be marked for deletion (`PTDEL='D'`) using the `KD` key.
    - Deleted transactions are not processed further but remain in `APPYTR` with the deletion flag.

---

### **Summary**

The `AP150` RPG program is a critical component of the A/P payment transaction process, providing an interactive interface for entering and editing payment details. It progresses through five screens to collect and validate company, bank, date, vendor, voucher, and payment information. The program enforces strict business rules to ensure data integrity, including validations for company, vendor, voucher, payment amounts, and payment methods. It supports multiple payment types (checks, ACH, wire transfers, employee expenses, utility auto-pay) and handles special cases like prepaid checks and 13 accounting periods. The program interacts with multiple files (`APPYTR`, `APCONT`, `GLMAST`, `APVEND`, `APOPEN`, `GSCONT`, `GSTABL`) and uses subroutines to modularize validation and processing logic.