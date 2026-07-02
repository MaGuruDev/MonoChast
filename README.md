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

## Message Queue Setup

MonoChast publishes order confirmation to Magento's message queue instead of calling the Monobank API synchronously from the observer. **The queue does nothing until a consumer processes it** — Magento does not run consumers automatically out of the box.

| Consumer | Purpose |
|----------|---------|
| `mono_chast.order.confirm` | Confirm Chast order on shipment |

**Recommended:** register the consumer in `app/etc/env.php` so Magento's own cron picks it up automatically (no extra process to manage):

```php
'cron_consumers_runner' => [
    'cron_run' => true,
    'max_messages' => 200,
    'consumers' => [
        'mono_chast.order.confirm',
    ]
],
```

**Alternative:** run a persistent process under supervisor/systemd:

```bash
bin/magento queue:consumers:start mono_chast.order.confirm
```

Without one of the above, order confirmation on shipment never reaches Monobank — messages just accumulate unprocessed in the `queue_message` table.

---

## Configuration

**Stores → Configuration → Payment Methods → Monobank Частинами (MaGuru)**

| Field | Description |
|-------|-------------|
| Enabled | Enable/disable the payment method |
| Title | Display name on checkout (default: "Monobank Частинами (BNPL)") |
| Checkout Button Label | Text on the payment submit button in checkout. Default: "Pay in installments" |
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
- **View Guarantee Letter Data (JSON)** button — visible when `mono_order_id` is present; calls `POST /api/order/data/for/guarantee/letter`, opens structured data (customer PII, passport, bank agreement) in a new tab

### Guarantee Letter
- `ChastOrderService::downloadGuaranteeLetter(string $monoOrderId, string $invoiceNumber = '', string $invoiceDate = ''): string` — returns raw PDF bytes
- `ChastOrderService::getGuaranteeLetterData(string $monoOrderId, string $version = '1', ...): array` — returns structured JSON data (customer PII, passport, bank agreement); `version = '2'` calls `/api/v2/order/data/for/guarantee/letter`
- Admin controller `mono_chast/order/guaranteeletter` — sends PDF as `Content-Disposition: attachment`
- Admin controller `mono_chast/order/guaranteeletterdata` — returns `getGuaranteeLetterData()` as JSON (v1 by default; `?version=2` for the v2 endpoint)

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
| `SUCCESS` | `ACTIVE` / `DONE` / `RETURNED` | `Processing` |
| `FAIL` | any | `Canceled` |
| `EARLY_CLOSED_BY_CLIENT` | — | `Canceled` |

Raw codes above are translated to human-readable labels (admin order view, admin grid, order comments) by `Model\Config\Source\ChastOrderState` / `ChastOrderSubState`. FAIL sub-states, per Monobank's official reference:

| Sub-State | Meaning | Recommended Action |
|-----------|---------|---------------------|
| `CLIENT_NOT_FOUND` | Client not found or not a Monobank customer | Check the phone number format |
| `EXCEEDED_SUM_LIMIT` | Client exceeded their installment limit | Reduce the order total or use another payment method |
| `EXISTS_OTHER_OPEN_ORDER` | Client has another unfinished installment order | Wait 15 minutes or ask the client to cancel their existing order |
| `NOT_ENOUGH_MONEY_FOR_INIT_DEBIT` | Insufficient funds for the first payment | Client needs to top up their card |
| `REJECTED_BY_CLIENT` | Client declined the installment offer | Try again later or use another payment method |
| `PAY_PARTS_ARE_NOT_ACCEPTABLE` | Installment count not acceptable to Monobank | Contact Monobank support |
| `FRAUD_REJECTED` | Rejected by fraud prevention | Contact Monobank support |
| `RESTRICTED_BY_RISKS` | Restricted by risk management | Contact Monobank support |
| `CLIENT_PUSH_TIMEOUT` | Client did not respond within 15 minutes | Contact the client or retry |
| `REJECTED_BY_STORE` | Store cancelled the order | None — order was cancelled by the store |
| `FAIL` | Internal bank error | Retry in 5 minutes |

---

## Tests

```bash
bin/clinotty php vendor/bin/phpunit -c app/code/MaGuru/MonoChast/Test/Unit/phpunit.xml
```

221 unit tests · 339 assertions · PHPStan Level 8 ✅

---

## Support

- Email: maguru.sup@gmail.com
- Issues: via Magento Marketplace order page

---

## License

Proprietary — one-time license per domain
