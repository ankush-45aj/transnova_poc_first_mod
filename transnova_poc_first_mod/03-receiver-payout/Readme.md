Step 1 — Crypto reaches the destination side

Suppose the settlement asset is ETH.

Ethereum
     │
     │ ETH transfer
     ▼
Crypto.com wallet

TransNova's backend tracks the blockchain transaction and waits for the required confirmation/status.

Crypto.com's institutional API has a private/create-withdrawal endpoint for crypto withdrawals, and withdrawal functionality has to be enabled for the API key.

4. Step 2 — TransNova tells Crypto.com to sell the crypto

Once the asset is available on the Crypto.com side, TransNova's backend can use the trading API to execute a sell order.

Conceptually:

ETH
 │
 │ SELL
 ▼
Crypto.com Exchange
 │
 ▼
USD

Crypto.com's institutional API provides private/create-order for BUY/SELL orders.

So your backend might conceptually say:

"Sell X ETH for USD."

Crypto.com returns an order identifier.

The operation is asynchronous, so TransNova should then monitor the order status rather than assuming it completed immediately.

5. Step 3 — Wait until the conversion is complete

Your system might have:

CRYPTO_RECEIVED
       ↓
SELL_ORDER_CREATED
       ↓
SELL_ORDER_FILLED
       ↓
USD_AVAILABLE

This is where the Crypto.com WebSocket infrastructure becomes useful.

Instead of constantly asking:

"Did the order finish?"

TransNova can subscribe to relevant account/order updates and update its internal transaction state.

6. Step 4 — USD now exists in Crypto.com

Now the crypto has effectively been converted:

ETH
 ↓
Crypto.com
 ↓
SELL
 ↓
USD balance

At this point, the blockchain part of the transaction is finished.

We are back in the traditional financial system.

7. Step 5 — USD withdrawal to Rishu

This is the actual fiat off-ramp.

Crypto.com's current institutional Fedwire documentation says USD withdrawals require a linked bank account, with the account verified through the required setup process. Once linked, USD can be withdrawn to a Fedwire bank account.

Conceptually:

Crypto.com USD Balance
          │
          │ USD withdrawal
          ▼
       Fedwire
          │
          ▼
 Rishu's US Bank Account

Crypto.com currently lists the US among jurisdictions where institutional USD transfers are available.


This is important for your UPI-like UX.

Rishu shouldn't have to enter his bank details every time.

Instead, during onboarding:

Rishu
  │
  ├── Identity
  ├── KYC
  └── Bank account
          │
          ▼
    TransNova securely
    stores payout information

Then Rishu's TransNova payment ID / QR maps to his verified payout destination.

So when Ganesha scans:

QR
 ↓
Rishu's TransNova ID
 ↓
Verified payout destination

The sender doesn't need to see or manually enter:

Routing number
Account number
SWIFT
Bank name
etc.

That's where your abstraction layer creates the UPI-like experience.

Instead, for a payments company, I'd design:

                  TRANSNOVA
                      │
               Payment Engine
                      │
               Settlement Engine
                      │
                      ▼
               Crypto.com API
                      │
              Crypto → USD
                      │
                      ▼
              USD Liquidity
                      │
                      ▼
              Payout Provider
                      │
                      ▼
              Rishu's Bank

Why?

Because you don't necessarily want Crypto.com to be responsible for every individual customer payout.

You want Crypto.com to potentially provide:

crypto liquidity + conversion + settlement

while a specialized banking/BaaS/payout provider handles:

USD → customer bank account

This gives you much greater flexibility.


Instead of:

Crypto.com → Rishu's bank

you could have:

Ethereum
    ↓
Crypto.com
    ↓
Crypto → USD
    ↓
TransNova's US USD account
    ↓
US payout/BaaS API
    ↓
Rishu

So:

Crypto.com

handles:

Crypto → USD

TransNova's US banking partner

handles:

USD → Rishu


Now put everything together:

                 🇮🇳 INDIA
                     │
                 Ganesha
                     │
                Scan Rishu QR
                     │
                     ▼
              TRANSNOVA APP
                     │
                     ▼
             TRANSNOVA BACKEND
                     │
              KYC / AML / FX
                     │
                     ▼
              INR COLLECTION
                     │
                     ▼
             INR ON-RAMP
                     │
                     ▼
                 ETH / USDC
                     │
                     ▼
             CRYPTO.COM API
                     │
                     ▼
             BLOCKCHAIN
                     │
                     ▼
             CRYPTO.COM API
                     │
                SELL CRYPTO
                     │
                     ▼
                  USD
                     │
                     ▼
          US BANKING / PAYOUT API
                     │
                     ▼
              🇺🇸 RISHU'S BANK
12. Where Crypto.com's APIs are used

There are essentially three important interactions:

① Trading API
Crypto → USD

TransNova sends a sell-order request through the Crypto.com API.

② Wallet API

For crypto deposits/withdrawals and blockchain-related movement.

Crypto.com's institutional API exposes private/create-withdrawal for crypto withdrawals.

③ Fiat withdrawal infrastructure
USD → Bank

Crypto.com currently supports institutional USD withdrawals through supported rails such as Fedwire.

