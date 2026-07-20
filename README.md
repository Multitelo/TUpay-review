# TUpay-review

This is the Paystack webhook handler for Clinimu's subscription billing. I verify the webhook signature before trusting any payload, wrap the status update and subscription activation in a DB transaction with lockForUpdate() to prevent race conditions, and check status === 'success' first as an idempotency guard since Paystack can send the same webhook more than once. I chose this because it's the piece most likely to break silently — signature bypass, duplicate charges, or partial writes — so getting it right mattered more than almost anything else in the codebase.
