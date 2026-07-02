# Changelog — MaGuru_MonoChast

## 1.1.0 — 2026-07-02

### Added
- `Checkout Button Label` config field — text on the checkout submit button was hardcoded as "Pay in installments" with no admin control, unlike MonoPayment's equivalent setting; now configurable per store view
- **View Guarantee Letter Data (JSON)** button in admin order view — `ChastOrderService::getGuaranteeLetterData()` (structured customer PII, passport, bank agreement) existed only as an unreachable service method; added `Controller/Adminhtml/Order/GuaranteeLetterData` and a button next to the existing PDF download
- `ChastOrderState` / `ChastOrderSubState` source models — human-readable labels (and recommended-action hints for FAIL sub-states) for raw Monobank status codes like `EXCEEDED_SUM_LIMIT`, sourced from Monobank's official status reference table; wired into the order view block, the admin grid (now select/filterable columns), and order comment history

### Fixed
- **"Monobank Частинами" section never rendered in admin order view** — `ChastInfo::getOrder()` read block `data('order')`, which nothing ever set, instead of the `Registry` `current_order` key used by `MonoPayment`'s equivalent block
- **Download Guarantee Letter 404** — `getGuaranteeLetterUrl()` built the action URL as `guarantee_letter`; Magento's action router converts underscores in the full route path to namespace separators, so it looked for a nested `Guarantee\Letter` controller instead of the real `GuaranteeLetter` class

## 1.0.10 — 2026-07-02

> Note: `composer.json` `version` had drifted out of sync with this file since 1.0.1 (stuck at `1.0.0` while releases up to 1.0.9 already shipped). This release corrects `composer.json` to match reality in addition to the changes below.

### Changed
- `Chast` config group (Environment, Store ID/Secret for production/sandbox/stage, Chast Callback URL note) moved here from `MonoCore` — same admin section/path, same config values; only appears once `MonoChast` is installed
- Added missing DocBlocks to `_construct()` overrides, `configure()`/helper methods in Console Commands, `Cron::execute()`, `OrderStatusHandler` private methods, `Callback::verifySignature()`, `ChastInfo::getOrder()` per project DocBlock conventions
- Documented required message queue consumer setup (`app/etc/env.php` → `cron_consumers_runner`) in README — without it, order confirmation on shipment silently never reaches Monobank

## 1.0.9 — 2026-05-19

### Added
- `OrderStatusHandlerTest::testHandleSuccessWithDoneSubStateUpdatesOrderToProcessing` — verifies `STATE_SUCCESS` + `SUB_STATE_DONE` applies `STATE_PROCESSING` (substate-agnostic branch) (+1 test)
- `ChastOrderServiceTest::testCreateThrowsWhenClientThrows` — `ApiException` from `client->post()` propagates through `create()` without swallowing (+1 test)
- `ChastInfoTest` type fix: `$block` property declared as `ChastInfo&MockObject`; positive URL tests updated to use `expects($this->any())->method('getUrl')` (PHPStan Level 8 compliance)

## 1.0.8 — 2026-05-19

### Added
- `ChastInfo` positive URL tests: `getConfirmUrl()`, `getRejectUrl()`, `getGuaranteeLetterUrl()` success paths (3 tests)

## 1.0.7 — 2026-05-19

### Added
- `ChastOrderService::getGuaranteeLetterData(string $monoOrderId, string $version = '1', string $invoiceNumber = '', string $invoiceDate = ''): array` — structured JSON data for guarantee letter; `version = '2'` targets `/api/v2/order/data/for/guarantee/letter`, default targets `/api/order/data/for/guarantee/letter`; optional `invoice.number` / `invoice.date` fields
- 3 unit tests: V1 endpoint, V2 endpoint, invoice params included when provided

## 1.0.6 — 2026-05-19

### Added
- `Controller/Adminhtml/Order/Confirm` — admin action to confirm BNPL delivery (`POST /api/order/confirm`); empty `mono_order_id` → redirect with error; `ApiException` → redirect with error; success → redirect with success message
- `Controller/Adminhtml/Order/Reject` — admin action to reject BNPL order (`POST /api/order/reject`); identical pattern to Confirm
- **Confirm Delivery** and **Reject Order** buttons in admin order view (`chast_info.phtml`) — conditional visibility based on state/sub-state
- 8 unit tests: 4 per controller (constant, empty ID, success, ApiException)

## 1.0.5 — 2026-05-19