But: the public documentation I found does not establish a public API endpoint equivalent to private/create-withdrawal specifically for USD fiat payouts. So for your production design, you'd need Crypto.com to confirm whether institutional API-based USD payout initiation is available to your account, or whether this leg is handled through their interface/another payout provider.



your architecture could become much better with USDC

I'd seriously consider changing:

ETH → Ethereum → ETH → USD

to:

USDC → blockchain → USDC → USD

for the payment settlement layer.

Why?

Because you're building a payments network, and ETH has market-price volatility.

For example:

₹10,000
   ↓
USDC
   ↓
Blockchain
   ↓
USDC
   ↓
USD

The settlement asset remains approximately dollar-denominated during the journey.

That reduces one major source of FX/asset-price risk.

Whether USDC and the chosen network are the right production choice still requires regulatory, liquidity, fee and Crypto.com-support analysis.

The key idea

Your system should ultimately look like:

Ganesha doesn't know or care about ETH, Ethereum, Crypto.com, Fedwire, or BaaS.

He sees:

Scan Rishu's QR → ₹10,000 → Confirm

Behind the scenes:

INR → crypto settlement asset → blockchain → crypto → USD → US payout rail → Rishu

And Crypto.com is one infrastructure component in the middle, rather than being the entire payment system.


NEW MODULE 3 BASED ON 1 AND 2--
# Module 3 — Recipient Payout & Local Currency Delivery

---

# 1. Module Overview

Module 1 answers:

> "Did the sender actually pay?"

Module 2 answers:

> "How do we convert and settle that value and manage it through the TransNova wallet?"

Module 3 answers:

> "How do we finally deliver the recipient's local currency?"

Therefore:

```text
MODULE 1
Payment Collection & Verification
        ↓
PAYMENT_VERIFIED
        ↓
READY_FOR_SETTLEMENT
        ↓
MODULE 2
Settlement + Liquidity + Conversion
        ↓
Custodial Wallet
        ↓
Stablecoin → Fiat
        ↓
READY_FOR_PAYOUT
        ↓
MODULE 3
Recipient Payout
        ↓
Payout Provider
        ↓
Local Payment Rail
        ↓
Recipient Bank / Wallet
        ↓
PAYOUT_COMPLETED
Main responsibility of Module 3

Module 3 takes the fiat value prepared by Module 2 and safely delivers it to the recipient's local financial account.

For example:

Module 2
USD available
     ↓
Module 3
USD payout
     ↓
Rishu's US bank account

Module 3 is therefore the last-mile delivery layer of TransNova.

2. The Core Idea

TransNova should hide country-specific banking infrastructure from the user.

The user should simply see:

Rishu
     ↓
Verified payout destination
     ↓
Receive $XXX.XX

The user should not have to understand:

ACH
Fedwire
SEPA
Routing Number
Bank Account Number
SWIFT
Bank APIs
BaaS
Payout Provider

These are backend responsibilities.

The architecture is:

                    TRANSNOVA
                        │
                  Payout Engine
                        │
                  Payout Router
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
       India          USA          Europe
          │             │             │
          ▼             ▼             ▼
        UPI/Bank       ACH          SEPA

This is what creates the abstraction layer.

3. Module 3 Input

Module 3 should only begin when Module 2 reaches:

READY_FOR_PAYOUT

This means that the previous stages have completed.

Conceptually:

Payment
   ↓
Verification
   ↓
Settlement
   ↓
Liquidity
   ↓
Conversion
   ↓
Custody
   ↓
Fiat Available
   ↓
READY_FOR_PAYOUT

Module 3 should not assume that simply receiving a request means the money is ready.

The transaction must be in the correct state.

4. Example — Ganesha → Rishu

Suppose:

Sender:
Ganesha

Country:
India

Amount:
₹10,000

Recipient:
Rishu

Country:
USA

Module 1:

Ganesha
   ↓
₹10,000
   ↓
India Payment Provider
   ↓
PAYMENT_VERIFIED

Module 2:

READY_FOR_SETTLEMENT
   ↓
INR → USDC
   ↓
Settlement
   ↓
USDC → USD
   ↓
USD Available
   ↓
READY_FOR_PAYOUT

Now Module 3 begins.

READY_FOR_PAYOUT
        ↓
Payout Engine
        ↓
US Payout Provider
        ↓
US Payment Rail
        ↓
Rishu's Bank
5. Recipient Identity vs Payout Destination

One of the most important ideas from Module 1 must continue into Module 3.

A TransNova identity is not the same thing as a bank account.

Conceptually:

TRANSNOVA ID
     ↓
Recipient Identity
     ↓
Payout Profile
     ↓
Verified Bank Account

For example:

Rishu
  │
  ├── TransNova ID
  │
  ├── Identity / KYC
  │
  └── US Payout Profile
          │
          └── Verified US Bank Account

This means the sender does not need to know the recipient's bank details.

6. Why This Matters for the UPI-Like Experience

Traditional international payments may require the sender to provide information such as:

Bank Name
Account Number
Routing Number
SWIFT Code
IBAN
Recipient Address

TransNova hides this complexity.

The sender performs:

Scan Rishu's QR
       ↓
Rishu's TransNova ID
       ↓
Recipient Resolved
       ↓
Verified Payout Profile
       ↓
Send ₹10,000

Therefore:

The sender interacts with a TransNova identity, not with the underlying banking infrastructure.

This is one of the main reasons the architecture can provide a UPI-like user experience.

7. Payout Profile

Every recipient who wants to receive money should have a payout profile.

Conceptually:

PayoutProfile

profile_id
user_id
country
currency
payout_method
provider
status
created_at
updated_at

Example:

profile_id: PP_001
user_id: USR_002
country: USA
currency: USD
payout_method: BANK_ACCOUNT
provider: US_PAYOUT_PROVIDER
status: VERIFIED

The actual sensitive bank information should not be exposed to the sender.

8. Payout Destination

The payout profile points to a destination.

Conceptually:

Payout Profile
       ↓
Payout Destination
       ↓
Bank Account / Payment Account

The destination can contain or reference:

destination_id
account_type
country
currency
provider_reference
status
verified_at

The actual bank details should be securely stored or tokenized according to the provider's architecture.

TransNova should preferably store a secure provider reference rather than unnecessarily storing raw banking credentials.

9. Payout Provider Abstraction

Just as Module 2 uses:

SettlementProvider
        ↓
CryptoComAdapter

Module 3 should use:

PayoutProvider
        ↓
Provider Adapter

Conceptually:

                    PayoutProvider
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
      USPayoutAdapter  IndiaAdapter  SEPAAdapter
             │           │           │
             ▼           ▼           ▼
           ACH/etc.     UPI/Bank     SEPA

This prevents TransNova from becoming dependent on one payout provider.

10. Why Use a Payout Adapter?

Without an adapter:

TransNova
    ↓
Provider-specific API
    ↓
Bank

The entire application becomes tightly coupled to one provider.

With an adapter:

TransNova
    ↓
PayoutProvider Interface
    ↓
Provider Adapter
    ↓
Payout Provider
    ↓
Local Payment Rail

This allows TransNova to replace providers without changing the core payout engine.

11. PayoutProvider Interface

The core interface can conceptually contain:

createPayout()
getPayoutStatus()
verifyDestination()
cancelPayout()
getPayoutFee()
getSupportedCurrencies()
getSupportedCountries()

For example:

PayoutProvider

    createPayout()
    getPayoutStatus()
    verifyDestination()
    cancelPayout()
    getPayoutFee()
    getCapabilities()

The TransNova payout engine only communicates with this interface.

12. Payout Router

The Payout Router determines which provider and payment rail should be used.

Conceptually:

READY_FOR_PAYOUT
        ↓
Payout Router
        ↓
Determine:
    Country
    Currency
    Payout Method
    Provider Availability
    Cost
    Speed
    Risk
        ↓
Select Provider
        ↓
Create Payout

Example:

USA
 ↓
USD
 ↓
Bank Account
 ↓
US Payout Provider
 ↓
ACH

Another example:

Europe
 ↓
EUR
 ↓
Bank Account
 ↓
SEPA Provider
 ↓
SEPA
13. Payout Route Selection

TransNova should not simply choose a provider because it is available.

It should evaluate:

Compliance
Liquidity
Currency Support
Country Support
Provider Availability
Payout Cost
Expected Speed
Risk
Reliability

Conceptually:

Provider A
Cost = ₹80
Time = 1 day

Provider B
Cost = ₹50
Time = minutes

Provider C
Cost = ₹100
Time = 1 hour

If all providers satisfy the required compliance and risk rules:

Provider B

may be selected.

Therefore:

The Payout Router chooses the best available payout route.

14. Payout Quote

Before executing the payout, TransNova should calculate the final payout amount.

Example:

USD Available:
$114.45

Payout Fee:
$1.00

Recipient Receives:
$113.45

Or if the fee is paid by the sender:

USD Available:
$114.45

Payout Fee:
$1.00

Recipient Receives:
$114.45

Sender/Transaction Fee:
$1.00

The exact fee model depends on TransNova's business rules.

The important principle is:

The payout amount must be known before the payout is executed.

15. Payout Cost

Module 3 contributes another cost to the overall transaction.

The total transaction cost from Module 2 was:

Payment Fee
+
FX Cost
+
Crypto Conversion
+
Settlement
+
Custody
+
Payout
+
TransNova Fee

Module 3 is primarily responsible for:

Payout Fee
+
Network / Rail Fee
+
Provider Fee

These should be recorded separately.

16. Payout Request

Once the recipient and payout destination are validated, TransNova creates a payout request.

Example:

Payout

payout_id: PO_001

transaction_id: TXN001

recipient_id: USR_002

currency: USD

amount: 114.45

destination: DP_001

provider: US_PAYOUT_PROVIDER

status: CREATED
17. Payout Validation

Before sending money, the backend should validate:

Recipient Exists
        ↓
Payout Profile Exists
        ↓
Destination Verified
        ↓
Currency Supported
        ↓
Country Supported
        ↓
Amount Valid
        ↓
Risk Check
        ↓
Compliance Check
        ↓
Funds Available
        ↓
Create Payout

This prevents invalid payout requests.

18. Payout Funds Reservation

Before sending the payout, the relevant funds should be reserved.

Example:

USD Available = $500

Payout Request = $100

After reservation:

Available = $400
Reserved  = $100

The payout is then processed.

If successful:

Reserved
   ↓
Payout Completed

If it fails:

Reserved
   ↓
Released
   ↓
Available

This follows the same balance-state principle used in Module 2.

19. Payout State Machine

The payout should have explicit states.

READY_FOR_PAYOUT
        ↓
PAYOUT_CREATED
        ↓
DESTINATION_VALIDATED
        ↓
FUNDS_RESERVED
        ↓
PAYOUT_SUBMITTED
        ↓
PROVIDER_PROCESSING
        ↓
PAYOUT_CONFIRMED
        ↓
PAYOUT_COMPLETED

Possible failure states:

PAYOUT_VALIDATION_FAILED
PAYOUT_SUBMISSION_FAILED
PAYOUT_REJECTED
PAYOUT_FAILED
PAYOUT_CANCELLED
20. Example — US Payout

Suppose:

Recipient:
Rishu

Currency:
USD

Amount:
$114.45

The flow becomes:

READY_FOR_PAYOUT
        ↓
Validate Rishu
        ↓
Find US Payout Profile
        ↓
Verify Bank Destination
        ↓
Check USD Amount
        ↓
Reserve Funds
        ↓
US Payout Provider
        ↓
ACH / Supported US Rail
        ↓
Rishu's Bank
        ↓
PAYOUT_COMPLETED

The sender never needs to know the banking details.

21. Payout Provider API

The TransNova backend should communicate with the payout provider.

Conceptually:

TransNova Backend
       ↓
Payout Adapter
       ↓
Payout Provider API
       ↓
Banking Network
       ↓
Recipient Bank

The provider API may support operations such as:

Create Payout
Check Payout Status
Validate Bank Account
Get Fees
Get Supported Currencies
Get Supported Countries
Receive Webhooks

The exact API depends on the selected provider.

22. Webhooks

Payouts are often asynchronous.

Therefore, TransNova should not assume:

Create Payout
      ↓
Completed

Instead:

Create Payout
      ↓
Provider Processing
      ↓
Webhook / Status Update
      ↓
TransNova
      ↓
Update Payout State
23. Secure Webhook Handling

Never blindly trust a payout webhook.

The correct process is:

Provider
   ↓
Webhook
   ↓
TransNova Webhook Endpoint
   ↓
Signature Verification
   ↓
Event ID Check
   ↓
Duplicate Check
   ↓
Payout Lookup
   ↓
Validate Current State
   ↓
Update Payout
   ↓
Update Ledger

For example:

Provider says:

PAYOUT_COMPLETED

TransNova should verify that the webhook is authentic and belongs to the correct payout before updating the user's transaction state.

24. Idempotency

Payouts must be idempotent.

Example:

idempotency_key:

TXN001_PAYOUT

Suppose TransNova accidentally sends the same request three times:

Request 1
    ↓
Create payout

Request 2
    ↓
Existing payout

Request 3
    ↓
Existing payout

Only one actual payout should be created.

This is extremely important because duplicate payouts can cause real financial loss.

25. Payout Confirmation

Do not mark:

PAYOUT_COMPLETED

simply because the payout request was accepted.

There is a difference between:

PAYOUT_SUBMITTED

and:

PAYOUT_COMPLETED

For example:

PAYOUT_SUBMITTED
        ↓
Provider Processing
        ↓
Bank Processing
        ↓
PAYOUT_COMPLETED

Only the appropriate provider confirmation should move the transaction to the completed state.

26. Transaction Lifecycle

The complete TransNova transaction now looks like:

MODULE 1
        ↓
PAYMENT_VERIFIED
        ↓
READY_FOR_SETTLEMENT
        ↓
MODULE 2
        ↓
SETTLEMENT
        ↓
LIQUIDITY
        ↓
CONVERSION
        ↓
STABLECOIN
        ↓
CUSTODY
        ↓
STABLECOIN → FIAT
        ↓
READY_FOR_PAYOUT
        ↓
MODULE 3
        ↓
PAYOUT CREATED
        ↓
DESTINATION VERIFIED
        ↓
FUNDS RESERVED
        ↓
PAYOUT SUBMITTED
        ↓
PROVIDER PROCESSING
        ↓
PAYOUT CONFIRMED
        ↓
PAYOUT COMPLETED
27. Complete Ganesha → Rishu Example
Step 1 — Sender
Ganesha
India

scans:

Rishu's QR
Step 2 — Module 1

Ganesha enters:

₹10,000

Payment is verified.

PAYMENT_VERIFIED
        ↓
READY_FOR_SETTLEMENT
Step 3 — Module 2

TransNova determines the settlement route.

Conceptually:

INR
 ↓
USDC
 ↓
Blockchain / Settlement
 ↓
USDC
 ↓
USD

The resulting fiat value becomes available for payout.

READY_FOR_PAYOUT
Step 4 — Module 3

TransNova resolves:

Rishu
 ↓
TransNova ID
 ↓
US Payout Profile
 ↓
Verified US Bank Account
Step 5 — Payout
USD
 ↓
Payout Provider
 ↓
US Payment Rail
 ↓
Rishu's Bank
Step 6 — Completion

The provider confirms the payout.

PAYOUT_COMPLETED

The complete transaction is now finished.

28. The Recipient Experience

From Rishu's perspective, the process should be simple.

Before receiving money:

Rishu
   ↓
Create TransNova Account
   ↓
Complete KYC
   ↓
Add Bank Account
   ↓
Verify Bank Account
   ↓
Payout Profile ACTIVE

Then:

Ganesha scans Rishu's QR
        ↓
Ganesha sends ₹10,000
        ↓
TransNova processes transaction
        ↓
Rishu receives USD

Rishu does not need to manually provide bank details for every transaction.

29. Payout Destination Verification

A major security requirement is ensuring that the payout destination actually belongs to the recipient.

Conceptually:

Recipient
    ↓
Payout Account
    ↓
Verification
    ↓
Payout Profile = VERIFIED

Only verified destinations should be eligible for automatic payouts.

If a recipient wants to change their bank account:

Existing Bank
      ↓
Change Request
      ↓
Verification
      ↓
Risk Check
      ↓
New Bank Activated

Changing payout destinations should be treated as a sensitive operation.

30. Risk Controls

Before executing a payout, TransNova can check:

Transaction Amount
Recipient History
Payout Destination
Country
Currency
Transaction Frequency
Velocity
Risk Score
KYC Status
AML Status
Account Status

Example:

Large Payout
      ↓
Risk Score High
      ↓
Step-Up Verification
      ↓
Manual Review / Additional Verification

This prevents suspicious transactions from being automatically processed.

31. Payout Limits

Each payout route can have limits.

For example:

Minimum Payout
Maximum Payout
Daily Limit
Monthly Limit
Per-Transaction Limit

The system should therefore check:

Requested Amount
       ↓
Payout Limits
       ↓
Allowed?

If not:

PAYOUT_LIMIT_EXCEEDED
32. Payout Router Architecture

The complete routing layer can be:

                    READY_FOR_PAYOUT
                           │
                           ▼
                    Payout Router
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
          India           USA          Europe
             │             │             │
             ▼             ▼             ▼
        UPI / Bank         ACH          SEPA
             │             │             │
             ▼             ▼             ▼
        Recipient       Recipient     Recipient
          Account         Bank          Bank

This is the country-specific abstraction layer.

33. Payout Engine

The Payout Engine coordinates the entire Module 3 process.

Conceptually:

PayoutEngine
     │
     ├── Validate Recipient
     │
     ├── Validate Destination
     │
     ├── Check Risk
     │
     ├── Check Limits
     │
     ├── Calculate Fee
     │
     ├── Reserve Funds
     │
     ├── Select Provider
     │
     ├── Create Payout
     │
     ├── Monitor Status
     │
     └── Update Ledger

The Payout Engine should contain TransNova's business logic.

The actual provider-specific implementation stays inside adapters.

34. Payout Database Design

Important tables:

payouts
payout_destinations
payout_attempts
payout_provider_transactions
payout_webhook_events
payout_fees
34.1 payouts
payouts

payout_id
transaction_id
recipient_id
destination_id
currency
amount
fee
provider_id
status
created_at
updated_at
completed_at
34.2 payout_destinations
payout_destinations

destination_id
user_id
country
currency
type
provider_reference
status
verified_at
created_at
34.3 payout_attempts

Used when the system needs to retry or switch providers.

payout_attempts

attempt_id
payout_id
provider_id
attempt_number
status
error_code
error_message
created_at

Example:

Payout
  ↓
Provider A
  ↓
FAILED
  ↓
Provider B
  ↓
SUCCESS

The history of both attempts should be preserved.

35. Provider Transaction Record

The external provider will usually have its own transaction identifier.

For example:

TransNova Payout ID:
PO_001

Provider Payout ID:
PROV_982173

Both should be stored.

This allows TransNova to correlate:

TransNova Transaction
        ↕
Provider Transaction
36. Payout Reconciliation

Reconciliation must continue into Module 3.

At the end of the payout:

TransNova Ledger
       │
       ├── Payout Record
       │
       └── Provider Record
              ↓
        Reconciliation
              ↓
        MATCH / MISMATCH

Example:

TransNova:
$114.45 payout

Provider:
$114.45 payout

Result:
RECONCILED

If:

TransNova:
$114.45

Provider:
$113.45

the system should raise:

PAYOUT_RECONCILIATION_ALERT
37. Failed Payout

A payout can fail.

For example:

READY_FOR_PAYOUT
       ↓
PAYOUT_SUBMITTED
       ↓
PAYOUT_FAILED

Possible causes:

Invalid Bank Account
Provider Rejection
Insufficient Provider Liquidity
Compliance Block
Unsupported Destination
Bank Rejection
Technical Error
Timeout

The system must then determine whether the payout should:

Retry
   OR
Switch Provider
   OR
Return Funds
   OR
Require Manual Review
38. Retry Architecture

A failed payout should not simply be retried indefinitely.

Use controlled retries.

Example:

Payout Attempt 1
       ↓
FAILED
       ↓
Check Error
       ↓
Retryable?
   ┌───┴───┐
  YES      NO
   ↓        ↓
Retry    Manual Review

For example:

Temporary Network Error
        ↓
Retry

But:

Invalid Bank Account
        ↓
Do NOT automatically retry
        ↓
Require correction
39. Provider Failover

If a provider becomes unavailable:

Provider A
   ↓
Unavailable

TransNova can potentially route through:

Provider B

provided that:

Provider B
supports:

Country
Currency
Payout Method
Compliance
Limits
Liquidity

Therefore:

Payout Router
      ↓
Provider A
      ↓
FAILED
      ↓
Provider B
      ↓
SUCCESS

This improves system resilience.

40. Payout Notifications

The recipient should receive clear status updates.

For example:

Payout Created
      ↓
Processing
      ↓
Completed

The UI could display:

Your payout

$114.45

Status:
Processing

Then:

$114.45

Status:
Completed ✓

The notification should only say "Completed" after the required confirmation.

41. User-Facing Transaction Status

The user should not see internal technical states such as:

PAYOUT_PROVIDER_WEBHOOK_RECEIVED
PROVIDER_TRANSACTION_CONFIRMED
RECONCILIATION_PENDING

Instead, the application can map them to:

Processing
Completed
Failed

For example:

Internal State:
PROVIDER_PROCESSING

User sees:
Processing...

This is another abstraction layer.

42. Security Architecture

Module 3 handles real financial payouts, so security is critical.

The architecture should include:

Authentication
Authorization
KYC
AML / Risk Checks
Payout Destination Verification
Idempotency
Webhook Verification
Encryption
Secrets Management
Audit Logs
Rate Limiting
Transaction Limits
Fraud Detection
Reconciliation
43. Never Trust the Client

The frontend should never be allowed to decide:

Payout Amount
Recipient Bank Account
Provider
Payout Fee
Payout Status

Instead:

Frontend
   ↓
TransNova Backend
   ↓
Validation
   ↓
Risk / Compliance
   ↓
Payout Engine
   ↓
Payout Provider

The backend is the source of truth.

44. Payout API Endpoints

A possible Module 3 API design:

Recipient Payout Profile
POST /api/payout-profiles

GET /api/payout-profiles

GET /api/payout-profiles/{id}

POST /api/payout-profiles/{id}/verify
Payout Destination
POST /api/payout-destinations

GET /api/payout-destinations

GET /api/payout-destinations/{id}
Payout
POST /api/payouts

GET /api/payouts/{id}

POST /api/payouts/{id}/cancel
Payout Quote
POST /api/payouts/quote
Webhook
POST /api/webhooks/payout/{provider}
45. Example Payout Request

Conceptually:

{
    "transaction_id": "TXN001",
    "recipient_id": "USR_002",
    "destination_id": "DP_001",
    "currency": "USD",
    "amount": "114.45",
    "idempotency_key": "TXN001_PAYOUT"
}

The backend then determines:

Which provider?
Which rail?
What fee?
Is destination valid?
Is recipient eligible?
Is the transaction allowed?

The frontend should not make these decisions.

46. Example Payout Response
{
    "payout_id": "PO_001",
    "transaction_id": "TXN001",
    "currency": "USD",
    "amount": "114.45",
    "fee": "1.00",
    "status": "PROCESSING",
    "estimated_completion": "..."
}

The actual response fields will depend on the provider and TransNova's API design.

47. Module 3 State Machine

The complete state machine can be:

READY_FOR_PAYOUT
        ↓
PAYOUT_CREATED
        ↓
DESTINATION_VALIDATED
        ↓
RISK_CHECKED
        ↓
FUNDS_RESERVED
        ↓
PAYOUT_SUBMITTED
        ↓
PROVIDER_PROCESSING
        ↓
PAYOUT_CONFIRMED
        ↓
PAYOUT_COMPLETED

Failures:

DESTINATION_VALIDATION_FAILED
RISK_CHECK_FAILED
PAYOUT_LIMIT_EXCEEDED
PAYOUT_SUBMISSION_FAILED
PAYOUT_REJECTED
PAYOUT_FAILED
PAYOUT_CANCELLED
48. Complete Three-Module Architecture
                         MODULE 1
                 PAYMENT COLLECTION
                     + VERIFICATION
                           │
                           ▼
                  PAYMENT_VERIFIED
                           │
                           ▼
                READY_FOR_SETTLEMENT
                           │
                           ▼
                         MODULE 2
                SETTLEMENT + LIQUIDITY
                    + CONVERSION
                           │
                           ▼
                    STABLECOIN
                           │
                           ▼
                 CUSTODIAL WALLET
                           │
                           ▼
                   STABLECOIN → FIAT
                           │
                           ▼
                  READY_FOR_PAYOUT
                           │
                           ▼
                         MODULE 3
                     PAYOUT ENGINE
                           │
                           ▼
                    PAYOUT ROUTER
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
           INDIA           USA         EUROPE
             │             │             │
             ▼             ▼             ▼
          UPI/Bank        ACH          SEPA
             │             │             │
             ▼             ▼             ▼
        RECIPIENT       RECIPIENT     RECIPIENT
         ACCOUNT         BANK          BANK
49. Complete Ganesha → Rishu Architecture
🇮🇳 INDIA

Ganesha
   │
   │ Scan Rishu QR
   ▼
TransNova App
   │
   ▼
TransNova Backend
   │
   ├── Identity
   ├── KYC / AML
   ├── Risk
   └── Transaction Engine
   │
   ▼
India Payment Provider
   │
   ▼
PAYMENT_VERIFIED
   │
   ▼
READY_FOR_SETTLEMENT
   │
   ▼
Settlement Engine
   │
   ├── Liquidity Engine
   ├── Conversion Engine
   └── Fee Engine
   │
   ▼
Crypto.com Adapter
   │
   ▼
Crypto Infrastructure
   │
   ▼
USDC Settlement
   │
   ▼
TransNova Custodial Wallet
   │
   ▼
USDC → USD
   │
   ▼
READY_FOR_PAYOUT
   │
   ▼
Payout Engine
   │
   ▼
Payout Router
   │
   ▼
US Payout Provider
   │
   ▼
US Payment Rail
   │
   ▼
🇺🇸 RISHU'S BANK
   │
   ▼
PAYOUT_COMPLETED
50. The Abstraction Achieved by TransNova

The most important thing is what the user does NOT have to know.

Ganesha sees:

Scan Rishu's QR
        ↓
₹10,000
        ↓
Confirm

Rishu sees:

$XXX.XX
        ↓
Received

Behind the scenes:

INR
 ↓
Indian Payment Rail
 ↓
Settlement Engine
 ↓
Stablecoin
 ↓
Crypto Infrastructure
 ↓
USD
 ↓
Payout Router
 ↓
US Payment Provider
 ↓
ACH / Other US Rail
 ↓
Rishu's Bank

Therefore:

TransNova converts a complicated multi-system international payment into a single user-facing transaction.

51. POC Implementation Phases

Do not implement every country and every payout rail immediately.

Phase 1 — Payout Profile

Implement:

User
 ↓
Payout Profile
 ↓
Mock Bank Account
 ↓
Verified
Phase 2 — Mock Payout Provider

Implement:

PayoutProvider
      ↓
MockPayoutAdapter
      ↓
Mock Bank

Test:

Payout Creation
Payout Status
Payout Completion
Payout Failure
Phase 3 — Payout Engine

Build:

PayoutService
PayoutRouter
PayoutValidation
PayoutFeeService
Phase 4 — Webhooks

Implement:

Provider
   ↓
Webhook
   ↓
TransNova
   ↓
Validation
   ↓
Payout State Update
Phase 5 — Reconciliation

Implement:

TransNova Ledger
       ↓
Payout Records
       ↓
Provider Records
       ↓
Reconciliation
Phase 6 — Real Provider

Only after the POC works:

TransNova
   ↓
PayoutProvider
   ↓
Real Payout Provider
   ↓
Real Banking Rail

with the required commercial, technical and regulatory onboarding.

52. What NOT to Implement in the First POC

Do not start with:

❌ Multiple countries

❌ Multiple payout providers

❌ Multiple banking rails

❌ Complex beneficiary management

❌ Automated provider failover

❌ Real bank credentials

❌ Production banking integrations

❌ Complex fraud engine

Instead implement:

✅ One recipient country

✅ One payout currency

✅ One payout provider

✅ Mock bank account

✅ Payout profile

✅ Payout destination

✅ Payout engine

✅ Payout router

✅ Payout state machine

✅ Idempotency

✅ Webhooks

✅ Reconciliation

✅ Fee calculation
53. POC Demonstration

The final demonstration should look like this.

Screen 1 — Rishu Onboarding
Create TransNova Account

Name:
Rishu

Country:
USA

Currency:
USD
Screen 2 — Add Payout Account
Add Bank Account

Account Type:
Bank Account

Currency:
USD

Status:
Verification Required
Screen 3 — Verification
Bank Account

Verification:
✓ VERIFIED

Payout Profile:
ACTIVE
Screen 4 — Ganesha Sends Money
Send Money

To:
Rishu

Amount:
₹10,000

Destination:
USA
Screen 5 — Quote
You Pay:
₹10,000

Fees:
₹XXX

Recipient Receives:
$XXX.XX
Screen 6 — Module 2
Payment Verified       ✓
Liquidity Available    ✓
Conversion             ✓
Settlement             ✓
USD Available          ✓
Screen 7 — Module 3
Recipient:
Rishu

Amount:
$XXX.XX

Payout Destination:
Verified Bank Account

Status:
Processing
Screen 8 — Completed
$XXX.XX

Payout Status:
COMPLETED ✓
54. Failure Scenarios to Demonstrate

A good POC should also demonstrate failure handling.

Invalid Destination
Payout
   ↓
Destination Invalid
   ↓
PAYOUT_VALIDATION_FAILED
Insufficient Funds
Payout
   ↓
Balance Check
   ↓
Insufficient Funds
   ↓
PAYOUT_FAILED
Provider Failure
Payout
   ↓
Provider
   ↓
FAILED
   ↓
Retry / Alternative Route
Duplicate Request
Request 1
   ↓
Payout Created

Request 2
   ↓
Existing Payout

No duplicate payout
55. Module 3 Backend Structure

The backend can be organized as:

transnova-backend/

├── controllers/
│
├── services/
│   ├── payout/
│   ├── recipient/
│   ├── destination/
│   ├── risk/
│   ├── fees/
│   └── reconciliation/
│
├── providers/
│   └── payout/
│       ├── mock/
│       ├── usa/
│       ├── india/
│       └── europe/
│
├── models/
│   ├── Payout
│   ├── PayoutProfile
│   ├── PayoutDestination
│   ├── PayoutAttempt
│   └── PayoutFee
│
├── webhooks/
│
├── middleware/
│
└── config/
56. Relationship Between All Three Modules

The clean architecture is:

MODULE 1
COLLECT + VERIFY
       │
       ▼
PAYMENT_VERIFIED
       │
       ▼
MODULE 2
SETTLE + CONVERT + MANAGE
       │
       ▼
READY_FOR_PAYOUT
       │
       ▼
MODULE 3
DELIVER LOCAL FIAT
       │
       ▼
PAYOUT_COMPLETED

Each module has a clear responsibility.

57. Module Boundaries
Module 1

Responsible for:

Payment Initiation
Identity
QR Resolution
KYC
Risk
Limits
INR Collection
Payment Verification

Ends at:

READY_FOR_SETTLEMENT
Module 2

Responsible for:

Settlement
Liquidity
Conversion
Stablecoin
Custody
Wallet
Ledger
Internal Transfer
Stablecoin → Fiat

Ends at:

READY_FOR_PAYOUT
Module 3

Responsible for:

Recipient Payout Profile
Payout Destination
Payout Provider
Payout Routing
Payout Validation
Payout Execution
Payout Status
Payout Webhooks
Payout Reconciliation

Ends at:

PAYOUT_COMPLETED
58. Final TransNova Transaction Lifecycle
                    TRANSNOVA
                        │
                        ▼
              ┌─────────────────┐
              │    MODULE 1     │
              │                 │
              │ COLLECT + VERIFY│
              └────────┬────────┘
                       │
                       ▼
               PAYMENT_VERIFIED
                       │
                       ▼
             READY_FOR_SETTLEMENT
                       │
                       ▼
              ┌─────────────────┐
              │    MODULE 2     │
              │                 │
              │ SETTLE          │
              │ LIQUIDITY       │
              │ CONVERT         │
              │ CUSTODY         │
              │ WALLET          │
              └────────┬────────┘
                       │
                       ▼
              READY_FOR_PAYOUT
                       │
                       ▼
              ┌─────────────────┐
              │    MODULE 3     │
              │                 │
              │ PAYOUT ENGINE   │
              │ PAYOUT ROUTER   │
              │ PROVIDER        │
              │ BANKING RAIL    │
              └────────┬────────┘
                       │
                       ▼
                PAYOUT_COMPLETED
59. Final Definition of Module 3
Module 3 — Recipient Payout & Local Currency Delivery

Module 3 begins when Module 2 reaches READY_FOR_PAYOUT. It resolves the recipient's verified payout profile, selects an appropriate country-specific payout provider and payment rail, validates the payout destination, checks risk and limits, reserves the required funds, submits the payout, monitors provider status through APIs/webhooks, and finally records the completed payout in the TransNova ledger and reconciliation system.

The purpose is to hide country-specific banking complexity from the sender and provide a simple experience:

Scan Recipient
      ↓
Enter Amount
      ↓
Confirm
      ↓
Recipient receives local currency

while the backend handles:

Payout Provider
Payout Rail
Bank Account
Currency
Fees
Risk
Compliance
Status
Reconciliation
60. Module 3 in One Sentence
Module 3 takes FIAT READY FOR PAYOUT from Module 2,
resolves the recipient's verified payout destination,
routes the money through the appropriate local payment provider and rail,
and completes delivery into the recipient's bank or payment account.
61. Complete TransNova Architecture in One Flow
🇮🇳 Ganesha
      │
      │ Scan Rishu QR
      ▼
TRANSNOVA APP
      │
      ▼
TRANSNOVA BACKEND
      │
      ▼
MODULE 1
COLLECT + VERIFY
      │
      ▼
PAYMENT_VERIFIED
      │
      ▼
MODULE 2
SETTLE + LIQUIDITY + CONVERT
      │
      ▼
STABLECOIN
      │
      ▼
CUSTODIAL WALLET
      │
      ▼
STABLECOIN → USD
      │
      ▼
READY_FOR_PAYOUT
      │
      ▼
MODULE 3
PAYOUT ENGINE
      │
      ▼
PAYOUT ROUTER
      │
      ▼
US PAYOUT PROVIDER
      │
      ▼
US PAYMENT RAIL
      │
      ▼
🇺🇸 RISHU'S BANK
      │
      ▼
PAYOUT_COMPLETED
Final Principle
Ganesha sees:

SCAN QR
   ↓
₹10,000
   ↓
CONFIRM


Rishu sees:

$XXX.XX
   ↓
RECEIVED


TransNova handles:

Identity
+
Payment
+
KYC / AML
+
Settlement
+
Liquidity
+
Stablecoin
+
Custody
+
Conversion
+
Payout Routing
+
Banking Rails
+
Reconciliation

The user sees one payment. TransNova orchestrates the entire financial infrastructure behind it.