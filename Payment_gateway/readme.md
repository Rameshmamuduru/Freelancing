Here's the complete production flow that a senior engineer would explain in a design review.

# One-Time Setup (Done Once)

## 1. Create Merchant Account

Register with Razorpay

Provide:

```text
Company Details
PAN
GST
Bank Account
KYC Documents
```

Razorpay creates:

```text
Merchant ID
Key ID
Key Secret
Webhook Secret
```

Example:

```text
Key ID = rzp_live_xxxxx
Key Secret = xxxxx
Webhook Secret = xxxxx
```

---

## 2. Store Secrets Securely

Never hardcode them.

```text
AWS Secrets Manager
      ↓
Payment Service
```

Store:

```text
RAZORPAY_KEY_ID
RAZORPAY_SECRET
WEBHOOK_SECRET
```

---

# Runtime Flow

## Step 1: User Clicks Buy

```text
User
 ↓
Buy Premium ₹499
```

---

## Step 2: Create Order in Your Database

```text
POST /orders
```

Database:

```sql
orders
--------------------------------
id      = ORD001
amount  = 499
status  = PENDING
```

---

## Step 3: Create Razorpay Order

Backend calls Razorpay:

```text
POST /v1/orders
```

Request:

```json
{
  "amount": 49900,
  "currency": "INR"
}
```

(using your Key ID + Secret)

Razorpay returns:

```json
{
  "id": "order_RAZ123"
}
```

Store:

```sql
orders
--------------------------------
id                 = ORD001
razorpay_order_id  = order_RAZ123
status             = PENDING
```

---

## Step 4: Show Payment Page

```text
User
 ↓
Razorpay Checkout
 ↓
UPI/Card/Net Banking
```

Customer pays.

---

## Step 5: Razorpay Captures Payment

Razorpay records:

```text
Payment ID = pay_123
Amount     = 499
Status     = CAPTURED
Merchant   = YOUR_ACCOUNT
```

---

## Step 6: Razorpay Sends Webhook

```http
POST /webhook/razorpay
```

Payload:

```json
{
  "event": "payment.captured",
  "payment_id": "pay_123"
}
```

---

## Step 7: Verify Webhook Signature

Backend:

```text
Verify Signature
```

If invalid:

```text
Reject Request
```

If valid:

```text
Process Payment
```

---

## Step 8: Update Database

```sql
orders.status='PAID'
payments.status='SUCCESS'
```

Now:

```text
ORD001
Status = PAID
```

---

## Step 9: Deliver Product

Examples:

### SaaS

```text
Enable Subscription
```

### E-Commerce

```text
Create Shipment
```

### Course Platform

```text
Grant Course Access
```

---

## Step 10: Settlement

Razorpay knows:

```text
Payment → Merchant Account
```

because your API keys were used.

Settlement:

```text
Customer Pays ₹499
        ↓
Razorpay
        ↓
Fee Deducted
        ↓
Your Bank Account
```

Usually T+1 or according to your settlement schedule.

---

# Production Architecture

```text
                ┌─────────────┐
                │    User     │
                └──────┬──────┘
                       │
                       ▼
                ┌─────────────┐
                │ Frontend    │
                └──────┬──────┘
                       │
                       ▼
                ┌─────────────┐
                │ API Gateway │
                └──────┬──────┘
                       │
                       ▼
                ┌─────────────┐
                │ Order Svc   │
                └──────┬──────┘
                       │
                       ▼
                ┌─────────────┐
                │ Payment Svc │
                └──────┬──────┘
                       │
                       ▼
             ┌──────────────────┐
             │    Razorpay      │
             └──────┬───────────┘
                    │
         Payment    │    Webhook
                    ▼
             ┌─────────────┐
             │ Payment Svc │
             └──────┬──────┘
                    │
                    ▼
             ┌─────────────┐
             │ PostgreSQL  │
             └──────┬──────┘
                    │
                    ▼
             ┌─────────────┐
             │ Business    │
             │ Services    │
             └─────────────┘
```

### Interview Summary (2-minute answer)

> User creates an order in our system with status `PENDING`. The payment service creates an order in Razorpay using merchant API credentials and redirects the user to Razorpay Checkout. After payment, Razorpay sends a signed webhook to our backend. We verify the webhook signature, update the order status to `PAID`, store the payment transaction, and then trigger downstream business actions such as subscription activation or order fulfillment. Razorpay later settles the funds to the merchant bank account configured in the Razorpay dashboard. This design ensures security, auditability, idempotency, and reliability.
