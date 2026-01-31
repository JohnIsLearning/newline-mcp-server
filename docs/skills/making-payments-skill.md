# Making Payments Skill Guide

**Prerequisite**: [Managing Users and Accounts Skill](managing-users-and-accounts-skill.md) must be completed first

**Related Skills**: [Payment Operations Skill](payment-operations-skill.md) for advanced workflows (returns, reversals, combined transfers)

This guide provides step-by-step instructions for agents to initiate transfers in Newline's payment system. Three transfer types are available: Wire transfers, ACH transfers, and Instant Payments (RTP). Each has different requirements and processing characteristics.

## Overview: Transfer Types

| Aspect | Wire Transfer | ACH Transfer | Instant Payment (RTP) |
|--------|---------------|--------------|----------------------|
| **Processing Speed** | Same-day or next business day | 1-2 business days | Real-time (seconds) |
| **Amount Limits** | High | Varies by program | Varies by program |
| **Use Case** | Urgent transfers, large amounts | Routine business payments | Urgent, lower amounts |
| **Returns Supported** | Yes | Yes | No |
| **Reversals Supported** | No | Yes (Newline-originated only) | No |
| **Setup Complexity** | Requires intermediary bank details (optional) | Requires ACH setup (company ID, originator name) | Requires transmitter address |

## Prerequisites for All Transfers

Before initiating any transfer, you must have:

