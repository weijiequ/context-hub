---
name: subscriptions
description: "Zuora subscription billing guide covering the full order-to-cash lifecycle: product catalog setup, subscription creation via Orders API, billing runs, invoicing, payments, and revenue recognition"
metadata:
  languages: "javascript"
  versions: "2025-08-12"
  revision: 1
  updated-on: "2026-03-31"
  source: maintainer
  tags: "zuora,subscriptions,billing,orders,invoices,payments,revenue"
---
# Zuora Subscription Billing Guide

## 1. Golden Rules

**Use the Orders API for all subscription operations.** The Orders API is the modern, recommended approach for creating, amending, renewing, and canceling subscriptions. It replaces the legacy Subscribe and Amend calls.

**Always preview before creating.** Use `preview_order()` to validate your order and see projected billing documents before committing.

**Follow the order-to-cash flow:** Product Catalog -> Account -> Order (Subscription) -> Bill Run -> Invoice -> Payment.

## 2. Order-to-Cash Overview

```
Product Catalog          Account              Order
┌──────────────┐    ┌──────────────┐    ┌──────────────────┐
│ Product       │    │ Name         │    │ Order Date       │
│  └─ Rate Plan │    │ Bill-To      │    │ Account          │
│     └─ Charge │    │ Currency     │    │ Subscriptions    │
│       └─ Tier │    │ Payment Info │    │  └─ Order Actions│
└──────────────┘    └──────────────┘    └──────────────────┘
                                              │
                    ┌─────────────────────────┘
                    ▼
              Bill Run / Invoice              Payment
         ┌──────────────────────┐     ┌──────────────────┐
         │ Invoice Number       │     │ Amount           │
         │ Invoice Items        │     │ Payment Method   │
         │ Amount / Balance     │     │ Applied Invoices │
         └──────────────────────┘     └──────────────────┘
```

## 3. Product Catalog Setup

Before creating subscriptions, you need a product catalog: **Product > Rate Plan > Charge > Tier**.

### Create a Product

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

```java
CreateProductRequest productRequest = new CreateProductRequest()
    .name("Gold Membership")
    .effectiveStartDate(LocalDate.of(2024, 1, 1))
    .effectiveEndDate(LocalDate.of(2034, 1, 1))
    .SKU("SKU-GOLD-001")
    .description("Premium subscription tier");

ProxyCreateOrModifyResponse response = zuoraClient.productsApi()
    .createProductApi(productRequest).execute();
```

```javascript
let createProductRequest = new ZuoraAPI.CreateProductRequest();
createProductRequest.Name = 'Gold Membership';
createProductRequest.Description = 'Premium subscription tier';
createProductRequest.EffectiveStartDate = '2024-01-01';
createProductRequest.EffectiveEndDate = '2034-01-01';

let product = await zuoraClient.productsApi.createProduct(createProductRequest);
```

```csharp
var product = zuoraClient.ProductsApi.CreateProduct(
    new CreateProductRequest(
        name: "Gold Membership",
        effectiveStartDate: DateOnly.Parse("2024-01-01"),
        effectiveEndDate: DateOnly.Parse("2034-01-01")));
```

### Create a Rate Plan

```python
# Rate plans are created via the CRUD API
rate_plan = client.product_rate_plans_api().create_product_rate_plan({
    'Name': 'Monthly Plan',
    'ProductId': product_id,
    'EffectiveStartDate': '2024-01-01',
    'EffectiveEndDate': '2034-01-01',
    'Description': 'Monthly billing plan',
    'activeCurrencies': ['USD']
})
```

```java
CreateProductRatePlanRequest planRequest = new CreateProductRatePlanRequest()
    .name("Monthly Plan")
    .productId(productId)
    .effectiveStartDate(LocalDate.of(2024, 1, 1))
    .effectiveEndDate(LocalDate.of(2034, 1, 1))
    .description("Monthly billing plan")
    .activeCurrencies(List.of("USD"));

ProxyCreateOrModifyResponse ratePlan = zuoraClient.productRatePlansApi()
    .createProductRatePlanApi(planRequest).execute();
```

