# GPT-Line Payment Service Repository Specification
**Service owner:** Payment / PCI-flow developer  
**Primary runtime:** Node.js 22 + TypeScript  
**Primary role:** Manage package purchase sessions, orchestrate PCI-isolated card entry during the phone call, execute CardCom settlement flow, and credit caller minutes through the Core API exactly once.

---

## 1. Mission

Build the production Payments service for GPT-Line. This repository must completely implement the “press 3 to buy minutes” flow so that a developer can work from this document alone.

The finished service must:

- expose the package list for telephony purchase menus
- create payment sessions keyed by caller phone number and selected package
- orchestrate a PCI-isolated card-entry subflow during the live phone call
- settle the transaction with CardCom
- handle provider callbacks/webhooks or provider result polling as needed
- apply credits to the Core API exactly once on approved transactions
- let Telephony poll purchase result status
- ensure raw card details never enter logs, DBs, caches, or ordinary application memory outside the PCI-isolated capture boundary
- provide admin-facing payment inspection and reconciliation endpoints

This service owns payment orchestration and settlement, but not account balance truth.

---

## 2. Hard decisions already locked

Do not change these decisions.

- Runtime: **Node.js 22**
- Language: **TypeScript**
- Framework: **Fastify**
- Settlement provider: **CardCom**
- Caller must enter credit-card details during the phone call
- Raw PAN/CVV must never enter Asterisk, Core API, Redis, Admin, browser code, or ordinary logs
- Account identifier: `phone_e164`
- Purchase-session identifier: `payment_session_id`
- Provider-settlement uniqueness identifier: `provider_transaction_id`
- Package catalog authority: **Core API**
- This service may cache package catalog data but must validate against Core
- Credits are applied only after approved provider result
- Credit application must be idempotent

---

## 3. Critical compliance boundary

### 3.1 Prohibited outcomes

The implementation must never allow:

- full card number logged anywhere
- CVV logged anywhere
- raw DTMF card-entry digits stored in ordinary app logs
- raw PAN or CVV stored in PostgreSQL
- raw PAN or CVV written into Redis
- support staff being able to retrieve raw card data later
- card details being routed through Core or Telephony

### 3.2 Required architecture

The service must orchestrate a **PCI-isolated card-entry component**.

That component may be:

- a vendor-hosted secure IVR capture service
- a PCI-compliant separate capture microcomponent/network segment
- or another secure telephony payment capture mechanism

In all cases, GPT-Line’s ordinary app stack must receive only a tokenized or outcome-level result, never raw PAN/CVV.

---

## 4. Product behavior this service must support

Caller flow:

1. Caller presses `3`
2. Telephony asks this service for the package list
3. Caller selects a package digit
4. Telephony asks this service to create a payment session
5. This service returns a `payment_session_id` and a structured Asterisk `transfer_target`
6. Telephony transfers the caller there
7. Caller enters card details in the secure leg
8. The PCI leg captures the card data securely and drives settlement via CardCom
9. This service receives the payment result
10. If approved, it applies purchased seconds through Core
11. Telephony polls for the outcome and plays the proper result prompt
12. Caller returns to the main menu

---

## 5. Package catalog

The package catalog is fixed initially.

| Digit | package_code | Hebrew name     | Price | price_agorot | granted_seconds |
|------:|--------------|-----------------|------:|-------------:|----------------:|
| 1     | P05          | חמש דקות        | 30 ₪  | 3000         | 300             |
| 2     | P10          | עשר דקות        | 50 ₪  | 5000         | 600             |
| 3     | P20          | עשרים דקות      | 90 ₪  | 9000         | 1200            |
| 4     | P40          | ארבעים דקות     | 160 ₪ | 16000        | 2400            |

Source of truth:
`GET /internal/catalog/packages` on Core.

This service must validate local package view against Core.

---

## 6. Repository deliverables

The repository must include:

- application source code
- payment-session state machine
- package-catalog client
- card-entry orchestration logic
- CardCom adapter
- callback/webhook endpoints
- result-polling endpoint for Telephony
- Core crediting client
- admin endpoints
- tests
- Dockerfile
- docker-compose
- `.env.example`
- README
- deployment runbook
- security/logging redaction rules

---

## 7. Data model

