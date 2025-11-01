# Laravel PayMob Integration

This project integrates a Laravel application with PayMob — a secure online payment gateway that enables merchants to accept payments via credit/debit cards and mobile wallets.

Contents
- Overview
- Requirements
- Installation
- Environment / PayMob credentials
- Quick usage example (create order and open IFrame)
- Handling callbacks / webhooks
- Testing (Sandbox)
- Common issues & troubleshooting
- Contributing & contact
- License

---

## Overview
This repository provides a simple integration pattern for PayMob in a Laravel app to:
- Create payment orders on PayMob
- Obtain a payment key (payment_token)
- Redirect or show PayMob payment IFrame
- Handle payment notifications and update local order status

---

## Requirements
- PHP >= 8.0
- Laravel 8, 9 or 10 (should work with modern versions)
- Composer
- A PayMob account (Sandbox or Live)

---

## Installation (quick)
1. Clone the repository or use it as an example:
   ```bash
   git clone https://github.com/Hatem-Mohammed-toma/laravel-paymob-integration.git
   cd laravel-paymob-integration
   composer install
   ```
2. Copy .env.example to .env and adjust values:
   ```bash
   cp .env.example .env
   ```
3. Run migrations if there are any:
   ```bash
   php artisan migrate
   ```
4. Start local server:
   ```bash
   php artisan serve
   ```

If you want to extract this into a reusable package, move the relevant files into a package structure and publish config/routes as needed.

---

## Environment / PayMob configuration
Add the following to your `.env` (replace with your credentials):

```
PAYMOB_API_KEY=your_paymob_api_key
PAYMOB_INTEGRATION_ID=your_integration_id
PAYMOB_IFRAME_ID=your_iframe_id
PAYMOB_MERCHANT_ID=your_merchant_id   # if required
PAYMOB_API_HOST=https://accept.paymob.com
PAYMOB_MODE=sandbox   # or "live"
```

You can also create a config file `config/paymob.php` to read these values (recommended).

---

## Payment flow (quick example)
1. Create a local Order record.
2. Create an order on PayMob and get the PayMob `order_id`.
3. Request a payment key (payment_token / payment_key) using the PayMob order_id and your integration_id.
4. Redirect the user to the PayMob IFrame URL:

IFrame payment URL format:
```
https://accept.paymob.com/api/acceptance/iframes/{IFRAME_ID}?payment_token={PAYMENT_TOKEN}
```

Example Controller (illustrative):

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Http;
use App\Models\Order;

class PaymobController extends Controller
{
    public function pay(Order $order)
    {
        // 1. Create order in PayMob (get PayMob order id)
        $authResponse = Http::post(config('paymob.api_host') . '/api/auth/tokens', [
            'api_key' => config('paymob.api_key'),
        ]);
        $auth = $authResponse->json();

        $orderResponse = Http::withToken($auth['token'])->post(config('paymob.api_host') . '/api/ecommerce/orders', [
            'amount_cents' => $order->amount * 100,
            'currency' => 'EGP',
            'merchant_order_id' => $order->id,
            // additional fields...
        ]);
        $paymobOrder = $orderResponse->json();

        // 2. Request payment key
        $paymentKeyResponse = Http::withToken($auth['token'])->post(config('paymob.api_host') . '/api/acceptance/payment_keys', [
            'amount_cents' => $order->amount * 100,
            'expiration' => 3600,
            'order_id' => $paymobOrder['id'],
            'integration_id' => config('paymob.integration_id'),
            'billing_data' => [
                // customer billing data
            ],
        ]);

        $paymentKey = $paymentKeyResponse->json()['token'];

        // 3. Redirect to IFrame
        $iframeId = config('paymob.iframe_id');
        return redirect("https://accept.paymob.com/api/acceptance/iframes/{$iframeId}?payment_token={$paymentKey}");
    }

    public function notify(Request $request)
    {
        // Handle PayMob webhook / callback data here
        // Verify payload and update local order status accordingly
    }
}
```

Notes:
- Some PayMob endpoints require obtaining an auth token first (POST /api/auth/tokens) using your API key.
- Ensure amounts are in cents (amount * 100) and currency matches your merchant account.

---

## Handling callbacks / webhooks
- Create a route to receive PayMob notifications (POST).
- Verify the payload according to PayMob's docs (check status fields or signature if provided).
- Update the order status in your database (paid, failed, void, etc.) and perform post-payment actions (emails, fulfillment).

Example route:
```php
Route::post('/paymob/notify', [\App\Http\Controllers\PaymobController::class, 'notify']);
```

For local testing, use a tunnel (ngrok) so PayMob can reach your local endpoint.

---

## Testing (Sandbox)
- Use PayMob sandbox mode and sandbox credentials.
- Use PayMob's test card numbers and test flows to validate integration.
- Never use sandbox keys in production.

---

## Common issues & troubleshooting
- 401 Unauthorized: Verify PAYMOB_API_KEY and that you use it in the correct endpoint (some calls require auth token).
- 400 Bad Request: Check that amount_cents is amount * 100 and that currency is correct.
- Webhook not received: Ensure your webhook endpoint is publicly reachable (use ngrok for local development).
- Missing or invalid integration_id / iframe_id: confirm values from PayMob dashboard.

---

## Contributing
Contributions are welcome. Open an issue or submit a pull request with a clear description and any tests or steps to reproduce. Keep changes focused and document new behavior.

---

## License
This project is licensed under the MIT License. See the LICENSE file for details.

---

## Author / Contact
Author: Hatem-Mohammed-toma  
Repository: https://github.com/Hatem-Mohammed-toma/laravel-paymob-integration