### Create a Charge with Pricing Tiers

```java
CreateProductRatePlanChargeRequest chargeRequest = new CreateProductRatePlanChargeRequest()
    .name("Monthly Fee")
    .chargeModel(ChargeModelProductRatePlanChargeRest.FLAT_FEE_PRICING)
    .chargeType(ChargeType.RECURRING)
    .triggerEvent(TriggerEventProductRatePlanChargeRest.CONTRACTEFFECTIVE)
    .productRatePlanId(ratePlanId)
    .billCycleType(BillCycleType.DEFAULTFROMCUSTOMER)
    .billingPeriod(BillingPeriodProductRatePlanChargeRest.MONTHLY)
    .productRatePlanChargeTierData(new ProductRatePlanChargeTierData()
        .addProductRatePlanChargeTierItem(new ProductRatePlanChargeTier()
            .currency("USD")
            .price(29.99)));

zuoraClient.productRatePlanChargesApi()
    .createProductRatePlanChargeApi(chargeRequest).execute();
```

```javascript
let chargeRequest = new ZuoraAPI.CreateProductRatePlanChargeRequest();
chargeRequest.Name = 'Monthly Fee';
chargeRequest.ChargeModel = 'Flat Fee Pricing';
chargeRequest.ChargeType = 'Recurring';
chargeRequest.TriggerEvent = 'ContractEffective';
chargeRequest.ProductRatePlanId = ratePlanId;
chargeRequest.BillCycleType = 'DefaultFromCustomer';
chargeRequest.BillingPeriod = 'Monthly';
chargeRequest.UseDiscountSpecificAccountingCode = false;

let tierData = new ZuoraAPI.ProductRatePlanChargeTierData();
tierData.ProductRatePlanChargeTier = [{ Currency: 'USD', price: 29.99 }];
chargeRequest.ProductRatePlanChargeTierData = tierData;

let charge = await zuoraClient.productRatePlanChargesApi
    .createProductRatePlanCharge(chargeRequest, {});
```

### Query the Catalog

```python
# Get full catalog hierarchy
rate_plans = client.object_queries_api().query_product_rate_plans(
    filter=['productId.EQ:' + product_id],
    expand=['productrateplancharges',
            'productrateplancharges.productrateplanchargetiers'])

for rp in rate_plans.data:
    print(f"Rate Plan: {rp.name}")
    for charge in rp.product_rate_plan_charges:
        print(f"  Charge: {charge.name}, Period: {charge.billing_period}")
        for tier in charge.product_rate_plan_charge_tiers:
            print(f"    {tier.currency}: ${tier.price}")
```

## 4. Account Management

### Create a Billing Account

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
account_number = account.account_number
```

### Create Account with Payment Method

```java
CreateAccountContact contact = new CreateAccountContact()
    .firstName("Jane").lastName("Doe")
    .workEmail("jane@acme.com")
    .country("United States").state("CA");

CreateAccountPaymentMethod paymentMethod = new CreateAccountPaymentMethod()
    .type("CreditCard").cardType("Visa")
    .cardNumber("4111111111111111")
    .expirationMonth(10).expirationYear(2030)
    .securityCode("123")
    .cardHolderInfo(new CreatePaymentMethodCardholderInfo()
        .cardHolderName("Jane Doe"));

CreateAccountRequest request = new CreateAccountRequest()
    .name("Acme Corp")
    .billToContact(contact)
    .paymentMethod(paymentMethod)
    .billCycleDay(1)
    .soldToSameAsBillTo(true)
    .autoPay(false)
    .currency("USD");

ApiResponse<CreateAccountResponse> response = zuoraClient.accountsApi()
    .createAccountApi(request).executeWithHttpInfo();
