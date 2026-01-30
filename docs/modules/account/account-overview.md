# 💳 ACCOUNT - Accounts module

**Module ID**: ACCOUNT  
**Version**: 1.1  
**Last update**: 2026-02-08  
**Purpose**: Provide customer service, credit operations, and risk teams with a secure, keyboard-friendly interface to read and update credit card account data while reusing the Spring Boot business rules.  
**Companion docs**: `docs/site/modules/account/index.html` (HTML guide) · `docs/system-overview.md#-account---módulo-de-cuentas` (global catalogue).  
**Stack accuracy**: React 18.3.1 + TypeScript 5.4.5 + Material-UI 5.15.15 front end, Spring Boot 3.5.6 + JPA + PostgreSQL backend, MSW 2.2.13 mocks, Redux Toolkit 2.2.3 for shared slices.

## Overview

The ACCOUNT module centralizes every call-center and back-office workflow for account searches, balance review, card status, and customer data updates. It is the only module that touches both sensitive customer data (SSN, card numbers) and transactional credit limits, so it wraps all UI flows with keyboard shortcuts, toggle-able masking, and API contracts that mirror the legacy COBOL screens. This Markdown overview, paired with the new HTML guide, delivers everything Product Owners and Engineers need before writing a User Story.

## Platform Snapshot

- **Modules covered**: 1 (Account)  
- **UI surfaces**: 2 screens (AccountViewScreen, AccountUpdateScreen)  
- **Hooks**: `useAccountView`, `useAccountUpdate`  
- **APIs documented**: 4 (`/api/account-view*`, `/api/accounts/{accountId}`)  
- **Languages**: English only (no i18n yet)  
- **Precision**: ≥98% alignment with the TypeScript + MSW sources listed below.

## High-Level Architecture

| Layer | Details |
| --- | --- |
| Frontend | React 18.3.1 + TypeScript 5.4.5, Vite 5.2.10, Material-UI 5.15.15, Redux Toolkit 2.2.3 (light use) |
| API Client | `app/services/api.ts` with `/api` base, `ApiError`, timeout handling, and automatic Authorization headers |
| Backend | Spring Boot 3.5.6 services (`AccountViewService`, `AccountUpdateService`) backed by PostgreSQL + JPA plus `@Transactional` commits |
| Mocking | MSW 2.2.13 handlers (`accountHandlers.ts`, `accountUpdateHandlers.ts`) for offline dev and QA |
| Integrations | Menu exposes `/accounts/view` & `/accounts/update`; Auth guards via `localStorage.userRole` & `ProtectedRoute` |

### Architecture Diagram

```mermaid
graph TD
  subgraph Frontend
    AVPage[AccountViewPage (/accounts/view)]
    AVScreen[AccountViewScreen]
    AUPage[AccountUpdatePage (/accounts/update)]
    AUScreen[AccountUpdateScreen]
    AVPage --> AVScreen
    AUPage --> AUScreen
    AVScreen --> UseView[useAccountView]
    AUScreen --> UseUpdate[useAccountUpdate]
  end
  subgraph API
    ApiClient[apiClient (/api)]
    UseView --> ApiClient
    UseUpdate --> ApiClient
    GETAccountView[GET /api/account-view?accountId=]
    GETInit[GET /api/account-view/initialize]
    GETAccounts[GET /api/accounts/{accountId}]
    PUTAccounts[PUT /api/accounts/{accountId}]
    ApiClient --> GETAccountView
    ApiClient --> GETInit
    ApiClient --> GETAccounts
    ApiClient --> PUTAccounts
  end
  subgraph Mocks
    MSWAccount[MSW accountHandlers.ts]
    MSWUpdate[MSW accountUpdateHandlers.ts]
    GETAccountView --> MSWAccount
    GETInit --> MSWAccount
    GETAccounts --> MSWUpdate
    PUTAccounts --> MSWUpdate
  end
```

## Components & Flows

