# Managing Users and Accounts Skill Guide

**Prerequisite for**: Making Payments, Payment Operations, and Transaction Reporting skills

This guide provides step-by-step instructions for agents to manage customers and synthetic accounts in Newline's payment system. Both customers and synthetic accounts must be properly set up before initiating transfers or other payment operations.

## Overview

Two key entity types form the foundation of payment operations:

1. **Customers** — Represent individuals or organizations using Newline's payment services
2. **Synthetic Accounts** — Logical accounts created for customers that support fund transfers and balance tracking

### Synthetic Account Types

Synthetic accounts serve different purposes based on their type:

| Account Type | Purpose | Use Case | Category |
|---|---|---|---|
| **General Synthetic Account** | Represents your customer's account (source/sending account) | Create for each of your customers who need to send funds | General use |
| **External Synthetic Account** | Represents an external counterparty's account (destination/receiving account) | Create for external recipients you'll repeatedly send funds to; avoids re-entering details each transfer | External account |

**Choosing the Right Account Type**:
- **One-time or infrequent recipient** → Use external account details directly (no synthetic account needed)
- **Repeated recipient** → Create an external synthetic account first; reuse the account UID in future transfers
- **Your customer sending funds** → Always create a general synthetic account for each customer

**Link to Documentation**: [Synthetic Accounts Reference](https://developers.newline53.com/reference/synthetic-accounts) and [Synthetic Account Categories](https://developers.newline53.com/reference/synthetic-account-categories)

## Step 1: Authenticate

Before performing any account management operations, you must authenticate with valid credentials.

**Tool**: `newline-authenticate`

**Required Parameters**:
- `api_key` — Your Newline API key (typically from environment variable `NEWLINE_API_KEY`)

**Output**: Authentication token (automatically handled by the MCP server)

**Common Issues**:
- Missing or invalid `api_key` — Verify credentials are set correctly
- Expired credentials — Refresh authentication and retry

See [Authentication Configuration](../README.md#configuration) for setup details.

## Step 2: List Existing Customers

Query all customers in the system to identify existing customer records or verify customer setup.

**Tool**: `newline-list-customers`

**Optional Parameters**:
- `limit` — Number of results to return (default: 10)
- `offset` — Pagination offset (default: 0)

**Example Response Fields**:
```json
{
  "customers": [
    {
      "uid": "cust_123abc",
      "external_uid": "customer_id_from_your_system",
      "name": "John Doe",
      "email": "john@example.com",
      "created_at": "2026-01-15T10:30:00Z"
    }
  ],
  "total_count": 5
}
```

**Link to Source**: [src/customers.ts](../../src/customers.ts)

**Common Scenarios**:
- Paginate through results using `limit` and `offset` to find a specific customer
- Check `external_uid` to map your system's customer IDs to Newline UIDs

## Step 3: Get Customer Details

Retrieve detailed information about a specific customer.

**Tool**: `newline-get-customer`

**Required Parameters**:
- `uid` — Customer unique identifier (from list-customers or customer creation response)

**Example Response Fields**:
```json
{
  "uid": "cust_123abc",
  "external_uid": "customer_id_from_your_system",
  "name": "John Doe",
  "email": "john@example.com",
  "created_at": "2026-01-15T10:30:00Z"
}
```

**Link to Source**: [src/customers.ts](../../src/customers.ts)

**Common Issues**:
- `unknown_customer` error — Verify the customer UID exists (use list-customers to confirm)
- Invalid UID format — Ensure the UID is correct (typically starts with `cust_`)

## Step 4: Create a New Customer

Create a new customer record in Newline's system.

**Tool**: `newline-create-customer`

**Required Parameters**:
- `external_uid` — Your system's customer identifier (string, max 255 characters)
- `name` — Customer's full name (string, max 255 characters)

**Optional Parameters**:
- `email` — Customer's email address (string, valid email format)

**Example Request**:
```json
{
  "external_uid": "my_customer_001",
  "name": "Jane Smith",
  "email": "jane.smith@example.com"
}
```

**Example Response Fields**:
```json
{
  "uid": "cust_456def",
  "external_uid": "my_customer_001",
  "name": "Jane Smith",
  "email": "jane.smith@example.com",
  "created_at": "2026-01-31T14:22:00Z"
}
```

**Link to Source**: [src/customers.ts](../../src/customers.ts)

**Important Notes**:
- The `external_uid` must be unique within your system
- Once created, the customer UID cannot be changed
- Email is optional but recommended for communication

**Common Issues**:
- `duplicate_external_uid` — The external_uid already exists; use a different identifier
- `invalid_email_format` — Verify email address is valid if provided
- `missing_required_field` — Ensure `external_uid` and `name` are provided

## Step 5: List Synthetic Account Types

Before creating synthetic accounts, view available account types supported by your program.

**Tool**: `newline-list-synthetic-account-types`

**Optional Parameters**:
- `limit` — Number of results to return (default: 10)
- `offset` — Pagination offset (default: 0)

**Example Response Fields**:
```json
{
  "synthetic_account_types": [
    {
      "uid": "type_001",
      "name": "Standard Checking",
      "description": "Standard operational checking account",
      "supports_wire": true,
      "supports_ach": true,
      "supports_rtp": true
    }
  ]
}
```

**Link to Source**: [src/synthetic_accounts.ts](../../src/synthetic_accounts.ts)

**Usage**: Note the type UID to use when creating synthetic accounts in Step 6.

## Step 6: Create a Synthetic Account

Create a synthetic account for a customer to enable fund transfers and balance tracking. Synthetic accounts can represent:
- **Your customer's account** (general account type) — Used as the sending/source account for transfers
- **An external counterparty's account** (external account type) — Used as the receiving/destination account for transfers to repeated recipients

**Tool**: `newline-create-synthetic-account`

**Required Parameters**:
- `customer_uid` — UID of the customer who will own this account (from Step 2 or 3)
- `synthetic_account_type_uid` — Type of account to create (from Step 5)

**Optional Parameters**:
- `external_uid` — Your system's account identifier (string, max 255 characters) — recommended for tracking

### Creating a General Synthetic Account (Your Customer's Account)

Use this for customers who will be **sending funds**.

**Example Request**:
```json
{
  "customer_uid": "cust_456def",
  "synthetic_account_type_uid": "type_general_001",
  "external_uid": "account_jane_001"
}
```

**Example Response**:
```json
{
  "uid": "syn_789ghi",
  "customer_uid": "cust_456def",
  "synthetic_account_type_uid": "type_general_001",
  "external_uid": "account_jane_001",
  "status": "active",
  "account_number": "9876543210",
  "routing_number": "123456789",
  "transfer_methods": {...},
  "created_at": "2026-01-31T14:25:00Z"
}
```

### Creating an External Synthetic Account (Recipient's Account)

Use this for external counterparties you'll **repeatedly send funds to**. This allows you to reuse the account UID in future transfers instead of entering their details each time.

**Example Request**:
```json
{
  "customer_uid": "ext_counterparty_001",
  "synthetic_account_type_uid": "type_external_001",
  "external_uid": "recipient_acme_corp"
}
```

**Example Response**:
```json
{
  "uid": "syn_external_001",
  "customer_uid": "ext_counterparty_001",
  "synthetic_account_type_uid": "type_external_001",
  "external_uid": "recipient_acme_corp",
  "status": "active",
  "created_at": "2026-01-31T14:30:00Z"
}
```

**Link to Source**: [src/synthetic_accounts.ts](../../src/synthetic_accounts.ts)

**Important Notes**:
- A customer can have multiple synthetic accounts (both general and external types)
- **General accounts** — Get `account_number` and `routing_number` for use in transfers (returned in response)
- **External accounts** — Use the account UID directly in transfers when sending to repeated recipients
- Account status should be "active" for normal operations
- For **one-time recipients**, you can skip creating an external synthetic account and provide their details directly in the transfer request (see [Making Payments Skill](making-payments-skill.md#step-2a-initiate-a-wire-transfer))

**When to Create External Synthetic Accounts**:
- Creating an external synthetic account is optional
- **Create one if** you'll send funds to the same external recipient multiple times (reuse the account UID)
- **Skip it if** the recipient is a one-time destination (provide account details directly in transfer request)
- **Alternative** — Use Combined Transfers endpoint to create the external account and transfer in one call (see [Making Payments Skill: Combined Transfers](making-payments-skill.md#combined-transfers-create-destination-and-transfer-in-one-call))

**Common Issues**:
- `unknown_customer` — Verify customer UID exists (use get-customer to confirm)
- `unknown_synthetic_account_type` — Verify the type UID is correct (use list-synthetic-account-types)
- `duplicate_external_uid` — The external_uid already exists; use a different identifier

## Step 7: List All Synthetic Accounts

Query all synthetic accounts to verify account setup or find accounts for a customer.

**Tool**: `newline-list-synthetic-accounts`

**Optional Parameters**:
- `limit` — Number of results to return (default: 10)
- `offset` — Pagination offset (default: 0)

**Example Response Fields**:
```json
{
  "synthetic_accounts": [
    {
      "uid": "syn_789ghi",
      "customer_uid": "cust_456def",
      "synthetic_account_type_uid": "type_001",
      "external_uid": "account_jane_001",
      "status": "active",
      "created_at": "2026-01-31T14:25:00Z"
    }
  ],
  "total_count": 3
}
```

**Link to Source**: [src/synthetic_accounts.ts](../../src/synthetic_accounts.ts)

## Step 8: Get Synthetic Account Details

Retrieve detailed information about a specific synthetic account, including routing information needed for transfers.

**Tool**: `newline-get-synthetic-account`

**Required Parameters**:
- `uid` — Synthetic account unique identifier (from create or list operations)

**Example Response Fields**:
```json
{
  "uid": "syn_789ghi",
  "customer_uid": "cust_456def",
  "synthetic_account_type_uid": "type_001",
  "external_uid": "account_jane_001",
  "status": "active",
  "account_number": "9876543210",
  "routing_number": "123456789",
  "transfer_methods": {
    "wire": {
      "enabled": true,
      "address": {...}
    },
    "ach": {
      "enabled": true,
      "company_id": "000000001",
      "originator_name": "NEWLINE INC"
    },
    "rtp": {
      "enabled": true,
      "address": {...}
    }
  },
  "created_at": "2026-01-31T14:25:00Z"
}
```

**Link to Source**: [src/synthetic_accounts.ts](../../src/synthetic_accounts.ts)

**Important Notes**:
- The `account_number` and `routing_number` are required for ACH transfers
- The `transfer_methods` object shows which transfer types are enabled for this account
- This information is essential for the Making Payments and Payment Operations skills

**Common Issues**:
- `unknown_synthetic_account` — Verify the account UID exists (use list-synthetic-accounts to confirm)
- Account status not "active" — Cannot use account for transfers until status is active

## Step 9: Delete a Synthetic Account (Optional)

Remove a synthetic account from the system.

**Tool**: `newline-delete-synthetic-account`

**Required Parameters**:
- `uid` — Synthetic account unique identifier

**Important Notes**:
- This operation is typically irreversible
- Only delete accounts that are no longer needed
- Verify no pending transfers exist before deletion

**Common Issues**:
- `account_has_pending_transfers` — Wait for transfers to complete before deletion
- `unknown_synthetic_account` — Verify the account UID is correct

## Workflow Summary

For a new customer:

1. **Authenticate** (Step 1)
2. **Create customer** (Step 4)
3. **Check account types** (Step 5)
4. **Create synthetic account** (Step 6)
5. **Verify account details** (Step 8) — Note routing information
6. **Proceed to Making Payments skill** to initiate transfers

For existing customers:

1. **Authenticate** (Step 1)
2. **List customers** (Step 2) and identify the customer UID
3. **Get customer details** (Step 3) — Verify customer exists
4. **List synthetic accounts** (Step 7) and identify the account UID
5. **Get account details** (Step 8) — Note routing information
6. **Proceed to Making Payments skill** or Payment Operations skill

## Next Steps

Once you have customers and synthetic accounts configured, proceed to:
- [Making Payments Skill](making-payments-skill.md) — Initiate wire, ACH, and instant payment transfers
- [Payment Operations Skill](payment-operations-skill.md) — Handle returns, reversals, and complex payment workflows
- [Transaction Reporting Skill](transaction-reporting-skill.md) — Monitor transfers and query transaction history
