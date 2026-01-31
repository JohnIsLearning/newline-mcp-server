# Payment Operations Skill Guide

**Prerequisites**: 
- [Managing Users and Accounts Skill](managing-users-and-accounts-skill.md)
- [Making Payments Skill](making-payments-skill.md)

**Related Skills**: [Transaction Reporting Skill](transaction-reporting-skill.md) for monitoring transfer and return status

This guide provides step-by-step instructions for agents to handle advanced payment operations including: Wire and ACH transfers with detailed workflows, ACH reversals (for Newline-originated payments), Wire and ACH returns (for received payments), combined transfer tracking, and error recovery specific to each payment type.

## Overview: Transfer Types and Return Support

| Aspect | Wire Transfer | ACH Transfer | Instant Payment (RTP) |
|--------|---------------|--------------|----------------------|
| **Return Support** | ✓ Yes (received wires only) | ✓ Yes (received ACH only) | ✗ No returns |
| **Reversal Support** | ✗ No reversals | ✓ Yes (Newline-originated only) | ✗ No reversals |
| **Processing Speed** | Same-day/next-day | 1-2 business days | Real-time (seconds) |
| **Return Window** | Varies by system | Time-bound by SEC code | N/A |
| **Reversal Window** | N/A | 5 business days (Newline-originated) | N/A |

---

# Wire Transfer Operations

## Step 1: Initiate a Wire Transfer (Detailed)

