---
name: api
description: "Subscription billing and revenue management platform with comprehensive REST APIs for accounts, subscriptions, orders, invoicing, payments, product catalog, usage, and Object Query"
metadata:
  languages: "javascript"
  versions: "2025-08-12"
  revision: 1
  updated-on: "2026-03-31"
  source: maintainer
  tags: "zuora,api,billing,subscriptions,payments,invoicing,revenue"
---
# Zuora API Coding Guide

## 1. Golden Rules

**Always use the official Zuora SDK packages:**
- Python: `zuora-sdk` (PyPI)
- JavaScript: `zuora-sdk-js` (npm)
- Java: `zuora-sdk-java` (Maven Central)
- C#: `ZuoraSDK` (NuGet)

**Never call the REST API directly when an SDK is available.** The SDKs handle authentication, token refresh, retries, and serialization automatically.

**Current API Version:** `2025-08-12` (date-based versioning). Specify via the `Zuora-Version` header or SDK configuration.

**Always use ZuoraClient with auto-auth.** The `ZuoraClient` class manages OAuth2 token lifecycle automatically, including background refresh and thread-safe access.

## 2. Environments

Zuora provides multiple environments across regions:

| Environment | Base URL |
|---|---|
| US Production (Cloud 2) | `https://rest.zuora.com` |
| US Sandbox (Cloud 2) | `https://rest.apisandbox.zuora.com` |
| US Production (Cloud 1) | `https://rest.na.zuora.com` |
| EU Production | `https://rest.eu.zuora.com` |
| EU Sandbox | `https://rest.sandbox.eu.zuora.com` |

All SDKs support environment selection via enum constants (e.g., `ZuoraEnvironment.SBX`, `ZuoraClient.ZuoraEnv.SBX`).

## 3. Authentication

Zuora uses **OAuth 2.0 Client Credentials** flow exclusively.

### Environment Variables

```bash
# Required
ZUORA_CLIENT_ID=your-client-id        # OAuth client ID from Zuora Admin
ZUORA_CLIENT_SECRET=your-client-secret  # OAuth client secret

# Optional
ZUORA_BASE_URL=https://rest.apisandbox.zuora.com  # Override environment URL
ZUORA_ENTITY_IDS=entity-id-1,entity-id-2          # Multi-entity scoping
ZUORA_ORG_IDS=org-id-1                             # Multi-org scoping
```

**CRITICAL:** Never commit client secrets to version control. Use environment variables or secure secret management.

### Authentication Flow

1. SDK sends `POST /oauth/token` with `client_id`, `client_secret`, and `grant_type=client_credentials`
2. Zuora returns a Bearer token (valid ~60 minutes)
3. SDK auto-refreshes tokens in the background (every 10 minutes by default)
4. All subsequent API calls include `Authorization: Bearer {token}` header

### Quick Start (Python)

```python
import os
from zuora_sdk.zuora_client import ZuoraClient, ZuoraEnvironment

client = ZuoraClient(
    client_id=os.environ['ZUORA_CLIENT_ID'],
    client_secret=os.environ['ZUORA_CLIENT_SECRET'],
    env=ZuoraEnvironment.SBX
)
client.initialize()  # Authenticates and starts background token refresh
```

### Quick Start (JavaScript)

```javascript
const { ZuoraClient } = require('zuora-sdk-js');

const client = new ZuoraClient({
    clientId: process.env.ZUORA_CLIENT_ID,
    clientSecret: process.env.ZUORA_CLIENT_SECRET,
    env: ZuoraClient.SBX,
});
await client.initialize();
```

### Quick Start (Java)

```java
import com.zuora.ZuoraClient;

ZuoraClient client = new ZuoraClient(
    System.getenv("ZUORA_CLIENT_ID"),
    System.getenv("ZUORA_CLIENT_SECRET"),
    ZuoraClient.ZuoraEnv.SBX
);
client.initialize();
```

### Quick Start (C#)

```csharp
using ZuoraSDK.Client;

var client = new ZuoraClient(
    Environment.GetEnvironmentVariable("ZUORA_CLIENT_ID"),
    Environment.GetEnvironmentVariable("ZUORA_CLIENT_SECRET"),
    ZuoraEnv.SBX
);
client.Initialize();
```

## 4. Core API Surfaces

### 4.1 Accounts

Create and manage customer billing accounts.

**Create Account:**

