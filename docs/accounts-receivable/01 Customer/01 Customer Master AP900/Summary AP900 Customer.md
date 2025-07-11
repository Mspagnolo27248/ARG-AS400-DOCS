Based on the provided `AR900.ocl36.txt` and `AR900.rpgle.txt` files, the RPGLE program `AR900`, orchestrated by the OCL program, implements a customer master file maintenance system for an Accounts Receivable (AR) application. Below, I’ll identify the use cases implemented by the program and then provide a **Function Requirement Document** for a non-interactive version of the primary use case, assuming it processes inputs programmatically rather than through screen interactions.

---

### Use Cases Implemented by the Program

The `AR900` program primarily supports a single overarching use case with multiple sub-functions, all centered around maintaining customer master data. The use case and its sub-functions are:

**Use Case: Customer Master File Maintenance**
- **Description**: This use case allows for the addition, update, and reactivation of customer records in the Accounts Receivable system, including core customer data (`ARCUST`), supplemental data (`ARCUSP`), and related history files (`ARCUSHS`, `ARCUSPH`). It also synchronizes credit problem flags across grouped accounts (`ARCLGR`) and supports ship-to product maintenance via an external program (`BI907AC`).
- **Sub-Functions**:
  1. **Add New Customer**: Creates a new customer record in `ARCUST` and `ARCUSP`, with validations for mandatory fields, unique customer numbers, and valid codes (e.g., salesman, terms, customer class).
  2. **Update Existing Customer**: Modifies existing customer records in `ARCUST` and `ARCUSP`, ensuring valid data updates and maintaining history records.
  3. **Reactivate Inactive Customer**: Restores an inactive customer (marked with `'I'` in `ARCUST` and `ARCUSP`) to active status (`'A'`).
  4. **Mark Customer as Inactive**: Marks a customer as inactive (`'I'`) instead of deleting, provided there are no outstanding invoices.
  5. **Synchronize Credit Problem Flag**: Updates the credit problem flag (`CLPB`) across all grouped accounts in `ARCLGR` when modified for a customer.
  6. **Maintain Ship-to Products**: Calls `BI907AC` to manage ship-to product data in `ARCUPR` for the customer.
  7. **Record History**: Logs changes to customer and supplemental data in `ARCUSHS` and `ARCUSPH`, including system date, time, user ID, and workstation ID.

While the program supports these sub-functions, they are all part of the broader **Customer Master File Maintenance** use case, as the program is designed to handle all aspects of customer data management within a single interactive workflow.

---

### Function Requirement Document

**Document Title**: Customer Master File Maintenance Function  
**Document Purpose**: To define the requirements for a non-interactive function that performs customer master file maintenance (add, update, reactivate, and mark inactive) in the Accounts Receivable system, processing inputs programmatically instead of through screen interactions.

---

#### 1. Function Overview

**Function Name**: `CustomerMasterMaintenance`  
**Purpose**: To programmatically add, update, reactivate, or mark inactive customer records in the Accounts Receivable system, ensuring data integrity through validations and maintaining history and group synchronization.  
**Scope**: The function processes customer data for `ARCUST` (customer master), `ARCUSP` (supplemental data), `ARCUSHS` (customer history), `ARCUSPH` (supplemental history), and `ARCLGR` (group ledger), and interfaces with `BI907AC` for ship-to product maintenance.  
**Objective**: To provide a reliable, automated way to manage customer master data without user interaction, suitable for batch processing or API integration.

---

#### 2. Functional Requirements

##### 2.1 Inputs
The function accepts a structured input containing all necessary customer data fields, mode of operation, and system metadata. The input is a single data structure with the following fields:

- **Mode** (3 characters):
  - `ADD`: Add a new customer.
  - `UPD`: Update an existing customer.
  - `REA`: Reactivate an inactive customer.
  - `INA`: Mark a customer as inactive.
