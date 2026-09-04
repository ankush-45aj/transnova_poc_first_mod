1. Module Objective

The purpose of Module 1 is to answer:

How does a user securely initiate a real-money transaction from India into the TransNova transaction system?

This module does not perform crypto settlement or international payout. Those belong to later modules.

Module 1 ends here:

PAYMENT_VERIFIED
       ↓
READY_FOR_SETTLEMENT

The next module begins:

READY_FOR_SETTLEMENT
       ↓
SETTLEMENT PROCESSING
2. Core Architecture
┌───────────────────┐
│   SENDER INDIA    │
│   TransNova App   │
└─────────┬─────────┘
          │
          │ 1. Authenticate
          ▼
┌───────────────────┐
│ Identity & Access │
└─────────┬─────────┘
          │
          │ 2. Scan QR
          ▼
┌───────────────────┐
│ TransNova Identity│
│ Resolution        │
└─────────┬─────────┘
          │
          │ 3. Generate Quote
          ▼
┌───────────────────┐
│ Transaction Engine│
└─────────┬─────────┘
          │
          │ 4. Security Checks
          ▼
┌───────────────────┐
│ KYC / Risk /      │
│ Limits            │
└─────────┬─────────┘
          │
          │ 5. User Authorizes
          ▼
┌───────────────────┐
│ India Payment Rail│
│ Provider / Partner│
└─────────┬─────────┘
          │
          │ 6. Payment Verification
          ▼
┌───────────────────┐
│ TransNova Backend │
│ Transaction Ledger│
└─────────┬─────────┘
          │
          ▼
READY_FOR_SETTLEMENT
3. Two Different Identities

One important design principle is that every user has two different identities.

A. TransNova Identity

This is your internal universal identity.

Example:

User ID:
usr_8f92ab31

TransNova ID:
TNV_8F92AB31

This identity works globally.

India
USA
Europe
Other countries

        ↓

Same TransNova ID
B. Payment Profile

The user's actual payment destination depends on their country.

TNV_8F92AB31
       │
       ├── India
       │     └── UPI / Bank payment profile
       │
       ├── USA
       │     └── USD bank payout profile
       │
       ├── Europe
       │     └── EUR/IBAN payout profile
       │
       └── Other countries
             └── Local payout profile

This gives TransNova a universal identity layer without exposing the user's bank information in the QR.

4. QR Flow

The receiver creates or displays their TransNova QR.

Example:

┌──────────────────────┐
│                      │
│     TRANSNOVA QR     │
│                      │
│    TNV_8F92AB31      │
│                      │
└──────────────────────┘

The QR should contain only a safe identifier or signed reference.

Conceptually:

transnova://pay/TNV_8F92AB31

The QR should not expose:

Bank account number
VPA credentials
API keys
Private keys
KYC documents
Sensitive financial information

Flow:

Sender scans QR
       │
       ▼
TNV_8F92AB31
       │
       ▼
TransNova Backend
       │
       ▼
Resolve Recipient
       │
       ▼
Return safe information

Response:

{
  "recipient_id": "TNV_8F92AB31",
  "display_name": "Rishu",
  "country": "US",
  "receive_currency": "USD",
  "status": "ACTIVE"
}

The mobile app never decides where money ultimately goes.

The backend resolves:

TransNova ID
       ↓
Recipient
       ↓
Country
       ↓
Payment/Payout Profile
       ↓
Supported Destination
5. Complete Sender Flow

Let's use your example:

Sender: Ganesha 🇮🇳
Receiver: Rishu 🇺🇸
Amount: ₹10,000
Step 1 — Authentication
Ganesha
   │
   ▼
TransNova App
   │
   ▼
Login / Authentication
   │
   ▼
Backend

Backend verifies:

✓ User exists
✓ Account active
✓ Session valid
✓ Device/session acceptable

For sensitive transactions:

Normal Login
      ↓
Transaction exceeds risk threshold
      ↓
Step-up verification
      ↓
Transaction authorization
6. Step 2 — QR Scan
Rishu displays QR
        │
        ▼