```

```csharp
var contact = new CreateAccountContact(
    firstName: "Jane", lastName: "Doe",
    country: "United States", state: "CA");

var paymentMethod = new CreateAccountPaymentMethod(
    type: "CreditCard", cardType: "Visa",
    cardNumber: "4111111111111111",
    expirationMonth: 10, expirationYear: 2030,
    securityCode: "123",
    cardHolderInfo: new CreatePaymentMethodCardholderInfo(cardHolderName: "Jane Doe"));

var request = new CreateAccountRequest(
    name: "Acme Corp",
    billToContact: contact,
    paymentMethod: paymentMethod,
    billCycleDay: 1,
    soldToSameAsBillTo: true,
    autoPay: false,
    currency: "USD");

CreateAccountResponse response = zuoraClient.AccountsApi.CreateAccount(request);
```

## 5. Subscription Operations

### Create Subscription (Evergreen)

An evergreen subscription has no end date and continues until explicitly canceled.

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
```

```csharp
CreateOrderResponse response = zuoraClient.OrdersApi.CreateOrder(
    new CreateOrderRequest(
        orderDate: DateOnly.FromDateTime(DateTime.Today),
        existingAccountNumber: accountNumber,
        subscriptions: [
            new CreateOrderSubscription(
                orderActions: [
                    new CreateOrderAction(
                        type: OrderActionType.CreateSubscription,
                        createSubscription: new CreateOrderCreateSubscription(
                            terms: new OrderActionCreateSubscriptionTerms(
                                initialTerm: new InitialTerm(termType: TermType.EVERGREEN)),
                            subscribeToRatePlans: [
                                new CreateOrderRatePlanOverride(
                                    productRatePlanNumber: "PRP-00000774")
                            ]))])],
        processingOptions: new ProcessingOptionsWithDelayedCapturePayment(
            runBilling: true,
            billingOptions: new BillingOptions(
                targetDate: DateOnly.FromDateTime(DateTime.Today)))));
```

### Create Subscription (Termed)

A termed subscription has a fixed duration.

```python
order = client.orders_api().create_order(
    CreateOrderRequest(
        order_date=date.today().strftime('%Y-%m-%d'),
        existing_account_number='A00000001',
        subscriptions=[{
            'order_actions': [{
                'type': 'CreateSubscription',
                'create_subscription': {
                    'terms': {
                        'initial_term': {
                            'term_type': 'TERMED',
                            'period': 12,
                            'period_type': 'Month'
                        },
                        'renewal_terms': [{
                            'period': 12,
                            'period_type': 'Month'
                        }],
                        'auto_renew': True
                    },
                    'subscribe_to_rate_plans': [
                        {'product_rate_plan_number': 'PRP-00000151'}
                    ]
                }
            }]
        }]
    ))
```

### Preview Order

**ALWAYS preview before creating** to validate and see projected billing:

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
            'preview_number_of_periods': 3,
            'preview_types': ['BillingDocs', 'ChargeMetrics']
        }
    ))
# Inspect projected invoices before committing
```

### Cancel Subscription

```python
# Cancel via Order API
order = client.orders_api().create_order(
    CreateOrderRequest(
        order_date=date.today().strftime('%Y-%m-%d'),
        existing_account_number='A00000001',
        subscriptions=[{
            'subscription_number': 'A-S00000001',
            'order_actions': [{
                'type': 'CancelSubscription',
                'cancel_subscription': {
                    'cancel_date': date.today().strftime('%Y-%m-%d'),
                    'cancel_policy': 'SpecificDate'
                }
            }]
        }]
    ))
```

### Renew Subscription

```python
# Renew via Order API
order = client.orders_api().create_order(
    CreateOrderRequest(
        order_date=date.today().strftime('%Y-%m-%d'),
        existing_account_number='A00000001',
        subscriptions=[{
            'subscription_number': 'A-S00000001',
            'order_actions': [{
                'type': 'RenewSubscription',
                'renew_subscription': {
                    'terms': {
                        'renewal_terms': [{
                            'period': 12,
                            'period_type': 'Month'
                        }]
                    }
                }
            }]
        }]
    ))
