# MaGuru MonoChast — User Guide

**Version:** 1.1.0  
**Compatible with:** Adobe Commerce / Magento Open Source 2.4.4 – 2.4.x  
**PHP:** 8.1 or higher  
**Requires:** MaGuru Monobank Core  
**License:** Paid

---

## Overview

MaGuru MonoChast adds **Monobank Частина** — a Buy Now, Pay Later (BNPL) installment payment method — to your Adobe Commerce store. Customers who have a Monobank card can spread the cost of their purchase across 3 to 24 monthly installments at no extra interest.

Installment orders flow through the standard Magento checkout with a dedicated "Pay in installments" payment option. Everything else — approval, payment processing, and installment management — is handled entirely by Monobank.

> **Fiscal receipts:** if you also use **MaGuru Monobank Fiscal**, note that it currently fiscalizes **MaGuru Monobank Payment** orders only. Installment orders placed through MonoChast are not automatically fiscalized by that module.

---

## What It Does

- Adds **Monobank Частина** as a separate payment method at checkout
- Lets you configure which installment durations are available (3, 6, 10, 12, 18, or 24 months)
- Supports optional phone validation at checkout — confirms the customer's eligibility via Monobank before they proceed
- Provides **Confirm Delivery** and **Reject Order** actions in the admin order view (required by Monobank to release or cancel the installment)
- Automatically confirms delivery when a Magento shipment is created (optional)
- Provides a **Guarantee Letter (PDF)** download — a legally-binding document from Monobank about the installment agreement
- Admin grid showing all installment orders with their Monobank states

---

## Requirements

| Requirement | Version / Note |
|---|---|
| Adobe Commerce / Magento Open Source | 2.4.4 or higher |
| PHP | 8.1 or higher |
| MaGuru Monobank Core | Must be installed and configured with MonoChast credentials |
| Monobank Частина account | Active MonoChast merchant account from Monobank |

---

## Installation

```bash
composer require maguru/module-mono-chast
bin/magento setup:upgrade
bin/magento setup:di:compile
bin/magento cache:flush
```

Verify:

```bash
bin/magento module:status MaGuru_MonoChast
```

### Message Queue Setup (required)

Order confirmation on shipment goes through Magento's message queue and **does nothing until a consumer is running**; Magento does not start consumers by itself.

Add the following to `app/etc/env.php`:

```php
'cron_consumers_runner' => [
    'cron_run' => true,
    'max_messages' => 200,
    'consumers' => [
        'mono_chast.order.confirm',
    ]
],
```

Magento's own cron will then run this consumer automatically — no extra process needed. If this is skipped, Chast order confirmation on shipment will never reach Monobank, with no error shown anywhere.

---

## Configuration

Navigate to **Stores → Configuration → Sales → Payment Methods → Monobank Частинами (MaGuru)**.

> The Monobank Chast API environment (Production / Sandbox / Stage) is set in **Stores → Configuration → MaGuru → Monobank Integration → BNPL "Частинами" API**.

---

### General Settings

| Setting | Description | Default |
|---|---|---|
| **Enabled** | Activates Monobank Частина at checkout | No |
| **Chast API Connection** | Read-only "Check Connection" button that tests the configured Chast API credentials | — |
| **Title** | Name shown to customers at checkout (radio button label) | Monobank Частинами (BNPL) |
| **Checkout Button Label** | Text on the payment submit button in checkout | Pay in installments |
| **Available Installment Parts** | Multiselect of allowed installment durations (months): 3, 6, 10, 12, 18, 24 | 3, 6, 10, 12 (18 and 24 not selected by default) |
| **Minimum Order Total (UAH)** | Minimum order total to show this payment method (leave blank for no minimum) | — |
| **Maximum Order Total (UAH)** | Maximum order total to show this payment method (leave blank for no maximum) | — |
| **Auto-Confirm on Shipment** | Automatically confirms delivery to Monobank when a Magento shipment is created | Yes |
| **New Order Status** | Magento order status assigned when the Chast order is first created (before Monobank confirmation) | Pending |
| **Pre-validate Client on Checkout** | Prompts the customer to enter their phone number and checks eligibility via Monobank before checkout proceeds | No |
| **Debug Logging** | Write detailed API data to the log | No |