```python
from zuora_sdk import CreateAccountRequest

account = client.accounts_api().create_account(
    CreateAccountRequest(
        name='Acme Corp',
        bill_to_contact={
            'first_name': 'Jane',
            'last_name': 'Doe',
            'state': 'California',
            'country': 'USA'
        },
        auto_pay=False,
        currency='USD',
        bill_cycle_day='1'
    ))
print(f"Account created: {account.account_number}")
```

**Query Account with Expansion:**

```python
account = client.object_queries_api().query_account_by_key(
    'A00000001',
    expand=['billTo', 'subscriptions'])
```

### 4.2 Product Catalog

Build your product catalog with Products, Rate Plans, and Charges.

**Create Product:**

```python
from zuora_sdk import CreateProductRequest

product = client.products_api().create_product(
    CreateProductRequest(
        name='Gold Membership',
        description='Premium subscription tier',
        effective_start_date='2024-01-01',
        effective_end_date='2034-01-01'
    ))
product_id = product.id
```

**Query Products:**

```python
products = client.object_queries_api().query_products()
for prod in products.data:
    print(f"Product: {prod.product_number} - {prod.name}")
```

**Query Rate Plans with Charges and Tiers:**

```python
rate_plans = client.object_queries_api().query_product_rate_plans(
    filter=['productId.EQ:8ad097b4917efc7701917f0d297d01b7'],
    expand=['productrateplancharges', 'productrateplancharges.productrateplanchargetiers'])
for rp in rate_plans.data:
    print(f"Rate Plan: {rp.product_rate_plan_number} - {rp.name}")
```

### 4.3 Orders & Subscriptions

Use the Orders API to create, amend, renew, and cancel subscriptions.

**Create Subscription via Order:**

```python
from datetime import date
from zuora_sdk import CreateOrderRequest, ProcessingOptionsWithDelayedCapturePayment

order = client.orders_api().create_order(
    CreateOrderRequest(
        order_date=date.today().strftime('%Y-%m-%d'),
        existing_account_number='A00000001',
        subscriptions=[{
            'order_actions': [{
                'type': 'CreateSubscription',
                'create_subscription': {
                    'terms': {
                        'initial_term': {'term_type': 'EVERGREEN'}
                    },
                    'subscribe_to_rate_plans': [
                        {'product_rate_plan_number': 'PRP-00000151'}
                    ]
                }
            }]
        }],
        processing_options=ProcessingOptionsWithDelayedCapturePayment(
            run_billing=True,
            billing_options={'target_date': date.today().strftime('%Y-%m-%d')}
        )
    ))
print(f"Order: {order.order_number}, Subscription: {order.subscription_numbers}")
```

**Preview Order (dry run):**

```python
from zuora_sdk import PreviewOrderRequest

preview = client.orders_api().preview_order(
    PreviewOrderRequest(
        order_date=date.today().strftime('%Y-%m-%d'),
        existing_account_number='A00000001',
        subscriptions=[{
            'order_actions': [{
                'type': 'CreateSubscription',
                'create_subscription': {
                    'terms': {'initial_term': {'term_type': 'EVERGREEN'}},
                    'subscribe_to_rate_plans': [
                        {'product_rate_plan_number': 'PRP-00000151'}
                    ]
                }
            }]
        }],
        preview_options={
            'preview_thru_type': 'NumberOfPeriods',
            'preview_number_of_periods': 1,
            'preview_types': ['BillingDocs']
        }
    ))
```

### 4.4 Invoicing

**Create Standalone Invoice:**

```python
from zuora_sdk import CreateInvoiceRequest, CreateInvoiceItem

invoice = client.invoices_api().create_standalone_invoice(
    CreateInvoiceRequest(
        account_number='A00000001',
        auto_pay=False,
        invoice_date='2024-01-01',
        status='Posted',
        invoice_items=[
            CreateInvoiceItem(
                amount=100.0,
                charge_name='Set Up Fee',
                description='One-time setup charge',
                quantity=1.0,
                service_start_date='2024-01-01',
                uom='Each'
            )
        ]
    ))
print(f"Invoice {invoice.invoice_number} created, balance: {invoice.balance}")
```

**Query Invoices:**

```python
# By invoice number
inv = client.object_queries_api().query_invoice_by_key(
    'INV00000315',
    expand=['invoiceItems'])

# By account
invoices = client.object_queries_api().query_invoices(
    filter=['accountId.EQ:2c92c0f96db4d8cc016db9400a4e4c16'])
```

### 4.5 Payments

**Create Payment and Apply to Invoice:**