Reference the [Making Payments Skill: Wire Transfers](making-payments-skill.md#step-2a-initiate-a-wire-transfer) for basic wire transfer initiation.

**Tool**: `newline-create-transfer`

**Transfer Type**: `"wire"`

**Key Requirements for Wire Transfers**:
- **Sending Account** — Must be synthetic account with wire enabled
- **Recipient Details** — Account number, routing number, account holder name
- **Amount** — In cents (integer)
- **Wire Code** (optional) — For special routing requirements
- **Intermediary Bank** (optional) — Required for international wires or correspondent routing

**Link to Source**: [src/transfers.ts](../../src/transfers.ts) and [API Reference](https://developers.newline53.com/reference/transfers)

---

## Step 2: Monitor Wire Transfer Status

Use [Transaction Reporting Skill: Monitor Transaction Status](transaction-reporting-skill.md#step-3-monitor-transaction-status) to track wire settlement.

**Expected Timeline**:
- `queued` → `pending` → `settled` (typically 5 minutes to same business day)

---

## Step 3: Handle Wire Transfer Errors

**Common Issues & Wire Transfer Errors**:

| Error Code | Error Name | Cause | Solution |
|------------|-----------|-------|----------|
| 400 | Invalid account number format | Malformed account number | Verify account number format with recipient bank |
| 400 | Invalid routing number | Non-existent or incorrect routing | Verify 9-digit routing number matches recipient's bank |
| 400 | Sending account not found | Account UID incorrect or deleted | Use list-synthetic-accounts to verify account exists |
| 400 | Transfer method not enabled | Account doesn't support wire | Use different account or contact support |
| 31009 | extra_payment_information | Unnecessary wire fields provided | Remove intermediary_bank or wire_transmitter if not needed |
| 31010 | insufficient_information_for_return | Missing required wire data | Ensure account_number and routing_number provided |
| 31008 | network_code_unsupported | Wire code not supported | Try without wire_code or verify code with support |

**Recovery Steps**:
1. Verify recipient details (account number, routing number) with recipient
2. Confirm sending account has wire enabled
3. Retry with corrected information
4. If persistent, contact support with transfer UID

---

# ACH Transfer Operations

## Step 1: Initiate an ACH Transfer (Detailed)

Reference the [Making Payments Skill: ACH Transfers](making-payments-skill.md#step-2b-initiate-an-ach-transfer) for basic ACH transfer initiation.

**Tool**: `newline-create-transfer`

**Transfer Type**: `"ach"`

**Key Requirements for ACH Transfers**:
- **Sending Account** — Must be synthetic account with ACH enabled
- **Recipient Details** — Account number, routing number (must be valid ACH routing), account holder name
- **Amount** — In cents (integer)
- **Prenote Flag** (optional) — Set `true` for first-time recipients
- **Effective Date** (optional) — Future business day for settlement
- **Company ID, Originator Name, Entry Description** (optional) — ACH metadata

**Link to Source**: [src/transfers.ts](../../src/transfers.ts) and [API Reference](https://developers.newline53.com/reference/transfers)

---

## Step 2: Implement ACH Prenote Workflow

Prenote transfers validate recipient accounts before sending full amounts.

**Prenote Step-by-Step**:

1. **Initiate prenote transfer** (Step 1)
   ```json
   {
     "external_uid": "ach_prenote_20260131_001",
     "customer_uid": "cust_456def",
     "initiator_type": "customer",
     "sending_account_uid": "syn_789ghi",
     "receiving_account": {
       "account_number": "123456789",
       "routing_number": "987654321",
       "account_holder_name": "New Recipient Inc"
     },
     "transfer_type": "ach",
     "amount_cents": 0,
     "ach": {
       "prenote_flag": true
     }
   }
   ```

2. **Monitor prenote settlement**
   - Use transaction monitoring from [Transaction Reporting Skill: Step 3](transaction-reporting-skill.md#step-3-monitor-transaction-status)
   - Poll every 1-2 minutes for ACH settlement status
   - Wait for `status: settled` (typically 1 business day)

3. **Once prenote settled, initiate full payment**
   ```json
   {
     "external_uid": "ach_payment_20260131_001",
     "customer_uid": "cust_456def",
     "initiator_type": "customer",
     "sending_account_uid": "syn_789ghi",
     "receiving_account": {
       "account_number": "123456789",
       "routing_number": "987654321",
       "account_holder_name": "New Recipient Inc"
     },
     "transfer_type": "ach",
     "amount_cents": 50000,
     "ach": {
       "effective_date": "2026-02-05"
     }
   }
   ```

4. **Monitor full payment settlement**
   - Poll until `status: settled` (typically 1-2 business days)

**Best Practice**:
- Always use prenote for first-time ACH recipients
- Prenote catches invalid account numbers before sending real funds
- Prenote is $0.00 amount — no charge to either party

---

## Step 3: Monitor ACH Transfer Status

Use [Transaction Reporting Skill: Monitor Transaction Status](transaction-reporting-skill.md#step-3-monitor-transaction-status) to track ACH settlement.

**Expected Timeline**:
- `queued` → `pending` → `settled` (typically 1-2 business days)

---

## Step 4: Handle ACH Transfer Errors

**Common Issues & ACH Transfer Errors**:

| Error Code | Error Name | Cause | Solution |
|------------|-----------|-------|----------|
| 400 | Invalid routing number for ACH | Routing number not valid for ACH | Verify routing is 9 digits and ACH-enabled |
| 400 | Account number format invalid | Account number wrong length/format | Verify account number with recipient (typically 8-17 digits) |
| 400 | Invalid effective date | Effective date is past or weekend | Use next business day or omit for immediate processing |
| 400 | Originator name too long | Field exceeds 16 character limit | Abbreviate company name to 16 chars max |
| 400 | Entry description invalid | Description exceeds 10 character limit | Shorten description to 10 chars (e.g., "SALARY", "PAYMENT") |
| 400 | Sending account not active | Account status not "active" | Verify account status or use different account |
| 31006 | unsupported_return | Returns not supported for this transaction | This indicates invalid ACH setup; verify with support |

**Recovery Steps**:
1. Verify recipient routing number is valid for ACH (not a wire-only routing)
2. Confirm account number format with recipient
3. Verify originator_name and entry_description lengths
4. Use valid future business day for effective_date
5. Retry with corrected information

---

# ACH Reversals (Newline-Originated Payments Only)

ACH reversals reverse payments that you (Newline customer) originated. Only Newline-originated settled ACH transactions can be reversed, and only within 5 business days of settlement.

## Step 1: Pre-Flight Transaction Eligibility Check

Before attempting to reverse an ACH transfer, verify it meets all eligibility criteria.

**Tool**: `newline-get-transaction`

**Required Parameters**:
- `uid` — Transaction UID of the ACH to reverse

**Eligibility Verification Checklist**:

1. **Transaction Type Check**
   - Must be `transaction_type: "ach"` (not wire or RTP)
   - ✓ Proceed to Step 2

2. **Transaction Status Check**
   - Must be `status: "settled"` (fully completed)
   - ✗ If queued or pending: Wait for settlement; cannot reverse pending transfers
   - ✗ If failed: Cannot reverse failed transfers

3. **Origination Check**
   ```json
   Check response metadata:
   {
     "originated_by": "customer"  // Must be "customer" (Newline customer-originated)
   }
   ```
   - ✓ If you originated the payment: Eligible for reversal
   - ✗ If externally originated: Must use "return" instead (Step 3 below)

4. **Time Window Check** — Must be within 5 business days of settlement
   - Get `settled_at` timestamp from transaction details
   - Calculate days elapsed: TODAY - settled_at
   - ✓ If ≤ 5 business days: Eligible for reversal
   - ✗ If > 5 business days: Expired; cannot reverse (error 31007)

5. **Reversal Existence Check**
   - Check if reversal already exists for this transaction
   - ✗ If reversal exists: Cannot create another (error 31004)

**Example Eligibility Response**:
```json
{
  "uid": "txn_001",
  "external_uid": "ach_payment_20260131_001",
  "transaction_type": "ach",
  "status": "settled",
  "settled_at": "2026-01-30T10:00:00Z",
  "originated_by": "customer",
  "created_at": "2026-01-29T08:00:00Z"
}
```
✓ All checks pass — Eligible for reversal

**Common Eligibility Issues**:

| Issue | Error Code | Message | Solution |
|-------|-----------|---------|----------|
| Transaction not settled | 31005 | cannot_initiate_return | Wait for transaction to settle |
| Externally originated | 31006 | unsupported_return | Use "return" workflow instead (Step 3 below) |
| Outside 5-day window | 31007 | expired_transaction | Cannot reverse; too much time elapsed |
| Reversal already exists | 31004 | return_already_exists | Check for existing reversal |
| Wrong transaction type | 31006 | unsupported_return | Only ACH can be reversed; wire/RTP cannot |

---

## Step 2: Initiate ACH Reversal

Once pre-flight checks pass, create the reversal.

**Tool**: `newline-create-return`

**Required Parameters**:
- `transaction_uid` — UID of the settled ACH to reverse
- `requestor_type` — Always `"customer"`

**Optional Parameters for ACH Reversals**:
- Leave `ach` object empty or omit it (reversal-specific)
- Leave other payment type objects empty

**Example Request**:
```json
{
  "transaction_uid": "txn_001",
  "requestor_type": "customer"
}
```

**Link to Source**: [src/returns.ts](../../src/returns.ts) and [API Reference](https://developers.newline53.com/reference/post_returns)

**Expected Response**:
```json
{
  "uid": "return_001",
  "transaction_uid": "txn_001",
  "transaction_type": "initiated_ach_reversal",
  "status": "queued",
  "amount_cents": 50000,
  "created_at": "2026-01-31T15:30:00Z"
}
```

**Important Notes**:
- The `transaction_type` in the response will be `"initiated_ach_reversal"` (not `"ach"`)
- This creates a new return transaction that reverses the original payment
- A new transaction UID is generated for the reversal itself

---

## Step 3: Monitor ACH Reversal Status

Use [Transaction Reporting Skill: Monitor Transaction Status](transaction-reporting-skill.md#step-3-monitor-transaction-status) to track the reversal.

**Tool**: `newline-get-return`

**Required Parameters**:
- `uid` — Reversal UID (from Step 2 response)

**Status Progression**:
```
created_at: 2026-01-31T15:30:00Z, status: queued
↓ (Processing)
status: pending
↓ (Settlement)
status: settled
```

**Expected Timeline**:
- Reversal processes similarly to normal ACH (1-2 business days)
- Original payment is reversed and funds returned to sending account

**Monitoring Tool**: `newline-list-returns` — Query all returns with pagination

---

## Step 4: Handle ACH Reversal Errors

**Common Issues & ACH Reversal Errors**:

| Error Code | Error Name | Cause | Solution |
|------------|-----------|-------|----------|
| 31005 | cannot_initiate_return | Transaction not settled yet | Wait for ACH to settle before reversing |
| 31007 | expired_transaction | Beyond 5-day reversal window | Contact recipient for alternative resolution |
| 31004 | return_already_exists | Reversal already created for this transaction | Check existing reversal status instead of creating new one |
| 31006 | unsupported_return | Not a Newline-originated ACH | Use "return" for externally-originated payments instead |
| 31014 | ineligible_transaction | Transaction ineligible for reversal | Verify transaction meets all criteria from Step 1 |

**Recovery Steps**:
1. Re-run pre-flight eligibility checks (Step 1)
2. Verify transaction is fully settled and within 5 business days
3. Verify transaction was originated by your company (not received from external party)
4. Verify no existing reversal already created
5. If error persists, contact support with transaction UID

---

# Wire and ACH Returns (Received Payments Only)

Returns reverse payments you **received** (originated by external parties). Wire returns are available immediately for settled received wires. ACH returns are available for settled received ACH within the SEC-code-specific time window.

## Step 1: Pre-Flight Transaction Eligibility Check

Before attempting to return a wire or ACH, verify it meets all eligibility criteria.

**Tool**: `newline-get-transaction`

**Required Parameters**:
- `uid` — Transaction UID of the wire or ACH to return

**Eligibility Verification Checklist**:

### For Wire Returns:

1. **Transaction Type Check**
   - Must be `transaction_type: "wire"` (received wire)
   - ✓ Proceed to next check

2. **Transaction Status Check**
   - Must be `status: "settled"` (fully completed)
   - ✗ If queued or pending: Wait for settlement

3. **Origination Check**
   - Must be externally originated (received wire)
   - Check metadata: `originated_by: "external"` or similar
   - ✗ If you originated (Newline-originated): Use reversal instead (not return)

4. **UETR Code Check**
   - Wires with UETR (Unique End-to-End Transaction Reference) numbers may not support returns
   - ✗ If error 31008 network_code_unsupported: Wire has UETR, cannot return

**Example Wire Eligibility Response**:
```json
{
  "uid": "txn_002",
  "transaction_type": "wire",
  "status": "settled",
  "settled_at": "2026-01-31T14:00:00Z",
  "amount_cents": 100000
}
```
✓ All checks pass — Eligible for wire return

### For ACH Returns:

1. **Transaction Type Check**
   - Must be `transaction_type: "ach"` (received ACH)
   - ✓ Proceed to next check

2. **Transaction Status Check**
   - Must be `status: "settled"` (fully completed)
   - ✗ If queued or pending: Wait for settlement

3. **Origination Check**
   - Must be externally originated (received ACH)
   - ✗ If you originated (Newline-originated): Use reversal instead (not return)

4. **Time Window Check** — Depends on SEC code of the original ACH
   - ACH return windows vary by SEC code (see SEC Code Return Windows table below)
   - Get the original ACH's SEC code from transaction metadata
   - Calculate: TODAY - settled_at ≤ SEC code window days
   - ✓ If within window: Eligible for return
   - ✗ If expired: Cannot return (error 31007)

**Example ACH Eligibility Response**:
```json
{
  "uid": "txn_003",
  "transaction_type": "ach",
  "status": "settled",
  "settled_at": "2026-01-30T09:00:00Z",
  "amount_cents": 50000,
  "metadata": {
    "sec_code": "PPD"
  }
}
```
✓ For PPD (5-day window): If today is within 5 days of settled_at, eligible for return

**Common Eligibility Issues**:

| Issue | Error Code | Message | Solution |
|-------|-----------|---------|----------|
| Transaction not settled | 31005 | cannot_initiate_return | Wait for transaction to settle |
| Wire has UETR | 31008 | network_code_unsupported | Wire cannot be returned; UETR present |
| Return already exists | 31004 | return_already_exists | Check existing return status instead |
| ACH return window expired | 31007 | expired_transaction | Too much time elapsed; contact recipient |
| Not externally originated | 31006 | unsupported_return | Reversal (not return) for your own payments |

---

## Step 2: Initiate Wire or ACH Return

Once pre-flight checks pass, create the return.

**Tool**: `newline-create-return`

**Required Parameters**:
- `transaction_uid` — UID of the settled wire or ACH to return
- `requestor_type` — Always `"customer"`

**Optional Parameters**:
- `wire` or `ach` — Omit or leave empty for basic return
- `return_reason` — Free-text reason for return (optional but recommended)

**Example Request - Wire Return**:
```json
{
  "transaction_uid": "txn_002",
  "requestor_type": "customer",
  "return_reason": "Customer requested reversal"
}
```

**Example Request - ACH Return**:
```json
{
  "transaction_uid": "txn_003",
  "requestor_type": "customer",
  "return_reason": "Erroneous payment - duplicate transaction"
}
```

**Link to Source**: [src/returns.ts](../../src/returns.ts) and [API Reference](https://developers.newline53.com/reference/post_returns)

**Expected Response**:
```json
{
  "uid": "return_002",
  "transaction_uid": "txn_002",
  "transaction_type": "initiated_wire_return",
  "status": "queued",
  "amount_cents": 100000,
  "return_reason": "Customer requested reversal",
  "created_at": "2026-01-31T15:45:00Z"
}
```

**Important Notes**:
- The `transaction_type` will be `"initiated_wire_return"` or `"initiated_ach_return"`
- A new return transaction is created with its own UID
- Return reason is optional but helps with audit trail

---

## Step 3: Monitor Wire or ACH Return Status

Use [Transaction Reporting Skill: Monitor Transaction Status](transaction-reporting-skill.md#step-3-monitor-transaction-status) to track the return.

**Tool**: `newline-get-return`

**Required Parameters**:
- `uid` — Return UID (from Step 2 response)

**Status Progression**:
```
created_at: 2026-01-31T15:45:00Z, status: queued
↓ (Processing)
status: pending
↓ (Settlement)
status: settled (or failed)
```

**Expected Timeline**:
- Wire returns: Typically settle same-day or next business day
- ACH returns: Typically settle 1-2 business days

**Return Status Values**:
- `queued` — Return created and queued for processing
- `pending` — Return being processed by clearing house
- `settled` — Return completed; funds returned to your account
- `failed` — Return failed (e.g., outside time window); see error details

**Monitoring Tool**: `newline-list-returns` — Query all returns with pagination, optional filters by status/date

---

## Step 4: Handle Wire or ACH Return Errors

**Common Issues & Return Errors**:

| Error Code | Error Name | Cause | Solution |
|------------|-----------|-------|----------|
| 31005 | cannot_initiate_return | Transaction not settled yet | Wait for settlement before returning |
| 31007 | expired_transaction | Return window elapsed (ACH) or too much time (Wire) | Contact recipient or issuer for alternative resolution |
| 31004 | return_already_exists | Return already created for this transaction | Check existing return status instead |
| 31008 | network_code_unsupported | Wire has UETR; cannot return | Wire cannot be returned; contact originating bank |
| 31006 | unsupported_return | Transaction type not returnable | Only wires and ACH supported; RTP cannot be returned |
| 31009 | extra_payment_information | Unnecessary fields provided in request | Omit `wire` or `ach` objects; provide only required fields |
| 31010 | insufficient_information_for_return | Original transaction data missing | Contact support; transaction may have incomplete metadata |
| 31014 | ineligible_transaction | Transaction ineligible for return | Verify transaction meets all criteria from Step 1 |

**Recovery Steps**:
1. Re-run pre-flight eligibility checks (Step 1)
2. Verify transaction is fully settled
3. For ACH: Verify within SEC-code time window (see table below)
4. For Wire: Verify wire doesn't have UETR number
5. Verify no existing return already created
6. Simplify request (omit optional parameters if error indicates extra fields)
7. If error persists, contact support with transaction UID

---

## SEC Code Return Windows Reference

ACH return windows are determined by the SEC (Standard Entry Class) code of the original transaction. Return the ACH only if within the specified days of settlement.

| SEC Code | Description | Return Window | Example Use Case |
|----------|-------------|----------------|------------------|
| PPD | Prearranged Payment & Deposit | 5 calendar days | Consumer bill payments, payroll |
| CCD | Corporate Credit or Debit | 2 calendar days | B2B payments, business-to-business |
| CTX | Corporate Trade Exchange | 2 calendar days | Complex B2B transactions |
| IAT | International ACH Transaction | Varies (typically 5-10 days) | Cross-border ACH transfers |
| ACK | Acknowledgment Entry | Same as original | Echo/verification transactions |
| ARC | Accounts Receivable Entry | Return not supported | Lockbox/check conversion |
| BOC | Back Office Conversion | Return not supported | Point-of-sale conversion |
| POP | Point of Purchase | Return not supported | Retail point-of-sale |
| TEL | Telephone-Initiated Entry | Return not supported | Phone-initiated payments |
| WEB | Website-Initiated Entry | 5 calendar days | Online bill pay |

**Important Notes**:
- Return windows are **calendar days** (including weekends/holidays), not business days
- After the window expires, the transaction cannot be returned
- If unclear, reference external API documentation: https://developers.newline53.com/docs/returns-1
- For unusual SEC codes, contact support for return eligibility

---

# Combined Transfer Tracking

Combined transfers are logical groupings of multiple individual transfers, created by combining the synthetic accounts and transfers endpoints.

## Step 1: Understand Combined Transfers

**What are Combined Transfers?**
- Grouping mechanism for tracking multiple transfers as a cohesive batch
- Aggregate status tracking across multiple transfer UIDs
- Useful for bulk payment operations and reporting

**How to Create Combined Transfer Grouping**:
1. Create individual transfers via [Making Payments Skill](making-payments-skill.md)
2. Group transfer UIDs together in your system
3. Track aggregate amount (sum of all transfer amounts)
4. Track overall status (all settled = group settled, one failed = group partially failed)

## Step 2: Query Combined Transfers

**Tool**: `newline-list-combined-transfers`

**Optional Parameters**:
- `limit` — Number of combined transfers to return (default: 10)
- `offset` — Pagination offset (default: 0)

**Example Response Fields**:
```json
{
  "combined_transfers": [
    {
      "uid": "combined_001",
      "transfer_uids": ["transfer_001", "transfer_002", "transfer_003"],
      "status": "settled",
      "total_amount_cents": 150000,
      "currency": "USD",
      "created_at": "2026-01-31T15:00:00Z",
      "completed_at": "2026-01-31T15:30:00Z"
    }
  ],
  "total_count": 5
}
```

**Link to Source**: [src/combined_transfers.ts](../../src/combined_transfers.ts)

## Step 3: Get Combined Transfer Details

**Tool**: `newline-get-combined-transfer`

**Required Parameters**:
- `uid` — Combined transfer group UID

**Example Response**:
```json
{
  "uid": "combined_001",
  "transfer_uids": ["transfer_001", "transfer_002", "transfer_003"],
  "status": "settled",
  "total_amount_cents": 150000,
  "currency": "USD",
  "created_at": "2026-01-31T15:00:00Z",
  "completed_at": "2026-01-31T15:30:00Z"
}
```

**Link to Source**: [src/combined_transfers.ts](../../src/combined_transfers.ts)

**Important Notes**:
- Returns are NOT included in combined transfer grouping
- Only individual transfers are grouped
- Use combined transfers for aggregate reporting and status tracking

---

# Sandbox Testing: Simulating Failed Returns

Test error scenarios and recovery workflows in Sandbox by simulating failed returns.

## Step 1: Create Return with Failure Trigger

To simulate a failed return in Sandbox, include the value `"fail"` in a free-text field of your return request.

**Tool**: `newline-create-return`

**Request with Failure Trigger**:
```json
{
  "transaction_uid": "txn_002",
  "requestor_type": "customer",
  "return_reason": "fail"  // Include "fail" string to trigger failure
}
```

**Alternative Failure Trigger**:
```json
{
  "transaction_uid": "txn_003",
  "requestor_type": "customer",
  "memo": "fail"  // Can use any free-text field
}
```

## Step 2: Monitor Failed Return Status

**Tool**: `newline-get-return`

**Expected Response** for failed return:
```json
{
  "uid": "return_003",
  "transaction_uid": "txn_002",
  "transaction_type": "initiated_wire_return",
  "status": "failed",  // Status shows "failed"
  "amount_cents": 100000,
  "created_at": "2026-01-31T16:00:00Z"
}
```

**Use Case**: Test your error handling and recovery workflows in Sandbox before Production.

---

# Error Handling by Payment Type Summary

## Wire Transfer Errors
- Focus on: Account number/routing validation, intermediary bank details
- Recovery: Verify recipient details, retry with valid routing
- Cannot reverse; use return for received wires

## ACH Transfer Errors
- Focus on: Routing number (ACH-specific), effective date, field lengths
- Recovery: Validate ACH routing, abbreviate descriptions, use business day dates
- Can reverse (Newline-originated) within 5 days or return (externally-originated) per SEC window

## ACH Reversal Errors
- Focus on: 5-day window compliance, settlement status, origination verification
- Recovery: Verify transaction settled, within 5-day window, check pre-flight eligibility
- Time-sensitive: No extension available

## Wire/ACH Return Errors
- Focus on: Settlement status, SEC code windows (ACH), UETR presence (Wire)
- Recovery: Verify settled, within time window, check pre-flight eligibility
- Time-sensitive: Windows vary by SEC code or are permanent once window expires

## RTP (Instant Payment) Errors
- Focus on: Cannot return or reverse RTP transfers
- Recovery: Verify recipient before sending; contact recipient for alternative resolution if error
- No reversal/return available

---

# Workflow Summary: Return Processing

For **Received Wire**:
1. **Authenticate**
2. **Pre-flight check** (Step 1 under Wire/ACH Returns) — Verify settled, not UETR
3. **Initiate return** (Step 2) — `newline-create-return`
4. **Monitor return** (Step 3) — Check status progression

For **Received ACH**:
1. **Authenticate**
2. **Pre-flight check** (Step 1 under Wire/ACH Returns) — Verify settled, within SEC window
3. **Initiate return** (Step 2) — `newline-create-return`
4. **Monitor return** (Step 3) — Check status progression

For **Newline-Originated ACH**:
1. **Authenticate**
2. **Pre-flight check** (Step 1 under ACH Reversals) — Verify settled, within 5 days
3. **Initiate reversal** (Step 2) — `newline-create-return` (reversal type)
4. **Monitor reversal** (Step 3) — Check status progression

For **RTP**:
- No returns or reversals available
- Contact recipient directly for alternative resolution

---

## Next Steps

For complete payment workflows:
- [Making Payments Skill](making-payments-skill.md) — Initiate transfers
- [Transaction Reporting Skill](transaction-reporting-skill.md) — Monitor all transfer and return status
- [Managing Users and Accounts Skill](managing-users-and-accounts-skill.md) — Manage customers and accounts