```

### Query Subscriptions

```python
# Get subscription details with rate plans
subscriptions = client.object_queries_api().query_subscriptions(
    filter=['accountId.EQ:account-id'],
    expand=['rateplans', 'rateplans.rateplancharges'])
```

```csharp
// Query subscription with expansion
var sub = zuoraClient.ObjectQueriesApi.QuerySubscriptionByKey(
    "A-S00000001");
```

## 6. Invoicing

### Standalone Invoices

Create invoices directly without a subscription:

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
print(f"Invoice {invoice.invoice_number}: ${invoice.balance}")
```

```csharp
var invoice = zuoraClient.InvoicesApi.CreateStandaloneInvoice(
    new CreateInvoiceRequest(
        accountNumber: accountNumber,
        autoPay: false,
        invoiceDate: DateOnly.Parse("2024-01-01"),
        status: BillingDocumentStatus.Posted,
        invoiceItems: [
            new CreateInvoiceItem(
                amount: 100,
                chargeName: "Set Up Fee",
                description: "One-time setup charge",
                quantity: 1.0m,
                serviceStartDate: DateOnly.Parse("2024-01-01"),
                uom: "Each")]));
```

### Bill Runs

Generate invoices for subscriptions in bulk:

```javascript
const billRunFilter = new ZuoraAPI.BillRunFilter();
billRunFilter.filterType = 'Account';
billRunFilter.accountId = 'A00000001';

const billRunRequest = new ZuoraAPI.CreateBillRunRequest();
billRunRequest.billRunFilters = [billRunFilter];
billRunRequest.targetDate = '2025-01-31';

const billRun = await zuoraClient.billRunApi.createBillRun(billRunRequest);
```

### Query Invoices

```python
# By invoice number with line items
invoice = client.object_queries_api().query_invoice_by_key(
    'INV00000315',
    expand=['invoiceItems'])

# By account
invoices = client.object_queries_api().query_invoices(
    filter=['accountId.EQ:account-uuid'])
```

## 7. Payments

### Create Payment and Apply to Invoice

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
                invoice_id=invoice_id
            )
        ]
    ))
```

```csharp
zuoraClient.PaymentsApi.CreatePayment(new CreatePaymentRequest(
    accountNumber: accountNumber,
    amount: 100.0,
    currency: "USD",
    effectiveDate: "2024-11-30",
    type: PaymentType.External,
    paymentMethodType: "Check",
    invoices: [
        new CreatePaymentInvoiceApplication(
            amount: 100.0,
            invoiceId: invoice.Id)]));
