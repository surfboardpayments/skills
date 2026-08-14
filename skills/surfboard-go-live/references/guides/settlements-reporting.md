---
title: "Settlements & Reporting"
source: https://www.surfboardpayments.com/developers/guides/settlements-reporting
category: online
tags: [Online, API, Settlements, Reporting, Charges, Adjustments]
generated: true
---

# Settlements & Reporting

> Retrieve settlement reports, view adjustments, manage merchant charges, and register customer profiles for reconciliation and billing.

## Overview

Once transactions are processed, you need visibility into what was settled, what fees were applied, and how to bill merchants for additional services. The Settlements and Reporting APIs give you that visibility.

This guide covers four related capabilities:

1. **Settlement reports** -- retrieve payout summaries for a merchant over a given period.
2. **Adjustments** -- view tips, surcharges, insurance, and other amounts applied to orders.
3. **Merchant charges** -- create, update, and list one-time or recurring charges billed to a merchant.
4. **Customer details** -- register customer profiles with addresses, contact information, and linked payment cards.

## Prerequisites

- A valid `partnerId` and `merchantId`
- API credentials (API key, API secret)

## Settlement Reports

Settlement reports summarize a merchant's settled transactions for a selected time period. Reports can be configured as `DAILY` or `MONTHLY` depending on the merchant's setup.

### Fetch settlement reports

```
GET /partners/:partnerId/merchants/:merchantId/reports
```

**Response:**

```json
{
  "status": "SUCCESS",
  "data": [
    {
      "payoutId": "po_83a1f...",
      "merchantId": "m_91b2c...",
      "transactionStartDate": "2026-01-01",
      "transactionEndDate": "2026-01-31",
      "settlementDate": "2026-02-03",
      "reportType": "MONTHLY",
      "url": "https://reports.surfboardpayments.com/settlements/po_83a1f...",
      "totalSale": 1250000,
      "totalRefund": 35000,
      "fee": 18750,
      "payout": 1196250
    }
  ],
  "message": "Settlement reports fetched successfully"
}
```

### Response fields

| Field | Type | Description |
|-------|------|-------------|
| `payoutId` | string | Identifies this specific payout |
| `transactionStartDate` | string | First transaction date covered (`YYYY-MM-DD`) |
| `transactionEndDate` | string | Last transaction date covered (`YYYY-MM-DD`) |
| `settlementDate` | string | Date the payout was issued (`YYYY-MM-DD`) |
| `reportType` | string | `MONTHLY` or `DAILY` |
| `url` | string | Direct link to view the full report |
| `totalSale` | number | Total sales amount in smallest currency unit |
| `totalRefund` | number | Total refunded amount |
| `fee` | number | Total fees deducted |
| `payout` | number | Net payout to the merchant |

Use the `url` field to download or redirect merchants to a detailed breakdown of every transaction in the settlement period.

## Adjustments

Adjustments represent additional amounts applied to orders during a transaction -- tips, surcharges, insurance payments, and similar line items. The Adjustments API lets you retrieve all adjustments at the merchant level for tracking and reconciliation.

### Fetch adjustments

```
GET /partners/:partnerId/merchants/:merchantId/adjustments?startDate=2026-01-01&endDate=2026-01-31
```

Both `startDate` and `endDate` are required query parameters in `YYYY-MM-DD` format.

**Response:**

```json
{
  "status": "SUCCESS",
  "data": [
    {
      "adjustmentId": "adj_44c2e...",
      "adjustmentType": "TIP",
      "amount": "2500"
    },
    {
      "adjustmentId": "adj_55d3f...",
      "adjustmentType": "SURCHARGE",
      "amount": "1500"
    }
  ],
  "message": "Adjustments fetched successfully"
}
```

### Response fields

| Field | Type | Description |
|-------|------|-------------|
| `adjustmentId` | string | Unique identifier for the adjustment |
| `adjustmentType` | string | Type of adjustment (e.g. `TIP`, `SURCHARGE`, `INSURANCE`) |
| `amount` | string | Adjustment amount in smallest currency unit |

## Merchant Charges

Merchant charges let partners bill merchants for services, fees, or subscriptions. A charge can be one-time or recurring, and supports VAT.

### Create a charge

```json
POST /partners/:partnerId/merchants/:merchantId/charges
{
  "description": "Monthly platform fee",
  "currency": "752",
  "amount": 5000000,
  "vat": 35,
  "billingDate": "2026-03-01",
  "recurring": {
    "frequency": "monthly",
    "billingEndDate": "2027-03-01"
  }
}
```

