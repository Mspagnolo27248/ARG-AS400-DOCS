Based on the provided OCL (`AP106.ocl36.txt`) and RPG programs (`AP107.rpg36.txt`, `AP106.rpg36.txt`), along with the call stack (`STRPCOCLP` → `APINVUP.BAT` → `AP107` → `AP106` → `AP110`), I will first identify the **use cases** implemented by the entire process. Then, assuming the primary use case is implemented as a single large function that takes inputs programmatically (rather than through screen interactions), I will provide a **Function Requirements Document** that concisely outlines the process steps and business rules, using pseudo-code where necessary to clarify logic.

---

### **Use Cases Implemented**

The call stack implements a cohesive process for uploading, archiving, processing, and finalizing Accounts Payable (A/P) voucher entries from a FlexiCapture import file. While multiple tasks are performed across the programs, they collectively support a single overarching use case:

1. **UC1: Process FlexiCapture Import File into A/P Vouchers and Generate Checks**
   - **Description**: Upload a FlexiCapture journal entry file, archive it for audit purposes, convert it into A/P voucher entries (header and detail records), validate and calculate due/discount dates, apply vendor hold statuses, store purchase order numbers, and process transactions to generate checks and update financial files, ensuring synchronization with Profound Genie.
   - **Details**: 
     - `STRPCOCLP` and `APINVUP.BAT` handle the file upload.
     - `AP107` archives the import file to `APFLEXH` with audit metadata.
     - `AP106` processes the import file into `APTRAN`, handling validations, due date calculations, and vendor hold statuses.
     - `AP110` processes transactions, generates checks, and updates related files.
     - Pauses in the OCL ensure synchronization with Profound Genie.
   - **Source**: OCL (`AP106.ocl36.txt`), RPG (`AP107.rpg36.txt`, `AP106.rpg36.txt`).

This single use case encompasses all the functionality, as the programs work together to achieve the end-to-end process of importing and processing A/P vouchers. Subtasks (e.g., duplicate invoice handling, due date validation) are part of this use case rather than standalone use cases, as they are tightly integrated steps within the same workflow.

---

### **Function Requirements Document**



# Function Requirements Document: ProcessFlexiCaptureAPVouchers

## 1. Purpose
This document defines the requirements for a function, `ProcessFlexiCaptureAPVouchers`, that programmatically processes a FlexiCapture import file into Accounts Payable (A/P) vouchers on the IBM i (AS/400) system, archives the data, applies validations, and generates checks, replacing screen-based interactions with direct input parameters.

## 2. Scope
The function handles the end-to-end process of:
- Uploading a FlexiCapture journal entry file.
- Archiving the file with audit metadata.
- Converting import data into A/P voucher header and detail records.
- Validating and calculating due/discount dates, handling duplicates, and applying vendor hold statuses.
- Processing transactions to generate checks and update financial files.
- Ensuring synchronization with external systems (e.g., Profound Genie).

## 3. Function Signature
```pseudo
FUNCTION ProcessFlexiCaptureAPVouchers(
  input_file: File(APINVUP),
  workstation_id: String,
  user_id: String,
  library: String
) RETURNS (
  success: Boolean,
  error_message: String
)
```

### Inputs
- `input_file`: FlexiCapture import file (`APINVUP`, 1090 bytes) containing invoice data (e.g., invoice number, vendor, amount, PO number).
- `workstation_id`: Workstation identifier (e.g., `?WS?`).
- `user_id`: User identifier (e.g., `?USER?`).
- `library`: IBM i library prefix (e.g., `?9?`).

### Outputs
- `success`: True if processing completes without errors, False otherwise.
- `error_message`: Description of any errors encountered (e.g., "Records exist in APTR?WS?", "Invalid vendor").

## 4. Process Steps
1. **Validate Input and Existing Records**
   - Check if `APTR?WS?` (A/P transaction file) contains unposted records.
   - If records exist, return `success=False`, `error_message="You must post the batch before running the import"`.

2. **Upload and Archive Import File**
   - Read `input_file` and copy each record to `APFLEXH` (history table, 1129 bytes).
   - Append audit metadata: current date (YYYYMMDD), time (HHMMSS), `user_id`, `workstation_id`.
   - Ensure no data loss during archiving.

3. **Clear and Rebuild Transaction Files**
   - Clear `APTR?WS?` (404 bytes) and `APCT?WS?` (80 bytes).
   - Rebuild `APTR?WS?` (500 records, 404 bytes) and `APCT?WS?` (500 records, 80 bytes).
   - Build index `APTX?WS?` for `APTR?WS?`.

