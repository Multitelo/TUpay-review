# TUpay-review

public function handle(Request $request)
{
    $payload   = $request->getContent();
    $signature = $request->header('x-paystack-signature');

    if (!$this->paystackService->validWebhookSignature($signature, $payload)) {
        return response()->json(['ok' => false, 'message' => 'Invalid webhook signature.'], 401);
    }

    $event = $request->json()->all();

    return match (data_get($event, 'event')) {
        'charge.success' => $this->handleChargeSuccess($event),
        // ...other event types handled similarly
        default => response()->json(['ok' => true, 'message' => 'Event ignored.']),
    };
}

private function handleChargeSuccess(array $event): JsonResponse
{
    $reference = data_get($event, 'data.reference');
    if (!$reference) return response()->json(['ok' => true]);

    DB::transaction(function () use ($reference, $event) {
        $payment = Payment::where('reference', $reference)
            ->lockForUpdate()
            ->first();

        if (!$payment) return;

        if ($payment->status === 'success') {
            // Idempotency guard — Paystack can send this webhook more than once
            return;
        }

        $payment->update([
            'status' => 'success',
            'paid_at' => now(),
            'provider_payload_json' => $event,
        ]);

        $this->subscriptionService->activateFromPlanPrice(
            $payment->clinic, $payment->plan, $payment->planPrice, $payment, $event
        );
    });

    return response()->json(['ok' => true, 'message' => 'Webhook processed.']);
}





This is the Paystack webhook handler for Clinimu's subscription billing. I verify the webhook signature before trusting any payload, wrap the status update and subscription activation in a DB transaction with lockForUpdate() to prevent race conditions, and check status === 'success' first as an idempotency guard since Paystack can send the same webhook more than once. I chose this because it's the piece most likely to break silently — signature bypass, duplicate charges, or partial writes — so getting it right mattered more than almost anything else in the codebase.


<img width="1024" height="768" alt="WhatsApp Image 2026-07-20 at 2 20 17 AM" src="https://github.com/user-attachments/assets/13526b78-c8c2-40fa-8c19-e2c958db5edc" />