```python
from zuora_sdk import CreatePaymentRequest, CreatePaymentInvoiceApplication

payment = client.payments_api().create_payment(
    CreatePaymentRequest(
        account_number='A00000001',
        amount=100.0,
        currency='USD',
        effective_date='2024-11-30',
        type='External',
        payment_method_type='Check',
        invoices=[
            CreatePaymentInvoiceApplication(
                amount=100.0,
                invoice_id=invoice.id
            )
        ]
    ))
print(f"Payment created: {payment.number}")
```

## 5. Object Query API

The Object Query API provides a powerful, consistent way to query 40+ Zuora object types with filtering, sorting, expansion, and pagination.

### Endpoint Pattern

```
GET /object-query/{objectType}           # List objects
GET /object-query/{objectType}/{key}     # Get by key
```

### Filtering

Use dot-notation operators in the `filter` parameter:

| Operator | Description | Example |
|---|---|---|
| `EQ` | Equal | `currency.EQ:EUR` |
| `NE` | Not equal | `status.NE:Draft` |
| `LT` | Less than | `amount.LT:100` |
| `GT` | Greater than | `amount.GT:0` |
| `LE` | Less than or equal | `balance.LE:500` |
| `GE` | Greater than or equal | `createdDate.GE:2024-01-01` |
| `SW` | Starts with | `name.SW:Test` |
| `IN` | In set | `status.IN:Active,Cancelled` |

```python
# Filter accounts by currency
accounts = client.object_queries_api().query_accounts(
    filter=['currency.EQ:USD', 'status.EQ:Active'],
    sort=['accountNumber.ASC'],
    page_size=20)
```

### Expansion (Joins)

Include related objects in a single response using `expand`:

```python
# Get account with billing contact and subscriptions
account = client.object_queries_api().query_account_by_key(
    'A00000001',
    expand=['billTo', 'subscriptions', 'subscriptions.rateplans'])

# Get invoice with line items
invoice = client.object_queries_api().query_invoice_by_key(
    'INV00000315',
    expand=['invoiceItems'])

# Get product catalog with full charge details
rate_plans = client.object_queries_api().query_product_rate_plans(
    filter=['productId.EQ:prod-id'],
    expand=['productrateplancharges',
            'productrateplancharges.productrateplanchargetiers'])
```

### Pagination

Object Query uses cursor-based pagination:

```python
# First page
response = client.object_queries_api().query_accounts(page_size=20)
print(f"Page 1: {len(response.data)} accounts")

# Next page (if available)
if response.next_page:
    response = client.object_queries_api().query_accounts(
        page_size=20,
        cursor=response.next_page)
```

### Field Selection

Reduce response payload by selecting specific fields:

```python
accounts = client.object_queries_api().query_accounts(
    fields=['id', 'accountNumber', 'name', 'balance'])
```

## 6. Common Request Headers

| Header | Purpose | Example |
|---|---|---|
| `Authorization` | Bearer token (auto-managed by SDK) | `Bearer eyJ...` |
| `Zuora-Version` | API version override | `2025-08-12` |
| `Zuora-Track-Id` | Custom request tracing ID | `my-trace-123` |
| `Zuora-Entity-Ids` | Multi-entity scoping | `entity-uuid-1` |
| `Zuora-Org-Ids` | Multi-org scoping | `org-uuid-1` |
| `Idempotency-Key` | Idempotent POST/PATCH operations | `uuid-v4` |

## 7. Error Handling

### Error Response Structure

All Zuora API errors return a consistent structure:

```json
{
  "reasons": [
    {
      "code": 53100020,
      "message": "The account number A99999999 is invalid."
    }
  ]
}
```

### Comprehensive Error Handler (Python)

```python
import json
from zuora_sdk.rest import ApiException

try:
    account = client.accounts_api().create_account(request)
except ApiException as e:
    print(f"HTTP Status: {e.status}")
    print(f"Reason: {e.reason}")

    if e.body:
        error_details = json.loads(e.body)
        for reason in error_details.get('reasons', []):
            print(f"  Code: {reason['code']}, Message: {reason['message']}")

    # Handle specific HTTP status codes
    if e.status == 401:
        # Token expired or invalid credentials
        client.initialize()  # Re-authenticate
    elif e.status == 404:
        print("Resource not found")
    elif e.status == 429:
        # Rate limited - implement backoff
        pass
```

### Error Handler (Java)

```java
import com.zuora.ApiException;
import com.zuora.model.CommonResponse;

try {
    CreateAccountResponse response = zuoraClient.accountsApi()
        .createAccountApi(request).execute();
} catch (ApiException e) {
    System.err.println("HTTP Status: " + e.getCode());
    CommonResponse errorResponse = CommonResponse.fromJson(e.getResponseBody());
    if (errorResponse.getReasons() != null) {
        errorResponse.getReasons().forEach(reason ->
            System.err.printf("Code: %s, Message: %s%n",
                reason.getCode(), reason.getMessage()));
    }
}
```