**Response:**

```json
{
  "status": "SUCCESS",
  "data": {
    "chargeId": "chg_72a4d..."
  },
  "message": "Charge created successfully"
}
```

### Create charge request fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `description` | string | Yes | Brief description of the charge |
| `currency` | string | Yes | Three-digit ISO currency code |
| `amount` | number | Yes | Charge amount in smallest currency unit |
| `vat` | number | No | VAT amount |
| `billingDate` | string | No | Effective date (`YYYY-MM-DD`) |
| `recurring.frequency` | string | No | Billing frequency (see table below) |
| `recurring.billingEndDate` | string | No | When to stop recurring charges (`YYYY-MM-DD`) |

### Frequency options

| Value | Cycle |
|-------|-------|
| `daily` | Every day |
| `twiceWeekly` | Twice per week |
| `weekly` | Every week |
| `tenDays` | Every 10 days |
| `fortNightly` | Every 2 weeks |
| `monthly` | Every month |
| `everyTwoMonths` | Every 2 months |
| `trimester` | Every 4 months |
| `quarterly` | Every 3 months |
| `twiceYearly` | Every 6 months |
| `annually` | Every year |
| `unscheduled` | No fixed schedule |

### Fetch a charge by ID

```
GET /partners/:partnerId/merchants/:merchantId/charges/:chargeId
```

The response includes subscription details, VAT, frequency, billing dates, and any associated `subCharges`. Sub-charges are individual billing instances generated from a recurring charge.

**Key response fields:**

| Field | Type | Description |
|-------|------|-------------|
| `isSubscriptionCharge` | boolean | Whether this is a recurring charge |
| `description` | string | Charge description |
| `amount` | number | Charge amount in smallest currency unit |
| `vat` | number | VAT applied |
| `frequency` | string | Billing frequency |
| `billingStartDate` | string | Start date (ISO 8601) |
| `billingEndDate` | string | End date (ISO 8601) |
| `subCharges` | array | Individual billing instances with their own `chargeId`, `amount`, `status`, and `billingDate` |

### Update a charge

Modify an existing charge's amount, VAT, or recurring configuration:

```json
PUT /partners/:partnerId/merchants/:merchantId/charges/:chargeId
{
  "amount": 650000,
  "vat": 15,
  "recurring": {
    "updateType": "onlyNext",
    "billingEndDate": "2027-10-23"
  }
}
```

The `recurring.updateType` field controls the scope of the update:

| Value | Behaviour |
|-------|-----------|
| `onlyNext` | Apply the change only to the next billing cycle |
| `allFuture` | Apply the change to all future billing cycles |

### List all merchant charges

```
GET /partners/:partnerId/merchants/:merchantId/charges
```

Returns a paginated list of all charges (one-time and recurring) for the merchant, including `chargeId`, `description`, `amount`, `vat`, `status`, `billingDate`, and whether the charge is subscription-based.

## Billing Plans

A merchant charge is what a merchant is billed. A billing plan is the pricing behind it: the rates that apply to a card brand, a payment method and a terminal type, broken down by where the card comes from and what kind of card it is. Plans are defined once at partner level and then assigned to merchants.

### Create billing plans

```json
POST /partners/:partnerId/billing-plans
{
  "plans": [
    {
      "id": "SP_STANDARD_CARD",
      "paymentMethod": "CARD",
      "cardBrand": "VISA",
      "terminalType": "STANDARD",
      "planType": "FIXED",
      "description": "Standard card pricing 2026",
      "domesticDebitNonCommercial": 0.6,
      "domesticCreditNonCommercial": 0.9,
      "eeaDebitNonCommercial": 0.8,
      "eeaCreditNonCommercial": 1.1,
      "internationalDebitNonCommercial": 1.9,
      "internationalCreditNonCommercial": 2.3,
      "fixedCost": 30,
      "vatPercentage": 25
    }
  ]
}
```

`plans` is an array, so a full price list goes up in one call.

| Field | Description |
|-------|-------------|
| `id` | Your identifier for the plan. |
| `paymentMethod`, `cardBrand`, `terminalType` | What the plan applies to. One plan per combination. |
| `planType` | `FIXED` for a flat percentage or amount, `VARIABLE` for pricing that depends on transaction type. |
| `domestic*`, `eea*`, `international*` | Percentage rates, split by debit or credit and commercial or non-commercial. |
| `minimumCeiling` | Minimum amount for the rate to apply. |
| `fixedCost` | Fixed cost per transaction, in minor units. |
| `fixedPercentage` | Flat percentage across the board. |
| `vatPercentage` | VAT applied to the plan. |