---

### Callback URL

A read-only field displayed in the configuration showing the URL that Monobank uses to send installment status updates. Enter this URL in your Monobank Частина merchant dashboard.

---

## Customer Checkout Experience

1. At checkout, the customer selects **Pay in installments** as the payment method
2. If **Validate Client** is enabled, they enter their Monobank phone number; the module checks eligibility in real time
3. The customer selects the desired number of installments (from the options you configured)
4. The order is placed in Magento (status **Pending**) and the customer is redirected to Monobank to confirm the installment agreement
5. Upon return, status updates are received via webhook (and background polling as a fallback) and the Magento order status is updated accordingly

---

## Admin Panels

### Chast Orders Grid

**Monobank → Chast Orders**

A grid showing all installment orders processed through your store.

| Column | Description |
|---|---|
| **Order #** | Magento order increment ID |
| **Mono Order ID** | Monobank's internal order identifier |
| **State** | Current Monobank installment state |
| **Sub-state** | Additional state detail from Monobank |
| **Parts** | Number of installments selected by the customer |
| **Total (UAH)** | Order total |
| **Customer Phone** | Phone number associated with the installment |
| **Created At** | Order creation time |

---

## Admin Order View

When an order was placed via MonoChast, a **Monobank Chast** block appears on the order view page showing:

- Mono Order ID, state, sub-state, number of installments, total (UAH), and customer phone

**Action buttons** (shown based on current state):

| Button | When Visible | What It Does |
|---|---|---|
| **Confirm Delivery** | Sub-state = `WAITING_FOR_STORE_CONFIRM` | Confirms to Monobank that goods have been delivered; releases the installment agreement |
| **Reject Order** | State = `IN_PROCESS` | Cancels the installment order in Monobank |
| **Download Guarantee Letter (PDF)** | A Mono Order ID exists (any state) | Downloads the legal PDF document confirming the installment agreement |
| **View Guarantee Letter Data (JSON)** | A Mono Order ID exists (any state) | Opens structured guarantee letter data — customer name, passport details, bank agreement terms — in a new tab |
| **Sync State** | A Mono Order ID exists | Forces an immediate state refresh from Monobank API |

---

## Installment Order States

The admin order view, the Chast Orders grid, and order comments all show plain-language labels (e.g. "Exceeded installment limit") instead of Monobank's raw codes — the table below lists the underlying codes for reference.

| State | Meaning |
|---|---|
| **IN_PROCESS** | Order created in Monobank, awaiting customer confirmation on their device |
| **SUCCESS** | Order completed — installment is active and running |
| **FAIL** | Order was rejected, cancelled, or failed |
| **EARLY_CLOSED_BY_CLIENT** | Customer closed the installment agreement early |

**IN_PROCESS sub-states:**

| Sub-State | Meaning |
|---|---|
| **WAITING_FOR_CLIENT** | Waiting for the customer to confirm in their Monobank app |
| **WAITING_FOR_STORE_CONFIRM** | Installment approved by Monobank; store must confirm delivery to proceed |

**SUCCESS sub-states:**

| Sub-State | Meaning |
|---|---|
| **ACTIVE** | Installment agreement is active and running |
| **DONE** | Installment has been fully paid off |
| **RETURNED** | Order was refunded (in full or in part) after activation |

**FAIL sub-states** (why the order failed, and what to do):

| Sub-State | Meaning | Recommended Action |
|---|---|---|
| **CLIENT_NOT_FOUND** | Client not found or not a Monobank customer | Check the phone number format |
| **EXCEEDED_SUM_LIMIT** | Client exceeded their installment limit | Ask the customer to reduce the order total or use another payment method |
| **EXISTS_OTHER_OPEN_ORDER** | Client already has another unfinished installment order | Wait 15 minutes, or ask the client to cancel their existing order |
| **NOT_ENOUGH_MONEY_FOR_INIT_DEBIT** | Insufficient funds for the first payment | Client needs to top up their card |
| **REJECTED_BY_CLIENT** | Client declined the installment offer | Customer can try again later or use another payment method |
| **PAY_PARTS_ARE_NOT_ACCEPTABLE** | The requested installment count is not acceptable to Monobank | Contact Monobank support |
| **FRAUD_REJECTED** | Rejected by Monobank's fraud prevention | Contact Monobank support |
| **RESTRICTED_BY_RISKS** | Restricted by Monobank's risk management | Contact Monobank support |
| **CLIENT_PUSH_TIMEOUT** | Client did not respond within 15 minutes | Contact the client, or ask them to retry |
| **REJECTED_BY_STORE** | The store cancelled the order | None — this was intentional |
| **FAIL** | Internal Monobank error | Retry in a few minutes |