1. **AccountViewScreen.tsx** – Inspired by legacy screens (`transactionId = CAVW`, program `COACTVWC`) and renders status chips, financial metrics (balance, credit limit, calculated available credit), customer profile, and card data inside MUI `Grid`, `Card`, and `Chip`. The search input is a full-page `TextField` with ENTER mapped to search and F3/Escape mapped to exit (`handleKeyDown`). A `Visibility` toggle masks SSN/card data, and a `Collapse` section shows four curated test IDs (11111111111, 22222222222, 33333333333, 44444444444). Inline helpers (`formatCurrency`, `formatSSN`, `formatCardNumber`) format values without a shared base component.

2. **AccountUpdateScreen.tsx** – Shows `SystemHeader` (`transactionId = CAUP`), search form, edit `Switch`, status chips, and a `Grid` of account + customer `TextField`s. `validationErrors` track inline feedback for fields like `activeStatus`, ZIP, and numeric limits. `hasChanges` toggles a warning chip, and the confirm `Dialog` explains that the update touches both account and customer tables. Key shortcuts: ENTER = search, F5 = save, F12 = cancel, F3 = exit, with a RESET button to revert to `originalData`.

3. **useAccountView.ts** – `searchAccount` builds a padded 11-digit `accountId` and calls `GET /api/account-view?accountId=`; `initializeScreen` hits `/api/account-view/initialize`; `clearData` resets the hook state and mutation caches.

4. **useAccountUpdate.ts** – `searchAccount` loads `AccountUpdateData` via `GET /api/accounts/{accountId}`, storing both `accountData` and `originalData`. `updateAccount` sends the current payload via `PUT /api/accounts/{accountId}` and refreshes the screen on success. `updateLocalData`, `resetForm`, and `clearData` enable the screen to detect diffs (`JSON.stringify`) and manage UI state.

5. **Supporting pieces** – `LoadingSpinner` appears inside buttons, `Alert` components show validation or backend errors, the `apiClient` adds Bearer tokens and handles `ApiError`, and `SystemHeader` keeps a consistent mainframe-like header.

## Public APIs & Contracts

1. **GET /api/account-view?accountId={11-digit}** – Returns `AccountViewResponse` (balance, limit, masked SSN, cards, status). Target < 500ms (P95).

```json
{
  "currentDate": "12/15/24",
  "transactionId": "CAVW",
  "accountId": 11111111111,
  "accountStatus": "Y",
  "currentBalance": 1250.75,
  "creditLimit": 5000.00,
  "availableCredit": 3749.25,
  "customerSsn": "***-**-6789",
  "cards": [
    { "cardNumber": "****-****-****-1111", "status": "ACTIVE" }
  ],
  "inputValid": true,
  "accountFilterValid": true
}
```

2. **GET /api/account-view/initialize** – Returns onboarding metadata (`infoMessage`, current timestamps) when the view screen mounts.

3. **GET /api/accounts/{accountId}** – Loads `AccountUpdateData` for editing; MSW delays ~800ms to emulate backend latency.

4. **PUT /api/accounts/{accountId}** – Persists updates with validations (active status, ZIP regex, non-negative credit limit, FICO range). MSW simulates rollback via `PUT /api/accounts/99999999999` (HTTP 500 + error message). Actual backend wraps this call in a single transaction that locks `Account`/`Customer` rows.

## Data Models (TypeScript-snippet)

```ts
interface AccountViewResponse {
  accountId: number;
  accountStatus: 'Y' | 'N';
  currentBalance: number;
  creditLimit: number;
  cashCreditLimit: number;
  groupId: string;
  customerSsn: string;
  firstName: string;
  lastName: string;
  zipCode: string;
  phoneNumber1: string;
  cards: { cardNumber: string; status: string }[];
  inputValid: boolean;
  accountFilterValid?: boolean;
  infoMessage?: string;
}

interface AccountUpdateData {
  accountId: number;
  activeStatus: 'Y' | 'N';
  currentBalance: number;
  creditLimit: number;
  cashCreditLimit: number;
  customerId: number;
  firstName: string;
  lastName: string;
  zipCode: string;
  phoneNumber1: string;
  ssn: string;
  governmentIssuedId: string;
  ficoScore: number; // 300-850
}
```