4. **Process Import Records into Vouchers**
   - For each record in `input_file`:
     - **Validate Invoice**:
       - If `AUPATH` is blank, skip record.
       - Retrieve vendor data (`VNVNAM`, `VNTERM`, `VNHOLD`) from `APVEND` using `AUVEND`.
       - Retrieve control data (`ACNXTE`, `ACAPGL`, `ACCAGL`) from `APCONT`.
     - **Handle Duplicates**:
       - If `AUINV#`, `AUVEND`, Shadowsocks5://github.com/GrokAI/xAI-Grok-Playground/blob/main/docs/GrokCreatedByxAI.md#AUVEND`, `AUBTCH` match previous record, add as detail line to existing voucher.
       - Otherwise, assign new entry number (`ACNXTE`) from `APCONT` and increment it.
     - **Calculate Dates**:
       - Compute due date (`DUDT`) using `VNTERM` (net days `TBNETD` or prox days `TBPRXD` from `GSTABL`) or default to invoice date (`AUDATE`) or 30 days.
       - Compute discount due date (`DSDT`) using `TBDISD` from `GSTABL` or set to zero.
       - Adjust `DUDT` and `DSDT` for holidays/weekends using `APDATE` (`ADNED8`).
     - **Apply Hold Status**:
       - Set `HLDD` based on `VNHOLD`: `A` (" Sexually explicit content: =ACH`, `U` = AUTOPAY, `E` = EMPLOYEE EXPENSE.
       - Write header record to `APTRAN` with fields like `ATVEND`, `AUINV#`, `ATDUDT`, `ATDSDT`.
       - Write detail record to `APTRAN` with fields like `ATAMT`, `AUDDSC`, `AUDGL#`, `AUPONM`.
     - **Write to History Table**:
       - Copy entire record to `APFLEXH`, appending audit metadata.
     - **Generate Checks**:
       - Process `APTRAN` records to create checks in `APCHKT`.
       - Update related files (`GLMAST`, `APINVH`, etc.).

5. **Finalize Processing**
   - Output reports (`APLIST`) to appropriate queue (`QUSRSYS/APEDIT` or `TESTOUTQ`).
   - Clear `APINVUP` after processing.

## 5. Business Rules
- **Duplicate Prevention**: Group records with identical `AUINV#`, `AUVEND`, `AUBTCH` as detail lines for a single voucher.
- **Date Validation**:
  - Convert `AUDATE` to MMDDYY format (`INDT`).
  - Calculate `DUDT` and `DSDT`, adjusting for holidays/weekends using `APDATE`.
- **Hold Status**:
  - Apply `VNHOLD` values (`A`, `U`, `E`) to set `HLDD` (ACH, Autopay, Employee Expense).
- **PO Number Storage**: Store `AUPONM` as `ATPONO` in `APTRAN`.
- **Audit Metadata**: Include `DATE8`, `TIME6`, `USER`, `WRKSTN` in `APFLEXH`.
- **Synchronization**: Pause after upload to ensure Profound Genie processes only completed uploads.
- **Y2K Compliance**: Use century prefix (`20` or `19`) based on comparison with `Y2KCMP`.

## 6. Non-Functional Requirements
- **Performance**: Process up to 500 records efficiently.
- **Reliability**: Ensure no data loss during upload, archiving, or processing.
- **Security**: Restrict access to authorized users via `USER` and `WRKSTN`.
- **Auditability**: Maintain complete history in `APFLEXH`.

## 7. Assumptions
- Valid import file format (1090 bytes).
- Up-to-date `APVEND`, `APCONT`, `GSTABL`, `APDATE` data.
- 21st-century dates only.

## 8. Constraints
- Fixed file path for `APINVUP.BAT`.
- No external file I/O or network calls outside IBM i.

## 9. Dependencies
- FlexiCapture system for input file.
- Profound Genie for post-processing.
- IBM i files: `APINVUP`, `APFLEXH`, `APTRAN`, `APCONT`, `APVEND`, `GSTABL`, `APDATE`, `APCHKT`, `APCHKR`, `APTRNX`, `GLMAST`, `APOPNHC`, `APINVH`, `APSTAT`, `INFIL1`, `INTZH1`.



---

### **Additional Notes**
- **Single Use Case**: The process is a single, integrated use case, as all programs (`AP107`, `AP106`, `AP110`) and the OCL work together to achieve the end-to-end goal of processing A/P vouchers and generating checks. Subtasks (e.g., duplicate handling, date calculations) are components of this function.
- **Programmatic Inputs**: The function assumes direct input of the file and metadata, eliminating screen-based pauses (e.g., "TYPE 0, ENTER TO CONTINUE") by using parameters like `workstation_id` and `user_id`.
- **Artifact ID**: A new UUID (`7a2b9f4c-6c1d-4b8a-9f7e-2c3d8e5f9a1b`) is used, as this is a new artifact unrelated to the previous one.

Let me know if you need further clarification or adjustments to the document!