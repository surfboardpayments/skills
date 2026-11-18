# surfboard-in-store: bundled guides

Generated from https://www.surfboardpayments.com/api/ai/guides.json (49 guides, updated 2026-11-17).
Do not edit these by hand; edit the guide on the marketing site and re-run `npm run skills`.

- [Create an Order](./create-an-order.md): Learn how to create orders with line items, tax, customer details, and control functions. The starting point for accepting payments with the Surfboard API.
- [EMV Terminal Integration](./emv-terminal-integration.md): Integrate traditional card-present terminals through Surfboard's unified API. From account setup to live payments in one guide.
- [CheckoutX SoftPOS](./checkoutx-softpos.md): Accept in-person payments on smartphones and tablets by pairing the CheckoutX app with your own POS app, a dual-app setup that requires no SDK integration.
- [Inter-App Integration](./interapp-integration.md): Integrate your POS app with CheckoutX using native app switch. Register terminals, process payments, and scan NFC tags through a bi-directional deep link flow -- from a native app or a browser-based POS.
- [Customer Identification on Card Tap](./customer-identification.md): Recognise a returning customer from the card they tap, before the payment runs. Apply their discount, points or price list to the open order, then take the money.
- [Tips Configuration](./tips-configuration.md): Configure tipping on Surfboard payment terminals at the merchant, store, or terminal level using a hierarchical override model.
- [Receipts](./receipts.md): Generate, email, print, and customise receipts for in-store transactions using the Surfboard Receipts API.
- [POS Templates](./pos-templates.md): Decide what the till shows and in what order. Lay out product categories, payment methods and terminals per store, and switch layouts automatically by time of day.
- [Partial Payments](./partial-payments.md): Split an order across multiple payment methods or transactions. Accept card, cash, and Swish in any combination to settle a single order.
- [NFC Tag Reading](./nfc-tag-reading.md): Use the NFC Reading API to create tag-reading sessions on payment terminals, scan NFC/RFID-tagged products, and retrieve scanned tag data.
- [Create Order Error Codes](./create-order-error-codes.md): Reference of error codes returned by the create order and initiate payment flows, with notes on when each is thrown and how to handle it.
- [Receipt Printing](./receipt-printing.md): How to print a receipt on a terminal with a built-in printer or on the FinPrinter: let Surfboard render it from a template, or build the layout yourself and send ESC/POS to a terminal. How to choose, and how to set up each.
- [ESC/POS Printing](./escpos-printing.md): Send custom ESC/POS receipts to any Surfboard terminal using the UTF-8 contract: opt in with one field, get validated payloads and consistent output across current and legacy hardware.
- [Product Catalog](./product-catalog.md): Create and manage product catalogs, products, variants, inventory levels, and analytics through the Catalog API.
- [Gift Cards & Promotions](./gift-cards-promotions.md): Issue and manage gift cards, track transactions, and create marketing promotions using the Surfboard Payments APIs.
- [Build Your Own ECR](./electronic-cash-register.md): Turn your ERP or business system into a compliant point of sale for Sweden. Product catalog, fiscal cash register with control unit, Skatteverket declaration, shifts, cash handling, orders, receipts, and Z-reports on the Surfboard platform.