## 8. Best Practices

### Idempotency

Use the `Idempotency-Key` header for POST/PATCH requests to prevent duplicate operations:

```python
# SDK handles this via request options
# For manual REST calls, always include:
# headers = {'Idempotency-Key': str(uuid.uuid4())}
```

### Rate Limiting

- Zuora enforces rate limits per tenant
- SDKs include built-in retry with exponential backoff (Python SDK: 3 attempts with 1s, 2s, 4s delays)
- Monitor `429 Too Many Requests` responses
- Use bulk/async endpoints for high-volume operations

### Multi-Entity & Multi-Org

For multi-entity tenants, scope API calls using headers:

```python
# Set entity context at client level
client.set_default_header('Zuora-Entity-Ids', 'entity-uuid-1')

# Or per-request via SDK options
```

### Async Operations

For long-running operations (bill runs, large orders), use async endpoints:

```python
# Async order creation
response = client.orders_api().create_order_async(request)
# Poll for completion using the job ID
```

### Custom Fields

Zuora supports custom fields (suffix `__c`) on most objects:

```python
# Include custom fields in create/update requests
account = CreateAccountRequest(
    name='Acme Corp',
    # ... standard fields ...
)
# Custom fields via additional properties
account.additional_properties = {'salesRegion__c': 'West'}
```

```java
// Java uses putAdditionalProperty
CreateAccountRequest request = new CreateAccountRequest()
    .name("Acme Corp")
    .putAdditionalProperty("salesRegion__c", "West");
```

## 9. Charge Models

Zuora supports 16 charge models for flexible pricing:

| Category | Models |
|---|---|
| **Recurring** | Flat Fee, Per Unit, Tiered, Volume |
| **One-Time** | Flat Fee, Tiered, Volume |
| **Usage-Based** | Per Unit, Flat Fee, Tiered, Volume, Tiered with Overage, Overage |
| **Discount** | Fixed Amount, Percentage |
| **Custom** | Multi-Attribute Pricing |

### Charge Types

- **Recurring** — Billed on a regular schedule (monthly, annual, etc.)
- **OneTime** — Billed once at subscription start
- **Usage** — Billed based on consumption data

## 10. Production Checklist

### Pre-Launch Verification

- [ ] Switch from Sandbox to Production environment URL
- [ ] Replace sandbox OAuth credentials with production credentials
- [ ] Set `ZUORA_CLIENT_ID` and `ZUORA_CLIENT_SECRET` via secure secret management
- [ ] Verify API version is explicitly set (`Zuora-Version` header)
- [ ] Enable auto-auth with token refresh (use `ZuoraClient.initialize()`)
- [ ] Implement error handling for all API calls with proper logging
- [ ] Use `Idempotency-Key` headers for all mutating POST/PATCH requests
- [ ] Set up `Zuora-Track-Id` headers for request tracing
- [ ] Configure rate limit handling with exponential backoff
- [ ] Test webhook endpoints receive and process events correctly
- [ ] Verify multi-entity/multi-org headers if applicable
- [ ] Review custom field mappings (`__c` fields)
- [ ] Validate product catalog (products, rate plans, charges) exists in production
- [ ] Test full order-to-cash flow: account creation, order, invoice, payment
- [ ] Enable SDK debugging in staging, disable in production

### Monitoring

- Track API response times and error rates
- Monitor OAuth token refresh cycles
- Alert on consecutive 401/403 errors (credential issues)
- Alert on 429 responses (rate limiting)
- Monitor billing run completion times
- Track payment success/failure rates

### Security

- **NEVER** expose OAuth client secrets in client-side code
- Rotate OAuth credentials periodically
- Use separate OAuth clients for different environments
- Restrict OAuth client permissions to minimum required scopes
- Audit API access logs in Zuora Admin

## 11. Key Resources

- **Developer Portal:** https://developer.zuora.com/
- **API Reference:** https://developer.zuora.com/v1-api-reference/introduction/
- **SDKs:** Python (`zuora-sdk`), JavaScript (`zuora-sdk-js`), Java (`zuora-sdk-java`), C# (`ZuoraSDK`)
- **MCP Server:** `npx zuora-mcp` for AI-assisted Zuora development
- **Knowledge Center:** https://knowledgecenter.zuora.com/
