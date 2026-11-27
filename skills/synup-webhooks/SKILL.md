---
name: synup-webhooks
description: Use when building, testing or debugging a server endpoint that receives Synup webhooks. Covers checking the Synup-Signature header (HMAC-SHA256 over "t.rawBody", with a five-minute limit), the event envelope and payloads such as review.received, listing.sync.failed and post.published, retries and duplicates, rotating the signing secret, and setting up the endpoint under Developer → Webhooks.
metadata:
  openclaw:
    homepage: https://synup.com/developers/webhooks/
    envVars:
      - name: SYNUP_WEBHOOK_SECRET
        required: false
        description: The endpoint's signing secret (whsec_...), used by the verifier examples. The skill itself needs none.
      - name: SYNUP_WEBHOOK_SECRET_OLD
        required: false
        description: Optional second secret the examples also accept while switching over after a rotation.
---

# Synup webhooks

## Set up

- In Synup, open **Developer → Webhooks**. A workspace has one endpoint: an HTTPS URL that accepts `POST` and returns a 2xx status within 10 seconds.
- On creation, the endpoint is subscribed to every live event and gets a signing secret starting with `whsec_`. The secret is shown only once. Put it in your server's environment, for example `SYNUP_WEBHOOK_SECRET`, and never commit it.
- Saving sends a signed test event (`ping.sent`), and **Send test event** sends another whenever you need one. Pick events on the Events tab; only live events can be selected.
- Webhooks need a Synup plan that includes the Developer API. Deliveries stop if the workspace loses it.

## What arrives

A `POST` with a JSON body:

```json
{
  "id": "msg_...",
  "type": "review.received",
  "created_at": "2026-11-26T09:30:00.000Z",
  "api_version": "2026-06-01",
  "data": { "review_id": "...", "client_id": "...", "platform": "google", "rating": 2, "author": "...", "review_date": "...", "client_location_id": "...", "body": "..." }
}
```

Headers:

- `Synup-Signature`: `t=<unix seconds>,v1=<hex>`, with a second `v1` for 7 days after the secret is rotated. See below.
- `Synup-Delivery`: the same value as `id`. Use it to drop duplicates.
- `Synup-Event`: the same value as `type`, so you can route before parsing.

`api_version` is fixed for an endpoint when it's created.

## Check every request

1. Read the raw body bytes before any JSON parser or body middleware touches them. Re-serialized JSON won't match.
2. Split `Synup-Signature` on commas into `t` and one or more `v1` values. Accept the request if any `v1` matches.
3. Reject the request if `t` is more than 300 seconds from now.
4. Compute HMAC-SHA256 over `<t>.<raw body>`, using the whole secret string (including `whsec_`) as the key. Hex-encode it and compare it with `v1` in constant time.
5. On a mismatch, return `401`; Synup retries non-2xx responses. Otherwise return 2xx at once and do the work in a background job.

Node (Express):

```ts
import crypto from "node:crypto";
import express from "express";

// The second secret is optional (see Rotating the secret below).
const SECRETS = [process.env.SYNUP_WEBHOOK_SECRET, process.env.SYNUP_WEBHOOK_SECRET_OLD].filter(
  (s): s is string => Boolean(s),
);

export function verifySynupSignature(header: string | undefined, rawBody: Buffer, toleranceSeconds = 300): boolean {
  if (!header) return false;
  let t = 0;
  const signatures: Buffer[] = [];
  for (const part of header.split(",")) {
    const [key, value] = part.split("=");
    if (key === "t") t = Number(value);
    if (key === "v1" && value) signatures.push(Buffer.from(value, "hex"));
  }
  if (!t || signatures.length === 0) return false;
  if (Math.abs(Date.now() / 1000 - t) > toleranceSeconds) return false;
  const signed = Buffer.concat([Buffer.from(`${t}.`), rawBody]);
  return SECRETS.some((secret) => {
    const expected = crypto.createHmac("sha256", secret).update(signed).digest();
    return signatures.some((given) => given.length === expected.length && crypto.timingSafeEqual(given, expected));
  });
}

const app = express();
app.post("/webhooks/synup", express.raw({ type: "application/json" }), (req, res) => {
  if (!verifySynupSignature(req.header("Synup-Signature"), req.body)) return res.status(401).send("invalid signature");
  const event = JSON.parse(req.body.toString("utf8"));
  res.sendStatus(200);
  // Drop the event if event.id was seen before, then hand it to a background job.
});
```

Python (Flask):

```python
import hashlib, hmac, os, time
from flask import Flask, abort, request

SECRETS = [s for s in (os.environ.get("SYNUP_WEBHOOK_SECRET"), os.environ.get("SYNUP_WEBHOOK_SECRET_OLD")) if s]

def verify_synup_signature(header, raw_body, tolerance=300):
    if not header:
        return False
    t, signatures = 0, []
    for part in header.split(","):
        key, _, value = part.partition("=")
        if key == "t" and value.isdigit():
            t = int(value)
        elif key == "v1" and value:
            signatures.append(value)
    if not t or not signatures or abs(time.time() - t) > tolerance:
        return False
    signed = f"{t}.".encode() + raw_body
    for secret in SECRETS:
        expected = hmac.new(secret.encode(), signed, hashlib.sha256).hexdigest()
        if any(hmac.compare_digest(expected, s) for s in signatures):
            return True
    return False

app = Flask(__name__)

@app.post("/webhooks/synup")
def synup_webhook():
    if not verify_synup_signature(request.headers.get("Synup-Signature"), request.get_data()):
        abort(401)
    event = request.get_json()
    # Drop the event if event["id"] was seen before, then hand it to a background job.
    return "", 200
```