### 7.1 payment_sessions
```sql
CREATE TABLE payment_sessions (
  payment_session_id TEXT PRIMARY KEY,
  phone_e164 TEXT NOT NULL,
  package_code TEXT NOT NULL,
  package_price_agorot INTEGER NOT NULL,
  package_seconds INTEGER NOT NULL,
  status TEXT NOT NULL CHECK (status IN (
    'created','ivr_in_progress','submitted','approved','declined','cancelled','failed','credited'
  )),
  provider_name TEXT NOT NULL DEFAULT 'cardcom',
  provider_transaction_id TEXT,
  provider_result_code TEXT,
  provider_result_message TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 7.2 payment_attempt_events
```sql
CREATE TABLE payment_attempt_events (
  event_id BIGSERIAL PRIMARY KEY,
  payment_session_id TEXT NOT NULL REFERENCES payment_sessions(payment_session_id),
  event_type TEXT NOT NULL,
  payload_json JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Optional: unique index on `provider_transaction_id` where not null.

---

## 8. Payment session state machine

States:

- `created`
- `ivr_in_progress`
- `submitted`
- `approved`
- `declined`
- `cancelled`
- `failed`
- `credited`

Allowed transitions:

- `created -> ivr_in_progress`
- `ivr_in_progress -> submitted`
- `submitted -> approved`
- `submitted -> declined`
- `submitted -> failed`
- `ivr_in_progress -> cancelled`
- `approved -> credited`

Duplicate provider callbacks must not create illegal transitions or duplicate credits.

---

## 9. Internal APIs for Telephony

### 9.1 Package list

`GET /internal/telephony/packages`

**Response**
```json
{
  "packages": [
    { "digit": 1, "package_code": "P05", "name_he": "חמש דקות", "price_agorot": 3000 },
    { "digit": 2, "package_code": "P10", "name_he": "עשר דקות", "price_agorot": 5000 },
    { "digit": 3, "package_code": "P20", "name_he": "עשרים דקות", "price_agorot": 9000 },
    { "digit": 4, "package_code": "P40", "name_he": "ארבעים דקות", "price_agorot": 16000 }
  ]
}
```

This endpoint is for Telephony consumption and must always be derived from the Core catalog.

### 9.2 Start payment session

`POST /internal/telephony/payment/session/start`

**Request**
```json
{
  "phone_e164": "+972501234567",
  "package_code": "P10",
  "provider_call_id": "PJSIP-abc-00001234"
}
```

**Behavior**

1. Validate `phone_e164`
2. Validate package exists and is active against Core-backed catalog
3. Create `payment_session_id`
4. Store session with status `created`
5. Move status to `ivr_in_progress`
6. Return a fixed-structure Asterisk transfer target

**Response**
```json
{
  "payment_session_id": "pay_01JPKAT7D3W1K6R9F0N0F4Y8S2",
  "flow_type": "ivr_card_entry",
  "transfer_target": {
    "type": "asterisk_route",
    "context": "pci_capture",
    "extension": "start",
    "priority": 1
  }
}
```

This `transfer_target` contract is fixed and must not be replaced by an opaque string in public service contracts.

### 9.3 Poll payment result

`GET /internal/telephony/payment/session/:payment_session_id`

**Approved and credited**
```json
{
  "payment_session_id": "pay_01JPKAT7D3W1K6R9F0N0F4Y8S2",
  "status": "credited",
  "result_prompt": "payment_success"
}
```

**Declined**
```json
{
  "payment_session_id": "pay_01JPKAT7D3W1K6R9F0N0F4Y8S2",
  "status": "declined",
  "result_prompt": "payment_failed"
}
```

**Cancelled**
```json
{
  "payment_session_id": "pay_01JPKAT7D3W1K6R9F0N0F4Y8S2",
  "status": "cancelled",
  "result_prompt": "payment_cancelled"
}
```

**Failure**
```json
{
  "payment_session_id": "pay_01JPKAT7D3W1K6R9F0N0F4Y8S2",
  "status": "failed",
  "result_prompt": "payment_unavailable"
}
```

---

## 10. Core API contracts this service must consume

### 10.1 Fetch package catalog

`GET /internal/catalog/packages`

### 10.2 Apply approved credit

`POST /internal/payments/credit`

**Request**
```json
{
  "payment_txn_id": "txn_20260316_123",
  "phone_e164": "+972501234567",
  "package_code": "P10",
  "amount_agorot": 5000,
  "granted_seconds": 600,
  "provider_name": "cardcom",
  "provider_status": "approved"
}
```

**Response**
```json
{
  "ok": true,
  "phone_e164": "+972501234567",
  "remaining_seconds": 887
}
```

This endpoint is idempotent by `payment_txn_id`, but this service must also protect itself from duplicate local processing.

---

## 11. Card-entry orchestration requirements

### 11.1 Secure leg fields

The secure capture leg must collect:

- card number
- expiry month/year
- CVV
- Israeli ID if required by acquirer flow

### 11.2 Rules

- Raw digits must not pass through Telephony’s ordinary logs or Core
- Ordinary logs in this repo must not include raw card fields
- Vendor tokens or opaque references may be stored if they do not expose PAN/CVV

### 11.3 Result outcomes

The secure leg must return one of:

- approved with `provider_transaction_id`
- declined with result code/message
- cancelled by caller
- failed technically

### 11.4 Timeout behavior

If caller does not complete card entry in time:

- mark `cancelled` or `failed` according to the reason
- expose the result to Telephony through the poll endpoint

---

## 12. CardCom settlement adapter requirements

Implement a dedicated adapter layer.

Responsibilities:

- submit or finalize a charge from the secure capture result
- parse approval/decline responses
- verify callbacks where applicable
- normalize provider result into GPT-Line internal statuses

Provider normalization:

- approved -> `approved`
- issuer/card decline -> `declined`
- user cancel -> `cancelled`
- provider/system/network error -> `failed`

Store:

- `provider_transaction_id`
- `provider_result_code`
- sanitized `provider_result_message`

Do not store:

- PAN
- CVV
- raw secure-entry DTMF

---

## 13. Callback/webhook endpoint

`POST /provider/cardcom/callback`

Required behavior:

1. Verify authenticity of callback
2. Find `payment_session_id`
3. Upsert provider result
4. Transition state safely
5. If approved and not yet credited:
   - call Core credit endpoint
   - on success set state `credited`
6. Return HTTP 200 quickly

Idempotency:

- duplicate callbacks must not duplicate credits
- duplicate callbacks must not break the state machine

---

## 14. Failure handling

### 14.1 Core unavailable after approved payment

If charge approved but Core crediting fails temporarily:

- keep session in `approved`, not `credited`
- retry credit application with backoff
- never re-charge
- never credit twice

### 14.2 Provider unavailable before settlement

- mark `failed`
- expose `payment_unavailable`

### 14.3 Caller cancels during secure entry

- mark `cancelled`
- expose `payment_cancelled`

### 14.4 Declined payment

- mark `declined`
- expose `payment_failed`

---

## 15. Admin APIs

### 15.1 List payments

`GET /admin/payments?page=...&phone=...&status=...`

### 15.2 Get payment detail

`GET /admin/payments/:payment_session_id`

### 15.3 Reconcile payment

`POST /admin/payments/:payment_session_id/reconcile`

Behavior:

- re-check whether approved-but-not-credited should trigger credit retry
- log admin identity and result

### 15.4 Refund marker

If true refund execution is not in v1, provide an admin endpoint that marks a payment for refund workflow and records it.

---

## 16. Security and logging rules

Allowed logs:

- `payment_session_id`
- masked phone number
- package code
- high-level state transitions
- provider transaction ID if allowed
- sanitized result code

Forbidden logs:

- PAN
- CVV
- raw secure-entry DTMF
- raw card-entry request payloads
- full Authorization secrets

All request/response logging middleware must support redaction.

---

## 17. Configuration and environment variables

Provide `.env.example` including at least:

```env
PORT=4000
NODE_ENV=development

DATABASE_URL=postgres://...
CORE_API_BASE_URL=https://core.internal
CORE_API_TOKEN=replace_me

CARDCOM_API_BASE_URL=replace_me
CARDCOM_TERMINAL_ID=replace_me
CARDCOM_USERNAME=replace_me
CARDCOM_PASSWORD=replace_me

LOG_LEVEL=info
```

Include any secure-capture-leg configuration explicitly.

---

## 18. Required tests

### 18.1 Unit tests

- package validation against catalog
- state transitions
- result-prompt mapping
- idempotent callback handling
- duplicate provider transaction rejection
- safe redaction of logs
- transfer-target contract shape

### 18.2 Integration tests

With mocked Core and mocked CardCom:

- create payment session
- approved callback credits exactly once
- duplicate callback does not double-credit
- declined callback never credits
- cancelled flow returns correct prompt
- Core temporary failure after approval leaves session retryable

### 18.3 Security tests

- forbidden card fields are redacted from logs
- webhook handler does not echo sensitive request data

---

## 19. Definition of done

This repository is complete only when:

1. Telephony can fetch packages and start payment sessions
2. Caller card entry can be handed into a secure call-based payment flow
3. Approved payments are credited exactly once through Core
4. Telephony can poll a stable result prompt afterward
5. Duplicate provider callbacks do not double-credit
6. No raw card details are stored or logged in the ordinary app stack
7. The repo contains all source, tests, docs, and deployment instructions needed to run it end to end