- **CompanyNumber** (2 digits, numeric): Company number (`ARCO`, `CO`).
- **CustomerNumber** (6 digits, numeric): Customer number (`ARCUSN`, `CUSN`).
- **CustomerData** (structure for `ARCUST` fields):
  - **Name** (30 characters): Customer name (`ARNAME`).
  - **AddressLine1** (30 characters): Address line 1 (`ARADR1`).
  - **AddressLine2** (30 characters): Address line 2 (`ARADR2`).
  - **AddressLine3** (30 characters): Address line 3 (`ARADR3`).
  - **AddressLine4** (30 characters): Address line 4 (`ARADR4`).
  - **ZipCode5** (5 characters): ZIP code (`ARZIP5`).
  - **ZipCode9** (4 characters): ZIP + 4 (`ARZIP9`).
  - **AreaCode** (3 digits, numeric): Phone area code (`ARAREA`).
  - **Telephone** (4 digits, numeric): Telephone number (`ARTELE`).
  - **SalesmanNumber** (2 digits, numeric): Salesman code (`ARSLS#`).
  - **TermsCode** (2 digits, numeric): Terms code (`ARTERM`).
  - **CreditLimit** (7 digits, numeric): Credit limit (`ARCLMT`).
  - **StatementCode** (1 character): Statements Y/N (`ARSTMT`).
  - **FinanceChargeCode** (1 character): Finance charge Y/N (`ARFINC`).
  - **SalesTaxCode** (4 characters): State tax code (`ARSTAX`).
  - **SortName** (10 characters): Last name abbreviation (`ARLSTN`).
  - **GroupCode** (2 characters): Group code (`ARGRUP`).
  - **MilesFreight** (3 digits, numeric): Miles for freight (`ARMILS`).
  - **InterCompanyNumber** (2 digits, numeric): Inter-company number (`ARINTR`).
  - **StateTaxes** (2 characters): State for taxes (`ARSTAT`).
  - **EFTParticipant** (1 character): EFT participant Y/N (`AREFT`).
  - **GroupingCompany** (2 digits, numeric): Grouping company (`ARGCO`).
  - **GroupingCustomer** (6 digits, numeric): Grouping customer (`ARGCUS`).
  - **CustomerClass** (1 character): Customer class (`ARCUCL`).
  - **EDIParticipant** (1 character): EDI participant Y/N (`AREDI`).
  - **CreditProblem** (1 character): Credit problem Y/N (`ARCLPB`).
  - **WireTransfer** (1 character): Wire transfer Y/N (`ARWIRE`).
  - **FederalEIN** (9 digits, numeric): Federal EIN number (`ARFEIN`).
  - **PAPermitClass** (3 characters): PA permit class (`ARPMCL`).
  - **ProductMove** (1 character): Product move Y/N (`ARPRMV`).
- **SupplementalData** (structure for `ARCUSP` fields):
  - **TaxCodes** (array of 10 x 4 characters): Tax codes (`TC`).
  - **TaxExemptNumbers** (array of 10 x 12 characters): Tax-exempt numbers (`TE`).
  - **StartDate** (8 digits, numeric, `YYYYMMDD`): Customer start date (`CSSTDT`).
  - **FinancialStatementDate** (8 digits, numeric, `YYYYMMDD`): Last financial statement date (`CSFSDT`).
  - **InsuranceCertDate** (8 digits, numeric, `YYYYMMDD`): Insurance certification expiration date (`CSICDT`).
  - **LastAllocationGallons** (4 digits, numeric): Last allocated gallons (`CSAGAL`).
  - **CreditComment1** (25 characters): Credit comment line 1 (`CSCMT1`).
  - **CreditComment2** (25 characters): Credit comment line 2 (`CSCMT2`).
  - **CreditComment3** (25 characters): Credit comment line 3 (`CSCMT3`).
  - **ContactName** (25 characters): Contact name (`CSCNCT`).
  - **CreditCardCount** (2 digits, numeric): Number of credit cards (`CSCARD`).
  - **SeparateFreight** (1 character): Separate freight Y/N (`CSSFRT`).
  - **OrderMarks1-4** (4 x 60 characters): Order marks (`CSOMK1`–`CSOMK4`).
  - **InvoiceMarks1-2** (2 x 60 characters): Invoice marks (`CSIMK1`–`CSIMK2`).
  - **DispatchInfo1-4** (4 x 60 characters): Dispatch information (`CSDSP1`–`CSDSP4`).
  - **BillOfLadingMarks1-4** (4 x 60 characters): Bill of lading marks (`CSBMK1`–`CSBMK4`).
  - **FreightCode** (1 character): Freight code (`CSFRCD`, `C`, `P`, `A`).
  - **FreightBillName** (30 characters): Freight bill name (`CSFRNM`).
  - **FreightBillAddress1-3** (3 x 30 characters): Freight bill address (`CSFRA1`–`CSFRA3`).
  - **BlanketPO** (15 characters): Blanket purchase order (`CSBKPO`).
  - **DuplicateOrderMatchType** (1 character): Duplicate order match type (`CSDUPC`, `A`, `B`).
  - **DuplicateOverrideDays** (3 digits, numeric): Override days for duplicate check (`CSDUPD`).
  - **CrossReferenceSet** (6 characters): Cross-reference set name (`CSXSET`).
  - **ACHClass** (3 characters): ACH class (`CSACLS`).
  - **ACHAccountType** (1 character): ACH checking/savings (`CSACOS`).
  - **ACHBankRoutingCode** (9 digits, numeric): ACH bank routing code (`CSARTE`).
  - **ACHBankAccountNumber** (17 characters): ACH bank account number (`CSABK#`).