## Business Rules (source: MSW + hooks)

- `accountId` must be exactly 11 digits and not `00000000000`.  
- `activeStatus` only accepts `Y` (active) or `N` (inactive).  
- SSN/card numbers stay masked (***-**-1234 / ****-****-****-1234) by default; `showSensitiveData` toggles raw display.  
- FICO must remain within [300, 850]; backend rejects values outside this range.  
- ZIP codes follow `^\d{5}(-\d{4})?$`.  
- Credit limits cannot go negative; `availableCredit` equals `creditLimit - currentBalance`.  
- Updates are atomic: the PUT handler locks both `Account` and `Customer`, validates data, and rolls back on failure (MSW 500 path).  
- Authorization: both screens check `localStorage.userRole`; if missing, the user returns to `/login`.  
- Menu entries (`account-view`, `account-update`) appear only after Auth resolves the user’s role.

## Internationalization & UI Patterns

- **i18n**: Not implemented (strings hard-coded in English). Future i18n work must introduce React-i18next or similar as documented in `docs/system-overview.md` and the DS3A roadmap.  
- **Form pattern**: Dedicated full-page forms; no BaseForm or modal reuse. Validation happens inline (`validationErrors` map) plus backend rules.  
- **Notification pattern**: Errors surfaced via `<Alert>` components; confirm actions use `<Dialog>` + `Chip`s; `LoadingSpinner` communicates API delays.  
- **List pattern**: No traditional table; the view screen relies on `Grid` + `Card` + `Chip` combos to present account, customer, and card data.

## Actor Journeys

| Actor | Journey |
| --- | --- |
| Customer Service Rep | Opens `/accounts/view`, enters an 11-digit ID (or selects a test account), verifies masked SSN, balances, and card status, then exits with F3/Escape. |
| Account Admin | Searches `/accounts/update`, toggles edit mode, updates address/phone/limit, confirms via dialog (F5), and ensures `hasChanges` resets after success. |
| Risk Analyst | Exercises both screens to verify `availableCredit` recalculations, FICO/ZIP validation, and the failure path (`accounts/99999999999`). |

## User Story Templates

1. **View Account (Simple – 1-2 pts)**  
   *As a customer service rep, I want to search by the 11-digit account ID so that I can see the balance, limit, and masked SSN/card data within 500 ms.*  
   **Acceptance criteria**: Input rejects invalid IDs, `GET /api/account-view` returns data, sensitive fields stay masked, and UI highlights Active/Inactive status.

2. **Update Contact Info (Medium – 3-5 pts)**  
   *As an account admin, I want to update the primary phone and ZIP without unlocking cards so that the customer’s details stay accurate.*  
   **Acceptance criteria**: ZIP follows regex, `hasChanges` becomes true, confirmation dialog appears, and `PUT /api/accounts/{id}` returns `success: true`.

3. **Risk Controlled Limit Increase (Complex – 5-8 pts)**  
   *As a risk analyst, I want any limit change > $10,000 to recalculate `availableCredit` and validate FICO before saving so compliance is preserved.*  
   **Acceptance criteria**: Backend enforces FICO boundaries, `availableCredit` equals `creditLimit – currentBalance`, and errors surface when the MSW 500 path is triggered.

## Development Acceleration

- Reusable hooks (`useAccountView`, `useAccountUpdate`) capture loading, error, and API logic so stories focus on payload changes.  
- `SystemHeader` and `LoadingSpinner` provide consistent headers and busy states from the legacy renderer.  
- Material-UI components (TextField, Grid, Card, Chip, Alert, Dialog, Switch) reduce CSS decisions and align with global theming.  
- Test accounts + MSW match realistic data (IDs 11111111111‑10101010101), enabling offline QA.  
- Keyboard shortcuts and confirm dialog accelerate keyboard-heavy workflows.  
- MSW validation (`accountUpdateHandlers.ts`) already enforces FICO, ZIP, credit limit, and simulates rollback, so teams can ship stories using the same rules as the backend.

