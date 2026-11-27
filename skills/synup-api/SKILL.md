---
name: synup-api
description: Use when writing, reviewing or debugging code that calls Synup's REST API at https://ai.synup.com/api/v1. Covers creating and storing API keys, the Authorization Bearer header and key scopes, finding the right endpoint and its fields in Synup's English API reference, paging through lists, rate limits and 429 retries, and error responses.
metadata:
  openclaw:
    homepage: https://synup.com/developers/
    envVars:
      - name: SYNUP_API_KEY
        required: false
        description: Synup API key used by the code examples, created under Developer → API Keys in Synup. The skill itself needs no key.
---

# Synup API

Base URL: `https://ai.synup.com/api/v1`. Requests and responses are JSON.

## Find the endpoint before writing code

Synup's API reference is in plain English, and every page has a Markdown copy for AI tools. Fetch these instead of guessing paths, parameters or fields:

- **Index of every page:** https://synup.com/developers/llms.txt
- **One area and all its endpoints:** `https://synup.com/developers/resources/<area>.md`, for example https://synup.com/developers/resources/reviews.md. Areas: `aeo`, `analytics`, `approvals`, `boosts`, `clients`, `connections`, `credits`, `custom-fields`, `data`, `distribution`, `duplicate-listings`, `inbox`, `listening`, `listings`, `locations`, `media`, `post-ideas`, `posts`, `review-invites`, `reviews`, `seo`, `team`, `workspace`.
- **One endpoint**, with its parameters, request body, response fields and required scope: follow the link on the area page and add `.md`, for example https://synup.com/developers/resources/reviews/list-reviews.md
- **Authentication, scopes and rate limits:** https://synup.com/developers/authentication.md

Don't take descriptions from the raw spec at `https://ai.synup.com/api/docs/v1`: its summaries are translation keys, not text. Its paths and schemas are fine for Postman or code generators.

## Keys and auth

- Create a key in Synup under **Developer → API Keys**. Keys start with `sy_`, are shown in full only once, and expire after 30, 90 (the default) or 365 days. An expired key gets `401`.
- Send the key on every request: `Authorization: Bearer sy_...`
- Choose the narrowest key that works. Access is **Read-only**, **Full access** or **Custom** (none, read, or read and write per area). Client access is **All clients** or **Specific clients**; a key limited to specific clients gets `403` for any other client. Each endpoint's page names the scope it needs, such as `reviews:read`.
- Keep keys out of source code. Read the key from an environment variable such as `SYNUP_API_KEY` or from a secret manager, and never log it.
- The REST API needs a Synup plan that includes the Developer API. A free-trial workspace can use the MCP server but gets refused here.

## Responses and paging

- Results come wrapped in `data`. For example, `GET /clients` returns `{ "data": { "clients": [...], "nextCursor": "...", "total": 42 } }`.
- Most lists page by cursor. Send `limit` (each endpoint documents its maximum) and pass `data.nextCursor` back as `cursor` until it comes back `null`.
- A few lists use page numbers instead: `page` with `perPage` or `pageSize`. These include posts, the social inbox, post ideas, review-invite lists and boosts. The endpoint's page says which style it uses.
- Many endpoints need a parent's ids. `GET /reviews`, for example, requires both `clientId` and `locationId`. Get clients from `GET /clients` and locations from `GET /locations`.

## Errors

Every error body has an `error` field, and some add more detail:

```json
{ "error": "locationId and clientId are both required" }
```

| Status | Meaning |
| --- | --- |
| 400 | A parameter is missing or malformed |
| 401 | The key is missing, invalid, expired or revoked |
| 403 | The key lacks the scope, or isn't allowed that client or location |
| 404 | It doesn't exist, or isn't in this workspace |
| 409 | It conflicts with the current state, for example it's already archived |
| 422 | Well-formed but fails validation |
| 429 | Rate limited (see below) |

## Rate limits

- Limits are per workspace, not per key. They grow with the number of active locations and come in per-minute, per-hour and per-day windows.
- Every response carries `X-RateLimit-Limit-Minute`, `-Hour` and `-Day`, plus matching `X-RateLimit-Remaining-*` and `X-RateLimit-Reset-*` headers.
- A `429` has a `Retry-After` header in seconds and a body like this:

  ```json
  {
    "error": "rate_limit_exceeded",
    "message": "This account has used its 600 API calls per hour ...",
    "window": "hour",
    "limit": 600,
    "retryAfterSeconds": 1200,
    "requestId": "..."
  }
  ```

  Wait `Retry-After` seconds, then retry. Never retry in a tight loop. For bulk jobs, watch `X-RateLimit-Remaining-*` and slow down before you hit zero.

## Debugging

- **Developer → API request logs** in Synup shows every request made with your keys, errors included.
- Quote the `requestId` from an error body when you contact support@synup.com.

## Example: every review for one location (TypeScript)

```ts
const BASE = "https://ai.synup.com/api/v1";
const KEY = process.env.SYNUP_API_KEY;
if (!KEY) throw new Error("Set SYNUP_API_KEY");

async function synup(path: string, params: Record<string, string> = {}) {
  const url = new URL(BASE + path);
  for (const [k, v] of Object.entries(params)) url.searchParams.set(k, v);
  for (let attempt = 1; ; attempt++) {
    const res = await fetch(url, { headers: { Authorization: `Bearer ${KEY}` } });
    if (res.status === 429 && attempt <= 5) {
      const waitSeconds = Number(res.headers.get("Retry-After") ?? "60");
      await new Promise((resolve) => setTimeout(resolve, waitSeconds * 1000));
      continue;
    }
    const body = await res.json();
    if (!res.ok) throw new Error(`Synup ${res.status}: ${body.error}`);
    return body;
  }
}

export async function allReviews(clientId: string, locationId: string) {
  const reviews = [];
  let cursor: string | null = null;
  do {
    const { data } = await synup("/reviews", {
      clientId,
      locationId,
      limit: "100",
      ...(cursor ? { cursor } : {}),
    });
    reviews.push(...data.reviews);
    cursor = data.nextCursor;
  } while (cursor);
  return reviews;
}
```