To unit-test a verifier, use Synup's published values (switch the five-minute check off for this test): secret `whsec_test_secret_key`, `t` `1700000000`, and body

```
{"id":"evt_test123","type":"review.received","created_at":"2023-11-14T22:13:20Z","api_version":"2026-06-01","data":{"review_id":"rev_test","rating":5}}
```

must give `v1=6204bc359e5eabdf2060a37634581c0276a9c99221d54a6063cb4373f43b2b04`.

## Events

`data` fields for common events:

| Event | `data` fields |
| --- | --- |
| `review.received` | `review_id`, `client_id`, `platform`, `rating`, `author`, `review_date`, `client_location_id`, `body` |
| `review.reply.posted` | `review_id`, `client_id`, `platform`, `client_location_id` |
| `review.reply.failed` | `review_id`, `client_id`, `platform`, `client_location_id`, `reason` |
| `listing.sync.completed` | `client_location_id`, `listing_id`, `site`, `live_link`, `listing_site_id` |
| `listing.sync.failed` | `client_location_id`, `listing_id`, `site`, `reason` |
| `post.published` | `post_id`, `platform`, `action`, `platform_post_id`, `post_link` |
| `post.failed` | `post_id`, `platform`, `action`, `reason` |
| `location.updated` | `location_id`, `client_id`, `changed_fields`, `new_values` |
| `connection.reconnect_needed` | `platform`, `provider_account_id`, `client_id`, `error_tag` |
| `review_invite.failed` | `campaign_id`, `error_reason` |

- `client_location_id` and `location_id` both hold the Synup location id, the same `id` that `GET /locations` returns.
- Treat any field as possibly `null`, and ignore fields you don't recognize, since payloads can gain fields.
- Payloads can be thin, so look up anything else you need through the API. `review_id` and `client_id` are the same ids the API uses; `client_id` is `null` when Synup doesn't know the client.

All events you can subscribe to today:

- **Clients:** `client.created`, `client.updated`, `client.archived`
- **Locations:** `location.created`, `location.updated`, `location.archived`
- **Listings:** `listing.sync.completed`, `listing.sync.failed`, `listing.submission.rejected`
- **Media:** `media.uploaded`, `media.processing.completed`, `media.processing.failed`
- **Reviews:** `review.received`, `review.reply.posted`, `review.reply.failed`
- **Review invites:** `review_invite.sent`, `review_invite.failed`
- **Posts:** `post.published`, `post.failed`, `post.scheduled.due`, `boost.status.changed`
- **Distribution campaigns:** `distribution.package.imported`, `distribution.approval.recorded`, `distribution.variant.scheduled`, `distribution.variant.unscheduled`, `distribution.variant.update_available`, `distribution.variant.blocked`, `distribution.variant.published`, `distribution.variant.failed`, `distribution.handoff.ready`, `distribution.campaign.paused`, `distribution.campaign.resumed`
- **Social inbox:** `inbox.item.received`, `inbox.item.replied`
- **Post ideas:** `post_idea.generated`, `post_idea.approved`
- **SEO:** `seo.rank_report.completed`, `seo.grid_scan.completed`
- **AI search visibility (AEO):** `aeo.report.completed`
- **Connections:** `connection.connected`, `connection.disconnected`, `connection.reconnect_needed`
- **Custom fields:** `custom_field.created`, `custom_field.updated`, `custom_field.deleted`
- **Team:** `team_member.invited`, `team_member.joined`, `team_member.removed`
- **Sydekick, Synup's assistant:** `agent.message.completed`, `agent.message.failed`, `agent.approval.requested`, `agent.approval.decided`

Some events are listed in Synup but not sent yet, so they can't be selected: `listing.duplicate.detected`, `review.flagged`, `review_invite.completed`, `inbox.message.received`, `inbox.mention.received`, `seo.keyword.rank_changed`, `aeo.visibility_alert`, `connection.wrong_account_detected`. Don't build features that depend on them.

## Delivery and retries

- A delivery fails on a non-2xx status, a network error, or no answer within 10 seconds. A failed delivery is retried after 5 seconds, 30 seconds, 2 minutes, 10 minutes, 1 hour, 4 hours and 19 hours: eight attempts over about a day. Point the URL at its final address so no redirects are involved.
- An endpoint that keeps failing for 72 hours is paused, and the person who created it gets an email. While it's paused, events are dropped, not queued. After fixing the receiver, resume it under **Developer → Webhooks**.
- An event can arrive more than once and out of order. Drop duplicates by `id` (or `Synup-Delivery`). `created_at` is when the delivery was created and stays the same on every retry of it, but events can still arrive out of order.
- Deliveries are kept for 30 days, and any of them can be replayed from the delivery log under **Developer → Webhooks**.

## Rotating the secret

Rotating (under **Developer → Webhooks**) takes effect at once. For the next 7 days every delivery, retries included, is signed with both secrets: the first `v1` with the new secret, the second with the old one. An endpoint still on the old secret keeps working, so deploy the new secret at any point in that window. **Revoke** the old secret on the same page to end the window early.

A verifier must accept a match on any `v1`, as the examples above do. One that checks only the first or only the last `v1` fails during the window. `SYNUP_WEBHOOK_SECRET_OLD` in the examples is optional: set it if you want an endpoint to accept either secret while you switch over.