- **SystemMetadata**:
  - **UserID** (8 characters): User performing the operation (`USERID`).
  - **WorkstationID** (2 characters): Workstation ID (`WSID`).
  - **TestPeriod** (1 character): Test period flag (`TSTPRD`).

##### 2.2 Outputs
- **StatusCode** (3 characters): Indicates success or failure (`SUC` for success, `ERR` for error).
- **ErrorMessage** (80 characters): Descriptive error message if the operation fails (combines `MSG1` and `MSG2` from the original program).
- **UpdatedCustomerNumber** (6 digits, numeric): The customer number processed, returned for confirmation.

##### 2.3 Processing Steps
1. **Initialization**:
   - Validate that `CompanyNumber` exists in `ARCONT`. If not, return `ERR` with "INVALID COMPANY NUMBER ENTERED".
   - Retrieve aging limits (`ACLMT1`–`ACLMT4`) from `ARCONT` for validation context.
   - Retrieve billing instructions (`BCINST`) from `BICONT`.

2. **Mode-Specific Validation**:
   - **ADD Mode**:
     - Ensure `CustomerNumber` is not zero and does not exist in `ARCUST` or `ARCUSP`. If it exists, return `ERR` with "CUSTOMER # IS ALREADY USED".
     - Validate all mandatory fields (e.g., `SortName`, `FederalEIN`, `StateTaxes`).
     - Validate codes against `GSTABL` (e.g., `SalesmanNumber`, `TermsCode`, `CustomerClass`).
     - Validate `StatementCode`, `FinanceChargeCode`, `SeparateFreight`, `FreightCode`, `EFTParticipant`, `EDIParticipant`, `CreditProblem`, `WireTransfer`, `DuplicateOrderMatchType`, `ProductMove` (must be `Y`, `N`, or blank, except `FreightCode` which can be `C`, `P`, `A`).
     - If `EFTParticipant = 'Y'`, ensure `ACHBankRoutingCode` and `ACHBankAccountNumber` are provided.
     - Validate dates (`StartDate`, `FinancialStatementDate`, `InsuranceCertDate`) for correct format (`YYYYMMDD`), valid months (1-12), days (1-31 or 1-28/29 for February), and leap year logic.

   - **UPD Mode**:
     - Ensure `CustomerNumber` exists in `ARCUST` and `ARCUSP`. If not, return `ERR` with "CUSTOMER NOT FOUND".
     - Validate all fields as in ADD mode.
     - Check that the customer is not inactive (`ARDEL ≠ 'I'` or `'D'`).

   - **REA Mode**:
     - Ensure `CustomerNumber` exists in `ARCUST` and `ARCUSP` and is inactive (`ARDEL = 'I'` or `'D'`). If not, return `ERR` with "THIS CUSTOMER WAS NOT PREVIOUSLY DELETED".
     - Validate minimal fields (e.g., `CompanyNumber`, `CustomerNumber`).

   - **INA Mode**:
     - Ensure `CustomerNumber` exists in `ARCUST` and `ARCUSP`. If not, return `ERR` with "CUSTOMER NOT FOUND".
     - Check for outstanding invoices by summing aging fields (`AGE`) in `ARCUST`. If non-zero, return `ERR` with "THIS CUSTOMER HAS OUTSTANDING INVOICES".
     - Validate minimal fields (e.g., `CompanyNumber`, `CustomerNumber`).