Ganesha scans QR
        │
        ▼
TNV_8F92AB31
        │
        ▼
POST /recipients/resolve

Backend checks:

Does recipient exist?

Is recipient active?

Can this recipient receive USD?

Does recipient have a valid payout profile?

Is the destination country supported?

Only after these checks:

Recipient Verified
7. Step 3 — Amount and Quote

Ganesha enters:

₹10,000

Request:

{
  "recipient_id": "TNV_8F92AB31",
  "amount": 10000,
  "currency": "INR"
}

The backend calculates:

Amount: ₹10,000

Transaction Fee: ₹X

FX Rate: X

Estimated Recipient Amount: $X

Quote Expiry: 30 seconds

Response:

{
  "quote_id": "qte_123456",
  "send_amount": 10000,
  "send_currency": "INR",
  "fee": 100,
  "receive_currency": "USD",
  "receive_amount": 114.45,
  "expires_at": "timestamp"
}

Security rule:

THE CLIENT DOES NOT DECIDE:

❌ Final FX rate
❌ Final fee
❌ Final receiver amount
❌ Settlement route

The backend owns those values.

8. Step 4 — Security and Risk Checks

Before allowing the payment:

TRANSACTION REQUEST
        │
        ▼
┌─────────────────────┐
│ Authentication Check│
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ User Status         │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ KYC Status          │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ Transaction Limits  │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ Risk Engine         │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ Compliance Check    │
└──────────┬──────────┘
           │
           ├── REJECT
           │
           ▼
        APPROVE

For the POC:

KYC = MOCK VERIFIED

Daily Limit = ₹100,000

Single Transaction Limit = ₹25,000

Risk Score = LOW

Compliance = PASS

In production these checks must connect to the appropriate regulated/compliance systems.

9. Step 5 — Create Transaction

Once approved:

Transaction ID:

txn_001

Initial transaction:

CREATED

Example:

{
  "transaction_id": "txn_001",
  "sender_id": "usr_123",
  "recipient_id": "TNV_8F92AB31",
  "send_amount": 10000,
  "send_currency": "INR",
  "receive_currency": "USD",
  "status": "CREATED"
}
10. Step 6 — Idempotency Protection

This is critical.

Imagine:

User presses PAY

Network freezes

User presses PAY again

App retries automatically

Without protection:

₹10,000
+
₹10,000
+
₹10,000

= Multiple charges

Instead:

PAY REQUEST
      │
      ▼
IDEMPOTENCY KEY
      │
      ▼
idem_8f72a
      │
      ▼
Database
      │
      ├── Exists?
      │       │
      │       └── Return original transaction
      │
      └── Does not exist?
              │
              ▼
       Create transaction

Result:

10 duplicate requests

        ↓

1 financial transaction
11. Step 7 — User Authorizes Payment

The user sees:

You are sending:

₹10,000

Recipient:
Rishu

Recipient receives:
$XXX

Fee:
₹XXX

[ CONFIRM ]

When the user confirms:

Transaction
       │
       ▼
PAYMENT_INITIATED

Then the TransNova backend creates the payment request through the configured India payment provider/rail.

The important architecture is:

Mobile App
     │
     ▼
TransNova Backend
     │
     ▼
Payment Provider Adapter
     │
     ▼
India Payment Rail

Not:

Mobile App
     │
     ▼
Direct provider secret/API access
12. Payment Provider Adapter

Do not put Razorpay-specific logic throughout the entire system.

Use:

PaymentProvider

Example interface:

createPayment()

getPaymentStatus()

verifyPayment()

refundPayment()

Then:

PaymentProvider
        │
        ├── RazorpayAdapter
        │
        ├── MockPaymentAdapter
        │
        └── FutureProviderAdapter

This means your TransNova transaction engine stays independent:

Transaction Engine
        │
        ▼
PaymentProvider
        │
        ▼
Selected Provider
13. Payment Verification

This is one of the most important security rules.

Never do this:

Mobile App:
"Payment Successful"

       ↓

Database:
COMPLETED

Instead:

Payment Initiated
       │
       ▼
Provider Processing
       │
       ▼
Provider Response / Webhook
       │
       ▼
Backend Verification
       │
       ▼
Transaction State Updated

The mobile app only displays the backend result.

14. Secure Webhook Flow
Payment Provider
        │
        ▼
POST /webhooks/payment
        │
        ▼
Verify Signature
        │
        ├── Invalid → Reject
        │
        ▼
Check Event ID
        │
        ├── Already Processed → Ignore
        │
        ▼
Validate Payment Reference
        │
        ▼
Save Webhook Event
        │
        ▼
Update Transaction

The webhook should be:

✓ Signature verified
✓ Idempotent
✓ Logged
✓ Retry-safe
✓ Asynchronous
15. Module 1 Transaction State Machine
CREATED
   │
   ▼
RECIPIENT_VERIFIED
   │
   ▼
QUOTE_CREATED
   │
   ▼
RISK_CHECK_PENDING
   │
   ├──────────────→ COMPLIANCE_REJECTED
   │
   ▼
PAYMENT_AUTHORIZED
   │
   ▼
PAYMENT_INITIATED
   │
   ▼
PAYMENT_PROCESSING
   │
   ├──────────────→ PAYMENT_FAILED
   │
   ▼
PAYMENT_VERIFICATION_PENDING
   │
   ▼
PAYMENT_VERIFIED
   │
   ▼
READY_FOR_SETTLEMENT

This matches your report's emphasis on explicit transaction states, failure handling, and lifecycle visibility.

16. Security Architecture

Your system should have multiple layers.

┌─────────────────────────────────────┐
│ LAYER 1                             │
│ Network Security                    │
│ TLS / HTTPS                         │
├─────────────────────────────────────┤
│ LAYER 2                             │
│ Authentication                      │
│ Secure session / MFA where required │
├─────────────────────────────────────┤
│ LAYER 3                             │
│ Authorization                       │
│ User owns requested resource        │
├─────────────────────────────────────┤
│ LAYER 4                             │
│ API Protection                      │
│ Rate limits / validation            │
├─────────────────────────────────────┤
│ LAYER 5                             │
│ Transaction Security                │
│ Idempotency / quote expiry          │
├─────────────────────────────────────┤
│ LAYER 6                             │
│ Risk Controls                       │
│ KYC / limits / compliance           │
├─────────────────────────────────────┤
│ LAYER 7                             │
│ Data Protection                     │
│ Encryption / RBAC                   │
├─────────────────────────────────────┤
│ LAYER 8                             │
│ Secrets Management                  │
│ Backend only                        │
├─────────────────────────────────────┤
│ LAYER 9                             │
│ Monitoring                          │
│ Logs / alerts / reconciliation      │
└─────────────────────────────────────┘

The report specifically recommends backend-only provider credentials, restricted access, MFA, encrypted communication, rate limiting, transaction limits, audit logging, and monitoring.

17. API Security Rules

Example:

POST /quotes

Requires:

✓ Authentication
✓ Valid request
✓ Supported currency
✓ Valid recipient
POST /transactions

Requires:

✓ Authentication
✓ Idempotency key
✓ Valid quote
✓ Quote not expired
✓ Recipient verified
✓ KYC status
✓ Transaction limits
GET /transactions/:id

Requires:

✓ Authentication
✓ Ownership check

Never allow:

User A

GET /transactions/transaction_of_user_B
18. Secrets Management

Never put production secrets inside:

❌ Flutter application
❌ Android APK
❌ GitHub repository
❌ README
❌ QR code
❌ Frontend JavaScript

Instead:

Developer
    │
    ▼
Secrets Manager / Environment
    │
    ▼
TransNova Backend
    │
    ▼
Payment Provider

The same principle applies to your later Crypto.com integration: provider credentials remain on the backend.

19. Database Records for Module 1

You need at least these tables:

users
payment_identities
payment_profiles
quotes
transactions
transaction_events
payment_attempts
idempotency_keys
webhook_events
audit_logs

Relationship:

USER
 │
 ├── PAYMENT IDENTITY
 │
 ├── PAYMENT PROFILES
 │
 ├── QUOTES
 │
 └── TRANSACTIONS
          │
          ├── EVENTS
          ├── PAYMENT ATTEMPTS
          └── WEBHOOK EVENTS
20. Attack and Protection Model
Attack / Failure	Protection
User clicks Pay multiple times	Idempotency key
Fake QR	Backend recipient validation
User modifies recipient ID	Authorization and server-side resolution
Client changes amount	Server-side quote
Quote expires	Quote expiry validation
Fake payment success	Backend/provider verification
Duplicate webhook	Webhook event deduplication
Stolen API key	Backend-only secrets
Brute-force requests	Rate limiting
Unauthorized admin access	RBAC + MFA
Database update fails	Transaction events + retry/reconciliation
Provider timeout	Pending state + status verification
21. Final Module 1 Flow

This is the complete flow I recommend documenting:

                         SENDER 🇮🇳
                              │
                              ▼
                    TRANSNOVA MOBILE APP
                              │
                     Authentication
                              │
                              ▼
                         Scan QR
                              │
                              ▼
                  TRANSNOVA ID RESOLUTION
                              │
                              ▼
                     Recipient Verified
                              │
                              ▼
                        Enter Amount
                              │
                              ▼
                         FX QUOTE
                              │
                              ▼
                  KYC / RISK / LIMIT CHECK
                              │
                              ▼
                     CREATE TRANSACTION
                              │
                              ▼
                   IDEMPOTENCY PROTECTION
                              │
                              ▼
                       USER CONFIRMATION
                              │
                              ▼
                  INDIA PAYMENT PROVIDER
                              │
                              ▼
                    REAL PAYMENT PROCESS
                              │
                              ▼
                    PROVIDER WEBHOOK/API
                              │
                              ▼
                   BACKEND VERIFICATION
                              │
                              ▼
                       PAYMENT VERIFIED
                              │
                              ▼
                   READY FOR SETTLEMENT
                              │
                              ▼
                           MODULE 2
What you should actually build for the POC

Start with this order:

Phase 1
├── User authentication
├── TransNova ID
├── QR generation
├── QR scanning
└── Recipient resolution

Phase 2
├── Quote engine
├── Transaction creation
├── Transaction state machine
└── Idempotency

Phase 3
├── Payment provider adapter
├── Razorpay integration/sandbox
├── Webhook receiver
└── Payment verification

Phase 4
├── Rate limiting
├── RBAC
├── Audit logs
├── Secrets management
└── Failure testing

The clean boundary for your architecture is:

MODULE 1
"Can TransNova securely identify users, resolve a recipient,
create a transaction, collect/verify the source payment,
and produce one trusted transaction ready for settlement?"

                            ↓

MODULE 2
"How does that verified value enter the settlement and
liquidity process?"

This makes Module 1 a complete and independent foundation for the rest of your TransNova POC.


Ganesha's Summary of Mod 1
# Module 1 — India-Side Payment Initiation & Verification

## Objective

Securely initiate and verify a real-money transaction from India and
produce a trusted transaction that is ready for settlement.

Crypto settlement and international payout happen in Module 2.

---

## Simple Flow

Sender
  ↓
TransNova App
  ↓
Authentication
  ↓
Scan QR
  ↓
Recipient Resolution
  ↓
Quote
  ↓
KYC / Risk / Limits
  ↓
User Confirmation
  ↓
India Payment Provider
  ↓
Payment Verification
  ↓
READY_FOR_SETTLEMENT

---

## Main Components

### 1. TransNova Identity

A universal ID such as:

TNV_8F92AB31

identifies the recipient globally.

---

### 2. Payment Profile

Maps the TransNova ID to the recipient's actual payment destination.

Examples:

India
  → UPI / Bank payment profile

USA
  → USD bank payout profile

Europe
  → EUR / IBAN payout profile

---

### 3. QR Code

The QR contains only a safe TransNova identifier/reference.

Example:

transnova://pay/TNV_8F92AB31