1. **Completed** [Managing Users and Accounts Skill](managing-users-and-accounts-skill.md)
2. **Source Synthetic Account** — UID of the account sending funds (your customer's account)
3. **Destination Account** — Either:
   - **Destination Synthetic Account UID** — For established recipients (pre-created external synthetic account)
   - **External Account Details** — For one-time recipients (account number, routing number, account holder name)
4. **Transfer amount** — USD amount to transfer (in cents)
5. **External reference** — External identifier for your system (optional but recommended)

## Source and Destination Account Fields

All transfers require two accounts:

| Field | Exact API Name | Purpose | Where It Comes From |
|-------|----------------|---------|---------------------|
| **Source** | `source_synthetic_account_uid` | Account funds are debited from (your customer's account) | Create via [Managing Users and Accounts Skill - General Account](managing-users-and-accounts-skill.md#creating-a-general-synthetic-account-your-customers-account) |
| **Destination Option 1** | `destination_synthetic_account_uid` | Account funds are credited to (pre-created external account for repeated recipients) | Create via [Managing Users and Accounts Skill - External Account](managing-users-and-accounts-skill.md#creating-an-external-synthetic-account-recipients-account) or [Combined Transfers](#combined-transfers-create-destination-and-transfer-in-one-call) |
| **Destination Option 2** | `destination_account` (object) | Account details for one-time recipients | Provide directly: `account_number`, `routing_number`, `account_holder_name` |

**Key Rule**: Always use `source_synthetic_account_uid`. For destination, choose between `destination_synthetic_account_uid` OR `destination_account` object — never both.

**IMPORTANT**: This guide uses the exact API field names. Older examples or different documentation may use simplified terms like `sending_account_uid` or `receiving_account` — these are simplified aliases for the exact API fields listed above.

## Step 1: Verify Account Details

Retrieve and confirm the sending account details before initiating a transfer.

**Tool**: `newline-get-synthetic-account`

**Required Parameters**:
- `uid` — Sending synthetic account UID

**Key Fields to Verify**:
- `status` — Must be "active" for transfers
- `account_number` — Required for receiving wire/ACH details
- `routing_number` — Required for wire/ACH setup
- `transfer_methods` — Confirm the transfer type you want is enabled

**Example**:
```json
{
  "uid": "syn_789ghi",
  "status": "active",
  "account_number": "9876543210",
  "routing_number": "123456789",
  "transfer_methods": {
    "wire": { "enabled": true },
    "ach": { "enabled": true },
    "rtp": { "enabled": true }
  }
}
```

**Common Issues**:
- Account status not "active" — Contact support to activate account
- Transfer method not enabled — Use a different account or contact support

---

# Wire Transfers

Wire transfers provide same-day or next-day settlement for domestic and international payments.

## Step 2a: Initiate a Wire Transfer

Create a wire transfer request.

**Tool**: `newline-create-transfer`

**Required Parameters**:
- `external_uid` — Your system's transfer reference ID (string, max 255 characters)
- `initiating_customer_uid` — Customer initiating the transfer (UID of customer owning the source account)
- `initiator_type` — Always `"customer"` for wire transfers
- `source_synthetic_account_uid` — Source account UID (your customer's account; funds come FROM here)
- **EITHER** `destination_synthetic_account_uid` **OR** `destination_account`:
  - `destination_synthetic_account_uid` — UID of pre-created external synthetic account (if using pre-created account)
  - `destination_account` — Object with recipient details (if providing account info directly):
    - `account_number` — Recipient's account number
    - `routing_number` — Recipient's routing number
    - `account_holder_name` — Recipient's name
- `transfer_type` — Set to `"wire"`
- `usd_transfer_amount` — Amount in cents as string (e.g., "50000" for $500.00)

**Optional Parameters**:
- `wire` — Object with wire-specific details:
  - `wire_code` — Wire code for routing (string)
  - `intermediary_bank` — Object with intermediary bank details (if needed):
    - `name` — Bank name (string)
    - `routing_number` — Bank routing number (string)
    - `address_line_1` — Address line 1 (string)
    - `address_line_2` — Address line 2 (string)
    - `address_line_3` — Address line 3 (string)
  - `wire_transmitter` — Object with transmitter information:
    - `name` — Transmitter name (string)
    - `identifier` — Transmitter identifier (string)
    - `street_address` — Street address (string)
    - `city` — City (string)
    - `state` — State code (string)
    - `postal_code` — Postal code (string)

**Example Request - Wire to One-Time Recipient (Using Account Details)**:
```json
{
  "external_uid": "wire_20260131_001",
  "initiating_customer_uid": "cust_456def",
  "initiator_type": "customer",
  "source_synthetic_account_uid": "syn_789ghi",
  "destination_account": {
    "account_number": "123456789",
    "routing_number": "987654321",
    "account_holder_name": "Acme Corp"
  },
  "transfer_type": "wire",
  "usd_transfer_amount": "50000"
}
```

**Example Request - Wire to Pre-Created External Account (Using Account UID)**:
```json
{
  "external_uid": "wire_to_acme_established",
  "initiating_customer_uid": "cust_456def",
  "initiator_type": "customer",
  "source_synthetic_account_uid": "syn_789ghi",
  "destination_synthetic_account_uid": "syn_external_acme",
  "transfer_type": "wire",
  "usd_transfer_amount": "50000"
}
```

**Example Request - Wire with Intermediary Bank (One-Time Recipient)**:
```json
{
  "external_uid": "wire_20260131_002",
  "initiating_customer_uid": "cust_456def",
  "initiator_type": "customer",
  "source_synthetic_account_uid": "syn_789ghi",
  "destination_account": {
    "account_number": "123456789",
    "routing_number": "987654321",
    "account_holder_name": "International Corp"
  },
  "transfer_type": "wire",
  "usd_transfer_amount": "100000",
  "wire": {
    "wire_code": "INTL",
    "intermediary_bank": {
      "name": "Correspondent Bank",
      "routing_number": "111222333",
      "address_line_1": "456 Banking Plaza",
      "address_line_2": "New York, NY 10001",
      "address_line_3": "USA"
    },
    "wire_transmitter": {
      "name": "John Smith",
      "identifier": "SMITH123",
      "street_address": "123 Main St",
      "city": "Boston",
      "state": "MA",
      "postal_code": "02101"
    }
  }
}
```

**Link to Source**: [src/transfers.ts](../../src/transfers.ts)

**Expected Response Fields**:
```json
{
  "uid": "transfer_001",
  "external_uid": "wire_20260131_001",
  "status": "queued",
  "transfer_type": "wire",
  "amount_cents": 50000,
  "sending_account_uid": "syn_789ghi",
  "receiving_account": {...},
  "created_at": "2026-01-31T15:00:00Z"
}
```

**Common Issues & Wire Transfer Errors**:

| Error Code | Error Name | Solution |
|------------|-----------|----------|
| 31009 | extra_payment_information | Remove unnecessary payment info fields and retry |
| 31010 | insufficient_information_for_return | Ensure all required account fields are provided |
| 31008 | network_code_unsupported | Wire code may not be supported; verify with support |
| 400 | Invalid account number format | Verify account and routing numbers are correct |
| 400 | Sending account not found | Verify sending_account_uid exists and is active |
| 400 | Unsupported transfer type | Confirm transfer type is "wire" and account supports wire |

---

# ACH Transfers

ACH transfers provide cost-effective batch processing for routine business payments.

## Step 2b: Initiate an ACH Transfer

Create an ACH transfer request.

**Tool**: `newline-create-transfer`

**Required Parameters**:
- `external_uid` — Your system's transfer reference ID
- `initiating_customer_uid` — Customer initiating the transfer (UID of customer owning the source account)
- `initiator_type` — Always `"customer"` for ACH transfers
- `source_synthetic_account_uid` — Source account UID (your customer's account; funds come FROM here)
- **EITHER** `destination_synthetic_account_uid` **OR** `destination_account`:
  - `destination_synthetic_account_uid` — UID of pre-created external synthetic account (if using pre-created account)
  - `destination_account` — Object with recipient details (if providing account info directly):
    - `account_number` — Recipient's account number
    - `routing_number` — Recipient's routing number (must be valid ACH routing)
    - `account_holder_name` — Recipient's name
- `transfer_type` — Set to `"ach"`
- `usd_transfer_amount` — Amount in cents as string

**Optional Parameters**:
- `ach` — Object with ACH-specific details:
  - `prenote` — Boolean; if `true`, sends zero-amount prenote before actual transfer (recommended for first-time recipients)
  - `effective_entry_date` — Desired settlement date (string, YYYY-MM-DD format; must be future business day)
  - `company_id` — ACH originating company ID (string; typically 9-10 digits)
  - `originator_name` — Company name on ACH file (string, max 16 characters)
  - `entry_description` — Description of ACH entry (string, max 10 characters)
  - `id_number` — Identification number for the transaction (string)

**Example Request - ACH to One-Time Recipient (Using Account Details)**:
```json
{
  "external_uid": "ach_20260131_001",
  "initiating_customer_uid": "cust_456def",
  "initiator_type": "customer",
  "source_synthetic_account_uid": "syn_789ghi",
  "destination_account": {
    "account_number": "123456789",
    "routing_number": "987654321",
    "account_holder_name": "Vendor LLC"
  },
  "transfer_type": "ach",
  "usd_transfer_amount": "25000"
}
```

**Example Request - ACH to Pre-Created External Account with Prenote**:
```json
{
  "external_uid": "ach_20260131_002",
  "initiating_customer_uid": "cust_456def",
  "initiator_type": "customer",
  "source_synthetic_account_uid": "syn_789ghi",
  "destination_synthetic_account_uid": "syn_external_vendor",
  "transfer_type": "ach",
  "usd_transfer_amount": "50000",
  "ach": {
    "prenote": true,
    "effective_entry_date": "2026-02-05",
    "company_id": "0000000001",
    "originator_name": "ACME CORP",
    "entry_description": "SALARY",
    "id_number": "12345678"
  }
}
```

**Link to Source**: [src/transfers.ts](../../src/transfers.ts)

**Expected Response Fields**:
```json
{
  "uid": "transfer_002",
  "external_uid": "ach_20260131_002",
  "status": "queued",
  "transfer_type": "ach",
  "amount_cents": 50000,
  "sending_account_uid": "syn_789ghi",
  "receiving_account": {...},
  "created_at": "2026-01-31T15:05:00Z"
}
```

**Common Issues & ACH Transfer Errors**:

| Error Code | Error Name | Solution |
|------------|-----------|----------|
| 400 | Invalid routing number for ACH | Use a valid ACH routing number (9 digits) |
| 400 | Account number format invalid | Verify account number length and format |
| 400 | Effective date in the past | Use a future business day for effective_date |
| 400 | Originator name too long | Keep originator_name to 16 characters max |
| 400 | Sending account not active | Verify account status is "active" |
| 31006 | unsupported_return | ACH returns are supported; error may indicate invalid recipient |

**Prenote Best Practice**:
- Set `prenote_flag: true` for first-time recipients to validate account before sending full amount
- Prenote transactions send $0.00 amount as verification
- Wait for prenote to settle before sending actual payment transfer

---

# Instant Payments (RTP)

Instant Payments provide real-time funds transfer for immediate settlement.

## Step 2c: Initiate an Instant Payment

Create an Instant Payment (RTP) transfer request.

**Tool**: `newline-create-transfer`

**Required Parameters**:
- `external_uid` — Your system's transfer reference ID
- `initiating_customer_uid` — Customer initiating the transfer (UID of customer owning the source account)
- `initiator_type` — Always `"customer"` for RTP transfers
- `source_synthetic_account_uid` — Source account UID (your customer's account; funds come FROM here)
- **EITHER** `destination_synthetic_account_uid` **OR** `destination_account`:
  - `destination_synthetic_account_uid` — UID of pre-created external synthetic account (if using pre-created account)
  - `destination_account` — Object with recipient details (if providing account info directly):
    - `account_number` — Recipient's account number
    - `routing_number` — Recipient's routing number
    - `account_holder_name` — Recipient's name
- `transfer_type` — Set to `"rtp"`
- `usd_transfer_amount` — Amount in cents as string

**Optional Parameters**:
- `instant_payment` — Object with RTP-specific details:
  - `instant_payment_transmitter` — Object with sender's address:
    - `name` — Sender name
    - `street_number` — Street number
    - `street1` — Street address (string)
    - `city` — City (string)
    - `state` — State code (string)
    - `postal_code` — Postal code (string)
  - `memo` — Payment memo or description (string)

**Example Request - RTP to One-Time Recipient (Using Account Details)**:
```json
{
  "external_uid": "rtp_20260131_001",
  "initiating_customer_uid": "cust_456def",
  "initiator_type": "customer",
  "source_synthetic_account_uid": "syn_789ghi",
  "destination_account": {
    "account_number": "123456789",
    "routing_number": "987654321",
    "account_holder_name": "Quick Payment Corp"
  },
  "transfer_type": "rtp",
  "usd_transfer_amount": "15000"
}
```

**Example Request - RTP to Pre-Created External Account**:
```json
{
  "external_uid": "rtp_to_quickpay_established",
  "initiating_customer_uid": "cust_456def",
  "initiator_type": "customer",
  "source_synthetic_account_uid": "syn_789ghi",
  "destination_synthetic_account_uid": "syn_external_quickpay",
  "transfer_type": "rtp",
  "usd_transfer_amount": "15000",
  "instant_payment": {
    "instant_payment_transmitter": {
      "name": "Jane Smith",
      "street_number": 123,
      "street1": "Main St",
      "city": "Boston",
      "state": "MA",
      "postal_code": "02101"
    },
    "memo": "Invoice #12345 Payment"
  }
}
```

**Link to Source**: [src/transfers.ts](../../src/transfers.ts)

**Expected Response Fields**:
```json
{
  "uid": "transfer_003",
  "external_uid": "rtp_20260131_002",
  "status": "queued",
  "transfer_type": "rtp",
  "amount_cents": 15000,
  "sending_account_uid": "syn_789ghi",
  "receiving_account": {...},
  "created_at": "2026-01-31T15:10:00Z"
}
```

**Important Notes**:
- RTP transfers settle in real-time (typically within seconds)
- **Returns are NOT supported for RTP transfers** — Verify recipient details carefully before submission
- Amount limits may apply depending on your program settings

**Common Issues & Instant Payment Errors**:

| Error Code | Error Name | Solution |
|------------|-----------|----------|
| 31006 | unsupported_return | RTP does not support returns; this error may indicate invalid setup |
| 400 | Invalid routing number | Verify routing number is valid and supports RTP |
| 400 | Amount exceeds limit | RTP may have per-transaction limits; reduce amount or contact support |
| 400 | Memo field too long | Keep memo field brief and under character limit |

---

## Step 3: Monitor Transfer Status

After initiating a transfer, query its status to verify processing.

**Tool**: `newline-get-transfer`

**Required Parameters**:
- `uid` — Transfer UID (from Step 2 response)

**Key Status Values**:
- `queued` — Transfer received and queued for processing
- `pending` — Transfer is being processed
- `settled` — Transfer completed successfully
- `failed` — Transfer failed; review error details

**Example Response**:
```json
{
  "uid": "transfer_001",
  "external_uid": "wire_20260131_001",
  "status": "settled",
  "transfer_type": "wire",
  "amount_cents": 50000,
  "sending_account_uid": "syn_789ghi",
  "receiving_account": {...},
  "created_at": "2026-01-31T15:00:00Z",
  "settled_at": "2026-01-31T15:30:00Z"
}
```

**Link to Source**: [src/transfers.ts](../../src/transfers.ts)

**Polling Recommendations**:
- Poll every 5-10 seconds for RTP (settles within seconds)
- Poll every 30-60 seconds for wire (same-day or next-day)
- Poll every 1-2 minutes for ACH (1-2 business days)

---

## Step 4: List Recent Transfers

Query all transfers for a customer or date range.

**Tool**: `newline-list-transfers`

**Optional Parameters**:
- `limit` — Number of results to return (default: 10)
- `offset` — Pagination offset (default: 0)

**Example Response**:
```json
{
  "transfers": [
    {
      "uid": "transfer_001",
      "external_uid": "wire_20260131_001",
      "status": "settled",
      "transfer_type": "wire",
      "amount_cents": 50000,
      "created_at": "2026-01-31T15:00:00Z"
    }
  ],
  "total_count": 15
}
```

**Link to Source**: [src/transfers.ts](../../src/transfers.ts)

---

# Alternative Workflows: External Synthetic Accounts and Combined Transfers

## Using External Synthetic Accounts for Repeated Recipients

If you're sending funds to the same external recipient multiple times, you can improve efficiency by creating an **external synthetic account** first, then referencing it in future transfers instead of re-entering recipient details each time.

### When to Use

- **Recurring recipient** — You'll send funds to the same external party multiple times
- **Efficiency** — Avoids re-entering account number, routing number, and account holder name for each transfer
- **Optional** — One-time recipients don't require an external synthetic account

### Step 1: Create External Synthetic Account

See [Managing Users and Accounts Skill: Step 6](managing-users-and-accounts-skill.md#creating-an-external-synthetic-account-recipients-account) for detailed instructions on creating an external synthetic account.

**Key points**:
- Use external account type (e.g., `type_external_001`)
- Create a customer record for the external counterparty or use a generic counterparty customer
- Note the returned account UID

**Example**:
```json
{
  "uid": "syn_external_recipient_001",
  "customer_uid": "ext_acme_corp_001",
  "synthetic_account_type_uid": "type_external_001",
  "external_uid": "recipient_acme_corp",
  "status": "active",
  "created_at": "2026-01-31T14:30:00Z"
}
```

### Step 2: Reference the Account in Future Transfers

Once the external synthetic account is created, reference it by UID in the `destination_synthetic_account_uid` field instead of providing account details:

**Example Wire Transfer to Established Recipient**:
```json
{
  "external_uid": "wire_to_acme_20260131_001",
  "initiating_customer_uid": "cust_456def",
  "initiator_type": "customer",
  "source_synthetic_account_uid": "syn_789ghi",
  "destination_synthetic_account_uid": "syn_external_recipient_001",
  "transfer_type": "wire",
  "usd_transfer_amount": "50000"
}
}
```

**Benefits**:
- Simplified transfer initiation (no need to repeat recipient details)
- Reduced error risk (account details are pre-validated)
- Faster transfer processing

---

## Combined Transfers: Create Destination Account and Transfer in One Call

The **Combined Transfers** endpoint allows you to create an external synthetic account AND initiate a transfer in a **single API call**. This eliminates the need to first create an external account, then create a separate transfer.

### When to Use

- **New recipients** — First-time payment to an external counterparty
- **Efficiency** — Skip the intermediate step of creating an external account separately
- **Simplicity** — Single operation instead of two sequential API calls

### Important Notes

- Combined Transfers creates the destination account and transfer together
- Use this when you don't need to store the external account for future use
- If you'll send to the same recipient again, consider creating the external account first and reusing it

### Step 1: Prepare Combined Transfer Request

**Tool**: `newline-create-combined-transfer` (or `newline-create-transfer` with combined request structure)

**Request Structure**:
```json
{
  "synthetic_account": {
    // External synthetic account creation details
  },
  "transfer": {
    // Transfer initiation details
  }
}
```

### Step 2: Wire Transfer via Combined Endpoint

Create external account and wire transfer in one call:

**Example Request**:
```json
{
  "synthetic_account": {
    "external_uid": "recipient_vendor_xyz",
    "synthetic_account_type_uid": "type_external_001",
    "customer_uid": "ext_vendor_xyz_001"
  },
  "transfer": {
    "external_uid": "combined_wire_20260131_001",
    "initiating_customer_uid": "cust_456def",
    "initiator_type": "customer",
    "source_synthetic_account_uid": "syn_789ghi",
    "destination_account": {
      "account_number": "123456789",
      "routing_number": "987654321",
      "account_holder_name": "Vendor XYZ"
    },
    "transfer_type": "wire",
    "usd_transfer_amount": "75000"
  }
}
```

**Expected Response**:
```json
{
  "uid": "combined_001",
  "synthetic_account_id": "syn_external_vendor_xyz",
  "synthetic_account_external_uid": "recipient_vendor_xyz",
  "transfer_id": "transfer_001",
  "transfer_external_uid": "combined_wire_20260131_001",
  "status": "queued",
  "total_amount_cents": 75000,
  "created_at": "2026-01-31T16:00:00Z"
}
```

**Key Benefits**:
- Single API call creates both account and transfer
- Faster onboarding for new recipients
- Automatic external account creation

**Link to Reference**: [Combined Transfers Reference](https://developers.newline53.com/reference/combined-transfers-1) and [QuickStart Guide: Completing a Transfer](https://developers.newline53.com/docs/quickstart-guide-completing-a-transfer)

### Step 3: Monitor Combined Transfer Status

**Tool**: `newline-get-combined-transfer`

**Required Parameters**:
- `uid` — Combined transfer UID (from Step 2 response)

**Status Progression**:
```
created_at: 2026-01-31T16:00:00Z, status: queued
↓ (Processing)
status: pending
↓ (Settlement)
status: settled (or failed)
```

**Expected Timeline**:
- Wire: Same-day or next business day
- ACH: 1-2 business days
- RTP: Real-time (seconds)

**Monitoring Tool**: `newline-list-combined-transfers` — Query all combined transfers with pagination

---

## Workflow Summary: Basic Payment

1. **Authenticate** (if not already done)
2. **Get sending account details** (Step 1) — Verify account is active and supports transfer type
3. **Choose transfer type** — Wire, ACH, or RTP based on use case
4. **Initiate transfer** (Step 2a, 2b, or 2c) — Provide recipient details and amount
5. **Monitor status** (Step 3) — Poll transfer status until settled or failed
6. **If failed** — Review error details and retry with corrected information

## Workflow Summary: Payment to Repeated Recipient (Using External Account)

1. **Authenticate**
2. **Create external synthetic account** (See [Managing Users and Accounts](managing-users-and-accounts-skill.md#creating-an-external-synthetic-account-recipients-account)) — One-time setup for the recipient
3. **Get sending account details** (Step 1) — Verify account is active and supports transfer type
4. **Choose transfer type** — Wire, ACH, or RTP
5. **Initiate transfer** (Step 2a, 2b, or 2c) — Reference the external account UID in `receiving_account_uid`
6. **Monitor status** (Step 3) — Poll transfer status until settled or failed
7. **For future transfers** — Repeat steps 1, 3, 4, 5, 6 (skip step 2; account already exists)

## Workflow Summary: Combined Transfer (Create Destination and Transfer in One Call)

1. **Authenticate**
2. **Get sending account details** (Step 1) — Verify account is active
3. **Choose transfer type** — Wire, ACH, or RTP
4. **Initiate combined transfer** — Create external account and transfer in one API call (see [Combined Transfers](#combined-transfers-create-destination-and-transfer-in-one-call))
5. **Monitor combined transfer status** — Poll until settled or failed
6. **If you need to send to the same recipient again** — Create an external synthetic account separately and reuse it in future transfers

## Workflow Summary: ACH with Prenote

1. **Authenticate**
2. **Get sending account details** (Step 1)
3. **Initiate prenote ACH** (Step 2b) — Set `prenote_flag: true`, amount 0
4. **Wait for prenote settlement** — Typically 1 business day
5. **Initiate full ACH transfer** (Step 2b) — With actual amount and recipient account
6. **Monitor status** — Poll until settled

## Next Steps

For advanced payment workflows:
- [Payment Operations Skill](payment-operations-skill.md) — Handle returns, reversals
- [Transaction Reporting Skill](transaction-reporting-skill.md) — Query transaction history and balances
