# REST API Reference

Reference for the Soroban Vesting Platform backend REST API. All shapes below
are taken directly from the route handlers in
[`backend/src/routes/`](../backend/src/routes/) — no fields are invented.

**Base URL:** `http://localhost:3001` (configurable via `PORT`)

**Route groups:**

| Group | Mount path | Source |
|-------|-----------|--------|
| Health | `/api/health` | [`backend/src/routes/health.js`](../backend/src/routes/health.js) |
| Vesting | `/api/vesting` | [`backend/src/routes/vesting.js`](../backend/src/routes/vesting.js) |
| Token | `/api/token` | [`backend/src/routes/token.js`](../backend/src/routes/token.js) |
| Multisig | `/api/multisig` | [`backend/src/routes/multisig.js`](../backend/src/routes/multisig.js) |

---

## Conventions

### Contract ID

Every route in the vesting, token, and multisig groups resolves a contract ID
with the same precedence:

1. `contractId` in the JSON request body
2. `contractId` as a query parameter (`?contractId=...`)
3. Environment variable — `VESTING_CONTRACT_ID`, `TOKEN_CONTRACT_ID`, or
   `MULTISIG_CONTRACT_ID` respectively

If none is available the request fails with:

```json
{ "error": "contractId is required" }
```

(HTTP 400.) `GET /api/health` and `GET /api/health/ready` do not require one.

### Success responses

- **Read endpoints (`GET`)** return HTTP `200` with
  `{ "success": true, "data": { ... } }`.
- **Mutating endpoints (`POST`)** return HTTP `202` with
  `{ "success": true, "message": "...", "data": { ... } }`. The backend does
  not submit transactions; the 202 means the transaction is ready for the
  caller to sign and submit.

### Error responses

| Status | Shape | Source |
|--------|-------|--------|
| `400` | `{ "error": "<message>" }` | Validation in the route handler |
| `404` | `{ "error": "Not found" }` | Unmatched route |
| `429` | `{ "error": "Too many requests, please slow down." }` | Rate limiter (120 requests/minute) |
| `500` | `{ "error": "<message>", "code": "INTERNAL_ERROR" }` | Central error handler |

Unhandled handler errors always produce `{ "error", "code" }` (`code` is
`INTERNAL_ERROR` for 500s, `REQUEST_ERROR` otherwise).

### Address validation

Address fields are validated with `validateStellarAddress()` (56-character
`G...` ed25519 public keys). An invalid value returns HTTP 400 with
`{ "error": "invalid Stellar address: <field>" }`.

### Stub data