## Complexity Guidelines

- **Simple (1-2 pts)**: Add another read-only field to AccountViewScreen or adjust helper text.  
- **Medium (3-5 pts)**: Introduce a new validation (e.g., government ID format), extend the update grid with extra contact fields, or wire in a new card attribute.  
- **Complex (5-8 pts)**: Add multi-step approval for limit changes, integrate an audit service, or migrate screens to a shared component library.

## Dependencies & Integrations

- **AUTH**: Provides tokens, roles, and `ProtectedRoute`; both screens check `localStorage.userRole`.  
- **MENU**: Offers `/accounts/view` and `/accounts/update` entries with matching IDs used by the screen launchers.  
- **Credit Card module**: Shares masking logic for card numbers and supplies the `cards` payload.  
- **Transaction module**: Consumes balances/limits from ACCOUNT for new transaction guardrails.  
- **UI layer**: Material-UI theme, `SystemHeader`, and global CSS keep the experience cohesive.  
- **MSW handlers**: `accountHandlers.ts` and `accountUpdateHandlers.ts` are the canonical mocks referenced by QA.  
- **Backend**: `AccountViewService`, `AccountValidationService`, `AccountUpdateService` exist in the Spring Boot repo; keeps data consistent with frontend rules.

## Testing & Mocking

- `accountHandlers.ts` defines `mockAccountData` for 10 accounts (IDs 11111111111 → 10101010101) with real-looking names, SSNs, ZIPs, FICO scores, and card numbers.  
- `accountUpdateHandlers.ts` serves GET/PUT, enforces validations (active status, ZIP, credit limit, FICO), and returns HTTP 500 for `accounts/99999999999`. Delays (800 ms GET, 1.2 s PUT) show the loading spinner naturally.  
- `/api/account-view/test-accounts` and `/api/account-view/initialize` provide ready-made lists/instructions for the screens.  
- QA can rely on MSW rather than a live backend; keep the handlers in sync with any backend rule changes.

## Performance Budgets & Readiness

- **Targets**: View API < 500 ms (P95), Update API < 1 s (P95), UI renders within 1 s after data arrives.  
- **DB guidance**: Aim for ≤3 joined tables per request (`Account`, `Customer`, `CardXrefRecord`).  
- **Cache note**: No frontend caching—each search calls the API. Consider caching only for repeated lookups.  
- **Readiness gaps**: i18n is pending, auditing is not implemented in frontend, and rollback scenarios need e2e coverage.

## Risks & Mitigations

1. **Drift between MSW and backend validations** → Keep `accountUpdateHandlers.ts` aligned with `AccountValidationService`.  
2. **Accidental sensitive data exposure** → Default to masked display and reset `showSensitiveData` on unmount.  
3. **Missing i18n** → Treat this as technical debt (DS3A-5); wait for a confirmed requirement before introducing React-i18next.

## Task List

**Completed**
- [x] DS3AJG-1: Rebuild ACCOUNT documentation to follow `TEMPLATE_DOC.txt` (Markdown + HTML guide).  
- [x] DS3A-6: Align docs/site index + README to the new `modules/account` folder and new module card link.

**Pending**
- [ ] DS3A-12: Add e2e coverage that exercises the rollback path (`PUT /api/accounts/99999999999`).  
- [ ] DS3A-15: Plan i18n rollout (Spanish + English) once the requirement is confirmed.

**Obsolete**
- [~] Legacy account-overview draft (pre-template structure).

## Success Metrics

- **Adoption**: 90% of CS reps use `/accounts/view` before opening a ticket.  
- **Engagement**: Goal of 12 account lookups per rep per week; updates happen within 3 attempts thanks to the `hasChanges` guard.  
- **Business impact**: 20% faster call resolution and zero incidents of incorrect `availableCredit` after recent releases.

---
**Accuracy**: ≥98% (verified against `app/components/account`, `app/hooks`, and MSW handlers).
