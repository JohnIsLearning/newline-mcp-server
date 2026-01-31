# Transaction Reporting Skill Guide

**Prerequisite**: [Managing Users and Accounts Skill](managing-users-and-accounts-skill.md) and [Making Payments Skill](making-payments-skill.md)

**Related Skills**: [Payment Operations Skill](payment-operations-skill.md) for monitoring returns and reversals

This guide provides step-by-step instructions for agents to query transaction history, monitor transfer status, check account balances, and track transaction events in Newline's payment system.

## Overview

Transaction reporting provides visibility into:
- **Transaction History** — Complete record of all payments and transfers
- **Transaction Status** — Current state of any transaction (queued, pending, settled, failed)
- **Transaction Events** — Audit trail of status changes and events
- **Account Balances** — Current balance via pool balances (aggregate of accounts)
- **Virtual Reference Numbers** — Tracking identifiers for transactions

## Step 1: List All Transactions

Query all transactions in the system with optional filtering.

**Tool**: `newline-list-transactions`

**Optional Parameters**:
- `limit` — Number of results to return (default: 10)
- `offset` — Pagination offset (default: 0)

**Example Response Fields**:
```json
{
  "transactions": [
    {
      "uid": "txn_001",
      "external_uid": "wire_20260131_001",
      "transaction_type": "wire",
      "status": "settled",
      "amount_cents": 50000,
      "currency": "USD",
      "customer_uid": "cust_456def",
      "created_at": "2026-01-31T15:00:00Z",
      "settled_at": "2026-01-31T15:30:00Z"
    }
  ],
  "total_count": 42
}
```

**Link to Source**: [src/transactions.ts](../../src/transactions.ts)

**Use Cases**:
- Retrieve recent transactions for a customer
- Paginate through transaction history using `limit` and `offset`
- Check if a transfer was recorded

**Common Scenarios**:
- List 20 most recent transactions: `limit: 20, offset: 0`
- Paginate to next 20: `limit: 20, offset: 20`

---

## Step 2: Get Transaction Details

Retrieve detailed information about a specific transaction.

**Tool**: `newline-get-transaction`

**Required Parameters**:
- `uid` — Transaction unique identifier (from list-transactions)

**Example Response Fields**:
```json
{
  "uid": "txn_001",
  "external_uid": "wire_20260131_001",
  "transaction_type": "wire",
  "status": "settled",
  "amount_cents": 50000,
  "currency": "USD",
  "customer_uid": "cust_456def",
  "sending_account_uid": "syn_789ghi",
  "receiving_account": {
    "account_number": "123456789",
    "routing_number": "987654321",
    "account_holder_name": "Acme Corp"
  },
  "created_at": "2026-01-31T15:00:00Z",
  "settled_at": "2026-01-31T15:30:00Z",
  "metadata": {...}
}
```

**Link to Source**: [src/transactions.ts](../../src/transactions.ts)

**Key Fields**:
- `status` — Current state (queued, pending, settled, failed)
- `settled_at` — When transaction completed (null if not yet settled)
- `receiving_account` — Recipient details for verification
- `transaction_type` — Type of transaction (wire, ach, rtp, etc.)

**Common Scenarios**:
- Verify a transfer was processed correctly
- Check settlement date of a payment
- Confirm recipient account details

**Common Issues**:
- `unknown_transaction` — Verify transaction UID exists (use list-transactions)
- Status is "failed" — Review transaction details for error information

---

## Step 3: Monitor Transaction Status

Track a transaction's progress from initiation to settlement.

**Tool**: `newline-get-transaction`

**Monitoring Strategy**:
1. Get transaction details with Step 2
2. Check `status` field for current state:
   - `queued` — Transaction received, waiting for processing
   - `pending` — Processing in progress
   - `settled` — Completed successfully
   - `failed` — Transaction failed
3. If `status` is `settled` or `failed`, monitoring is complete
4. Otherwise, wait and retry based on expected processing time:
   - RTP (Real-Time Payments): 5-10 second intervals
   - Wire: 30-60 second intervals (typically settles same-day)
   - ACH: 1-2 minute intervals (typically settles 1-2 business days)

**Example Status Flow**:
```
created_at: 2026-01-31T15:00:00Z, status: queued
↓ (5 seconds later)
status: pending
↓ (5 seconds later)
status: settled, settled_at: 2026-01-31T15:00:30Z
```

