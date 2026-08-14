# surfboard-testing: bundled guides

Generated from https://www.surfboardpayments.com/api/ai/guides.json (44 guides, updated 2026-08-13).
Do not edit these by hand; edit the guide on the marketing site and re-run `npm run skills`.

- [Payment Lifecycle](./payment-lifecycle.md): Manage the full payment lifecycle from order creation through capture, void, cancel, and refund operations using the Surfboard Payments API.
- [Create Order Error Codes](./create-order-error-codes.md): Reference of error codes returned by the create order and initiate payment flows, with notes on when each is thrown and how to handle it.
- [Capture a Payment](./capture-a-payment.md): Finalize a previously authorized payment by capturing funds. Covers delay capture and pre-authorization flows with step-by-step API examples.
- [Cancel a Payment](./cancel-a-payment.md): Stop an in-progress payment before it completes. Use cancellation when a customer abandons checkout or a payment needs to be halted mid-process.
- [Void a Payment](./void-a-payment.md): Reverse a completed payment before settlement. Voiding stops funds from transferring to the merchant's account, avoiding incorrect transactions.
- [Refund an Order](./refund-an-order.md): Process a full refund by creating a return order with negative quantities. Covers the complete refund flow with API examples and payment method requirements.
- [Partial Refund](./partial-refund.md): Refund specific items or a reduced amount from a completed order. Process partial returns by creating a return order with only the items to be refunded.