---

## Confirming Delivery

Monobank requires stores to explicitly confirm that the goods or services have been delivered before activating the installment agreement. There are two ways to do this:

**Option A — Automatic (recommended):** Enable **Auto-confirm on Shipment** in the configuration. When you create a Magento shipment for the order, the module automatically sends the delivery confirmation to Monobank.

**Option B — Manual:** When the order state reaches `WAITING_FOR_STORE_CONFIRM`, click the **Confirm Delivery** button in the admin order view.

> Until delivery is confirmed, the installment agreement is not activated and the customer has not committed to the installment schedule.

---

## CLI Commands

| Command | Description |
|---|---|
| `bin/magento mono:chast:status-check` | Force a status check for all pending installment orders |
| `bin/magento mono:chast:status-check --mono-order-id=X` | Force status check for a specific Mono Order ID |
| `bin/magento mono:chast:report` | Sync the latest daily report from Monobank |
| `bin/magento mono:chast:report --date=YYYY-MM-DD` | Sync the daily report for a specific date |

---

## Cron Jobs

| Job | Schedule | What It Does |
|---|---|---|
| Installment status check | Every 2 minutes | Polls pending installment orders for state updates |
| Daily report sync | Daily at 9:00 AM | Syncs the Monobank Chast daily report |

---

## Frequently Asked Questions

**Q: What is "Auto-confirm on Shipment" and should I enable it?**

When enabled, creating a Magento shipment for an installment order automatically sends a delivery confirmation to Monobank. This is the recommended setting for physical goods. If you sell services or digital products that are delivered immediately, you may prefer to confirm delivery manually using the admin button.

---

**Q: A customer says their installment was approved but the order is still pending in my store. What should I do?**

Click the **Sync State** button in the admin order view to fetch the latest status from Monobank. If the state is `WAITING_FOR_STORE_CONFIRM`, confirm delivery using the button or by creating a shipment (if Auto-confirm is enabled). If the problem persists, wait for the 2-minute background cron or run `bin/magento mono:chast:status-check` from the command line.

---

**Q: What is the Guarantee Letter?**

The Guarantee Letter is a legally-binding PDF document issued by Monobank confirming the installment agreement between the bank and the customer. It is available for download from the admin order view as soon as Monobank has assigned a Mono Order ID (there is also a **View Guarantee Letter Data (JSON)** button for the structured underlying data). You may be required to provide this document to the customer on request.

---

**Q: Can I restrict which installment durations are offered?**

Yes. In the **Available Installment Parts** multiselect setting, select only the durations you want to offer. For example, if you only want to offer 6-month and 12-month plans, select only those two options.

---

**Q: Can I set a minimum order amount for installment payments?**

Yes. Enter a value (in UAH) in **Minimum Order Total (UAH)**. Orders below this amount will not show the Monobank Частина payment option at checkout.

---

**Q: What environment should I use during testing?**

Set the MonoChast environment to **Sandbox** in **Stores → Configuration → MaGuru → Monobank Integration → BNPL "Частинами" API**, and use the Sandbox Store ID and Secret from Monobank. This allows full end-to-end testing without real financial transactions.

---

**Q: A customer changed their mind after the installment was approved. Can I reject it?**

Yes. While the order is in `IN_PROCESS` state (before delivery is confirmed), you can click **Reject Order** in the admin order view to cancel the installment in Monobank.

---

## Support

Before contacting support, please gather:

1. The Magento Order # and the Mono Order ID from the Chast Orders grid
2. The current order state and sub-state
3. Contents of `var/log/mono_chast.log` if Debug Logging is enabled
4. Whether the issue affects all installment orders or specific ones

---

*MaGuru MonoChast — part of the MaGuru Monobank Integration Suite for Adobe Commerce*
