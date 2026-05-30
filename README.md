# Pixel Verification API — Developer Documentation

> **Base URL**: `https://pixel-verification-api.likhonmain.workers.dev`
>
> **Version**: 1.1
> 
> **Owner**: `t.me/Tanvir124`
> 
> **FOR API Key, Contact OWNER TANVIR**
---

## Table of Contents

1. [Introduction](#introduction)
2. [Quick Start](#quick-start)
3. [Authentication](#authentication)
4. [Endpoints](#endpoints)
   - [Health Check](#1-health-check)
   - [Submit Order](#2-submit-order)
   - [Check Order Status](#3-check-order-status)
   - [Check Credits](#4-check-credits)
5. [Order Lifecycle](#order-lifecycle)
6. [Credit System](#credit-system)
7. [Error Codes](#error-codes)
8. [Rate Limiting](#rate-limiting)
9. [Code Examples](#code-examples)
   - [Python](#python)
   - [Node.js](#nodejs)
   - [cURL](#curl)
   - [Complete Bot Integration (Python)](#complete-bot-integration-python)
10. [Webhook Notifications (Coming Soon)](#webhook-notifications)
11. [Updating Your Integration](#updating-your-integration)
12. [FAQ](#faq)

---

## Introduction

The Pixel Verification API allows you to integrate Google account verification automation into your own bots and services. Submit an order with account credentials, and our system will process the verification and return the result link.

**How it works:**

```
Your Bot → POST order to API → Our system processes it → Your bot polls for result
```

Each API key comes with a **credit balance**. Each order costs **1 credit**. Contact the admin to purchase additional credits.

---

## Quick Start

**3 steps to get started:**

1. **Get your API key** from the admin (format: `pvsk_xxxxxxxxxxxx...`)
2. **Submit an order:**
   ```bash
   curl -X POST https://pixel-verification-api.likhonmain.workers.dev/api/order \
     -H "Content-Type: application/json" \
     -H "X-API-Key: YOUR_API_KEY" \
     -d '{"email": "user@gmail.com", "password": "pass123", "totp_secret": "JBSWY3DPEHPK3PXP"}'
   ```
3. **Poll for the result:**
   ```bash
   curl https://pixel-verification-api.likhonmain.workers.dev/api/order/ORDER_ID \
     -H "X-API-Key: YOUR_API_KEY"
   ```

That's it! The order will be processed automatically and you'll receive the verification link in the response.

---

## Authentication

All endpoints (except `/api/health`) require an API key.

**Send your key in the `X-API-Key` header:**

```
X-API-Key: pvsk_a3b4c5d6e7f8g9h0j1k2l3m4n5o6p7q8
```

| Scenario | HTTP Status | Response |
|----------|-------------|----------|
| Missing API key | `401` | `{ "error": "Missing X-API-Key header" }` |
| Invalid or revoked key | `401` | `{ "error": "Invalid or inactive API key" }` |
| No credits remaining | `402` | `{ "error": "Insufficient credits", "credits_remaining": 0 }` |

---

## Endpoints

### 1. Health Check

Check if the API is online. **No authentication required.**

```
GET /api/health
```

**Response** `200 OK`:
```json
{
  "status": "online",
  "timestamp": 1717020000000
}
```

---

### 2. Submit Order

Submit a new verification order. Requires **1 credit** (refunded automatically if the order fails).

```
POST /api/order
```

**Headers:**
```
Content-Type: application/json
X-API-Key: YOUR_API_KEY
```

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | string | ✅ | Gmail address to verify |
| `password` | string | ✅ | Account password |
| `totp_secret` | string | ✅ | 2FA TOTP secret key (base32 encoded) |

**Example Request:**
```json
{
  "email": "john.doe@gmail.com",
  "password": "MySecurePassword123",
  "totp_secret": "JBSWY3DPEHPK3PXP"
}
```

**Success Response** `201 Created`:
```json
{
  "success": true,
  "order_id": "e7c3a1f8-9d4b-4e2a-8f1c-6b5a3d2e1f0c",
  "credits_remaining": 49,
  "message": "Order queued for processing"
}
```

**Error Responses:**

| Status | Condition | Response |
|--------|-----------|----------|
| `400` | Missing/invalid fields | `{ "error": "Missing required field: email" }` |
| `401` | Invalid API key | `{ "error": "Invalid or inactive API key" }` |
| `402` | No credits left | `{ "error": "Insufficient credits", "credits_remaining": 0 }` |
| `429` | Rate limited | `{ "error": "Rate limit exceeded", "retry_after_seconds": 45 }` |

> **Important**: Save the `order_id` from the response — you'll need it to check the order status.

---

### 3. Check Order Status

Check the status and result of an existing order. You can only check your own orders.

```
GET /api/order/{order_id}
```

**Headers:**
```
X-API-Key: YOUR_API_KEY
```

**Success Response** `200 OK`:
```json
{
  "success": true,
  "order": {
    "order_id": "e7c3a1f8-9d4b-4e2a-8f1c-6b5a3d2e1f0c",
    "email": "john.doe@gmail.com",
    "status": "success",
    "flow_type": "normal",
    "created_at": 1717020000000,
    "started_at": 1717020005000,
    "completed_at": 1717020180000,
    "result": {
      "link": "https://one.google.com/offer/xxxxxxxx-xxxx",
      "link_type": "offer",
      "retries_used": 1
    },
    "error": null,
    "progress": {
      "current_step": 8,
      "total_steps": 8,
      "step_label": "Process complete",
      "status": "success"
    },
    "credit_refunded": false
  }
}
```

**Order Status Values:**

| Status | Meaning | What to do |
|--------|---------|------------|
| `pending` | Order is in queue, waiting to be processed | Keep polling (every 10-15 seconds) |
| `processing` | Order is being actively processed on a device | Keep polling — check `progress` for step updates |
| `success` | ✅ Order completed — result link is available | Read `result.link` — **you're done!** |
| `failed` | ❌ Order failed — credit auto-refunded | Read `error.code` and `error.user_message` |

**When status is `success`:**
```json
"result": {
  "link": "https://one.google.com/offer/xxxxxxxx-xxxx",
  "link_type": "offer",
  "retries_used": 1
}
```

**When status is `failed`:**
```json
"error": {
  "code": "WRONG_PASSWORD",
  "message": "The password entered is incorrect",
  "step": 3,
  "user_message": "Please double-check the password and try again"
},
"credit_refunded": true
```

> When an order fails, the `credit_refunded` field will be `true`, indicating your credit has been automatically returned. No action needed on your end.

**When status is `processing`** (live progress):
```json
"progress": {
  "current_step": 5,
  "total_steps": 8,
  "step_label": "Entering 2FA code",
  "status": "start"
}
```

**Error Responses:**

| Status | Condition | Response |
|--------|-----------|----------|
| `404` | Order not found | `{ "error": "Order not found" }` |
| `403` | Order belongs to different API key | `{ "error": "Access denied" }` |

> **Security**: The password and TOTP secret are **never** returned in status responses. Only the email, status, result, and error details are visible.

---

### 4. Check Credits

Check your remaining credit balance and account information.

```
GET /api/credits
```

**Headers:**
```
X-API-Key: YOUR_API_KEY
```

**Success Response** `200 OK`:
```json
{
  "success": true,
  "credits_remaining": 47,
  "rate_limit": 20,
  "total_orders": 53,
  "name": "Bot Developer X"
}
```

| Field | Description |
|-------|-------------|
| `credits_remaining` | Number of orders you can still submit |
| `rate_limit` | Maximum orders per minute allowed |
| `total_orders` | Total orders ever submitted with this key |
| `name` | Your API key label |

---

## Order Lifecycle

```
                   POST /api/order
                      (1 credit deducted)
                         │
                         ▼
                    ┌─────────┐
                    │ PENDING  │  ← Order is in the queue
                    └────┬────┘
                         │  (our system picks it up)
                         ▼
                   ┌──────────┐
                   │PROCESSING│  ← Automation is running on device
                   └─────┬────┘
                         │
                    ┌────┴────┐
                    ▼         ▼
              ┌─────────┐ ┌────────┐
              │ SUCCESS  │ │ FAILED │
              └─────────┘ └────────┘
               result.link   error.code
               is available  credit auto-refunded
```

**Typical processing time**: 2-5 minutes per order.

**Recommended polling strategy:**
- Poll every **10 seconds** while status is `pending`
- Poll every **5 seconds** while status is `processing`
- Stop polling once status is `success` or `failed`

---

## Credit System

Credits are the currency of the API. Here's how they work:

| Event | Credit Impact |
|-------|--------------|
| Order submitted (`POST /api/order`) | **-1 credit** deducted |
| Order succeeds (`status: "success"`) | No change — credit was consumed |
| Order fails (`status: "failed"`) | **+1 credit** auto-refunded |
| Rate limit hit (`429`) | No deduction |
| Invalid request (`400`) | No deduction |
| Auth failure (`401`) | No deduction |

**Key points:**
- **You are only charged for successful orders.** If an order fails for any reason (wrong password, wrong 2FA, device issues, automation errors), your credit is automatically refunded.
- Credits are deducted upfront when the order is created, and refunded automatically when the system detects a failure.
- The `credit_refunded` field in the order status response confirms the refund happened.
- Contact the admin to purchase additional credits at any time.

---

## Error Codes

When an order fails, the `error.code` field tells you exactly what went wrong. **All failed orders automatically refund the credit used.**

| Error Code | Meaning | Recommended Action |
|-----------|---------|-------------------|
| `WRONG_PASSWORD` | The account password is incorrect | Ask user to verify their password |
| `WRONG_2FA_CODE` | The TOTP secret/2FA key is invalid | Ask user to re-check their 2FA setup key |
| `2FA_WARNING` | 2FA is not set up correctly | Ask user to follow the 2FA setup guide |
| `PAYMENT_WARNING` | Payment profile issue detected | User needs to close payment profile first |
| `BROWSER_NOT_SECURE` | Google flagged the browser as insecure | Try with a different Gmail account |
| `CAPTCHA_DETECTED` | Captcha appeared during sign-in | User should manually check the account |
| `DEVICE_OFFLINE` | Processing device went offline | Retry the order later |
| `AUTOMATION_CRASH` | Unexpected crash during processing | Retry the order |
| `OTHER_ERROR` | General automation failure | Retry the order — if persistent, contact admin |

> **Tip**: Always check `error.user_message` for a human-friendly description you can show to your users.

---

## Rate Limiting

Each API key has a configurable rate limit (default: **20 orders per minute**).

When you exceed the limit, you'll receive:

```
HTTP 429 Too Many Requests
```

```json
{
  "error": "Rate limit exceeded",
  "retry_after_seconds": 45
}
```

**What this means**: Wait `retry_after_seconds` before submitting new orders.

> **Note**: Rate limiting only applies to `POST /api/order` (creating new orders). Checking order status (`GET /api/order/:id`) and checking credits (`GET /api/credits`) are **not** rate limited.
>
> **Note**: Hitting the rate limit does NOT deduct credits. You are only charged when an order is successfully created.

---

## Code Examples

### Python

```python
import requests
import time

API_KEY = "pvsk_your_api_key_here"
BASE_URL = "https://pixel-verification-api.likhonmain.workers.dev"

headers = {
    "Content-Type": "application/json",
    "X-API-Key": API_KEY
}

# 1. Submit an order
response = requests.post(f"{BASE_URL}/api/order", json={
    "email": "user@gmail.com",
    "password": "password123",
    "totp_secret": "JBSWY3DPEHPK3PXP"
}, headers=headers)

data = response.json()

if response.status_code == 201:
    order_id = data["order_id"]
    print(f"✅ Order created: {order_id}")
    print(f"   Credits remaining: {data['credits_remaining']}")
else:
    print(f"❌ Error: {data['error']}")
    exit()

# 2. Poll for result
print("⏳ Waiting for result...")
while True:
    status_resp = requests.get(
        f"{BASE_URL}/api/order/{order_id}",
        headers=headers
    )
    order = status_resp.json()["order"]

    if order["status"] == "success":
        print(f"✅ Done! Link: {order['result']['link']}")
        break
    elif order["status"] == "failed":
        print(f"❌ Failed: {order['error']['code']}")
        print(f"   Reason: {order['error']['user_message']}")
        if order.get("credit_refunded"):
            print("   💰 Credit has been refunded automatically.")
        break
    elif order["status"] == "processing":
        progress = order.get("progress") or {}
        step = progress.get("current_step", "?")
        total = progress.get("total_steps", "?")
        label = progress.get("step_label", "")
        print(f"   ⚙️ Step {step}/{total}: {label}")
        time.sleep(5)
    else:
        # pending
        time.sleep(10)
```

### Node.js

```javascript
const API_KEY = "pvsk_your_api_key_here";
const BASE_URL = "https://pixel-verification-api.likhonmain.workers.dev";

const headers = {
  "Content-Type": "application/json",
  "X-API-Key": API_KEY,
};

async function verifyAccount(email, password, totpSecret) {
  // 1. Submit order
  const createRes = await fetch(`${BASE_URL}/api/order`, {
    method: "POST",
    headers,
    body: JSON.stringify({ email, password, totp_secret: totpSecret }),
  });
  const createData = await createRes.json();

  if (!createData.success) {
    throw new Error(`Order failed: ${createData.error}`);
  }

  const orderId = createData.order_id;
  console.log(`✅ Order created: ${orderId}`);
  console.log(`   Credits remaining: ${createData.credits_remaining}`);

  // 2. Poll for result
  while (true) {
    const statusRes = await fetch(`${BASE_URL}/api/order/${orderId}`, {
      headers,
    });
    const statusData = await statusRes.json();
    const order = statusData.order;

    if (order.status === "success") {
      console.log(`✅ Done! Link: ${order.result.link}`);
      return order.result;
    }

    if (order.status === "failed") {
      console.log(`❌ Failed: ${order.error.code}`);
      if (order.credit_refunded) {
        console.log("   💰 Credit refunded automatically.");
      }
      throw new Error(order.error.user_message || order.error.message);
    }

    if (order.status === "processing" && order.progress) {
      const { current_step, total_steps, step_label } = order.progress;
      console.log(`   ⚙️ Step ${current_step}/${total_steps}: ${step_label}`);
    }

    // Wait before next poll
    const delay = order.status === "processing" ? 5000 : 10000;
    await new Promise((resolve) => setTimeout(resolve, delay));
  }
}

// Usage
verifyAccount("user@gmail.com", "password123", "JBSWY3DPEHPK3PXP")
  .then((result) => console.log("Verification link:", result.link))
  .catch((err) => console.error("Error:", err.message));
```

### cURL

**Submit an order:**
```bash
curl -X POST https://pixel-verification-api.likhonmain.workers.dev/api/order \
  -H "Content-Type: application/json" \
  -H "X-API-Key: pvsk_your_api_key_here" \
  -d '{
    "email": "user@gmail.com",
    "password": "password123",
    "totp_secret": "JBSWY3DPEHPK3PXP"
  }'
```

**Check order status:**
```bash
curl https://pixel-verification-api.likhonmain.workers.dev/api/order/ORDER_ID_HERE \
  -H "X-API-Key: pvsk_your_api_key_here"
```

**Check credits:**
```bash
curl https://pixel-verification-api.likhonmain.workers.dev/api/credits \
  -H "X-API-Key: pvsk_your_api_key_here"
```

**Health check (no auth needed):**
```bash
curl https://pixel-verification-api.likhonmain.workers.dev/api/health
```

### Complete Bot Integration (Python)

Here's a full example of how a Telegram bot might integrate with the API:

```python
"""
Example: Telegram Bot Integration with Pixel Verification API

This shows how a third-party bot developer would use
the API to offer verification services to their own users.
"""

import requests
import time
import threading

API_KEY = "pvsk_your_api_key_here"
BASE_URL = "https://pixel-verification-api.likhonmain.workers.dev"

HEADERS = {
    "Content-Type": "application/json",
    "X-API-Key": API_KEY
}


def check_credits():
    """Check how many credits are remaining."""
    r = requests.get(f"{BASE_URL}/api/credits", headers=HEADERS)
    return r.json()


def submit_verification(email, password, totp_secret):
    """Submit a verification order. Returns order_id or raises error."""
    r = requests.post(f"{BASE_URL}/api/order", json={
        "email": email,
        "password": password,
        "totp_secret": totp_secret
    }, headers=HEADERS)

    data = r.json()

    if r.status_code == 201:
        return data["order_id"], data["credits_remaining"]

    elif r.status_code == 402:
        raise Exception("No credits remaining! Contact admin to top up.")

    elif r.status_code == 429:
        wait = data.get("retry_after_seconds", 60)
        raise Exception(f"Rate limited. Try again in {wait} seconds.")

    else:
        raise Exception(f"API Error: {data.get('error', 'Unknown error')}")


def wait_for_result(order_id, on_progress=None, timeout=600):
    """
    Poll for order result. Blocks until complete or timeout.

    Args:
        order_id: The order ID from submit_verification()
        on_progress: Optional callback(step, total, label) for progress updates
        timeout: Max seconds to wait (default 10 minutes)

    Returns:
        dict with 'link' and 'link_type' on success

    Raises:
        Exception on failure or timeout
    """
    start = time.time()

    while time.time() - start < timeout:
        r = requests.get(f"{BASE_URL}/api/order/{order_id}", headers=HEADERS)
        data = r.json()

        if not data.get("success"):
            raise Exception(f"API Error: {data.get('error', 'Unknown')}")

        order = data["order"]
        status = order["status"]

        if status == "success":
            return order["result"]

        elif status == "failed":
            error = order.get("error", {})
            code = error.get("code", "UNKNOWN")
            msg = error.get("user_message", error.get("message", "Unknown error"))
            # Credit is auto-refunded — no need to handle refunds manually
            raise Exception(f"Order failed [{code}]: {msg}")

        elif status == "processing" and on_progress and order.get("progress"):
            p = order["progress"]
            on_progress(
                p.get("current_step", 0),
                p.get("total_steps", 0),
                p.get("step_label", "")
            )

        # Adaptive polling: faster when processing, slower when pending
        time.sleep(5 if status == "processing" else 10)

    raise Exception(f"Timeout: order {order_id} not completed in {timeout}s")


# ─── Example usage in a bot handler ─────────────────────────────

def handle_user_verification_request(email, password, totp_secret, reply_fn):
    """
    Called when a user in YOUR bot requests verification.
    reply_fn(text) sends a message back to the user.
    """

    # Check credits first
    credit_info = check_credits()
    if credit_info.get("credits_remaining", 0) < 1:
        reply_fn("⚠️ Service temporarily unavailable. Please try later.")
        return

    # Submit order
    try:
        order_id, credits_left = submit_verification(email, password, totp_secret)
        reply_fn(f"✅ Order submitted! Processing your request...\n"
                 f"Order ID: {order_id}")
    except Exception as e:
        reply_fn(f"❌ Could not submit order: {e}")
        return

    # Poll for result (run in background thread to not block bot)
    def poll():
        def on_progress(step, total, label):
            reply_fn(f"⚙️ Step {step}/{total}: {label}")

        try:
            result = wait_for_result(order_id, on_progress=on_progress)
            reply_fn(f"✅ Verification complete!\n"
                     f"🔗 Link: {result['link']}")
        except Exception as e:
            reply_fn(f"❌ Verification failed: {e}\n"
                     f"💰 Your credit has been refunded automatically.")

    threading.Thread(target=poll, daemon=True).start()
```

---

## Webhook Notifications

> 🚧 **Coming Soon** — Webhook support is planned for a future update.

When available, you'll be able to provide a callback URL when setting up your API key. We'll send a `POST` request to your URL when an order completes:

```json
{
  "event": "order.completed",
  "order_id": "e7c3a1f8-...",
  "status": "success",
  "result": {
    "link": "https://one.google.com/offer/...",
    "link_type": "offer"
  }
}
```

This will eliminate the need for polling. Stay tuned!

---

## Updating Your Integration

The API may receive updates to improve reliability, add features, or fix issues. Here's how to stay up to date:

### Versioning Policy

- The API uses URL-based versioning. The current version is **v1.1**.
- Breaking changes will only be introduced in new major versions.
- Non-breaking additions (new response fields, new endpoints) may be added at any time.

### What Developers Should Check After Updates

If you're notified of an API update, here's a quick checklist:

1. **Check the health endpoint** — Make sure the API is online:
   ```bash
   curl https://pixel-verification-api.likhonmain.workers.dev/api/health
   ```

2. **Check your credits** — Verify your balance is correct:
   ```bash
   curl https://pixel-verification-api.likhonmain.workers.dev/api/credits \
     -H "X-API-Key: YOUR_API_KEY"
   ```

3. **Test with a real order** — Submit a test order and verify the full flow works end-to-end.

4. **Check for new response fields** — New fields may be added to responses. Your code should handle unknown fields gracefully (ignore them, don't crash).

### Common Developer Mistakes

| Mistake | Fix |
|---------|-----|
| Hardcoding the base URL | Store it in a config variable |
| Not handling `429` rate limits | Add retry logic with `retry_after_seconds` |
| Polling too frequently | Use adaptive polling (10s for pending, 5s for processing) |
| Not saving `order_id` after submit | Always store the order ID — it's your only way to check the result |
| Ignoring `error.user_message` | Display it to your users — it's human-readable and helpful |
| Polling after `success` or `failed` | Stop polling once you get a terminal status |

### Example: Updating the Cloudflare Worker

If you're self-hosting the Cloudflare Worker, here's how to update it:

1. Download the latest `cloudflare-worker.js` from this repository
2. In your Cloudflare dashboard, go to **Workers & Pages → your worker**
3. Click **Edit Code** and replace the contents with the new file
4. Click **Deploy**
5. Verify with a health check: `GET /api/health`

---

## FAQ

### How long does an order take to process?
Typically **2-5 minutes**. Times may vary based on queue depth and network conditions.

### What happens if my order fails?
Your order is marked as `failed` with an error code explaining why. **Your credit is automatically refunded** — you are only charged for successful orders. Check the `error.code` and `error.user_message` fields for details on what went wrong.

### Can I cancel a pending order?
Not currently. Once submitted, orders will be processed in queue order.

### How do I get more credits?
Contact the admin to purchase additional credits. Credits can be added to your API key at any time.

### Is there a sandbox/test environment?
Not currently. All orders are processed on real devices. Use the `/api/health` and `/api/credits` endpoints for integration testing without consuming credits.

### What's the maximum queue size?
There is no hard limit on pending orders. However, each order takes 2-5 minutes to process, so a queue of 50 orders would take approximately 2-4 hours to complete.

### Do you support bulk orders?
Yes — just submit multiple orders via `POST /api/order`. They'll be processed sequentially in order of submission. Respect your rate limit (default: 20/minute).

### What about credential security?
Your credentials are transmitted over HTTPS (encrypted in transit) and stored only for the duration of processing. We recommend using dedicated/test accounts when possible.

### Are credits refunded for failed orders?
**Yes, always.** Credits are automatically refunded for any failed order, regardless of the failure reason (wrong password, wrong 2FA, device offline, etc.). You are only charged for orders that complete successfully.

---

## Support

For API key requests, credit purchases, or technical support:
- **Telegram**: t.me/Tanvir124
- **Include**: Your API key name (not the key itself) and order IDs for any issues

---

*Last updated: May 30, 2026 — API v1.1*
