
The RPG program `AP255.RPG36` is invoked by the OCL procedure `AP250.OCL36` as part of the Accounts Payable (A/P) Check Register process, specifically to generate the Cash Disbursements Journal and summarize A/P transactions. Below is a detailed explanation of the **process steps**, **business rules**, **tables/files used**, and **external programs called**, as requested.

---

### Process Steps

The `AP255.RPG36` program processes sorted records from the cash disbursements journal file (`APCDJR`) to produce a summarized journal report and update a general ledger file (`TEMGEN`). The processing is structured around a main program flow and a single subroutine (`JRNL`). The steps are as follows:

1. **Initialization (Level 4 Break - L4)**:
   - Executes at the start of the program (triggered by level break `L4`).
   - Retrieves the current system time and date using the `TIME` operation, storing them in `TIMDAT` (12 digits).
   - Splits `TIMDAT` into `TIME` (6 digits, time) and `DATE` (6 digits, date).
   - Converts the date to a year-month-day format (`DATYMD`) by multiplying `DATE` by 10000.01.
   - Formats the date into an 8-digit format (`DATE8`) by prefixing the century (`20`) and appending `DATYMD`.
   - Initializes the page counter (`PAGE`) to 0.
   - Sets a separator field (`SEP`) to `'* '`.
   - Chains to the `APCONT` file using the company number (`CDCONO`) to retrieve the company name (`ACNAME`).
   - Initializes debit and credit accumulators (`L4DR`, `L4CR`) to 0.

2. **Check Year/Period for Printing**:
   - Compares the year/period field (`CDYYPD`, positions 116–119) to 0.
   - If `CDYYPD` is non-zero (`N99`), sets indicator `98` to print the period and year on the journal report; otherwise, sets indicator `99`.

3. **Process Each Journal Record**:
   - Reads records from `APCDJR` (primary input file, indicator `01`).
   - Checks the credit/debit code (`CDCORD`, position 12) to identify debit entries (`'D'`, sets indicator `30`).
   - Checks the transaction type (`CDTYPE`, positions 106–115) to identify A/P transactions (`'AP      '`, sets indicator `20` for summarization).
   - Accumulates the transaction amount (`CDAMT`) into `L1AMT` (level 1 accumulator).
   - Converts the check date (`CDCKDT`, positions 51–56) to a year-month-day format (`YMD`) by multiplying by 10000.01.
   - Handles Y2K compliance for the check date:
     - Extracts the year (`YY`) from `YMD`.
     - If `YY` is greater than or equal to `Y2KCMP` (80), sets the century (`CN`) to `Y2KCEN` (19); otherwise, increments `Y2KCEN` by 1.
     - Formats the date into an 8-digit format (`CYMD`) by combining `CN` and `YMD`.

4. **Write Journal Entries (JRNL Subroutine)**:
   - Executes for non-summarized records (`N20`) or summarized A/P records at level 1 break (`L1` and `20`).
   - Increments the journal reference number (`JRREF#`) by 1.
   - Determines the credit/debit code (`CORD`):
     - Copies `CDCORD` to `CORD`.
     - If `L1AMT` is negative (indicator `10`):
       - Switches the sign of `L1AMT` using `Z-SUB`.
       - Sets `CORD` to `'C'` if `CDCORD` is `'D'` (indicator `30`), or `'D'` otherwise.
   - Accumulates amounts:
     - If `CORD` is `'D'` (indicator `11`), adds `L1AMT` to `L4DR` (debit total).
     - Otherwise, adds `L1AMT` to `L4CR` (credit total).
   - Sets `JRAMT` to `L1AMT` and resets `L1AMT` to 0.
   - Writes to the general ledger file (`TEMGEN`):
     - For non-summarized records (`01`, `N20`):
       - Writes a detail record with fields like `CDCONO`, `CDGLNO`, `CDJRNL`, `JRREF#`, `CORD`, `CDCHEK`, `CDDESC`, `YMD`, `JRAMT`, `CDNAME`, `CYMD`.
     - For summarized A/P records (`L1`, `20`):
       - Writes a summarized record with a fixed description (`'-SUMMARIZED A/P         '`), using `CDCKDT` instead of `CDCHEK`.

5. **Generate Cash Disbursements Journal Report (APPRINT)**:
   - Outputs a report to the printer file (`APPRINT`) with headers, detail lines, and totals:
     - **Headers** (level 4 break, `L4` or `OFNL4`):
       - Company name (`ACNAME`), page number (`PAGE`), date (`DATE`, formatted as `Y`), time (`TIME`), wire transfer description (`WIREDS`), and journal title.
       - Includes period/year (`CDPD`, `CDPDYY`) if `CDYYPD` is non-zero (indicator `98`).
       - Column headers for journal reference, paycheck, description, vendor name, G/L number, debit amount, and credit amount.
     - **Detail Lines** (non-summarized, `01`, `N20`):
       - Outputs `CDJRNL`, `JRREF#`, check number (`CDCHEK`), description (`CDDESC`), vendor name (`CDNAME`), G/L number (`CDGLNO`), and amount (`JRAMT`) as debit (`11`) or credit (`N11`).
     - **Summarized Lines** (level 1 break, `L1`, `20`):
       - Outputs `CDJRNL`, `JRREF#`, check date (`CDCKDTY`), fixed description (`'-SUMMARIZED A/P         '`), G/L number (`CDGLNO`), and amount (`JRAMT`) as debit or credit.
     - **Totals** (level 4 break, `T 2`, `L4`):
       - Outputs journal totals with debit (`L4DR`) and credit (`L audition for free, but I need to log in to view this.