---

## Step 4: List Transaction Events

Get the event history for a transaction showing all status changes.

**Tool**: `newline-list-transaction-events`

**Optional Parameters**:
- `limit` — Number of events to return (default: 10)
- `offset` — Pagination offset (default: 0)

**Example Response Fields**:
```json
{
  "transaction_events": [
    {
      "uid": "event_001",
      "transaction_uid": "txn_001",
      "event_type": "transfer_initiated",
      "status": "queued",
      "created_at": "2026-01-31T15:00:00Z"
    },
    {
      "uid": "event_002",
      "transaction_uid": "txn_001",
      "event_type": "transfer_processing",
      "status": "pending",
      "created_at": "2026-01-31T15:00:15Z"
    },
    {
      "uid": "event_003",
      "transaction_uid": "txn_001",
      "event_type": "transfer_settled",
      "status": "settled",
      "created_at": "2026-01-31T15:00:30Z"
    }
  ],
  "total_count": 3
}
```

**Link to Source**: [src/transaction_events.ts](../../src/transaction_events.ts)

**Key Fields**:
- `event_type` — Type of event (transfer_initiated, transfer_processing, transfer_settled, etc.)
- `status` — Status at time of event
- `created_at` — When the event occurred

**Use Cases**:
- Get full audit trail for a transaction
- Verify each processing step was completed
- Identify when a transaction settled
- Troubleshoot failed transfers by reviewing event sequence

---

## Step 5: Get Transaction Event Details

Retrieve detailed information about a specific event.

**Tool**: `newline-get-transaction-event`

**Required Parameters**:
- `uid` — Transaction event unique identifier (from list-transaction-events)

**Example Response Fields**:
```json
{
  "uid": "event_003",
  "transaction_uid": "txn_001",
  "event_type": "transfer_settled",
  "status": "settled",
  "created_at": "2026-01-31T15:00:30Z",
  "metadata": {
    "settlement_time": "2026-01-31T15:00:30Z",
    "confirmation_code": "CONF123456"
  }
}
```

**Link to Source**: [src/transaction_events.ts](../../src/transaction_events.ts)

---

## Step 6: List Pools (Account Balances)

Query available pools to check aggregate account balances.

**Tool**: `newline-list-pools`

**Optional Parameters**:
- `limit` — Number of pools to return (default: 10)
- `offset` — Pagination offset (default: 0)

**Example Response Fields**:
```json
{
  "pools": [
    {
      "uid": "pool_001",
      "customer_uid": "cust_456def",
      "pool_type": "operating",
      "balance_cents": 500000,
      "currency": "USD",
      "created_at": "2026-01-15T10:00:00Z"
    }
  ],
  "total_count": 3
}
```

**Link to Source**: [src/pools.ts](../../src/pools.ts)

**Key Fields**:
- `balance_cents` — Current balance in cents (e.g., 500000 = $5,000.00)
- `pool_type` — Type of pool (operating, reserve, etc.)
- `currency` — Currency (typically "USD")

**Use Cases**:
- Check available balance before initiating transfer
- Monitor aggregate balances across customer accounts
- Track balance changes over time

---

## Step 7: Get Pool Details

Retrieve detailed information about a specific pool.

**Tool**: `newline-get-pool`

**Required Parameters**:
- `uid` — Pool unique identifier (from list-pools)

**Example Response Fields**:
```json
{
  "uid": "pool_001",
  "customer_uid": "cust_456def",
  "pool_type": "operating",
  "balance_cents": 500000,
  "currency": "USD",
  "accounts": [
    {
      "account_uid": "syn_789ghi",
      "balance_cents": 250000
    },
    {
      "account_uid": "syn_xyz123",
      "balance_cents": 250000
    }
  ],
  "created_at": "2026-01-15T10:00:00Z"
}
```

**Link to Source**: [src/pools.ts](../../src/pools.ts)

**Key Fields**:
- `accounts` — Array of accounts in the pool with individual balances
- `balance_cents` — Total pool balance (sum of all accounts)

---

## Step 8: List Virtual Reference Numbers

Query Virtual Reference Numbers (VRNs) for transaction tracking.

**Tool**: `newline-list-virtual-reference-numbers`

**Optional Parameters**:
- `limit` — Number of VRNs to return (default: 10)
- `offset` — Pagination offset (default: 0)

