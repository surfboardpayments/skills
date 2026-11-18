---
title: "Build Your Own ECR"
source: https://www.surfboardpayments.com/developers/guides/electronic-cash-register
category: in-store
tags: [In-Store, ECR, Cash Register, POS, Skatteverket, Z-report, Sweden, API]
generated: true
---

# Build Your Own ECR

> Turn your ERP or business system into a compliant point of sale for Sweden. Product catalog, fiscal cash register with control unit, Skatteverket declaration, shifts, cash handling, orders, receipts, and Z-reports on the Surfboard platform.

## Overview

If you sell software to Swedish merchants, you already know the shape of the problem. Card, cash and Swish sales have to land in a certified cash register with a control unit, the register has to be declared to Skatteverket, and every trading day ends with a Z-report. Doing that in your own product means a control unit supplier, a certification cycle, and a fiscal journal you are responsible for from then on.

On Surfboard the register is a platform resource. You create it under a store, the platform pairs it with a control unit, keeps the journal, produces the Z-reports, X-reports and journal memory, and mails them to the merchant. Your side is the POS: the checkout, the product data, the stock and the customer records that already live in your ERP.

This guide is the end-to-end integration: catalog sync, register creation, the Skatteverket declaration, shifts and cash handling, orders and receipts, reports, and the certification steps before a merchant goes live. It assumes you are comfortable with the Surfboard order and terminal APIs; where it leans on them, it links to the guide that covers them.