```

### Payment Types

| Type | Description |
|---|---|
| **External** | Payment processed outside Zuora (check, wire, etc.) |
| **Electronic** | Payment processed through Zuora payment gateway |

### Payment Method Types

Common payment methods: `CreditCard`, `DebitCard`, `ACH`, `PayPal`, `Check`, `WireTransfer`, `Other`

## 8. Credit & Debit Memos

### Credit Memo

Apply credits to customer accounts for refunds, adjustments, or corrections.

### Debit Memo

Create additional charges outside the normal billing cycle.

Both support async operations for large volumes:
- `POST /v1/creditmemos` — Create credit memo
- `POST /v1/debitmemos` — Create debit memo
- Async variants available for bulk operations

## 9. Usage-Based Billing

Submit usage records for consumption-based charges:

```python
# Submit usage data
usage = client.usage_api().post_usage(
    account_number='A00000001',
    usage_data=[{
        'quantity': 150,
        'start_date_time': '2024-01-01T00:00:00',
        'uom': 'GB',
        'subscription_number': 'A-S00000001',
        'charge_number': 'C-00000001'
    }]
)
```

## 10. Order Actions Summary

| Action Type | Description |
|---|---|
| `CreateSubscription` | Create a new subscription |
| `AddProduct` | Add a rate plan to existing subscription |
| `RemoveProduct` | Remove a rate plan |
| `UpdateProduct` | Modify charge quantities/pricing |
| `RenewSubscription` | Renew a termed subscription |
| `CancelSubscription` | Cancel a subscription |
| `TermsAndConditions` | Update subscription terms |
| `OwnerTransfer` | Transfer subscription ownership |
| `Suspend` | Temporarily suspend a subscription |
| `Resume` | Resume a suspended subscription |

## 11. Complete Order-to-Cash Example (Python)

```python
import os
from datetime import date
from zuora_sdk.zuora_client import ZuoraClient, ZuoraEnvironment
from zuora_sdk import (
    CreateAccountRequest, CreateOrderRequest,
    CreateInvoiceRequest, CreateInvoiceItem,
    CreatePaymentRequest, CreatePaymentInvoiceApplication,
    ProcessingOptionsWithDelayedCapturePayment
)
from zuora_sdk.rest import ApiException

# 1. Initialize client
client = ZuoraClient(
    client_id=os.environ['ZUORA_CLIENT_ID'],
    client_secret=os.environ['ZUORA_CLIENT_SECRET'],
    env=ZuoraEnvironment.SBX
)
client.initialize()

# 2. Create account
account = client.accounts_api().create_account(
    CreateAccountRequest(
        name='Acme Corp',
        bill_to_contact={
            'first_name': 'Jane', 'last_name': 'Doe',
            'state': 'California', 'country': 'USA'
        },
        auto_pay=False, currency='USD', bill_cycle_day='1'
    ))
print(f"Account: {account.account_number}")

# 3. Create subscription via order
order = client.orders_api().create_order(
    CreateOrderRequest(
        order_date=date.today().strftime('%Y-%m-%d'),
        existing_account_number=account.account_number,
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
        processing_options=ProcessingOptionsWithDelayedCapturePayment(
            run_billing=True,
            billing_options={'target_date': date.today().strftime('%Y-%m-%d')}
        )
    ))
print(f"Order: {order.order_number}")

# 4. Create standalone invoice
invoice = client.invoices_api().create_standalone_invoice(
    CreateInvoiceRequest(
        account_number=account.account_number,
        auto_pay=False,
        invoice_date=date.today().strftime('%Y-%m-%d'),
        status='Posted',
        invoice_items=[
            CreateInvoiceItem(
                amount=100.0, charge_name='Set Up Fee',
                quantity=1.0, service_start_date=date.today().strftime('%Y-%m-%d'),
                uom='Each'
            )
        ]
    ))
print(f"Invoice: {invoice.invoice_number}, Balance: ${invoice.balance}")

# 5. Apply payment
payment = client.payments_api().create_payment(
    CreatePaymentRequest(
        account_number=account.account_number,
        amount=100.0, currency='USD',
        effective_date=date.today().strftime('%Y-%m-%d'),
        type='External', payment_method_type='Check',
        invoices=[CreatePaymentInvoiceApplication(
            amount=100.0, invoice_id=invoice.id
        )]
    ))
print(f"Payment: {payment.number}")
```

## 12. Best Practices

- **Always preview orders** before creating to catch validation errors early
- **Use processing_options** to control whether billing runs immediately or is deferred
- **Prefer account numbers** over account IDs for readability in order requests
- **Use product_rate_plan_number** (not ID) for portability across environments
- **Handle partial failures** — some order actions may succeed while others fail
- **Use async endpoints** for bulk bill runs and large order batches
- **Set up webhooks** for billing events (invoice posted, payment received, etc.)
- **Test the full flow** in Sandbox before Production: catalog -> account -> order -> bill -> pay
