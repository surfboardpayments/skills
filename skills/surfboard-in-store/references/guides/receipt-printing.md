---
title: "Receipt Printing"
source: https://www.surfboardpayments.com/developers/guides/receipt-printing
category: in-store
tags: [In-Store, API, Receipts, Printing, Templates, ESC/POS, FinPrinter, Cash Drawer]
generated: true
---

# Receipt Printing

> How to print a receipt on a terminal with a built-in printer or on the FinPrinter: let Surfboard render it from a template, or build the layout yourself and send ESC/POS to a terminal. How to choose, and how to set up each.

## Overview

Surfboard prints on two kinds of device: [terminals with a built-in printer](/products/?requirements=printer), such as SurfPrint Pro, and the [FinPrinter](/products/fins/finprinter/), the standalone cloud printer for a counter, a pickup point or a kitchen. There are two ways to get a receipt onto paper, and you pick whichever fits your POS better. Both paths work on both kinds of device. On the FinPrinter, the ESC/POS path is sent without the `codePages` opt-in; see [ESC/POS on the FinPrinter](#escpos-on-the-finprinter).

| | Printing templates | ESC/POS |
|---|---|---|
| Who renders the receipt | Surfboard | You |
| What you send | An ID: `paymentId`, `transactionId` or `orderId` | The complete ESC/POS byte stream |
| Layout control | Add or remove pre-defined receipt elements | Full control, line by line |
| Where the layout lives | Partner Portal, at partner, merchant or store level | Your code |
| Requirement | `orderLines` on the order | UTF-8 text inside the [ESC/POS contract](/developers/guides/escpos-printing/) |
| Fonts | Handled by the template | One font, with bold and size variations |
| Maintenance | Surfboard | You |
| Prints more than receipts | No | Yes: kitchen slips, bongs, pickup tickets |

If you already send `orderLines` on every order and want a receipt that looks right without owning its layout, use templates. If you need a layout Surfboard does not offer, or you want to print things that are not receipts at all, use ESC/POS. The two are not exclusive: many integrations print the customer receipt from a template and a kitchen slip from ESC/POS on the same device.

> **Note:** This guide is about paper. For email receipts, hosted receipt links and the fiscal receipt fields, see the [Receipts](/developers/guides/receipts/) guide.

## Prerequisites

- A Surfboard developer account with valid API credentials (`API-KEY` and `API-SECRET`)
- A [terminal with a built-in printer](/products/?requirements=printer) or a [FinPrinter](/products/fins/finprinter/), registered to a merchant and store, and its `terminalId`
- For templates: orders created with `orderLines`, and access to the Partner Portal

## The printers

Both kinds of device are registered the same way and get a `terminalId`, so the template print endpoint does not care which one it is talking to. The device must be online for a job to print; a `SUCCESS` response means the job was dispatched, not that paper came out.

| Device | Printer | Paper | Typical placement |
|---|---|---|---|
| [Terminals with a built-in printer](/products/?requirements=printer) | Built in | 58 mm | Handheld: prints at the table, on the floor or at the counter |
| [FinPrinter](/products/fins/finprinter/) | Standalone, cloud-connected | 80 mm | Counter, pickup point or kitchen, with no terminal beside it |

**Terminals with a built-in printer** print where the payment happens. The terminal that took the payment is the default target of the print call, so a receipt on the device the customer just tapped needs no `terminalId` at all. The current line-up is on the [products page](/products/?requirements=printer), filtered to devices with a printer.

**FinPrinter** has no screen and takes no payments. It connects over Wi-Fi or Ethernet, and you register it with the same device registration call as a terminal, using the serial number on the back of the unit as the registration identifier. See [Device Registration](/developers/guides/device-registration/). Once registered it shows up as a device next to the merchant's terminals in the portals and the API, and both print endpoints accept its `terminalId`. It also has a drawer port, so it can [open a cash drawer](#open-a-cash-drawer). Because every job routes through Surfboard's cloud, your POS needs no driver and no cable to the printer.

Templates adapt to the paper width on their own, so the same template prints correctly on both devices.

## Path 1: Printing templates

On this path you never render a receipt. You describe what the receipt should contain by choosing from Surfboard's pre-defined receipt fields, Surfboard lays it out, and at print time you send an ID. Surfboard maps that ID to the order, fills in the template and prints.

### Set up the template

Templates are configured in the Partner Portal. A template is a list of the receipt elements you want, built from the pre-defined fields: merchant details, the order lines with quantities and prices, VAT breakdown, the payment details from the card scheme, footer text and so on. Add and remove elements to fit what you and the merchant need.

A template can be set at three levels:

| Level | Use it for |
|---|---|
| Partner | The default for every merchant you board |
| Merchant | A merchant whose receipt should differ from your default |
| Store | One location that needs its own variant, for example a different footer or fiscal line |

The most specific level wins, so the usual setup is one partner-level default and overrides only where a merchant or store asks for one. If you would rather set the merchant default from code, the merchant configuration endpoint takes a `terminalReceiptTemplateId`:

```
PATCH /merchants/{merchantId}/configuration
```

```json
{
  "terminalReceiptTemplateId": "<template id from the Partner Portal>"
}
```

### Send order lines

The one requirement on this path is that the receipt data comes from the order, so `orderLines` must be present on the [create order](/developers/guides/create-an-order/) call. An order created without lines has nothing for the template to render. If your POS already sends lines, which it does if the terminal shows a basket, there is no extra work beyond setting up the template.

The fiscal fields that Nordic regulators expect on a receipt, such as the register name and control unit serial, are stored against the order separately with `PUT /receipts/{orderId}`. That call is covered in [Receipts](/developers/guides/receipts/), and the template picks the fields up when they are present.

### Print

```
POST /receipts/{id}/print
```

The `{id}` is a `paymentId`, `transactionId` or `orderId`, whichever your POS has to hand. Surfboard resolves it to the order and its receipt data.

**Request body:**

```json
{
  "terminalId": "trm_abc123",
  "templateType": "TRANSACTION_RECEIPT"
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `terminalId` | string | No | The device to print on. Defaults to the terminal that processed the transaction. Set it to send the receipt to a FinPrinter at the counter, or to a different terminal than the one that took the payment. |
| `templateType` | string | No | `TRANSACTION_RECEIPT` (default) prints the customer receipt. `SALES_SUMMARY` prints a summary instead. |

**Response:**

```json
{
  "status": "SUCCESS",
  "message": "Receipt sent to the printer successfully."
}
```

Which template renders is decided by the configuration in the Partner Portal, resolved from the store up through the merchant to the partner. Nothing in the print call names a template, which is what lets you change a merchant's receipt without touching the POS.

> **Note:** The older `PUT /receipts/{id}/print` form, which took a `templateId` and a `language` in the body, still works but is superseded by the `POST` above. New integrations should use `POST`.

For a register built on Surfboard, the order itself can carry `autoPrintReceipt` under `controlFunctions.receipt`, so the receipt prints when the payment completes with no print call at all. See [Build Your Own ECR](/developers/guides/electronic-cash-register/).

## Path 2: ESC/POS

On this path you build the whole receipt: every line, every column, every bold header. You send the finished ESC/POS byte stream to a specific terminal with a built-in printer, and Surfboard validates it and passes it through. This is the path when you already have a receipt renderer in your POS, when the templates cannot express the layout you need, or when the thing you are printing is not a receipt.

```
PUT /receipts/{terminalId}/escpos
```

Note that the path takes a `terminalId`, not a payment or order ID. Nothing ties the job to an order, which is what makes it usable for kitchen slips and pickup tickets.

> **Note:** The ESC/POS contract described here applies to terminals with a built-in printer. It does not apply to the FinPrinter, which takes ESC/POS on the same endpoint without `codePages`. See [ESC/POS on the FinPrinter](#escpos-on-the-finprinter).

**Request body:**

```json
{
  "escposCommands": "<base64 ESC/POS>",
  "codePages": "UTF-8"
}
```

Three things to know before you commit to this path:

- **One font.** The terminal printers support a single typeface, with bold, a condensed variant and two larger sizes. If a design calls for anything else, it has to be sent as an image, and images are capped at 384 dots wide.
- **The contract is enforced.** With `"codePages": "UTF-8"` set, the stream is validated before it reaches hardware. Text must be UTF-8, only a fixed set of commands is accepted, and an invalid stream is rejected whole with `PR_0006` rather than printed halfway. The full rules, a worked receipt and a preflight validator are in [ESC/POS Printing](/developers/guides/escpos-printing/).
- **Formatting and maintenance sit with you.** Line widths differ by text size, starting from 32 columns of normal text on a 58 mm terminal, so every line has to be fitted by your code. When a merchant wants a different footer, that is a change in your code, not in a portal.

The trade is the same one a template makes in the other direction: you own the layout, so you own the work of keeping it right.

## ESC/POS on the FinPrinter

The [FinPrinter](/products/fins/finprinter/) takes ESC/POS on the same endpoint, addressed by its own `terminalId`:

```
PUT /receipts/{terminalId}/escpos
```

**Request body:**

```json
{
  "escposCommands": "<base64 ESC/POS>"
}
```

Leave `codePages` out. The `"UTF-8"` opt-in is only for payment terminals with a built-in printer: with it set, the API prepares the stream for those printers and adds a command the FinPrinter is not meant to receive. The validated contract, its command list and its line widths in [ESC/POS Printing](/developers/guides/escpos-printing/) do not apply here.

As on every ESC/POS job, the decoded stream must start with the ESC byte (`0x1B`), so begin it with `ESC @` (`1B 40`).

### Open a cash drawer

The FinPrinter has a drawer port, so a cash drawer at the counter opens from the same API as the receipt:

```
Cash drawer  →  RJ12 cable  →  FinPrinter drawer port
```

1. Connect the cash drawer to the FinPrinter's drawer port with an RJ12 cable.
2. Add the drawer-kick commands to the ESC/POS job you send to the FinPrinter.
3. The FinPrinter prints the job and sends a 24 V pulse down the RJ12 cable. The drawer opens.

The drawer-kick command is `ESC p`. Drawers are wired to either pin 2 or pin 5 of the port, so send both pulses:

| Command | Bytes (hex) | Meaning |
|---|---|---|
| `ESC p 0 25 250` | `1B 70 00 19 FA` | Pulse drawer pin 2: on for 25 units, off for 250 |
| `ESC p 1 25 250` | `1B 70 01 19 FA` | Pulse drawer pin 5: on for 25 units, off for 250 |

A drawer opens on the pulse for its own pin and ignores the other, so the pair opens any 24 V drawer. Pin 2 alone leaves a drawer wired to pin 5 shut while the receipt prints normally.

To open the drawer as part of a receipt, append the ten bytes to the end of the receipt stream:

```python
import base64

ESC, GS = 0x1B, 0x1D
out = bytearray()
out.extend(bytes([ESC, 0x40]))                    # init
out.extend("Kvitto\n".encode("utf-8"))
out.extend(bytes([ESC, 0x64, 0x04]))              # feed past the tear bar
out.extend(bytes([GS, 0x56, 0x00]))               # cut
out.extend(bytes([ESC, 0x70, 0x00, 0x19, 0xFA]))  # open the cash drawer: pin 2
out.extend(bytes([ESC, 0x70, 0x01, 0x19, 0xFA]))  # and pin 5

print(base64.b64encode(bytes(out)).decode())
```

**Request body:**

```json
{
  "escposCommands": "G0BLdml0dG8KG2QEHVYAG3AAGfobcAEZ+g=="
}
```

To open the drawer without printing, for a float count or a no-sale, send the two commands on their own. Their Base64 is `G3AAGfobcAEZ+g==`:

```json
{
  "escposCommands": "G3AAGfobcAEZ+g=="
}
```

> **Note:** Do not send `"codePages": "UTF-8"` on these calls. It is meant for payment terminals with a built-in printer, not the FinPrinter.

## Choosing, in practice

- **A POS that already sends `orderLines` and has no receipt renderer.** Templates. Set one at partner level and you are done.
- **A POS with its own receipt engine, migrating to Surfboard.** ESC/POS, which lets you keep the receipts your merchants already know. Read the contract first; generic ESC/POS libraries often emit raster images and charset commands it rejects.
- **Customer receipt plus kitchen slip.** Both. Templates for the receipt on the terminal that took the payment, ESC/POS for the bong on a terminal with a built-in printer in the kitchen.
- **A receipt that must carry something the template fields do not have.** Check the available elements in the Partner Portal first; if the field is not there, ESC/POS.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `SUCCESS` but nothing printed | Device offline, or busy with another command | Check the device in the Merchant Portal and retry |
| Template receipt is empty or missing lines | Order was created without `orderLines` | Send lines on the create order call; they cannot be added after the fact |
| Receipt prints on the wrong device | `terminalId` omitted, so it defaulted to the terminal that took the payment | Pass the `terminalId` of the printer you want |
| Template changes do not show | A more specific level overrides yours | Check for a merchant or store template overriding the partner default |
| `PR_0006` on the ESC/POS path | A command outside the contract, or non-UTF-8 text | The message names the byte offset. See the [ESC/POS Printing](/developers/guides/escpos-printing/) error table |
| Cash drawer does not open, but the receipt prints | Drawer wired to the other pin, cable in the wrong port, drawer-kick bytes missing, or `codePages` sent | Send both pulses (`1B 70 00 19 FA 1B 70 01 19 FA`), use the drawer port, and leave `codePages` out for the FinPrinter. See [Open a cash drawer](#open-a-cash-drawer) |

## API Quick Reference

| Operation | Method | Endpoint |
|-----------|--------|----------|
| Print from a template | POST | `/receipts/{id}/print` |
| Print a custom ESC/POS job | PUT | `/receipts/{terminalId}/escpos` |
| Open a cash drawer on a FinPrinter | PUT | `/receipts/{terminalId}/escpos` |
| Set a merchant's default template | PATCH | `/merchants/{merchantId}/configuration` |
| Register a FinPrinter | POST | `/merchants/{merchantId}/stores/{storeId}/devices` |

For the full endpoint reference, see the [Receipts API](https://developers.surfboardpayments.com/api/receipts) documentation. For the ESC/POS contract itself, see [ESC/POS Printing](/developers/guides/escpos-printing/); for email, links and fiscal receipt data, see [Receipts](/developers/guides/receipts/).