### Added
- `ChastClientInterface::postRaw(string $path, array $data): string` — same HMAC auth as `post()` but returns raw response bytes (for binary/PDF responses)
- `ChastOrderService::downloadGuaranteeLetter(string $monoOrderId, string $invoiceNumber = '', string $invoiceDate = ''): string` — PDF bytes from `POST /api/order/guarantee/letter`
- `Controller/Adminhtml/Order/GuaranteeLetter` — admin action to download PDF; returns `application/pdf` with `Content-Disposition: attachment`
- `Block/Adminhtml/Order/ChastInfo`: instance-caching (prevents multiple repository hits per page load); `getGuaranteeLetterUrl(): ?string`; `getConfirmUrl(): ?string` (only when `WAITING_FOR_STORE_CONFIRM`); `getRejectUrl(): ?string` (only when `IN_PROCESS`)
- `ChastOrderService::getOrderData()` admin view uses `POST /api/order/data`
- Multi-environment config (P30): `environment` select (sandbox / stage / production); per-env `store_id` + `secret`; correct Base URLs per environment
- **Download Guarantee Letter** button in admin order view

## 1.0.4 — 2026-05-18

### Added
- `Controller/Adminhtml/Order/IndexTest` (2 tests) — ADMIN_RESOURCE constant + execute() returns page; uses `$resultPageFactory` (different property name vs MonoPayment); `Magento\Framework\View\Result\Page` (not Backend); no setActiveMenu

## 1.0.3 — 2026-05-18

### Added
- `Block/InfoTest` (3 tests): coverage for `getSpecificInformation()` — includes/omits `mono_order_id` and `mono_chast_parts`

## 1.0.2 — 2026-05-18

### Added
- `ChastOrderServiceInterface::checkPaid(string $monoOrderId): bool` — checks if all installments are paid (`POST /api/order/check/paid`)
- Response: `fully_paid` field; works only for orders in `SUCCESS` state
- 3 new unit tests for `checkPaid()`

## 1.0.1 — 2026-05-18

### Added
- `ChastOrderServiceInterface::getOrderData(string $monoOrderId): array` — new method for `POST /api/order/data` (detailed order info: pay_parts, commissions, status history)
- `ChastOrderService::getOrderData()` — implementation
- 1 new unit test `testGetOrderDataCallsCorrectEndpoint`

## 1.0.0 — 2026-05-17

### Added
- Payment method `mono_chast` via `Magento\Payment\Model\Method\Adapter`
- Checkout UI — installment part selector + phone input; Knockout.js renderer
- Waiting screen (`GET /mono/chast/wait`) with AJAX polling (`GET /mono/chast/poll`) every 3 sec, max 5 min
- `ChastOrderService` — create, getState, confirm, reject, return, getReport, validateClient
  - `create()` — `POST /api/order/create` with `total_sum` in UAH (decimal, NOT kopecks)
  - `validateClient()` — `POST /api/v2/client/validate`; returns `bool`
  - `getReport()` — `POST /api/store/report` for daily reconciliation
- `ChastOrderRepository` — CRUD + `getByMonoOrderId`
- `ChastOrderManagement` — `getForOrder`, `syncState`, `confirm`, `reject`
- `OrderStatusHandler` — maps Monobank state/sub_state → Magento order state; dispatches `mono_chast_order_success`
- **Callback controller** `POST /mono/chast/callback` — HMAC-verified; always HTTP 200
- **Client validation controller** `POST /mono/chast/validate` — fail-open; exposes `client_validate_enabled` + `validate_url` to checkout JS
- **Observers** — ShipmentSaveAfter (auto-confirm), CancelAfter (reject), CreditMemoSaveAfter (return in UAH)
- **Crons** — `mono_chast_status_check` (`*/2 * * * *`); `mono_chast_report_sync` (`0 9 * * *`)
- CLI command `mono:chast:report [--date=YYYY-MM-DD]`
- Admin grid **Chast Orders** (`System → Monobank → Chast Orders`)
- DB table `mono_chast_order` (`total_sum` DECIMAL 12,2 UAH)
- Config: `available_parts`, `auto_confirm`, `client_validate`, `debug`
- Virtual logger → `var/log/mono_chast.log`
- **Admin order view block** `Block/Adminhtml/Order/ChastInfo` — shows state, sub_state, installment parts, total (UAH) in order view
- **Checkout ConfigProvider** exposes `client_validate_enabled` + `validate_url` to frontend JS
- 113 unit tests, PHPStan Level 8
- `Gateway/Config/Config` — unit tests: getAvailableParts CSV→int[], default fallback, isAutoConfirmEnabled, isClientValidateEnabled
- `Controller/Chast/Wait` — unit tests: no order → redirect, order present → page