3. **Data Processing**:
   - **ADD/UPD Mode**:
     - Write or update records in `ARCUST` and `ARCUSP` with provided data.
     - Write history records to `ARCUSHS` and `ARCUSPH`, including `SystemMetadata` (`SYSCYM`, `SYSTIM`, `USERID`, `WSID`).
     - If `CreditProblem` is updated, synchronize `CLPB` across grouped accounts in `ARCLGR` using `ARCUST2`.
     - Call `BI907AC` with parameters (`CompanyNumber`, `CustomerNumber`, `ShipTo = '000'`, `Mode = 'MNT'`, `FileGroup = ' '`) to maintain ship-to products in `ARCUPR`.

   - **REA Mode**:
     - Update `ARCUST` and `ARCUSP` to set `ARDEL` and `CSDEL` to `'A'` (active).
     - Write history records to `ARCUSHS` and `ARCUSPH`.

   - **INA Mode**:
     - Update `ARCUST` and `ARCUSP` to set `ARDEL` and `CSDEL` to `'I'` (inactive).
     - Write history records to `ARCUSHS` and `ARCUSPH`.

4. **Output Generation**:
   - On success, return `StatusCode = 'SUC'`, `UpdatedCustomerNumber`, and empty `ErrorMessage`.
   - On failure, return `StatusCode = 'ERR'`, `UpdatedCustomerNumber` (if applicable), and the appropriate error message (e.g., from `MSG` array).

##### 2.4 Business Rules
- **Mandatory Fields**: `SortName`, `FederalEIN`, `StateTaxes` must not be blank.
- **Code Validations**:
  - `SalesmanNumber`, `TermsCode`, `CustomerClass` must exist in `GSTABL`.
  - `StatementCode`, `FinanceChargeCode`, `SeparateFreight`, `EFTParticipant`, `EDIParticipant`, `CreditProblem`, `WireTransfer`, `ProductMove` must be `Y`, `N`, or blank.
  - `FreightCode` must be `C`, `P`, `A`, or blank.
  - `DuplicateOrderMatchType` must be `A`, `B`, or blank.
- **EFT Validation**: If `EFTParticipant = 'Y'`, `ACHBankRoutingCode` and `ACHBankAccountNumber` must be provided.
- **Date Validation**: Dates must be in `YYYYMMDD` format, with valid months, days, and leap year handling.
- **Deletion Restriction**: Customers with outstanding invoices cannot be marked inactive.
- **Group Synchronization**: `CreditProblem` updates are propagated to all grouped accounts in `ARCLGR`.
- **History Logging**: All changes are logged in `ARCUSHS` and `ARCUSPH` with system metadata.

##### 2.5 Error Handling
- Returns descriptive error messages for invalid inputs, non-existent records, or validation failures.
- Logs errors in a system log (assumed external to the function) using `UserID` and `WorkstationID`.

##### 2.6 Performance Requirements
- Process a single customer record in under 1 second under normal system load.
- Handle up to 1000 records in a batch process without significant performance degradation.

##### 2.7 Security Requirements
- Restrict access to authorized users based on `UserID`.
- Validate `WorkstationID` against system configuration.
- Ensure sensitive data (e.g., `ACHBankAccountNumber`) is handled securely.

---

#### 3. Non-Functional Requirements

- **Platform**: IBM i (AS/400) with RPGLE support.
- **Files Accessed**:
  - `ARCUST`, `ARCUSP`, `ARCUSHS`, `ARCUSPH`, `ARCLGR`, `ARCONT`, `BICONT`, `GSTABL`.
- **External Program**: `BI907AC` for ship-to product maintenance.
- **Scalability**: Must support processing multiple records in a batch.
- **Reliability**: Must ensure data consistency across files with rollback on failure.
- **Maintainability**: Code must be modular, with clear documentation of validation logic and file operations.

---

#### 4. Assumptions
- Input data is provided in a structured format (e.g., JSON, XML, or data structure).
- The system has access to `GSTABL`, `ARCONT`, and `BICONT` for validations.
- `BI907AC` is available and correctly configured.
- The calling application handles logging and user authentication.

---

#### 5. Dependencies
- **Files**: `ARCUST`, `ARCUSP`, `ARCUSHS`, `ARCUSPH`, `ARCLGR`, `ARCONT`, `BICONT`, `GSTABL`.
- **Program**: `BI907AC`.
- **System**: IBM i environment with RPGLE runtime and database access.

---

#### 6. Constraints
- The function does not support physical deletion of customers (only marking as inactive, per `JB11`).
- Date fields must conform to `YYYYMMDD` format.
- Limited to processing one customer record at a time (batch processing requires iterative calls).

---

