TransNova-POC/

 README.md

 01-payment-flow-security/
  README.md

 02-crypto-settlement/
  README.md

 03-receiver-payout/
  README.md

 04-transaction-verification-consistency/
  README.md

 05-domestic-payment-razorpay/
  README.md

 06-database-bottlenecks/
  README.md

 architecture/
 system-architecture.md
 transaction-state-machine.md
 sequence-diagrams.md
I would define the modules like this:
Module 1 — Real Money Payment Flow, Security & TransNova VPA
Main question
How does a real-money transaction enter TransNova, and how do we secure it?
What this module should cover
1. End-to-end flow
Sender

Authentication

KYC / User Verification

Scan TransNova QR

Resolve Recipient

Risk / AML Checks

Generate Quote

User Confirmation

Collect Payment

Settlement Orchestration
2. TransNova payment identity
Here, I recommend you don’t call it a real VPA unless it is actually issued through a
UPI/PSP/banking arrangement.
For the POC, use:
TransNova ID
Example:
rishu@transnova
or internally:
TNV_USER_001
The QR should resolve to your internal recipient identity:
transnova://pay/TNV_USER_001
Your report supports the concept of a TransNova payment identity and QR-based
recipient resolution.
Security layers
Layer 1  Authentication
Layer 2  Device / Session Validation
Layer 3  KYC
Layer 4  Transaction Limits
Layer 5  AML / Risk Checks
Layer 6  Payment Authorization
Layer 7  API Security
Layer 8  Database Consistency
Layer 9  Audit Logs
Layer 10  Settlement Verification
Also explain:
JWT/session security
MFA for sensitive actions/admin
idempotency keys
rate limiting
encrypted data
RBAC
audit trail
backend-only API credentials
The report specifically emphasizes that exchange credentials must never be exposed to
the mobile app or frontend.

Module 2 — Crypto.com Integration & Crypto Settlement
Main question
How does TransNova interact with a crypto liquidity/settlement provider without making
the entire system dependent on that provider?
This module should not be titled only “How to use Crypto.com APIs.”
The stronger architecture is:
TransNova

Settlement Interface

Provider Adapter

Crypto.com / Other Provider
Your interface could conceptually support:
getQuote()
getBalance()
createOrder()
getOrderStatus()
getDepositAddress()
transferAsset()
getTransferStatus()
Then:
SettlementProvider

 CryptoComAdapter
 MockSettlementAdapter
 FutureProviderAdapter
This directly follows the adapter/orchestration model described in your report.
Wallet section
Document three possible concepts:
1. Custodial Wallet
Provider controls settlement wallet infrastructure.
2. Internal Ledger
TransNova records user balances/transaction ownership.
3. External Blockchain Wallet
Used when blockchain settlement actually occurs.
For the POC:
INR

Mock/Partner On-ramp

Settlement Asset

Testnet / Provider Sandbox

Settlement Confirmation
Do not assume that a production flow is automatically:
INR  ETH  USD
Your report explicitly recommends a more flexible settlement architecture rather than
hard-coding every transaction to ETH.
Module 3 — Receiver Payout
Main question
After settlement is complete, how does the receiver actually receive USD into their
account?
Flow:
Settlement Completed

Destination Amount Calculated

Recipient Payout Profile Loaded

Payout Request Created

Payout Provider / Banking Partner

Bank Processing

Webhook / Status Verification

Receiver Account Credited
For the POC, create a Mock US Payout Provider.
Example:
POST /payouts
Response:
{
“payout_id”: “PAY_US_001”,
“status”: “PROCESSING”
}
Later:
{
“payout_id”: “PAY_US_001”,
“status”: “COMPLETED”
}
Your transaction becomes:
BLOCKCHAIN_CONFIRMED

USD_CONVERSION_COMPLETED

PAYOUT_INITIATED

PAYOUT_PROCESSING

PAYOUT_COMPLETED
This module is important because settlement is not the same as the receiver getting
money in their bank account.
Module 4 — Transaction Verification & Database Consistency
This should be one of your strongest modules.
Main question
What happens if everything succeeds until the final payout step fails?
Example:
10,000 received 
KYC 
AML 
Settlement Asset 
Blockchain 
USD Conversion 
Payout  FAILED
You must not mark the entire transaction as simply FAILED and delete everything.
Instead:
CREATED
PAYMENT_PENDING
PAYMENT_RECEIVED
COMPLIANCE_CHECKED
SETTLEMENT_PROCESSING
SETTLEMENT_CONFIRMED
PAYOUT_PENDING
PAYOUT_PROCESSING
PAYOUT_COMPLETED
Failure states:
PAYMENT_FAILED
COMPLIANCE_REJECTED
SETTLEMENT_FAILED
PAYOUT_FAILED
MANUAL_REVIEW
REVERSAL_PENDING
The report strongly recommends a transaction state machine for exactly this type of
visibility and recovery.
The solution you should research and implement in the POC
Idempotency Key
+
Transaction State Machine
+
Immutable Transaction Ledger
+
Webhook Verification
+
Retry Queue
+
Reconciliation Job
Example:
User clicks Pay 3 times

Same idempotency key

Only ONE transaction created
For database consistency:
Do not delete failed transactions.
Keep:
Transaction ID
Current Status
Previous Status
Provider Reference ID
Blockchain TX Hash
Payout Reference
Failure Reason
Retry Count
Timestamps
For distributed operations, don’t try to use one giant database transaction across
Razorpay, a crypto provider, and a payout provider. Instead, document a
Saga/outbox/reconciliation approach:
Database Transaction

Save transaction + outbox event

Commit

Worker sends provider request

Webhook / polling confirms result

Advance transaction state
This is probably the most technically impressive part of your POC.
Module 5 — Domestic Payment with Razorpay
Main question
How does TransNova handle domestic payments separately from international
settlement?
Architecture:
Sender

TransNova App

TransNova Backend

Payment Order

Razorpay Checkout / Supported Payment Flow

Payment Completed

Webhook Verification

TransNova Transaction Updated
Important principle:
Frontend success
≠
Payment confirmed
Your backend should only finalize the transaction after trusted server-side verification
and/or provider webhook processing.
Suggested state:
CREATED

PAYMENT_INITIATED

PAYMENT_PROCESSING

PAYMENT_VERIFIED

COMPLETED
This module should demonstrate a real architectural separation:
Domestic Transaction

DomesticPaymentAdapter

Razorpay
International Transaction

InternationalSettlementOrchestrator

Settlement + Payout Providers
So your system doesn’t mix domestic and international transaction logic.
Module 6 — Database, Bottlenecks & System Challenges
This should be your engineering architecture module.
Suggested database structure
Users

 UserProfiles
 KYCRecords
 PaymentIdentities
 PayoutProfiles
Transactions

 TransactionEvents
 PaymentRecords
 FXQuotes
 SettlementRecords
 BlockchainTransactions
 PayoutRecords
System

 IdempotencyKeys
 WebhookEvents
 AuditLogs
 OutboxEvents
 ReconciliationRecords

 
Bottlenecks to discuss
Problem Solution
User clicks Pay multiple times Idempotency key
Provider API timeout Retry queue + status polling
Webhook arrives twice Deduplication
Blockchain confirmation delayed Pending state + confirmation worker
Payout fails after settlement Compensation/manual review
Provider is down Circuit breaker + retry
Database and provider become inconsistent Reconciliation job
Same money processed twice Immutable ledger + idempotency
Exchange rate expires Quote expiry + quote locking
High transaction volume Queue-based processing
The report also identifies transaction lifecycle management, failure handling, monitoring,
reconciliation, and provider abstraction as key architectural requirements.