# Accounts Module Documentation

> **Version:** 1.0  
> **Last Updated:** December 3, 2025  
> **Author:** Phoenix Development Team

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture Diagram](#architecture-diagram)
3. [Frontend (CaseAccount Module)](#frontend-caseaccount-module)
4. [Backend (Accounts Controller)](#backend-accounts-controller)
5. [System-Level Side Effects](#system-level-side-effects)
6. [Database Tables](#database-tables)
7. [Data Flow](#data-flow)

---

## Overview

The **Accounts Module** is a core domain in the Phoenix system that manages medical billing accounts associated with legal cases. It handles:

- Account creation, editing, and viewing
- Billing facility and service facility linkage
- Cost calculations and charges management
- Document management for accounts
- Integration with procedures, payments, and reductions
- PTF (Paid-To-Facility) status tracking
- Bulk operations and exports

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              FRONTEND (React/TypeScript)                     │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │   CaseAccountAdd    │  │   CaseAccountEdit   │  │   CaseAccountView   │ │
│  └──────────┬──────────┘  └──────────┬──────────┘  └──────────┬──────────┘ │
│             │                        │                        │             │
│             └────────────────────────┼────────────────────────┘             │
│                                      │                                       │
│  ┌───────────────────────────────────▼───────────────────────────────────┐  │
│  │                      RTK Query (accountApi)                            │  │
│  │  • useGetAccountQuery • useCreateAccountMutation                       │  │
│  │  • useUpdateAccountMutation • useLazyGetAccountAllQuery                │  │
│  └───────────────────────────────────┬───────────────────────────────────┘  │
└──────────────────────────────────────┼──────────────────────────────────────┘
                                       │ HTTP/REST
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                                BFF Layer                                      │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │                    AccountController (BFF)                              │  │
│  │  • GetById • Create • Update • Search • BulkUpdate • Delete            │  │
│  └────────────────────────────────────┬───────────────────────────────────┘  │
│                                       │                                       │
│  ┌────────────────────────────────────▼───────────────────────────────────┐  │
│  │                         IAccountService                                 │  │
│  └────────────────────────────────────┬───────────────────────────────────┘  │
└───────────────────────────────────────┼──────────────────────────────────────┘
                                        │ HTTP/REST (Internal)
                                        ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                               Core API Layer                                  │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │                   AccountController (Core API)                          │  │
│  └────────────────────────────────────┬───────────────────────────────────┘  │
│                                       │                                       │
│  ┌────────────────────────────────────▼───────────────────────────────────┐  │
│  │                         IAccountService                                 │  │
│  │  Dependencies:                                                          │  │
│  │  • IAccountRepository          • IAccountMessagingService               │  │
│  │  • IAccountAutomationService   • ICaseAutomationService                 │  │
│  │  • IValidationService          • IFacilityRepository                    │  │
│  │  • IContractRepository         • IProcedureRepository                   │  │
│  └────────────────────────────────────┬───────────────────────────────────┘  │
│                                       │                                       │
│  ┌────────────────────────────────────▼───────────────────────────────────┐  │
│  │                        IAccountRepository                               │  │
│  │  (Entity Framework Core - DebtorContext)                                │  │
│  └────────────────────────────────────┬───────────────────────────────────┘  │
└───────────────────────────────────────┼──────────────────────────────────────┘
                                        │
                                        ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                              SQL Server Database                              │
│  [debtor].[Accounts] • [debtor].[AccountCharges] • [debtor].[AccountChargeCpts]│
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Frontend (CaseAccount Module)

### Directory Structure

```
phoenix-ui/src/pages/CaseAccount/
├── accountSchema.ts              # Yup validation schemas
├── constants.ts                  # Module constants
├── helpers.ts                    # Utility functions
├── helpers.test.ts
├── index.ts                      # Module exports
├── useCaseAccountNavigation.ts   # Navigation hook
├── CaseAccountAdd/               # Add Account page
│   └── CaseAccountAdd.tsx
├── CaseAccountEdit/              # Edit Account page
│   └── CaseAccountEdit.tsx
├── CaseAccountView/              # View Account page
│   ├── CaseAccountView.tsx
│   ├── CaseAccountDetails/
│   ├── CaseAccountFacilityDetails/
│   ├── CaseAccountInfo/
│   └── CaseAccountTabs/
├── CaseAccountStepHeader/        # Wizard step header
├── CaseAccountSummary/           # Account summary component
├── CaseAccountTotalCost/         # Cost calculation display
├── CaseAccountWizardAsideTabs/   # Wizard sidebar tabs
├── hooks/                        # Custom hooks
│   ├── useAccountRestrictStatusChange.tsx
│   ├── useAccountSaveConfirmationDialog.tsx
│   ├── useDuplicateAccountConfirmationDialog.tsx
│   └── useChargesExceedConfirmationDialog.tsx
└── Steps/                        # Form wizard steps
    ├── caseAccountStepsMap.ts
    ├── AccountDetailsStep/
    ├── ChargesStep/
    ├── DocumentStep/
    └── FacilityDetailsStep/
```

### Components

| Component | File | Description |
|-----------|------|-------------|
| `CaseAccountAdd` | `CaseAccountAdd/CaseAccountAdd.tsx` | Page for creating new accounts with multi-step wizard |
| `CaseAccountEdit` | `CaseAccountEdit/CaseAccountEdit.tsx` | Page for editing existing accounts |
| `CaseAccountView` | `CaseAccountView/CaseAccountView.tsx` | Read-only view of account details |
| `CaseAccountSummary` | `CaseAccountSummary/CaseAccountSummary.tsx` | Summary component showing key account info |
| `CaseAccountTotalCost` | `CaseAccountTotalCost/CaseAccountTotalCost.tsx` | Displays calculated total cost |
| `CaseAccountDetails` | `CaseAccountView/CaseAccountDetails/` | Account detail section |
| `CaseAccountFacilityDetails` | `CaseAccountView/CaseAccountFacilityDetails/` | Facility information display |
| `CaseAccountTabs` | `CaseAccountView/CaseAccountTabs/` | Tab navigation for account views |
| `CaseAccountWizardAsideTabs` | `CaseAccountWizardAsideTabs/` | Sidebar tabs in wizard (Notes, Summary) |

### Wizard Steps

| Step | Component | Description |
|------|-----------|-------------|
| Facility Details | `FacilityDetailsStep` | Select billing/service facility, contract, location type |
| Account Details | `AccountDetailsStep` | Status, progress, dates, physician info, procedure linkage |
| Charges | `ChargesStep` | CPT codes, billing values, cost calculations |
| Documents | `DocumentsStep` | Upload Bill, Lien/LOP, Medical Records, General Package |

### Custom Hooks

| Hook | Purpose |
|------|---------|
| `useCaseAccountNavigation` | Handles navigation between account pages |
| `useAccountRestrictStatusChange` | Controls status transition validation |
| `useAccountSaveConfirmationDialog` | Confirmation dialog for saving drafts |
| `useDuplicateAccountConfirmationDialog` | Warns about potential duplicate accounts |
| `useChargesExceedConfirmationDialog` | Validates charges against procedure limits |

### State Management (RTK Query)

**File:** `phoenix-ui/src/store/bff-api/accounts/accounts.ts`

```typescript
// Key RTK Query Hooks
useGetAccountQuery            // GET /api/Account/{id}
useCreateAccountMutation      // POST /api/Account
useUpdateAccountMutation      // PUT /api/Account/{id}
useUpdateAccountBulkMutation  // PUT /api/Account/Bulk
useLazyGetAccountAllQuery     // POST /api/Account/Search
useLazyGetAccountProceduresQuery
useUnlinkAccountMutation
useLazyExportAccountSearchQuery
useUpdateBulkAccountsMutation
useLazyValidateForWriteOffQuery
useCheckDuplicateAccountMutation
useLazyCalculateAccountCostQuery
useRemoveAccountMutation
```

### Validation (Yup Schemas)

**File:** `accountSchema.ts`

| Schema | Description |
|--------|-------------|
| `draftAccountSchema` | Minimal validation for draft/pending accounts |
| `activeAccountSchema` | Full validation for open/active accounts |
| `documentStepSchema` | Document upload validation with file size limits |
| `facilityStepSchema` | Billing/Service facility validation |
| `accountStepSchema` | Core account field validation |
| `chargesStepSchema` | Charges and CPT validation |
| `chargeActiveSchema` | Strict charge validation for active accounts |

### Data Models / Interfaces

**Key Types from BFF API:**

```typescript
// Core Account Types
AccountDto              // Full account details
AccountListItemDto      // List view item
AccountSetDto           // Create/Update payload
AccountsSearchDto       // Search parameters

// Financial Types
AccountAggregatedTotalAmountsDto
AccountCostRequestDto / AccountCostResponseDto
AccountFundedAmountDto

// Status Types
AccountProgress         // Pending | Open | Closed
AccountStatus           // Open | Paid | Rejected | etc.
AccountSubStatus
AccountPTFStatus        // PTF tracking statuses
```

### Data Flow: UI → BFF

```
1. User fills form in CaseAccountAdd/Edit
                    ↓
2. Form validation via Yup schemas (accountSchema.ts)
                    ↓
3. getAccountMutationRequest() helper transforms data
                    ↓
4. RTK Query mutation (useCreateAccountMutation/useUpdateAccountMutation)
                    ↓
5. FormData sent to BFF: POST/PUT /api/Account
                    ↓
6. Response handled by useFormSubmit hook
                    ↓
7. Navigation to success page or error display
```

### Side Effects (Frontend)

| Effect | Trigger | Handler |
|--------|---------|---------|
| Loading States | API calls | RTK Query `isLoading`, `isFetching` |
| Error States | API failures | `submitResult.isError`, toast notifications |
| Cache Invalidation | Create/Update success | RTK Query tag invalidation |
| Document Upload | Form submission | FormData with multipart encoding |
| Duplicate Check | Before save | `useCheckDuplicateAccountMutation` |
| Cost Calculation | Field changes | `useLazyCalculateAccountCostQuery` |
| Procedure Linkage | Procedure selection | Updates linked billing facility, charges |

---

## Backend (Accounts Controller)

### BFF Layer

**File:** `bff/Phoenix.BFF/Controllers/AccountController.cs`

The BFF AccountController acts as an aggregation layer that:
- Calls Core API endpoints
- Enriches data with related entities (Facility, Case, Debtor, etc.)
- Handles document management
- Fills user details
- Applies UI-specific business logic

### Core API Layer

**File:** `core-api/Debtor/Phoenix.CoreAPI.Debtor/Controllers/AccountController.cs`

### Endpoints

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `GET` | `/api/Account/{id}` | Get account by ID |
| `POST` | `/api/Account/search` | Search accounts with pagination |
| `POST` | `/api/Account` | Create new account |
| `PUT` | `/api/Account/{id}` | Update account |
| `DELETE` | `/api/Account/{id}` | Delete account |
| `DELETE` | `/api/Account/{id}/unlink` | Unlink procedure from account |
| `POST` | `/api/Account/ProductLines/CaseIds` | Get product lines by case IDs |
| `POST` | `/api/Account/Search/Ids/Paginated` | Get account IDs paginated |
| `POST` | `/api/Account/GetAllIdsOnly` | Get all matching account IDs |
| `POST` | `/api/Account/GetAllIdsForCases` | Get account IDs for cases |
| `POST` | `/api/Account/GetAllParentIdsOnly` | Get parent account IDs |
| `POST` | `/api/Account/GetAllForRecon` | Get accounts for reconciliation |
| `GET` | `/api/Account/GetAllForIngestion` | Stream accounts for ingestion |
| `POST` | `/api/Account/AggregatedTotalAmounts` | Get aggregated totals |
| `POST` | `/api/Account/AggregatedTotalAmountsByAccount` | Per-account aggregates |
| `POST` | `/api/Account/FundedTotalAmount` | Total funded amount |
| `POST` | `/api/Account/FundedAmounts` | Funded amounts per account |
| `POST` | `/api/Account/BOSFundedAmounts` | Bill of Sale funded amounts |
| `POST` | `/api/Account/CreditAmounts` | Credit amounts |
| `POST` | `/api/Account/CostAmounts` | Cost amounts |
| `POST` | `/api/Account/BalanceRemainingAmounts` | Balance remaining |
| `POST` | `/api/Account/ReductionAmounts` | Reduction amounts |
| `POST` | `/api/Account/AssociatedWithBOS` | Check BOS association |
| `PUT` | `/api/Account/Bulk` | Bulk update accounts |
| `PUT` | `/api/Account/BulkStatusUpdate` | Bulk status update |
| `POST` | `/api/Account/Merge` | Merge accounts (case/facility merge) |
| `POST` | `/api/Account/Unmerge` | Unmerge accounts |
| `POST` | `/api/Account/Move` | Move accounts to different case |
| `GET` | `/api/Account/AppointmentRequestId/{id}` | Get by appointment request |
| `GET` | `/api/Account/Physicians` | Search physicians |
| `GET` | `/api/Account/Statuses` | Get available statuses |
| `GET` | `/api/Account/StatusToProgressMap` | Status to progress mapping |
| `GET` | `/api/Account/SecuritizationGroups` | Get securitization groups |
| `GET` | `/api/Account/RvpRegionsWithStates` | Get RVP regions |
| `GET` | `/api/Account/FilenameData/{id}` | Get filename metadata |
| `GET` | `/api/Account/BillingFacilityNames` | Get billing facility names |
| `POST` | `/api/Account/AccountContractTypes` | Get contract types for accounts |
| `POST` | `/api/Account/WriteOffAccounts` | Get write-off accounts |
| `POST` | `/api/Account/FirmIds` | Get unique firm IDs |
| `POST` | `/api/Account/CheckDuplicate` | Check for duplicate accounts |
| `GET` | `/api/Account/GetChargeId/{accountId}` | Get charge ID |
| `POST` | `/api/Account/Cost` | Calculate account cost |

### Request/Response DTOs

**Request DTOs:**

| DTO | Purpose |
|-----|---------|
| `AccountSetDto` | Create/Update account payload |
| `AccountSearchDto` | Search parameters with filters |
| `AccountBulkUpdateDto` | Bulk update fields |
| `AccountBulkUpdateSetDto` | Bulk status update |
| `AccountCheckDuplicateDto` | Duplicate check parameters |
| `AccountCostRequestDto` | Cost calculation request |
| `MergeDto` | Merge request |
| `UnmergeDto` | Unmerge request |
| `MultipleMoveDto` | Move accounts request |

**Response DTOs:**

| DTO | Purpose |
|-----|---------|
| `AccountDto` | Full account response |
| `AccountListItemDto` | List item for search results |
| `AccountAggregatedTotalAmountsDto` | Financial totals |
| `AccountFundedAmountDto` | Funded amount per account |
| `AccountCostResponseDto` | Cost calculation result |
| `AccountBulkUpdateResultDto` | Bulk update result |
| `AccountBulkStatusUpdateDto` | Bulk status result |
| `CreatedResponse<long>` | Create response with ID |

### Services

**IAccountService (Core):**

```csharp
// Primary operations
Task<Result<AccountEntity>> Get(long id);
Task<Result<PaginatedFilteredList<AccountEntity>>> GetAll(AccountSearchRequest request, CancellationToken ct);
Task<Result<long[]>> Insert(params AccountEntity[] entities);
Task<Result> SingleUpdate(AccountEntity entity);
Task<Result> DeleteAccount(long id, CancellationToken ct);

// Procedure operations
Task<Result> UnlinkProcedure(long id);
Task<Result<IList<AccountEntity>>> GetLinkedWithProcedure(long procedureId, IList<long> linkedBillingFacilityIds);

// Bulk operations
Task<Result<AccountBulkUpdateResultEntity>> BulkUpdate(AccountBulkUpdateRequest updateRequest, CancellationToken ct);
Task<Result<List<long>>> UpdateBulkProgress(IEnumerable<long> ids, AccountProgress SelectedProgress, AccountProgress UpdateProgress);

// Merge/Move operations
Task<Result> Merge(MergeRequest mergeRequest);
Task<Result> Unmerge(UnmergeRequest unmergeRequest);
Task<Result<EntitiesMovedResponse>> Move(MultipleMoveRequest request);

// Financial calculations
Task<AccountAggregatedTotalAmountsEntity> GetAggregatedTotalAmounts(AccountSearchRequest request, CancellationToken ct);
Task<Result<AccountCostResponseEntity>> CalculateAccountCost(AccountCostRequest request, CancellationToken ct);
Task<IList<AccountFundedAmountEntity>> GetFundedAmounts(IEnumerable<long> accountIds, CancellationToken ct);

// Validation
Task<Result<bool>> CheckDuplicate(CheckDuplicateAccountRequest request, CancellationToken ct);
```

**IAccountAutomationService:**

```csharp
// PTF status automation
Task UpdateAccountPTFStatusAfterPTFPaymentSave(IList<long> affectedAccountIds, DateTime now);
Task UpdateAccountPTFStatusAfterPTFReimbursementSave(IList<long> affectedAccountIds, DateTime now);
Task UpdateAccountPTFStatusAfterPTFRefundSave(IList<long> affectedAccountIds, DateTime now);
Task UpdateAccountPTFStatusAfterPTFRefundsReceivedSave(IList<long> affectedAccountIds, DateTime now);
Task UpdateAccountPTFStatusAfterPaymentTypeChange(IList<long> affectedAccountIds, DateTime now);
Task UpdateAccountPTFStatusOnPTFRecordDelete(IList<long> affectedAccountIds, DateTime now);

// Med Pay automation
Task UpdateAccountMedPayStatusAfterPTFPaymentOrRefund(IDictionary<long, bool> medPay);

// Status automation
Task UpdateAccountStatusBasedOnBalanceRemaining(IList<long>? affectedAccountIds, DateTime dateNow, bool updatePaidAccountsOnly = false);

// Pre-persist hooks
Task BeforeCreatePersist(AccountEntity entity);
Task BeforeUpdatePersist(AccountEntity originalEntity, AccountEntity entity, DateTime now);
```

### Repository Interface

**IAccountRepository:**

```csharp
// CRUD
Task<AccountEntity?> Get(long id);
Task<IList<AccountEntity>> GetByIds(IEnumerable<long> ids, CancellationToken ct);
Task<long> Insert(AccountEntity entity);
Task Update(AccountEntity entity);
Task Update(IEnumerable<AccountEntity> entities);
Task Delete(long id);

// Search
Task<Result<PaginatedFilteredList<AccountEntity>>> GetAll(AccountSearchRequest searchRequest, CancellationToken ct);
IAsyncEnumerable<AccountIngestionEntity> GetAllForIngestion(CancellationToken ct, AccountSearchIngestionRequest searchRequest);

// Financial queries
Task<AccountAggregatedTotalAmountsEntity> GetAggregatedTotalAmounts(AccountSearchRequest searchRequest, CancellationToken ct);
Task<decimal> GetFundedTotalAmount(IEnumerable<long> accountIds, CancellationToken ct);
Task<IList<AccountFundedAmountEntity>> GetFundedAmounts(IEnumerable<long> accountIds, CancellationToken ct);

// Relationship queries
Task<IList<AccountEntity>> GetLinkedWithProcedure(long procedureId, IList<long> linkedBillingFacilityIds);
Task<IList<AccountEntity>> GetAccountsByCaseIds(IList<long> caseIds);
Task<IList<AccountEntity>> GetAccountsByFacilityId(long id);
```

### Business Logic Flow

```
Create Account Flow:
1. Validate input (ValidationService)
2. Check duplicates if applicable
3. Set PTFStatusDate if PTFStatus provided
4. Verify/update ReasonForDenial based on feature flag
5. Run automation (BeforeCreatePersist)
   - Update ContractOwner from contract
   - Update State from service facility
6. Insert via Repository
7. Refetch entity for complete data
8. Update case progresses (CaseAutomationService)
9. Send messaging events (AccountMessagingService)
10. Return created ID

Update Account Flow:
1. Get original entity
2. Validate update (ValidationService)
3. Run automation (BeforeUpdatePersist)
   - Handle PTF status changes
   - Update DateReleased for progress changes
   - Update ContractOwner
   - Update State
4. Update via Repository
5. Refetch updated entity
6. Handle write-off case status updates
7. Send messaging events
8. Update case progresses
```

---

## System-Level Side Effects

### Messaging Service Events

**File:** `core-api/Debtor/Phoenix.CoreAPI.Debtor/Messaging/AccountMessagingService.cs`

The system uses **MassTransit** for message publishing. Events are published on:

| Event | Trigger | Payload |
|-------|---------|---------|
| `EntityCreated` | Account insert | `AccountSnapshot` with full entity data |
| `EntityUpdated` | Account update | Old and new `AccountSnapshot` |
| `EntityMoved` | Account move | Entity ID, old/new parent IDs |
| `EntityDeleted` | Account delete | Entity ID, timestamp |

**AccountSnapshot** includes enriched data:
- CPT code lookups
- Modality descriptions
- Service place names
- CPT modifier names
- Underwriter user names

### Event Consumers (Pub/Sub)

Messages are consumed by:

| Consumer | Purpose |
|----------|---------|
| Audit Service | Records all account changes in audit log |
| History Service | Maintains change history |
| Search/Indexing | Updates search indexes |
| Notification Service | Triggers notifications |

### Audit Logging

All account operations are automatically audited via:
- `AuditedDbContext` base class
- Change tracking on create/update/delete
- User context capture (`ICurrentUser`)
- Timestamp recording

### Cross-Service Interactions

| Service | Interaction |
|---------|-------------|
| **Case Service** | Updates case progress when accounts change |
| **Procedure Service** | Links/unlinks accounts to procedures |
| **Payment Service** | PTF payment status updates |
| **Reduction Service** | Applies reductions to accounts |
| **Document Service** | Manages account documents |
| **Facility Service** | Gets facility/contract details |
| **Write-Off Service** | Handles write-off processing |
| **MoveDocs** | Appointment status updates |

### Background Jobs

| Job | Trigger | Action |
|-----|---------|--------|
| PTF Status Update | Payment changes | Updates account PTF status |
| Case Progress Update | Account changes | Recalculates case progress |
| Balance Recalculation | Payment/Reduction changes | Updates balance remaining |

### Database Triggers

The system uses EF Core change tracking rather than database triggers. Calculated fields are updated:
- `BilledAmount` - Sum of charge amounts
- `CostAmount` - Sum of cost calculations
- `CollectedAmount` - Sum of payments
- `BalanceRemaining` - Calculated from above

### Error Propagation

```
Frontend → BFF → Core API → Repository
    ↓         ↓         ↓
  Toast   ValidationError  FluentResults Error
  Display  Response       with ErrorCode
```

Error codes are defined in `ErrorCode` enum and mapped to HTTP status codes:
- 400 Bad Request - Validation errors
- 404 Not Found - Entity not found
- 409 Conflict - Business rule violations
- 500 Server Error - Unexpected errors

---

## Database Tables

### Primary Account Tables

| Table Name | Schema | Description | CRUD Ops | Notes |
|------------|--------|-------------|----------|-------|
| `Accounts` | debtor | Main account records | CRUD | Primary account entity |
| `AccountCharges` | debtor | Account charge records | CRUD | One-to-many with Accounts |
| `AccountChargeCpts` | debtor | CPT codes per charge | CRUD | One-to-many with AccountCharges |
| `AccountPayments` | debtor | Payment allocations to accounts | CRUD | Links Payments to Accounts |
| `AccountPTFPayments` | debtor | PTF payment allocations | CRUD | Links PTFPayments to Accounts |
| `AccountPTFRefunds` | debtor | PTF refund allocations | CRUD | Links PTFRefunds to Accounts |
| `AccountBillOfSales` | debtor | BOS associations | CRUD | Links Accounts to Bill of Sales |
| `AccountReductions` | debtor | Reduction associations | CRUD | Links Accounts to Reductions |
| `AccountCalculatedAmounts` | debtor | Calculated financial amounts | R | Computed amounts view |

### Related Tables

| Table Name | Schema | Description | Relationship | Notes |
|------------|--------|-------------|--------------|-------|
| `Cases` | debtor | Parent case records | Account.CaseId → Cases.Id | Required FK |
| `Facilities` | debtor | Billing/Service facilities | Account.BillingFacilityId, ServiceFacilityId | Optional FK |
| `Contracts` | debtor | Facility contracts | Account.FacilityContractId | Optional FK |
| `Procedures` | debtor | Medical procedures | Account.ProcedureId | Optional FK |
| `Payments` | debtor | Incoming payments | Via AccountPayments | Many-to-many |
| `PTFPayments` | debtor | PTF payment records | Via AccountPTFPayments | Many-to-many |
| `PTFRefunds` | debtor | PTF refund records | Via AccountPTFRefunds | Many-to-many |
| `Reductions` | debtor | Reduction records | Via AccountReductions | Many-to-many |
| `BillOfSales` | debtor | Bill of Sale records | Via AccountBillOfSales | Many-to-many |
| `Documents` | document | Account documents | Via document service | External module |
| `WriteOffs` | debtor | Write-off records | Account.WriteOffId | Optional FK |

### Lookup/Enum Tables

| Table Name | Description |
|------------|-------------|
| `AccountStatuses` | Status enum values |
| `AccountSubStatuses` | Sub-status enum values |
| `AccountClasses` | Account class values |
| `AccountPTFStatuses` | PTF status enum values |
| `AccountProgresses` | Progress enum values |
| `SiteLocationTypes` | Location type enum |
| `CostTypes` | Cost type enum |
| `ServicingTypes` | Servicing type enum |
| `ProductLines` | Product line enum |
| `AccountRejectionUWPassedReasons` | Rejection reason enum |

### Key Entity Fields

**AccountRecord (Primary Entity):**

```csharp
// Identity
long Id
long CaseId
string AccountNumber

// Facility References
long? BillingFacilityId
long? ServiceFacilityId
long? LinkedBillingFacilityId
long? FacilityContractId
long? ContractVersionId
SiteLocationType? SiteLocationType
string? Zip, City
State? State

// Status Fields
AccountProgress Progress
AccountStatusNew Status
AccountSubStatus? SubStatus
AccountPTFStatus? PTFStatus
DateTime? PTFStatusDate

// Financial
decimal BilledAmount
decimal CostAmount
decimal CollectedAmount
decimal WriteOffAmount
decimal BalanceRemaining
decimal FundedAmount

// Rate Fields
decimal? CostRate, CostFee
CostType? CostType
decimal? Cap
decimal? InterestRate
decimal? ProviderRate
decimal? ContractOwnerRate

// Dates
DateOnly? DateOfService
DateOnly? PaymentDueDate
DateTime DateCreated
DateTime? DateReleased

// Other
long? ProcedureId
long? ReductionId
long? WriteOffId
long? AppointmentId
int CreatedByUserId
int? UnderwriterUserId
```

---

## Data Flow

### Complete Flow: UI → BFF → Core API

```
┌──────────────────────────────────────────────────────────────────────────┐
│ 1. USER ACTION                                                            │
│    User fills account form and clicks Save                                │
└────────────────────────────────────┬─────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 2. FRONTEND VALIDATION                                                    │
│    • Yup schema validation (activeAccountSchema)                          │
│    • Duplicate check (useCheckDuplicateAccountMutation)                   │
│    • Charges exceed confirmation dialog                                   │
└────────────────────────────────────┬─────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 3. REQUEST TRANSFORMATION                                                 │
│    • getAccountMutationRequest() helper                                   │
│    • FormData creation with documents                                     │
│    • RTK Query mutation trigger                                           │
└────────────────────────────────────┬─────────────────────────────────────┘
                                     │ POST /api/Account (FormData)
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 4. BFF CONTROLLER                                                         │
│    • AccountController.Create()                                           │
│    • Document validation                                                  │
│    • Map to CoreAPI.AccountSetDto                                         │
│    • Call CoreApiClient.AccountCreateAsync()                              │
│    • Document service upload                                              │
│    • Filename generation if auto-rename enabled                           │
└────────────────────────────────────┬─────────────────────────────────────┘
                                     │ POST /api/Account (JSON)
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 5. CORE API CONTROLLER                                                    │
│    • AccountController.Create()                                           │
│    • Permission check (HasPermission attribute)                           │
│    • Map to AccountEntity                                                 │
│    • Call IAccountService.Insert()                                        │
└────────────────────────────────────┬─────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 6. ACCOUNT SERVICE                                                        │
│    • Set PTFStatusDate if applicable                                      │
│    • Verify ReasonForDenial based on feature flag                         │
│    • Validate via IValidationService                                      │
│    • Call IAccountAutomationService.BeforeCreatePersist()                 │
│      - Update ContractOwner                                               │
│      - Update State from ServiceFacility                                  │
│    • Insert via IAccountRepository                                        │
└────────────────────────────────────┬─────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 7. REPOSITORY (Entity Framework)                                          │
│    • Insert AccountRecord                                                 │
│    • Insert AccountChargeRecords                                          │
│    • Insert AccountChargeCptRecords                                       │
│    • SaveChanges with audit tracking                                      │
└────────────────────────────────────┬─────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 8. POST-INSERT OPERATIONS                                                 │
│    • Refetch entity for complete data                                     │
│    • ICaseAutomationService.UpdateCasesProgresses()                       │
│    • IAccountMessagingService.SignalEntitiesCreated()                     │
│      - Publish to message bus                                             │
│      - Audit logging                                                      │
└────────────────────────────────────┬─────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 9. RESPONSE FLOW                                                          │
│    Core API → CreatedResponse<long>                                       │
│    BFF → CreatedResponse<long>                                            │
│    Frontend → Navigate to account view page                               │
│             → Show success toast                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

### Important Files

| Category | Key Files |
|----------|-----------|
| Frontend Pages | `CaseAccountAdd.tsx`, `CaseAccountEdit.tsx`, `CaseAccountView.tsx` |
| Validation | `accountSchema.ts` |
| API Client | `store/bff-api/accounts/accounts.ts` |
| BFF Controller | `bff/Phoenix.BFF/Controllers/AccountController.cs` |
| Core Controller | `core-api/Debtor/Phoenix.CoreAPI.Debtor/Controllers/AccountController.cs` |
| Service | `core-api/Debtor/Phoenix.CoreAPI.Debtor.Domain/Services/AccountService.cs` |
| Automation | `AccountAutomationService.cs` |
| Messaging | `AccountMessagingService.cs` |
| Repository | `core-api/Debtor/Phoenix.CoreAPI.Debtor/Data/Repositories/AccountRepository.cs` |
| Entity | `core-api/Debtor/Phoenix.CoreAPI.Debtor.Domain/Entities/AccountEntity.cs` |
| DB Context | `core-api/Debtor/Phoenix.CoreAPI.Debtor/Data/DebtorContext.cs` |

### Common Operations

| Operation | Frontend Hook | BFF Endpoint | Core Endpoint |
|-----------|--------------|--------------|---------------|
| Get Account | `useGetAccountQuery(id)` | `GET /Account/{id}` | `GET /api/Account/{id}` |
| Create | `useCreateAccountMutation` | `POST /Account` | `POST /api/Account` |
| Update | `useUpdateAccountMutation` | `PUT /Account/{id}` | `PUT /api/Account/{id}` |
| Delete | `useRemoveAccountMutation` | `DELETE /Account/{id}` | `DELETE /api/Account/{id}` |
| Search | `useLazyGetAccountAllQuery` | `POST /Account/Search` | `POST /api/Account/search` |
| Bulk Update | `useUpdateAccountBulkMutation` | `PUT /Account/Bulk` | `PUT /api/Account/Bulk` |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-12-03 | Initial documentation |

---

*This documentation is auto-generated based on codebase analysis. For the latest information, refer to the source code.*