#### 7. Acceptance Criteria
- **ADD**: Successfully creates new records in `ARCUST`, `ARCUSP`, `ARCUSHS`, `ARCUSPH`, and calls `BI907AC` without errors.
- **UPD**: Updates existing records in `ARCUST`, `ARCUSP`, logs history, and synchronizes `ARCLGR` correctly.
- **REA**: Reactivates inactive customers, setting `ARDEL` and `CSDEL` to `'A'`.
- **INA**: Marks customers as inactive if no outstanding invoices, updating `ARDEL` and `CSDEL` to `'I'`.
- All validations pass (e.g., mandatory fields, codes, dates).
- Error messages are returned for all failure scenarios.
- History records include correct system metadata (`SYSCYM`, `SYSTIM`, `USERID`, `WSID`).

---

#### 8. Example Input/Output

**Example Input** (ADD Mode):
```json
{
  "Mode": "ADD",
  "CompanyNumber": 01,
  "CustomerNumber": 123456,
  "CustomerData": {
    "Name": "ABC Corp",
    "AddressLine1": "123 Main St",
    "AddressLine2": "",
    "AddressLine3": "City",
    "AddressLine4": "",
    "ZipCode5": "12345",
    "ZipCode9": "6789",
    "AreaCode": 123,
    "Telephone": 4567,
    "SalesmanNumber": 01,
    "TermsCode": 01,
    "CreditLimit": 10000,
    "StatementCode": "Y",
    "FinanceChargeCode": "N",
    "SalesTaxCode": "TX01",
    "SortName": "ABCCORP",
    "GroupCode": "01",
    "MilesFreight": 100,
    "InterCompanyNumber": 0,
    "StateTaxes": "TX",
    "EFTParticipant": "N",
    "GroupingCompany": 01,
    "GroupingCustomer": 123456,
    "CustomerClass": "A",
    "EDIParticipant": "N",
    "CreditProblem": "N",
    "WireTransfer": "N",
    "FederalEIN": 123456789,
    "PAPermitClass": "P01",
    "ProductMove": "N"
  },
  "SupplementalData": {
    "TaxCodes": ["TX01", "", "", "", "", "", "", "", "", ""],
    "TaxExemptNumbers": ["EX123", "", "", "", "", "", "", "", "", ""],
    "StartDate": 20250101,
    "FinancialStatementDate": 20241231,
    "InsuranceCertDate": 20251231,
    "LastAllocationGallons": 1000,
    "CreditComment1": "Good credit",
    "CreditComment2": "",
    "CreditComment3": "",
    "ContactName": "John Doe",
    "CreditCardCount": 1,
    "SeparateFreight": "Y",
    "OrderMarks1": "Order Mark 1",
    "OrderMarks2": "",
    "OrderMarks3": "",
    "OrderMarks4": "",
    "InvoiceMarks1": "",
    "InvoiceMarks2": "",
    "DispatchInfo1": "Dispatch 1",
    "DispatchInfo2": "",
    "DispatchInfo3": "",
    "DispatchInfo4": "",
    "BillOfLadingMarks1": "",
    "BillOfLadingMarks2": "",
    "BillOfLadingMarks3": "",
    "BillOfLadingMarks4": "",
    "FreightCode": "C",
    "FreightBillName": "Freight Co",
    "FreightBillAddress1": "456 Freight St",
    "FreightBillAddress2": "",
    "FreightBillAddress3": "",
    "BlanketPO": "PO123",
    "DuplicateOrderMatchType": "A",
    "DuplicateOverrideDays": 30,
    "CrossReferenceSet": "SET001",
    "ACHClass": "",
    "ACHAccountType": "",
    "ACHBankRoutingCode": 0,
    "ACHBankAccountNumber": ""
  },
  "SystemMetadata": {
    "UserID": "USER123",
    "WorkstationID": "WS",
    "TestPeriod": "T"
  }
}
```

**Example Output** (Success):
```json
{
  "StatusCode": "SUC",
  "ErrorMessage": "",
  "UpdatedCustomerNumber": 123456
}
```

**Example Output** (Failure):
```json
{
  "StatusCode": "ERR",
  "ErrorMessage": "SORT NAME MAY NOT BE BLANK",
  "UpdatedCustomerNumber": 123456
}
```

---

This **Function Requirement Document** reimagines the `AR900` program as a programmatic function, maintaining all core functionality (add, update, reactivate, inactivate) while replacing screen interactions with structured input processing. It ensures compatibility with the existing IBM i environment and leverages the same files and external program (`BI907AC`) as the original implementation.