**Example Response Fields**:
```json
{
  "virtual_reference_numbers": [
    {
      "uid": "vrn_001",
      "synthetic_account_uid": "syn_789ghi",
      "reference_number": "REF-20260131-001",
      "status": "active",
      "transaction_uid": "txn_001",
      "created_at": "2026-01-31T14:00:00Z"
    }
  ],
  "total_count": 25
}
```

**Link to Source**: [src/virtual_reference_numbers.ts](../../src/virtual_reference_numbers.ts)

**Key Fields**:
- `reference_number` — External reference identifier
- `status` — Current status (active, inactive, etc.)
- `transaction_uid` — Associated transaction (if linked)

**Use Cases**:
- Track specific payments by reference number
- Link external reference IDs to transactions
- Generate reporting data for reconciliation

---

## Step 9: Get Virtual Reference Number Details

Retrieve detailed information about a specific VRN.

**Tool**: `newline-get-virtual-reference-number`

**Required Parameters**:
- `uid` — VRN unique identifier (from list-virtual-reference-numbers)

**Example Response Fields**:
```json
{
  "uid": "vrn_001",
  "synthetic_account_uid": "syn_789ghi",
  "reference_number": "REF-20260131-001",
  "status": "active",
  "transaction_uid": "txn_001",
  "created_at": "2026-01-31T14:00:00Z",
  "metadata": {...}
}
```

**Link to Source**: [src/virtual_reference_numbers.ts](../../src/virtual_reference_numbers.ts)

---

## Step 10: Query Customer Activities

View activity audit trail for a customer.

**Tool**: `newline-list-customer-activities`

**Optional Parameters**:
- `limit` — Number of activities to return (default: 10)
- `offset` — Pagination offset (default: 0)

**Example Response Fields**:
```json
{
  "customer_activities": [
    {
      "uid": "activity_001",
      "customer_uid": "cust_456def",
      "activity_type": "transfer_initiated",
      "description": "Wire transfer initiated for $500.00",
      "created_at": "2026-01-31T15:00:00Z"
    }
  ],
  "total_count": 87
}
```

**Link to Source**: [src/customer_activities.ts](../../src/customer_activities.ts)

**Key Fields**:
- `activity_type` — Type of activity (transfer_initiated, account_created, etc.)
- `description` — Human-readable description
- `created_at` — When activity occurred

**Use Cases**:
- Full audit trail of customer actions
- Compliance and reconciliation reporting
- Historical activity review

---

## Common Reporting Workflows

### Workflow 1: Verify a Transfer Completed

1. **List transactions** (Step 1) — Get recent transfers
2. **Get transaction details** (Step 2) — Find the specific transfer
3. **Check status** — Verify status is "settled"
4. **Get transaction events** (Step 4) — Review full event history
5. **Confirm settlement time** (Step 5) — Verify settled_at timestamp

### Workflow 2: Check Available Balance Before Transfer

1. **List pools** (Step 6) — Get all account pools
2. **Get pool details** (Step 7) — Review balance and account breakdown
3. **Compare balance to transfer amount** — Ensure sufficient funds
4. **Proceed with transfer** if balance is adequate

### Workflow 3: Generate Daily Activity Report

1. **List transactions** (Step 1) — Get transactions with pagination
2. **Get details for each transaction** (Step 2) — Collect full details
3. **List customer activities** (Step 10) — Get activity audit trail
4. **Compile report** with transaction summaries, statuses, amounts

### Workflow 4: Troubleshoot a Failed Transfer

1. **List transactions** (Step 1) — Find the failed transfer
2. **Get transaction details** (Step 2) — Review error details and timestamps
3. **Get transaction events** (Step 4) — Review event sequence to identify failure point
4. **Get event details** (Step 5) — Review metadata for specific error information
5. **Take corrective action** — Retry with corrected information or contact support

---

## Status Codes Reference

| Status | Meaning | Next Action |
|--------|---------|------------|
| `queued` | Transaction received and queued | Wait for processing |
| `pending` | Processing in progress | Continue monitoring |
| `settled` | Completed successfully | Task complete; no action needed |
| `failed` | Transaction failed | Review error details; retry if applicable |

---

## Next Steps

For advanced reporting and payment workflows:
- [Payment Operations Skill](payment-operations-skill.md) — Handle returns, reversals, track return status
- [Making Payments Skill](making-payments-skill.md) — Initiate new transfers based on balance/reporting data