The twelve rate fields are not padding. Interchange differs by card origin and card type, so a single blended rate either loses money on international commercial cards or overcharges on domestic debit. Price the grid.

### Manage plans

```
GET    /partners/:partnerId/billing-plans
GET    /partners/:partnerId/billing-plans/:id
DELETE /partners/:partnerId/billing-plans/:id
GET    /partners/:partnerId/merchants/:merchantId/plans
```

The last one is the useful one in support: it returns the plans actually assigned to a merchant, which is the answer to "why was I charged this". Plans are attached to a merchant during onboarding through the `transactionPricingPlan` and `displayProducts` control fields — see [Merchant Onboarding](/developers/guides/merchant-onboarding) and [Order and Return Terminals](/developers/guides/terminal-logistics).

## Customer Details

The Customer API lets you create and retrieve customer profiles. Profiles store personal information, addresses, contact details, and linked payment cards, enabling richer order data and streamlined checkout experiences.

### Create a customer

```json
POST /customers
{
  "firstName": "John",
  "middleName": "Doe",
  "birthDate": "1990/03/04",
  "countryCode": "SE",
  "address": [
    {
      "addressLine1": "Storgatan 12",
      "city": "Stockholm",
      "countryCode": "SE",
      "postalCode": "111 23",
      "role": "shipping"
    }
  ],
  "phoneNumbers": [
    {
      "phoneNumber": {
        "code": "46",
        "number": "701234567"
      },
      "role": "own"
    }
  ],
  "emails": [
    {
      "email": "john.doe@example.com",
      "role": "personal"
    }
  ],
  "cardIds": [
    "824c514bfe001805f0"
  ]
}
```

**Response:**

```json
{
  "status": "SUCCESS",
  "data": {
    "customerId": "cust_61e3b..."
  },
  "message": "Customer created successfully"
}
```

### Customer fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `firstName` | string | No | Customer's first name |
| `lastName` | string | No | Customer's last name |
| `birthDate` | string | No | Date of birth (`YYYY/MM/DD`) |
| `countryCode` | string | No | Two-letter ISO country code |
| `address` | array | No | Array of address objects (shipping, billing, etc.) |
| `address.addressLine1` | string | Yes | Primary address line |
| `address.city` | string | Yes | City |
| `address.countryCode` | string | Yes | Two-letter ISO country code |
| `address.postalCode` | string | Yes | Postal code |
| `address.role` | string | No | Address purpose (`shipping`, `billing`) |
| `emails` | array | No | Array of email objects with `email` and `role` |
| `phoneNumbers` | array | No | Array of phone objects with nested `phoneNumber` (`code`, `number`) and `role` |
| `cardIds` | array | No | Payment card identifiers to associate with the customer |

### Fetch a customer

```
GET /customers/:customerId
```

Returns the full customer profile including all addresses, emails, phone numbers, and linked card IDs.

## API Quick Reference

| Operation | Method | Endpoint |
|-----------|--------|----------|
| Fetch settlement reports | GET | `/partners/:partnerId/merchants/:merchantId/reports` |
| Fetch adjustments | GET | `/partners/:partnerId/merchants/:merchantId/adjustments` |
| Create merchant charge | POST | `/partners/:partnerId/merchants/:merchantId/charges` |
| Fetch charge by ID | GET | `/partners/:partnerId/merchants/:merchantId/charges/:chargeId` |
| Update merchant charge | PUT | `/partners/:partnerId/merchants/:merchantId/charges/:chargeId` |
| List all merchant charges | GET | `/partners/:partnerId/merchants/:merchantId/charges` |
| Create billing plans | POST | `/partners/:partnerId/billing-plans` |
| Fetch billing plans | GET | `/partners/:partnerId/billing-plans` |
| Fetch billing plan by ID | GET | `/partners/:partnerId/billing-plans/:id` |
| Remove billing plan | DELETE | `/partners/:partnerId/billing-plans/:id` |
| Fetch a merchant's plans | GET | `/partners/:partnerId/merchants/:merchantId/plans` |
| Create customer | POST | `/customers` |
| Fetch customer by ID | GET | `/customers/:customerId` |