> **Two meanings of "ECR".** Elsewhere in the Surfboard docs, "ECR" can also mean an external POS that drives a payment terminal over a cable or the API. This guide is about the fiscal cash register: the register record, its control unit, shifts, and reports. The API section for it is the [Electronic Cash Register (ECR) V2 API](https://developers.surfboardpayments.com/api/electronic-cash-register).

### What Surfboard provides, and what you build

| Layer | Surfboard provides | You build |
|-------|-------------------|-----------|
| Payments | Terminals (SurfTouch, SurfPad, SurfPrint, CheckoutX SoftPOS), card, Swish, Klarna, gift cards, refunds | The checkout flow that creates orders |
| Products | Product Catalog API: catalogs, products, variants, barcodes, prices, VAT, inventory | The sync from your ERP into the catalog |
| Fiscal register | Cash register record, paired control unit, register and control-unit details for Skatteverket | The declaration workflow your merchant follows |
| Daily operation | Open and close shift, petty cash, cash withdrawals, End of Day, scheduled close, register state | The cashier screens that call these |
| Receipts | Receipt data on the order, digital receipt link, email, terminal printing, ESC/POS | Where in your flow receipts are offered |
| Reporting | Z-report, X-report, journal memory files per register, SIE bookkeeping files per merchant, portal views | Any push into your own back office |

## How the Pieces Fit

The cash register sits inside the same hierarchy as everything else on the platform:

```
Partner (you)
└── Merchant (your customer, onboarded and KYB-approved)
    └── Store (a physical location)
        ├── Cash registers (one or more per store, each with a cashRegisterId)
        ├── Terminals (registered devices, each with a terminalId)
        └── Product catalog (products, variants, inventory for the store)
```

Cash registers belong to the store, not to a terminal. A store can have several. The link between a register and a terminal is made when a shift is opened on that terminal, and it is removed again when the shift is closed. Between shifts a register has no terminal, and a terminal has no register. This is a change from the first version of the ECR API, where a register was tied permanently to one terminal.

Your POS application can live in either of two places:

- **On the Surfboard terminal.** SurfTouch and SurfPrint run Android. Your app runs on the device beside the payment application and hands off payments with a native app switch. See [Inter-App Integration](/developers/guides/interapp-integration/).
- **On your own hardware.** A tablet, a PC, or your cloud backend creates orders over the API against the terminal, and the terminal takes the payment. See [Create an Order](/developers/guides/create-an-order/).

Either way the cash register is the same record, operated with the same twelve endpoints, and every sale is an order created against the terminal that currently holds an open shift on the register. A phone running CheckoutX SoftPOS is a terminal like any other here: it opens shifts, takes sales and closes shifts on a register exactly as the hardware terminals do.

## Prerequisites

1. A partner account with API credentials from the [integration portal](https://integrate.surfboardpayments.com/), and the base URL, key and secret in configuration as described in [API Conventions](/developers/guides/api-conventions/).
2. ECR enabled for your partner account. [Get Partner Config](https://developers.surfboardpayments.com/api/partners) returns `ecrEnabled`, which states whether the partner has access to the cash-register features. If it is `false`, ask your Surfboard contact to enable it before you start.
3. A Swedish merchant onboarded through [Merchant Onboarding](/developers/guides/merchant-onboarding/), with a store. You need the `merchantId` and `storeId`.
4. A terminal registered to that store, so you have its `terminalId`. See [Device Registration](/developers/guides/device-registration/). You can list a store's terminals with `GET /merchants/:merchantId/stores/:storeId/terminals`.

> **Note:** A charge applies for every cash register created under a merchant. Create the registers a store actually needs, not one per test run.

All ECR endpoints are merchant-scoped and take the standard header set:

```
Content-Type: application/json
API-KEY:      YOUR_API_KEY
API-SECRET:   YOUR_API_SECRET
MERCHANT-ID:  YOUR_MERCHANT_ID
```

## Step 1: Sync Your ERP into the Product Catalog

Your ERP is the source of truth for articles, barcodes, prices and stock. Mirror it into a Surfboard product catalog so the terminal, the merchant portal and the receipts all show the same products, and so sales statistics and inventory are tracked per store.

Create one catalog per store:

```json
POST /catalog
{
  "storeId": "8136a645a2c2d1bb0f"
}
```

```json
// Response
{
  "status": "SUCCESS",
  "data": {
    "catalogId": "8219688f18ebb8020a"
  },
  "message": "Product catalog created successfully"
}
```

Then create each article as a product. This is the request example from the reference, with the fields that matter for a POS sync:

```json
POST /catalog/:catalogId/products
{
  "storeId": "8136a645a2c2d1bb0f",
  "name": "SurfPad Purple Logo",
  "type": "PRODUCT",
  "unitType": "FIXED_UNIT",
  "costPrice": 2000,
  "sellingPrice": 4500,
  "currencyCode": "752",
  "tax": [
    {
      "type": "VAT",
      "percentage": "25"
    }
  ],
  "description": "SurfPad Payment Terminal in Purple",
  "category": "electronics",
  "unit": "nos",
  "productImages": [
    "https://res.cloudinary.com/martinsurf/image/upload/v1619101937/surfboardpayments/surfboard-icon.svg"
  ],
  "hsnCode": "723453",
  "barcode": "7812123454323"
}
```

How the ERP fields map:

| Your ERP holds | Catalog field | Notes |
|----------------|---------------|-------|
| Article name | `name` | Required. |
| Physical good or service | `type` | `PRODUCT` or `SERVICE`. Required. |
| Sold per piece, by weight, or open price | `unitType` | `FIXED_UNIT`, `VARIABLE_UNIT` or `FREE_UNIT`. Required. |
| Sales price | `sellingPrice` | Required. Integer in the smallest unit, so 45.00 SEK is `4500`. Tax-inclusive. |
| Cost price | `costPrice` | Optional, same unit rules. |
| VAT rate | `tax[].type` and `tax[].percentage` | `VAT` with the rate as a string. |
| Barcode / EAN | `barcode` | Scanned at the till, and carried onto the order line as `gtin`. |
| Unit of measure | `unit` | Required. `nos` for pieces, `kg`, `l`, and the other listed units. |
| Product group | `category` or `categoryId` | Free text, or an existing category identifier. |
| Your own article number or anything else | `metadata` | Free-form key/value object for your own use. |
| Sizes, colours | Variants | Add with `POST /catalog/:catalogId/products/:productId/variants`. |

`productImages` is required by the endpoint, so send at least one image URL per product.

Keep stock in step with your ERP with the inventory endpoint. Send a `STOCK_UP` when goods arrive and a `STOCK_DOWN` for shrinkage or manual corrections:

```json
PATCH /catalog/:catalogId/products/:productId/inventory
{
  "storeId": "8136a645a2c2d1bb0f",
  "operation": "STOCK_UP",
  "quantity": 10,
  "unit": "nos"
}
```

The full catalog lifecycle, including variants, related products and statistics, is in [Product Catalog](/developers/guides/product-catalog/).

## Step 2: Create the Cash Registers

A cash register is created under a store. Create as many as the store needs: typically one per till position, each with a name the cashier will recognise when choosing a register at shift open.

```json
POST /merchants/:merchantId/stores/:storeId/cash-register
{
  "cashRegisterName": "Main Register",
  "deviceId": "{{terminalId}}",
  "emails": [
    "store.manager@example.com"
  ]
}
```

| Parameter | Required | Description |
|-----------|----------|-------------|
| `cashRegisterName` | Yes | Name of the register, letters and digits only. Spaces and punctuation are rejected by the control unit provider's validation, so `Kassa1` works and `Kassa 1` does not. This is the name the merchant will see on reports. |
| `deviceId` | Yes | A terminal or device identifier. The reference lists it as required at creation. The terminal that actually operates the register is set when a shift is opened, see Step 4. |
| `emails` | No | Addresses that receive this register's reports, in addition to the merchant's own configured email. |

```json
// Response
{
  "status": "SUCCESS",
  "data": {
    "cashRegister": {
      "cashRegisterId": "{{cashRegisterId}}"
    }
  },
  "message": "Cash register subscribed successfully"
}
```

Store `cashRegisterId` in your system. Every shift, close, and report call is addressed to it, and it is what the cashier chooses from when opening a shift.

Creating the register also pairs it with a control unit. There is no separate control-unit call: the pairing is done by the platform, and the control unit's details come back in the register record in the next step. The control unit is shared by all of a merchant's registers, so a second or third register under the same merchant reuses it.

> **Note:** Report recipients can be added but not removed through the portal or the API. If an address is entered by mistake, write to Surfboard to have it removed.

To verify what exists under a merchant, or to rebuild your mapping after a reinstall, list the registers:

```
GET /merchants/:merchantId/cash-register
```

Each entry carries `cashRegisterId`, `storeId`, `terminalId`, `merchantId` and `cashRegisterName`. Filter on `storeId` to get the registers for one store. `terminalId` is the terminal currently linked by an open shift, and it is `null` while no shift is open on that register. This list is what your POS shows the cashier when they pick a register at shift open.

## Step 3: Declare the Register with Skatteverket

A cash register must be reported to Skatteverket before it is taken into use. The merchant does this in Skatteverket's e-service *Anmäl, ändra, felanmäl och avanmäl kassaregister* (report, change, report faults, and deregister cash registers). Only an authorised representative of the company can file, or a registration agent (*registreringsombud*) the company has appointed with Skatteverket's form. You as the partner cannot file on the merchant's behalf unless you have been appointed that way, so the declaration step belongs in your merchant onboarding flow, not in your backend.

Everything the form asks for about the register and its control unit is in the register record:

```
GET /merchants/:merchantId/stores/:storeId/cash-register/:cashRegisterId
```

```json
// Response
{
  "status": "SUCCESS",
  "data": {
    "cashRegisterId": "{{cashRegisterId}}",
    "cashRegisterName": "Main Register",
    "activationDate": "2026-04-15",
    "closingTime": "23:00",
    "designation": "Register 1",
    "modelOrProgram": "Surfboard ECR 1.0",
    "address": "Main Street 1, Stockholm",
    "email": [
      "store.manager@example.com"
    ],
    "controlUnitAddress": "Main Street 1, Stockholm",
    "controlUnitManufacturer": "Surfboard",
    "controlUnitType": "SOFTWARE",
    "controlUnitModel": "CU-100",
    "controlUnitSerialNumber": "{{controlUnitSerialNumber}}",
    "notifications": [
      {
        "email": "store.manager@example.com",
        "reports": [
          "Z_REPORT"
        ]
      }
    ]
  },
  "message": "Fetched cash register successfully"
}
```

What Skatteverket asks for, and where it comes from:

| Skatteverket asks for | Field in the register record |
|-----------------------|------------------------------|
| Cash register designation (*beteckning*) | `designation` |
| Model or program (*modell eller program*) | `modelOrProgram` |
| Address where the register is used | `address` |
| Control unit manufacturer | `controlUnitManufacturer` |
| Control unit type | `controlUnitType` |
| Control unit model | `controlUnitModel` |
| Control unit serial number (*tillverkningsnummer*) | `controlUnitSerialNumber` |
| Control unit address | `controlUnitAddress` |

The same details are shown on the register's **Info** tab in the partner portal, where a **Declare with Skatteverket** button opens the e-service. See [Cash Registers (ECR)](/partner-portal/cash-registers/) in the partner portal guide.

A practical onboarding flow for your merchants:

1. Create the register as soon as the store exists.
2. Show the merchant the register and control-unit details from the record above, or point them to the Info tab in the portal.
3. Have the merchant file the declaration in the e-service. Skatteverket sends a confirmation and later a registration certificate.
4. Only then open the first shift.

Declare production registers only. A register created in the demo environment carries demo control-unit details, and those are not valid for a Skatteverket declaration.

Changes must be reported to Skatteverket within two weeks. Moving the register to another address, replacing it, or taking it out of service are all changes, so pair every delete in Step 8 with a deregistration.

> **The 2027 deadline.** Skatteverket's regulation SKVFS 2021:17 applies to all cash registers from 1 January 2027, and cloud control systems certified under SKVFS 2020:9 are the alternative to a physical control unit. The register and control unit Surfboard provisions are what you declare; you do not source or certify a control unit yourself. Skatteverket's own pages cover [how to report a cash register](https://www.skatteverket.se/servicelankar/otherlanguages/englishengelska/businessesandemployers/startingandrunningaswedishbusiness/cashregisters/howtoreportacashregister.4.57cadbbd15a3688ff44de5a.html) and the [exemptions from the cash register requirement](https://www.skatteverket.se/servicelankar/otherlanguages/englishengelska/businessesandemployers/startingandrunningaswedishbusiness/cashregisters/exemptionsfromthecashregisterrequirement.4.57cadbbd15a3688ff44deba.html).

## Step 4: Run the Business Day

The ECR API follows the shape of a trading day. Your POS needs four screens for it: open shift, cash in and out, close shift, and end of day. Read the register state before each so the UI only offers what the register will accept.

The terminal enters the picture at shift open. Opening a shift links the terminal the POS is using to the register the cashier chose, and closing the shift removes that link. While the shift is open, the register is addressed either by its `cashRegisterId` or, for the cash calls, by the `terminalId` linked to it.

The rules that follow from that model, and that your POS has to respect:

- **One open shift per register.** Opening a second shift on a register that already has one is rejected.
- **One register per terminal at a time.** A terminal with an open shift cannot open a shift on another register until that shift is closed. The error names the register it is still mapped to.
- **No shift, no sales.** An order created on a terminal with no open shift is rejected.
- **Two tills trading at once need two registers.** A shop counter and a market stall selling at the same time is two terminals and two registers, each with its own shift and its own Z-report.
- **A terminal can move between registers.** Close the shift on one register, open a shift on another. The same phone can run the store register one day and the market register the next.

### Check the register state

```
GET /merchants/:merchantId/stores/:storeId/cash-register/:cashRegisterId/state-info
```

```json
// Response
{
  "status": "SUCCESS",
  "data": {
    "cashRegisterState": "CASH_REGISTER_OPEN",
    "cashRegisterId": "{{cashRegisterId}}",
    "cashRegisterName": "Main Register",
    "currencyCode": "SEK",
    "shiftInfo": {
      "shiftNumber": 1,
      "shiftState": "SHIFT_OPEN",
      "cashierName": "John Doe",
      "shiftOpenedAt": "2026-04-15T12:00:00Z",
      "shiftClosedAt": null
    },
    "pettyCash": 5000,
    "openedAt": "2026-04-15T08:00:00Z",
    "closedAt": null
  },
  "message": "Fetched cash register state successfully"
}
```

`cashRegisterState` is the register's lifecycle state and `shiftInfo.shiftState` the current or most recent shift. `pettyCash` is the cash float currently held. Use this call when the POS starts, after a crash, and before every shift action.

| Field | Values you will see | Meaning |
|-------|---------------------|---------|
| `cashRegisterState` | `CASH_REGISTER_OPEN` | Register is open for the trading day |
| | `CASH_REGISTER_CLOSED` | Register is closed, after End of Day or before its first shift |
| `shiftInfo.shiftState` | `SHIFT_OPEN` | A shift is active and the register accepts sales |
| | `SHIFT_CLOSED` | No active shift |

### Open a shift

A shift must be open before the register accepts payments. Your POS lists the store's registers, the cashier picks one, counts the float into the drawer, and you open the shift on that register with the ID of the terminal the POS is using. From this point the terminal is linked to the register.

```json
PUT /merchants/:merchantId/stores/:storeId/cash-register/:cashRegisterId/open-shift
{
  "terminalId": "{{terminalId}}",
  "pettyCash": 5000,
  "cashierName": "John Doe"
}
```

| Parameter | Required | Description |
|-----------|----------|-------------|
| `terminalId` | Yes | The terminal to link to this register for the shift. |
| `pettyCash` | Yes | Opening cash float for the shift. |
| `cashierName` | No | Name of the cashier. It is also what appears as the cashier on the shift record. |

```json
// Response
{
  "status": "SUCCESS",
  "data": {},
  "message": "Shift opened successfully"
}
```

### Deposit and withdraw cash

Every movement of cash in or out of the drawer that is not a sale goes through these two calls, so the fiscal journal and the drawer agree at close. Both are addressed by the terminal rather than the register: pass the `terminalId` that opened the shift, and the platform resolves the register linked to it.

Adding a float top-up:

```json
POST /merchants/:merchantId/stores/:storeId/cash-register/deposit-petty-cash
{
  "terminalId": "{{terminalId}}",
  "amount": 1000
}
```

```json
// Response
{
  "status": "SUCCESS",
  "data": {
    "cashRegisterId": "{{cashRegisterId}}",
    "newPettyCash": 6000
  },
  "message": "Petty cash inserted successfully"
}
```

Taking cash out, for a bank drop or a paid-out:

```json
PUT /merchants/:merchantId/stores/:storeId/cash-register/withdraw
{
  "amount": 500,
  "terminalId": "{{terminalId}}"
}
```

```json
// Response
{
  "status": "SUCCESS",
  "message": "Cash withdrawn successfully"
}
```

Both require an active shift.

### Close a shift

When the cashier goes off duty, close the shift. This also de-links the terminal from the register, so the terminal is free to open a shift on another register, and the register is free to be opened from another terminal. The response returns the shift number and the petty-cash balance at close, which is the figure to reconcile against the counted drawer:

```
PUT /merchants/:merchantId/stores/:storeId/cash-register/:cashRegisterId/close-shift
```

```json
// Response
{
  "status": "SUCCESS",
  "data": {
    "cashRegisterId": "{{cashRegisterId}}",
    "shiftNumber": 1,
    "pettyCash": 5000
  },
  "message": "Shift closed successfully"
}
```

A register can run several shifts in one day; open the next one with a fresh float, from the same terminal or a different one.

### End of Day

End of Day closes the register for the trading day and generates the Z-report. It runs automatically at the register's scheduled close time, or you trigger it from the POS when the store closes:

```
PUT /merchants/:merchantId/stores/:storeId/cash-register/:cashRegisterId/end-of-day
```

```json
// Response
{
  "status": "SUCCESS",
  "message": "Cash register closed successfully"
}
```

> **Warning:** End of Day is a one-time action and cannot be undone. Once it has been called, the register cannot accept payments for the rest of the day, and it cannot be reopened until the next day. Put a confirmation step in front of it, and never wire it to an automatic "logout".

The automatic close runs at the scheduled close time whether or not anyone called End of Day, and whether or not a shift is still open. From the next day the register is available again, and the first Open-Shift of the day starts the new trading day. A register that was closed by End of Day during the day stays closed for that day.

### Set the scheduled close time

Every register has an automatic End of Day. The reference documents the default as 00:00 local time. A store that trades past midnight, or one that wants its Z-report cut at closing time, sets its own:

```json
PATCH /merchants/:merchantId/stores/:storeId/cash-register/:cashRegisterId/closing-time
{
  "closingTime": "23:00"
}
```

```json
// Response
{
  "status": "SUCCESS",
  "data": {
    "cashRegisterId": "{{cashRegisterId}}",
    "newClosingTime": "23:00",
    "shiftCloseType": "CUSTOM",
    "scheduledClosingTime": "2026-04-16T23:00:00Z"
  },
  "message": "Cash register closing time updated successfully"
}
```

Sending a closing time sets the register's shift close type to `CUSTOM`. The current setting is also returned as `closingTime` on the register record.

### The day, end to end

| Moment | Call | What the POS shows |
|--------|------|--------------------|
| POS starts | Fetch Cash Registers, then Get Cash Register State | The store's registers, each with its status and float |
| Store opens | Open-Shift on the chosen register with this terminal's ID and the counted float | Terminal linked to the register, sales enabled |
| Float top-up or bank drop | Deposit Petty Cash / Withdraw Cash | Updated float |
| Cashier change | Close-Shift, then Open-Shift | Shift summary to reconcile, terminal de-linked and re-linked |
| Store closes | End of Day, or wait for the scheduled close | Z-report on its way to the report recipients |

## Step 5: Ring Up Sales

A sale on your POS is an order created against the terminal that has an open shift on the register. The order lines come straight from your catalog sync, and the order is flagged for the cash register with `controlFunctions.ecrEnabled`. The reference lists `ecrEnabled` as "whether ECR (electronic cash register) mode is enabled" on the order.

The example below follows the Create Order reference exactly. Two units of the same article, tax-inclusive prices, the barcode carried as `gtin`, your ERP line reference as `externalItemId`, and the cashier's name passed through to the receipt:

```json
POST /orders
{
  "terminal$id": "{{terminalId}}",
  "referenceId": "POS-2026-000123",
  "orderLines": [
    {
      "id": "1",
      "externalItemId": "ART-10442",
      "name": "SurfPad Purple Logo",
      "quantity": 2,
      "gtin": "7812123454323",
      "amount": {
        "regular": 4500,
        "total": 4500,
        "currency": "752",
        "tax": [
          {
            "amount": 900,
            "percentage": 25,
            "type": "VAT"
          }
        ]
      }
    }
  ],
  "totalOrderAmount": {
    "regular": 9000,
    "total": 9000,
    "currency": "752",
    "tax": [
      {
        "amount": 1800,
        "percentage": 25,
        "type": "VAT"
      }
    ]
  },
  "controlFunctions": {
    "ecrEnabled": true,
    "receipt": {
      "cashierName": "John Doe"
    },
    "initiatePaymentsOptions": {
      "paymentMethod": "CARD"
    }
  }
}
```

The response returns `orderId` and `paymentId`. Store both against your POS receipt so refunds and receipt lookups can find them later.

Three rules from [Create an Order](/developers/guides/create-an-order/) that matter more on a till than anywhere else:

- **`amount.total` is per unit, not per line.** The line above is two units at 45.00 SEK. `total` stays `4500`; only `totalOrderAmount` carries the `9000`.
- **Prices are tax-inclusive.** The `tax` array states how much VAT is inside the price, never an amount to add on top. Sweden's 25, 12 and 6 percent rates each go on their own lines with their own `tax` entries.
- **Currencies are numeric ISO 4217 codes as strings.** SEK is `"752"`.

Fields under `controlFunctions.receipt` are printed on the receipt. Besides `cashierName` you can send `autoPrintReceipt` and `autoSendReceiptIfEmailAvailable`. If you prefer to add receipt details after the payment, use `PUT /orders/:orderId/receipt` as described in [Receipts](/developers/guides/receipts/).

> **The receipt number is the register's, not yours.** On an order that goes through a Surfboard register, the receipt number shown on the receipt is generated by the register in line with the regulations, and the series is per cash register: two registers give two series. A `sequenceNumber` you send is stored on the order for your own reference but does not change the printed number. The `cashRegisterName` and `controlUnitSerialNumber` receipt fields are for the bring-your-own-control-unit path in Step 6, not for Surfboard registers. Keep the `orderId` on your side; it is printed on every Surfboard receipt and is the key to find a sale when a customer comes back with one.

### Returns and refunds

A return is an order with negative quantities. Set `purchaseOrderId` on each returned line to the `orderId` of the original sale; the reference marks it as mandatory for a return item. The full flow, including refunding to the original payment, is in [Refund an Order](/developers/guides/refund-an-order/) and [Partial Refund](/developers/guides/partial-refund/).

### Cash sales

Cash is a payment method like any other; a cash sale is still an order on the register, so it lands in the fiscal journal and on the Z-report. The [Partial Payments](/developers/guides/partial-payments/) guide shows a second payment taken with the `CASH` method. Cash movements that are not sales, such as floats and bank drops, are the petty-cash calls in Step 4.

### Split and mixed payments

Card plus gift card, or card plus cash, are handled with partial payments on the same order. See [Partial Payments](/developers/guides/partial-payments/) and [Gift Cards & Promotions](/developers/guides/gift-cards-promotions/).

## Step 6: Receipts

Once the payment completes, the receipt options are the same as for any Surfboard payment:

- **Digital receipt link.** `GET /receipts/{id}/link` returns a hosted receipt URL to show as a QR code or send in an SMS.
- **Email.** `PUT /receipts/{id}/email` sends the receipt to the customer.
- **Print on the terminal.** `POST /receipts/{id}/print` prints from a Surfboard template on a terminal with a built-in printer or on a FinPrinter. See [Receipt Printing](/developers/guides/receipt-printing/).
- **Your own layout.** `PUT /receipts/{terminalId}/escpos` prints a receipt you have rendered yourself. See [ESC/POS Printing](/developers/guides/escpos-printing/).

All of them are covered in [Receipts](/developers/guides/receipts/).

> **Already have a certified control unit?** If your existing POS keeps its own control unit and you only use Surfboard for payments and receipts, you can attach that unit's data to the receipt instead of creating a Surfboard register. `POST /receipts/:id` in the [Receipts API](https://developers.surfboardpayments.com/api/receipts) takes `receiptNumber`, `cashRegisterName`, `controlUnitSerial` and `cashierName`. That path is for partners who bring their own fiscal setup; everything else in this guide assumes Surfboard's.

## Step 7: Reports and Bookkeeping

Every End of Day produces a Z-report for the register, and the register also keeps X-reports and journal memory. They are emailed to the merchant's configured address and to every address in the register's notification list, and they can be fetched by API for your own back office:

```
GET /merchants/:merchantId/stores/:storeId/cash-register/:cashRegisterId/files?reportType=value&startDate=value&endDate=value
```

All three query parameters are optional and each is applied only when present. `reportType` filters to one report type, `startDate` and `endDate` bound the period.

```json
// Response
{
  "status": "SUCCESS",
  "data": [
    {
      "fileName": "z-report-2026-04-15.pdf",
      "fileType": "application/pdf",
      "reportType": "Z_REPORT",
      "sequence": 1,
      "fileUrl": "https://files.example.com/z-report-2026-04-15.pdf"
    }
  ],
  "message": "Cash register files fetched successfully."
}
```

`sequence` is the file's ordinal within its report type, so a gap in the Z-report sequence in your archive means a day you have not fetched yet. The partner portal offers the same files under the register's **Reports** tab, filtered by *Z report*, *X report* or *Journal Memory* and a date range.

For accounting, the merchant can also receive **SIE files**. These are generated per merchant, not per register, and sent for every payout Surfboard makes to the merchant. They are switched on with **Enable bookkeeping** on the merchant's Cash-Register tab in the partner portal. Settlement and payout reporting for the same money is in [Settlements & Reporting](/developers/guides/settlements-reporting/).

## Step 8: Decommission a Register

When a till is retired or a store closes, delete the register:

```
DELETE /merchants/:merchantId/stores/:storeId/cash-register/:cashRegisterId
```

```json
// Response
{
  "status": "SUCCESS",
  "message": "Cash register unsubscribed successfully."
}
```

> **Warning:** Deleting a register cannot be undone. Fetch and archive its files first, and remind the merchant to deregister it in Skatteverket's e-service within two weeks.

## Releasing Your ECR

The build order that gets a first merchant live with the least rework:

1. **Build in demo.** Demo credentials come with your developer account. Create a test merchant and store, register a terminal, and run the full day: create register, open shift, sales, cash movements, close shift, End of Day, fetch the Z-report. Demo terminals run in payment page mode and take test cards only, so exercise the register flow there and the terminal hardware once you have live credentials. See [API Conventions](/developers/guides/api-conventions/) for the environments.
2. **Wire the catalog sync.** Decide what triggers a product update in your ERP and push it to the catalog on that event. A nightly full sync plus event-driven price changes is the usual shape.
3. **Put the state check first.** Every register screen in your POS starts with Fetch Cash Registers for the store and Get Cash Register State for the chosen one. It is what keeps a cashier from opening a shift on a register that is already open elsewhere, or calling End of Day twice.
4. **Make End of Day deliberate.** Confirmation dialog, and a visible note that it cannot be undone. Set the scheduled close time per store so a forgotten End of Day still happens.
5. **Listen for payment events.** Subscribe to order and payment webhooks so the POS learns the outcome of a payment without polling. See [Webhooks](/developers/guides/webhooks-notifications/).
6. **Register your POS with the control unit provider.** Before the first production register can be created, Surfboard registers your POS software with the control unit provider. Send your Surfboard contact:
   - Company name and organisation number. The company name cannot be changed once the application is submitted, so use the entity that will own the product.
   - A unique cash register software identifier in reverse-domain form, 1 to 64 characters, for example `se.yourcompany`.
   - An application name that includes `SE`, for example `YourPOS SE 1.0.0`. Regulation requires the market code in the name. It is visible on internal dashboards only, not on receipts.
   - A point of contact: name, email and phone number.
7. **Self-certify.** Surfboard sends a test-case sheet covering the register flow and the payment flow. You run each case and fill in the IDs it asks for, and walk the full flow with Surfboard on a call. Two things are hard requirements from day one: support for every Swedish VAT rate including 12 percent, even if the merchant you start with never uses it, and a receipt for every sale. Paperless is fine: attach the digital receipt link as evidence and note that you do not print. Receipt copies are handled by Surfboard's receipt product, so your POS does not track copy limits or the "KOPIA" marking.
8. **Go live.** Live credentials and a new base URL are issued after certification. Keep host, key and secret in configuration so the switch is a config change.
9. **Roll out per merchant.** For each merchant: onboard, create the store, register the terminals, create the store's registers, hand the merchant each register's control-unit details, wait for the Skatteverket declarations, then open the first shift from a terminal.

## API Quick Reference

| Operation | Method | Endpoint |
|-----------|--------|----------|
| Create cash register | POST | `/merchants/:merchantId/stores/:storeId/cash-register` |
| Fetch cash registers | GET | `/merchants/:merchantId/cash-register` |
| Get cash register by ID | GET | `/merchants/:merchantId/stores/:storeId/cash-register/:cashRegisterId` |
| Get cash register state | GET | `/merchants/:merchantId/stores/:storeId/cash-register/:cashRegisterId/state-info` |
| Open shift | PUT | `/merchants/:merchantId/stores/:storeId/cash-register/:cashRegisterId/open-shift` |
| Close shift | PUT | `/merchants/:merchantId/stores/:storeId/cash-register/:cashRegisterId/close-shift` |
| End of day | PUT | `/merchants/:merchantId/stores/:storeId/cash-register/:cashRegisterId/end-of-day` |
| Deposit petty cash | POST | `/merchants/:merchantId/stores/:storeId/cash-register/deposit-petty-cash` |
| Withdraw cash | PUT | `/merchants/:merchantId/stores/:storeId/cash-register/withdraw` |
| Update scheduled close time | PATCH | `/merchants/:merchantId/stores/:storeId/cash-register/:cashRegisterId/closing-time` |
| Get cash register files | GET | `/merchants/:merchantId/stores/:storeId/cash-register/:cashRegisterId/files` |
| Delete cash register | DELETE | `/merchants/:merchantId/stores/:storeId/cash-register/:cashRegisterId` |
| Create order on the register | POST | `/orders` with `controlFunctions.ecrEnabled` |
| Add receipt information | PUT | `/orders/:orderId/receipt` |

## Reference

- [Electronic Cash Register (ECR) V2 API](https://developers.surfboardpayments.com/api/electronic-cash-register)
- [Orders API](https://developers.surfboardpayments.com/api/orders)
- [Receipts API](https://developers.surfboardpayments.com/api/receipts)
- [Cash Registers (ECR) in the partner portal](/partner-portal/cash-registers/)
- [Product Catalog](/developers/guides/product-catalog/)
- [Create an Order](/developers/guides/create-an-order/)
- [Receipts](/developers/guides/receipts/)
- [Device Registration](/developers/guides/device-registration/)
- [Merchant Onboarding](/developers/guides/merchant-onboarding/)
- [Skatteverket: How to report a cash register](https://www.skatteverket.se/servicelankar/otherlanguages/englishengelska/businessesandemployers/startingandrunningaswedishbusiness/cashregisters/howtoreportacashregister.4.57cadbbd15a3688ff44de5a.html)