It should NOT contain:

- Bank account number
- VPA credentials
- API keys
- Private keys
- KYC documents
- Sensitive financial information

---

### 4. Quote Engine

Calculates:

- INR amount
- Transaction fee
- FX rate
- Recipient amount
- Quote expiry

Example:

₹10,000
  ↓
Fee: ₹100
  ↓
FX conversion
  ↓
Recipient receives: $XXX

The backend controls the final:

- FX rate
- Fee
- Recipient amount
- Settlement route

---

### 5. KYC / Risk / Limits

Before payment, TransNova checks:

- User authentication
- User account status
- KYC status
- Transaction limits
- Risk score
- Compliance status

Only if the checks pass:

APPROVE
  ↓
Payment continues

---

### 6. Transaction Engine

Creates and manages the transaction lifecycle.

Example:

CREATED
  ↓
RECIPIENT_VERIFIED
  ↓
QUOTE_CREATED
  ↓
RISK_CHECK_PENDING
  ↓
PAYMENT_AUTHORIZED
  ↓
PAYMENT_INITIATED
  ↓
PAYMENT_PROCESSING
  ↓
PAYMENT_VERIFICATION_PENDING
  ↓
PAYMENT_VERIFIED
  ↓
READY_FOR_SETTLEMENT

---

### 7. Payment Provider Adapter

TransNova should not directly depend on one payment provider.

Instead:

Transaction Engine
       ↓
PaymentProvider Interface
       ↓
+-----------------------+
|                       |
RazorpayAdapter    FutureProviderAdapter
|                       |
+-----------------------+

This allows TransNova to change or add payment providers without
changing the core transaction engine.

---

### 8. Payment Verification

The mobile application should NOT decide whether a payment was successful.

Correct flow:

Payment Initiated
       ↓
Payment Provider
       ↓
Provider API / Webhook
       ↓
Backend Verification
       ↓
Transaction State Updated
       ↓
PAYMENT_VERIFIED

The backend is the source of truth.

---

### 9. Idempotency

Prevents duplicate payments.

Example:

User presses PAY
      ↓
Network freezes
      ↓
User presses PAY again
      ↓
Same Idempotency Key
      ↓
Backend detects duplicate request
      ↓
Original transaction returned

Result:

10 duplicate requests
        ↓
1 financial transaction

---

## Security

The system should include:

- HTTPS / TLS
- Authentication
- Authorization
- MFA where required
- Rate limiting
- Transaction limits
- KYC / AML checks
- Encryption
- Role-Based Access Control (RBAC)
- Audit logging
- Secrets management
- Transaction monitoring

Production API credentials must remain on the backend.

Never store them in:

- Flutter application
- Android APK
- Frontend JavaScript
- GitHub repository
- QR code
- README

---

## Database

Important records include:

users
payment_identities
payment_profiles
quotes
transactions
transaction_events
payment_attempts
idempotency_keys
webhook_events
audit_logs

---

## Final Module 1 Flow

SENDER 🇮🇳
    ↓
TRANSNOVA MOBILE APP
    ↓
Authentication
    ↓
Scan QR
    ↓
TransNova ID Resolution
    ↓
Recipient Verified
    ↓
Enter Amount
    ↓
FX Quote
    ↓
KYC / Risk / Limit Check
    ↓
Create Transaction
    ↓
Idempotency Protection
    ↓
User Confirmation
    ↓
India Payment Provider
    ↓
Real Payment Process
    ↓
Provider Webhook / API
    ↓
Backend Verification
    ↓
PAYMENT VERIFIED
    ↓
READY FOR SETTLEMENT
    ↓
MODULE 2

---

## Module Boundary

### Module 1

"Can TransNova securely identify users, resolve a recipient,
create a transaction, collect and verify the source payment,
and produce one trusted transaction ready for settlement?"

        ↓

### Module 2

"How does that verified value enter the settlement,
liquidity and international payout process?"

---

## One-Sentence Summary

Module 1 securely converts a user's intention to pay into one
verified INR transaction that TransNova can safely hand over
to the international settlement system.