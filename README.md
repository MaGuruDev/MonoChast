# MaGuru MonoChast — Monobank "Частинами" BNPL for Magento 2

Adds Monobank "Частинами" (Buy Now Pay Later) as a standalone payment method in Magento Checkout. Customers choose 3, 6, 10, or 12 equal interest-free installments and approve directly in the Monobank app.

**Requires:** `maguru/module-mono-core`  
**Separate Monobank contract required** — contact Monobank Business to enable "Частинами" for your merchant account.

> ⚠️ Amounts in Chast API are in **UAH (decimal)**, NOT kopecks. This module handles the distinction automatically.

---

## Requirements

| Component | Version |
|-----------|---------|
| Magento Open Source / Adobe Commerce | 2.4.4+ |
| PHP | 8.1+ |
| MaGuru MonoCore | ^1.0 |

---

## Installation

```bash
composer require maguru/module-mono-core maguru/module-mono-chast
bin/magento module:enable MaGuru_MonoCore MaGuru_MonoChast
bin/magento setup:upgrade
bin/magento setup:di:compile
bin/magento cache:flush
```

---

## Configuration

**Stores → Configuration → Payment Methods → Monobank Частинами (MaGuru)**

| Field | Description |
|-------|-------------|
| Enabled | Enable/disable the payment method |
| Title | Display name on checkout (default: "Monobank Частинами (BNPL)") |
| Available Installment Parts | Comma-separated list, e.g. `3,6,10,12,18,24` |
| Auto-Confirm on Shipment | Automatically call `order/confirm` when shipment is created |
| Pre-validate Client | Call Monobank API to check BNPL eligibility before showing the method (fail-open) |
| Debug Logging | Write detailed logs to `var/log/mono_chast.log` |

**Stores → Configuration → MaGuru → Monobank Integration:**

| Field | Description |
|-------|-------------|
| Chast Store ID | Your store ID from Monobank Частини contract |
| Chast Secret | HMAC-SHA256 signing key (encrypted) |

---

## Features

### BNPL Checkout Flow
1. Customer selects "Monobank Частинами", enters phone, chooses number of parts
2. Magento calls `POST /api/order/create` → Monobank sends confirmation to customer's app
3. Customer approves in Monobank app → callback hits `POST /mono/chast/callback`
4. Customer sees a waiting screen with live AJAX status polling (every 3 sec, max 5 min)
5. On approval: order moves to `Processing`; on rejection: order is `Canceled`

### Client Pre-Validation (Optional)
- Config flag `Pre-validate Client` — call `POST /api/v2/client/validate` on checkout
- AJAX endpoint: `POST /mono/chast/validate` — fails open if Monobank API is unavailable
- Frontend JS receives `client_validate_enabled` and `validate_url` via checkout config

### Auto-Confirm on Shipment
- When shipment is created and order sub-state is `WAITING_FOR_STORE_CONFIRM`, automatically calls `POST /api/order/confirm`
- Disable to handle confirmation manually

### Order Lifecycle
- **Shipment created** → auto-confirm (if enabled and status is `WAITING_FOR_STORE_CONFIRM`)
- **Order canceled** → `POST /api/order/reject`
- **Credit memo created** → `POST /api/order/return` (refund in UAH)

### Status Polling Cron
- Every 2 minutes: polls `POST /api/order/state` for all `IN_PROCESS` orders
- Fallback for missed or delayed callbacks

### Daily Reconciliation
- Cron at 09:00: `POST /api/store/report` for yesterday's transactions
- CLI: `bin/magento mono:chast:report [--date=YYYY-MM-DD]`

### Admin Order View
The Chast order details panel appears on the Magento order view page and includes:
- Monobank order ID, state, sub-state, installment parts, total (UAH), customer phone
- **Confirm Delivery** button — visible only when sub-state is `WAITING_FOR_STORE_CONFIRM`; calls `POST /api/order/confirm`
- **Reject Order** button — visible only when state is `IN_PROCESS`; calls `POST /api/order/reject`
- **Download Guarantee Letter (PDF)** button — visible when `mono_order_id` is present; calls `POST /api/order/guarantee/letter`

### Guarantee Letter
- `ChastOrderService::downloadGuaranteeLetter(string $monoOrderId, string $invoiceNumber = '', string $invoiceDate = ''): string` — returns raw PDF bytes
- `ChastOrderService::getGuaranteeLetterData(string $monoOrderId, string $version = '1', ...): array` — returns structured JSON data (customer PII, passport, bank agreement); `version = '2'` calls `/api/v2/order/data/for/guarantee/letter`
- Admin controller `mono_chast/order/guarantee_letter` — sends PDF as `Content-Disposition: attachment`

### Admin Grid
- **System → Monobank → Chast Orders** — all BNPL orders with state, sub-state, parts, total (UAH), phone

---

## Webhook / Callback URL

Configure in your Monobank Частини merchant settings as `result_callback`:

```
POST https://yourstore.com/mono/chast/callback
```

The module verifies the request using `store_order_id` from the payload. Always returns HTTP 200.

---

## REST API

```
GET  /V1/mono/chast/order/:magentoOrderId        [admin]     — get local chast order record
GET  /V1/mono/chast/order/:magentoOrderId/state  [customer]  — sync state from Monobank
POST /V1/mono/chast/order/:monoOrderId/confirm   [admin]     — manually confirm order
POST /V1/mono/chast/order/:monoOrderId/reject    [admin]     — manually reject order
```

---

## Status Reference

| Monobank State | Sub-State | Magento Order State |
|----------------|-----------|---------------------|
| `IN_PROCESS` | `WAITING_FOR_CLIENT` | `Pending Payment` |
| `IN_PROCESS` | `WAITING_FOR_STORE_CONFIRM` | `Payment Review` |
| `SUCCESS` | `ACTIVE` | `Processing` |
| `FAIL` | — | `Canceled` |

---

## Tests

```bash
bin/clinotty php vendor/bin/phpunit -c app/code/MaGuru/MonoChast/Test/Unit/phpunit.xml
```

152 unit tests · 245 assertions · PHPStan Level 8 ✅

---

## Support

- Email: maguru.sup@gmail.com
- Issues: via Magento Marketplace order page

---

## License

Proprietary — one-time license per domain