Unless the contract ID and signing environment variables are configured (see
[README — Backend API](../README.md#backend-api--live-data-note)), responses
contain documented stub values (e.g. `count: 0`, `claimableAmount: 0`) and a
`note` field telling you which environment variable to set.

---

## Health

### `GET /api/health`

Liveness check. Requires no contract ID.

**Response `200`**

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | Always `"ok"` |
| `timestamp` | string | ISO-8601 timestamp |
| `network` | string | `process.env.STELLAR_NETWORK` or `"testnet"` |
| `rpc` | string | Soroban RPC URL in use |

**Example**

```bash
curl http://localhost:3001/api/health
```

```json
{
  "status": "ok",
  "timestamp": "2026-10-19T12:00:00.000Z",
  "network": "testnet",
  "rpc": "https://soroban-testnet.stellar.org"
}
```

### `GET /api/health/ready`

Readiness check. Reports which contract IDs are configured.

**Response `200`** when at least one contract ID is set, **`503`** otherwise.

| Field | Type | Description |
|-------|------|-------------|
| `ready` | boolean | `true` if any contract ID is configured |
| `contracts` | object | `{ vesting, token, multisig }` — each a boolean |

**Example (not ready, HTTP 503)**

```json
{ "ready": false, "contracts": { "vesting": false, "token": false, "multisig": false } }
```

---

## Vesting — `/api/vesting`

### `POST /api/vesting/schedule`

Create a vesting schedule.

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `contractId` | string | see [Contract ID](#contract-id) | Vesting contract ID |
| `from` | string | yes | Valid Stellar address (funder/owner) |
| `beneficiary` | string | yes | Valid Stellar address |
| `tokenAddress` | string | yes | Valid Stellar address of the token contract |
| `totalAmount` | number | yes | Must be `> 0` |
| `startTime` | number | no | Unix seconds; defaults to now |
| `cliffDuration` | number | no | Seconds; defaults to `0`; must be `<= totalDuration` |
| `totalDuration` | number | yes\* | Total vesting duration in seconds |

\* `totalDuration` is not checked by the required-field guard but is echoed in
the response and used by the `cliffDuration <= totalDuration` check; always
send it.

**Success `202`**

```json
{
  "success": true,
  "message": "Schedule creation queued. Sign and submit the transaction.",
  "data": {
    "contractId": "C...",
    "from": "G...",
    "beneficiary": "G...",
    "tokenAddress": "C...",
    "totalAmount": 1000000,
    "startTime": 1760000000,
    "cliffDuration": 0,
    "totalDuration": 86400
  }
}
```

**Errors**

| Status | `error` |
|--------|---------|
| 400 | `Missing required fields` |
| 400 | `invalid Stellar address: from` / `beneficiary` / `tokenAddress` |
| 400 | `totalAmount must be positive` |
| 400 | `cliffDuration cannot exceed totalDuration` |
| 400 | `contractId is required` |

### `GET /api/vesting/schedule/:id`

Read a schedule by ID.

**Path params**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | integer string | yes | Must be a positive integer (`/^\d+$/` and `> 0`) |

**Success `200`**

```json
{
  "success": true,
  "data": { "id": 1, "contractId": "C...", "note": "Connect VESTING_CONTRACT_ID and SOURCE_SECRET_KEY to read live data." }
}
```

**Errors:** `400` `scheduleId must be a positive integer`; `400` `contractId is required`.

### `GET /api/vesting/claimable/:id`

Claimable amount for a schedule.

**Path params:** `id` (string, required) — a positive integer in the `u64`
range.

When `VESTING_CONTRACT_ID` and `SOURCE_SECRET_KEY` are configured, the route
simulates `get_claimable(schedule_id)` against the contract and returns the
on-chain `i128` amount as a decimal string. If either variable is absent, it
returns the documented stub amount (`0`).

**Success `200` — live response**

```json
{
  "success": true,
  "data": { "scheduleId": 1, "claimableAmount": "42000", "contractId": "C..." }
}
```

Stub response:

```json
{
  "success": true,
  "data": { "scheduleId": 1, "claimableAmount": 0, "note": "Set VESTING_CONTRACT_ID to query live contract." }
}
```

**Errors:** `400` `scheduleId must be a positive u64 integer`.

### `POST /api/vesting/claim`

Claim vested tokens.

**Request body**

| Field | Type | Required |
|-------|------|----------|
| `contractId` | string | see [Contract ID](#contract-id) |
| `scheduleId` | number/string | yes |
| `beneficiary` | string (valid Stellar address) | yes |

**Success `202`**

```json
{
  "success": true,
  "message": "Claim transaction ready for signing.",
  "data": { "scheduleId": 1, "beneficiary": "G...", "contractId": "C..." }
}
```

**Errors:** `400` `scheduleId and beneficiary are required`;
`400` `invalid Stellar address: beneficiary`.

### `POST /api/vesting/revoke`

Revoke a schedule (admin).

**Request body**

| Field | Type | Required |
|-------|------|----------|
| `contractId` | string | see [Contract ID](#contract-id) |
| `scheduleId` | number/string | yes |
| `recipient` | string (valid Stellar address) | yes |

**Success `202`**

```json
{
  "success": true,
  "message": "Revoke transaction ready for signing.",
  "data": { "scheduleId": 1, "recipient": "G...", "contractId": "C..." }
}
```

**Errors:** `400` `scheduleId and recipient are required`;
`400` `invalid Stellar address: recipient`.

### `POST /api/vesting/pause`

Pause the vesting contract (admin). No body fields beyond `contractId`.

**Success `202`**

```json
{ "success": true, "message": "Pause transaction ready." }
```

### `POST /api/vesting/unpause`

Unpause the vesting contract (admin). No body fields beyond `contractId`.

**Success `202`**

```json
{ "success": true, "message": "Unpause transaction ready." }
```

### `GET /api/vesting/beneficiary/:addr`

List schedule IDs for a beneficiary.

**Path params:** `addr` (string, required) — echoed back, no format validation
in this handler.

**Success `200`**

```json
{
  "success": true,
  "data": { "beneficiary": "G...", "scheduleIds": [], "contractId": "C..." }
}
```

### `GET /api/vesting/count`

Total schedule count.

**Success `200`**

```json
{ "success": true, "data": { "count": 0, "contractId": "C..." } }
```

**Example**

```bash
curl "http://localhost:3001/api/vesting/count?contractId=C..."
```

```json
{ "success": true, "data": { "count": 0, "contractId": "C..." } }
```

---

## Token — `/api/token`

### `GET /api/token/info`

Token metadata.

**Success `200`**

| Field | Type |
|-------|------|
| `contractId` | string |
| `name` | string (`"Vesting Token"` stub) |
| `symbol` | string (`"VEST"` stub) |
| `decimals` | number (`7`) |
| `totalSupply` | number (`0` stub) |
| `note` | string |

```json
{
  "success": true,
  "data": {
    "contractId": "C...",
    "name": "Vesting Token",
    "symbol": "VEST",
    "decimals": 7,
    "totalSupply": 0,
    "note": "Set TOKEN_CONTRACT_ID to read live data."
  }
}
```

### `GET /api/token/balance/:addr`

Token balance for an address.

**Path params:** `addr` (string, required) — valid Stellar address. When
`TOKEN_CONTRACT_ID` **and** `SOURCE_SECRET_KEY` are set, the handler validates
the address (400 on failure) and simulates `balance(account)` on-chain;
otherwise it returns a stub without address validation.

**Success `200` (live)**

```json
{
  "success": true,
  "data": { "address": "G...", "balance": "1000000", "contractId": "C..." }
}
```

**Success `200` (stub)**

```json
{
  "success": true,
  "data": {
    "address": "G...",
    "balance": 0,
    "contractId": "C...",
    "note": "Set TOKEN_CONTRACT_ID and SOURCE_SECRET_KEY to read live balance."
  }
}
```

Note: `balance` is a **string** in live mode (bigint serialization) and a
**number** in stub mode.

**Errors:** `400` `invalid Stellar address: addr` (live mode only);
`400` `contractId is required`; `500` if the RPC simulation fails.

### `GET /api/token/allowance`

Allowance between an owner and a spender.

**Query params**

| Param | Type | Required |
|-------|------|----------|
| `owner` | string | yes |
| `spender` | string | yes |
| `contractId` | string | see [Contract ID](#contract-id) |

**Success `200`**

```json
{
  "success": true,
  "data": { "owner": "G...", "spender": "G...", "allowance": 0, "contractId": "C..." }
}
```

**Errors:** `400` `owner and spender query params required`.

**Example**

```bash
curl "http://localhost:3001/api/token/allowance?owner=GAAA...&spender=GBBB..."
```

### `POST /api/token/mint`

Mint tokens (admin).

**Request body**

| Field | Type | Required |
|-------|------|----------|
| `contractId` | string | see [Contract ID](#contract-id) |
| `to` | string (valid Stellar address) | yes |
| `amount` | number, `> 0` | yes |

**Success `202`**

```json
{
  "success": true,
  "message": "Mint transaction ready.",
  "data": { "to": "G...", "amount": 1000, "contractId": "C..." }
}
```

**Errors:** `400` `to and amount required`; `400` `invalid Stellar address: to`;
`400` `amount must be positive`.

### `POST /api/token/burn`

Burn tokens.

**Request body**

| Field | Type | Required |
|-------|------|----------|
| `contractId` | string | see [Contract ID](#contract-id) |
| `from` | string (valid Stellar address) | yes |
| `amount` | number | yes |

**Success `202`**

```json
{
  "success": true,
  "message": "Burn transaction ready.",
  "data": { "from": "G...", "amount": 100, "contractId": "C..." }
}
```

**Errors:** `400` `from and amount required`; `400` `invalid Stellar address: from`.

### `POST /api/token/transfer`

Transfer tokens.

**Request body**

| Field | Type | Required |
|-------|------|----------|
| `contractId` | string | see [Contract ID](#contract-id) |
| `from` | string (valid Stellar address) | yes |
| `to` | string (valid Stellar address) | yes |
| `amount` | number | yes |

**Success `202`**

```json
{
  "success": true,
  "message": "Transfer transaction ready.",
  "data": { "from": "G...", "to": "G...", "amount": 100, "contractId": "C..." }
}
```

**Errors:** `400` `from, to and amount required`;
`400` `invalid Stellar address: from` / `to`.

### `POST /api/token/approve`

Approve a spender allowance.

**Request body**

| Field | Type | Required |
|-------|------|----------|
| `contractId` | string | see [Contract ID](#contract-id) |
| `owner` | string (valid Stellar address) | yes |
| `spender` | string (valid Stellar address) | yes |
| `amount` | number | yes — must be present; `0` is allowed (`amount === undefined` is the only failure) |

**Success `202`**

```json
{
  "success": true,
  "message": "Approve transaction ready.",
  "data": { "owner": "G...", "spender": "G...", "amount": 0, "contractId": "C..." }
}
```

**Errors:** `400` `owner, spender and amount required`;
`400` `invalid Stellar address: owner` / `spender`.

### `POST /api/token/pause`

Pause the token (admin). No body fields beyond `contractId`.

**Success `202`**

```json
{ "success": true, "message": "Pause transaction ready." }
```

### `POST /api/token/unpause`

Unpause the token (admin). No body fields beyond `contractId`.

**Success `202`**

```json
{ "success": true, "message": "Unpause transaction ready." }
```

---

## Multisig — `/api/multisig`

### `POST /api/multisig/proposal`

Submit a proposal.

**Request body**

| Field | Type | Required |
|-------|------|----------|
| `contractId` | string | see [Contract ID](#contract-id) |
| `proposer` | string (valid Stellar address) | yes |
| `description` | string | yes |

**Success `202`**

```json
{
  "success": true,
  "message": "Proposal transaction ready for signing.",
  "data": { "proposer": "G...", "description": "Increase threshold to 3", "contractId": "C..." }
}
```

**Errors:** `400` `proposer and description are required`;
`400` `proposer is required`; `400` `invalid Stellar address: proposer`.

**Example**

```bash
curl -X POST http://localhost:3001/api/multisig/proposal \
  -H 'Content-Type: application/json' \
  -d '{"proposer":"GAAA...","description":"Increase threshold to 3"}'
```

```json
{
  "success": true,
  "message": "Proposal transaction ready for signing.",
  "data": {
    "proposer": "GAAA...",
    "description": "Increase threshold to 3",
    "contractId": "C..."
  }
}
```

### `GET /api/multisig/proposal/:id`

Read a proposal by ID.

**Path params:** `id` (string, required) — echoed as `Number(id)`, no format
validation in this handler.

**Success `200`**

```json
{
  "success": true,
  "data": { "id": 1, "contractId": "C...", "note": "Set MULTISIG_CONTRACT_ID to read live data." }
}
```

### `POST /api/multisig/confirm`

Confirm a proposal.

**Request body**

| Field | Type | Required |
|-------|------|----------|
| `contractId` | string | see [Contract ID](#contract-id) |
| `owner` | string (valid Stellar address) | yes |
| `proposalId` | number/string | yes |

**Success `202`**

```json
{
  "success": true,
  "message": "Confirm transaction ready.",
  "data": { "owner": "G...", "proposalId": 1, "contractId": "C..." }
}
```

**Errors:** `400` `owner and proposalId are required`;
`400` `owner is required`; `400` `invalid Stellar address: owner`.

### `POST /api/multisig/revoke-confirm`

Revoke a confirmation.

**Request body:** identical to `POST /confirm` (`owner`, `proposalId`,
optional `contractId`).

**Success `202`**

```json
{
  "success": true,
  "message": "Revoke confirmation transaction ready.",
  "data": { "owner": "G...", "proposalId": 1, "contractId": "C..." }
}
```

**Errors:** `400` `owner and proposalId are required`; `400` `owner is required`;
`400` `invalid Stellar address: owner`.

### `POST /api/multisig/execute`

Execute a ready proposal.

**Request body**

| Field | Type | Required |
|-------|------|----------|
| `contractId` | string | see [Contract ID](#contract-id) |
| `proposalId` | number/string | yes |

**Success `202`**

```json
{
  "success": true,
  "message": "Execute transaction ready.",
  "data": { "proposalId": 1, "contractId": "C..." }
}
```

**Errors:** `400` `proposalId is required`.

### `POST /api/multisig/cancel`

Cancel a proposal.

**Request body**

| Field | Type | Required |
|-------|------|----------|
| `contractId` | string | see [Contract ID](#contract-id) |
| `caller` | string (valid Stellar address) | yes |
| `proposalId` | number/string | yes |

**Success `202`**

```json
{
  "success": true,
  "message": "Cancel transaction ready.",
  "data": { "caller": "G...", "proposalId": 1, "contractId": "C..." }
}
```

**Errors:** `400` `caller and proposalId are required`; `400` `caller is required`;
`400` `invalid Stellar address: caller`.

### `GET /api/multisig/owners`

List current owners.

**Success `200`**

```json
{ "success": true, "data": { "owners": [], "contractId": "C..." } }
```

### `GET /api/multisig/threshold`

Current confirmation threshold.

**Success `200`**

```json
{ "success": true, "data": { "threshold": 0, "contractId": "C..." } }
```

### `GET /api/multisig/count`

Total proposal count.

**Success `200`**

```json
{ "success": true, "data": { "count": 0, "contractId": "C..." } }
```

---

## Endpoint index

| Method | Path | Success |
|--------|------|---------|
| GET | `/api/health` | 200 |
| GET | `/api/health/ready` | 200 / 503 |
| POST | `/api/vesting/schedule` | 202 |
| GET | `/api/vesting/schedule/:id` | 200 |
| GET | `/api/vesting/claimable/:id` | 200 |
| POST | `/api/vesting/claim` | 202 |
| POST | `/api/vesting/revoke` | 202 |
| POST | `/api/vesting/pause` | 202 |
| POST | `/api/vesting/unpause` | 202 |
| GET | `/api/vesting/beneficiary/:addr` | 200 |
| GET | `/api/vesting/count` | 200 |
| GET | `/api/token/info` | 200 |
| GET | `/api/token/balance/:addr` | 200 |
| GET | `/api/token/allowance` | 200 |
| POST | `/api/token/mint` | 202 |
| POST | `/api/token/burn` | 202 |
| POST | `/api/token/transfer` | 202 |
| POST | `/api/token/approve` | 202 |
| POST | `/api/token/pause` | 202 |
| POST | `/api/token/unpause` | 202 |
| POST | `/api/multisig/proposal` | 202 |
| GET | `/api/multisig/proposal/:id` | 200 |
| POST | `/api/multisig/confirm` | 202 |
| POST | `/api/multisig/revoke-confirm` | 202 |
| POST | `/api/multisig/execute` | 202 |
| POST | `/api/multisig/cancel` | 202 |
| GET | `/api/multisig/owners` | 200 |
| GET | `/api/multisig/threshold` | 200 |
| GET | `/api/multisig/count` | 200 |
