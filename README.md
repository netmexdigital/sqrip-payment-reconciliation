# sqrip Auskunftsdienst — Client Integration Guide

**Build a payment-reconciliation client for any shop platform** (Shopify, Magento,
PrestaShop, a custom shop, …) against the sqrip Auskunftsdienst API.

> Status: v0.1 — feedback welcome (open an issue). Audience: developers building a client.
> The reference client is the WooCommerce plugin's `inc/class-sqrip-avis.php` (GPL) — read
> it alongside this guide for a complete, working example.

---

## 1. What the service does (mental model)

Swiss banks already e-mail their customers a **credit notification** whenever money
arrives on an account. The Auskunftsdienst turns those e-mails into an answer to one
question your shop cares about:

> "Which of my open orders just got paid?"

Flow:

1. The shop forwards its bank's credit notifications to a **per-shop mailbox**,
   `<name>@avis.sqrip.ch`.
2. The service reads and parses each notification (amount, currency, reference, dates).
3. Your **client** sends the service its **open orders** (reference + expected amount).
4. The service returns only the credits that **match one of those orders**, plus
   **warnings** for anything that needs a human. Everything else is only counted, never
   returned.
5. Your client books the clean matches, holds/flags the rest, and notifies the shop admin.

**Two properties define the whole design:**

- **Order-driven.** The client drives from its own open orders. The service never hands
  back the shop's entire bank activity — only what matches an order the client asked
  about. This is a hard privacy line (a monthly bank file is the shop's whole business).
- **Billing is server-side and platform-agnostic.** The service charges **1 credit per
  booked match** to the **shop's own sqrip account**. Your client does not handle money or
  pricing — it passes the shop's sqrip API token, and the service bills that account.

---

## 2. Safety Checklist (the money path — do not skip)

These are the non-negotiables. A client that gets any of these wrong will mis-book real
money and the shop will blame sqrip. Verify each one:

- [ ] **Order-driven only.** Send **only the shop's open orders**, each with `reference`
      **and** `amount` **and** `currency`. Never send closed orders. Never store, list, or
      log the bank data that did not match one of your orders — treat "unassigned" as a
      count, never a record.
- [ ] **Single delivery.** `matches` **and** `warnings` are delivered **exactly once** in
      the claim response, then the service **drops** them. You must act **and persist** on
      first sight (book the order / write the hold + note + admin e-mail). A second
      `/v2/claim` will **not** repeat them — an unhandled warning is lost.
- [ ] **Idempotency.** Never book the same payment twice. Guard each action with a stable
      key, e.g. `reference + amount + value_date`. The server bills idempotently
      (`customer:reference`); your side must be idempotent too.
- [ ] **Never auto-book on doubt.** Only an entry in **`matches[]`** may be marked paid
      automatically. Every **`warnings[]`** item (underpayment, low confidence, checksum,
      no-reference, …) is **held or flagged**, never booked without a human.
- [ ] **Always send `amount` (+ currency).** Without the expected amount the service
      cannot detect under- or overpayment — you lose the most important safety check.
- [ ] **Acknowledge the nudge fast.** Return `2xx` to the callback **immediately**; do the
      claim afterwards (a queue/async job is ideal). The service does not wait for your
      processing.
- [ ] **Distinguish gate failures from transient errors.** `402` = no credits, `403` =
      account inactive → tell the admin. `503` = upstream/network → retry later; do **not**
      show "account inactive" for a mere network hiccup.
- [ ] **Confirm-by-hand path is tamper-proof.** If you e-mail the admin one-click
      confirm/reject links, sign them (HMAC), make them **single-use** and **short-lived**
      (e.g. 24 h). Never book from an unsigned or reusable link.

---

## 3. The integration in five steps

1. **Register** the shop with the service (`POST /v2/register`). This stores the shop's
   callback URL and its sqrip token, and runs the account gate (active? credits?).
2. **Show the shop its mailbox address** `<name>@avis.sqrip.ch` and have it enter that as
   the destination for **credit notifications** in its e-banking.
3. **(Optional) Verification code.** Some banks send a one-time code to that address to
   confirm it. Proxy the onboarding calls so the shop can read the code and enter it back.
4. **Receive the nudge.** When a credit arrives, the service `POST`s to your callback.
   Acknowledge `2xx` immediately, then call claim.
5. **Claim & process.** `POST /v2/claim` with your open orders; act on the response
   (Section 5).

---

## 4. API surface (v2)

The authoritative, runnable contract is the **Postman collection "sqrip Auskunftsdienst
API"** — [browse the published docs](https://documenter.getpostman.com/view/57091140/2sBY4VLd8x)
(read online or fork into your own workspace). Summary:

### 4.1 Register — `POST /v2/register`
```json
{ "token": "<client token, your shared secret with the service>",
  "customer": "<mailbox localpart, e.g. \"timber\" for timber@avis.sqrip.ch>",
  "callback_url": "https://shop.example/whatever/reconcile",
  "sqrip_token": "<the shop's own sqrip API token>" }
```
- `token` is a secret **your client** mints once and reuses; it authenticates you on the
  callback and on claim.
- `sqrip_token` is the shop's real sqrip account token. **Strip whitespace/newlines**
  before sending (a trailing newline in a header breaks the upstream request).
- Responses: `200 {ok, customer, credits_left}` · `402 {error:"keine_credits"}` ·
  `403 {error:"sqrip_konto_inaktiv"}` · `409 {error:"mailbox_name_taken"}` ·
  `503 {error:"upstream_unavailable"}` (transient — retry).
- Register is **idempotent**; call it whenever settings are saved, on the health check, and
  before onboarding, so the service always has the current callback URL.

### 4.2 Callback (you expose this) — the service nudges you
```
POST <callback_url>    Body: { "token": "<client token>", "pending": <int> }
```
- Verify `token` against the stored client token (else `401`).
- If `pending` is present and `<= 0`, there is nothing to fetch — you may skip the claim.
- Return `2xx` **immediately**, then run the claim.

### 4.3 Claim — `POST /v2/claim`
```json
{ "token": "<client token>",
  "orders": [
    { "reference": "<QR reference / SCOR of the order>",
      "order_number": "4281",
      "amount": 50.00,
      "currency": "CHF",
      "customer_name": "<optional, raises the match score>" }
  ] }
```
- Send **one entry per open QR slip**. `reference` + `amount` + `currency` are the minimum.
- Response:
```json
{ "matches": { "<reference>": [ { "reference","amount","currency","value_date",
                                  "booking_date","score","matched_aspects":[...] } ] },
  "warnings": [ { "type": "...", ... } ],
  "dropped":  <int>,
  "last_seen": { ... },
  "pending_charges": <int> }
```
- `matches` → assigned **and billed**. `score` is 0–10 (gate ≥ 5); `matched_aspects` ⊆
  `["reference","order_number","payer","amount"]`.
- `pending_charges` is informational (internal billing backlog).
- **The service deletes this shop's candidates after answering.** See the checklist.

### 4.4 Onboarding (verification code) — `POST /v2/onboarding/{start|code|complete}`
Body `{ "token": "<client token>" }`. `start` → `{ok, address, status:"awaiting_code"}`;
`code` → `{code, status:"found"|"pending"}`; `complete` → `{ok}`. Gated on credits.

---

## 5. Acting on the claim response

### 5.1 `matches` → book the order as paid
Each match is assigned and already billed. Mark the order paid (idempotently). Use the
match's `reference` to find your order and `amount` as the credited amount.

### 5.2 `warnings` → never auto-book; act per `type`

| `type` | fields | what to do |
|---|---|---|
| `underpayment` | `reference, received, expected` | **Hold** the order (stays open). E-mail the admin: paid too little. |
| `overpayment` | `reference, received, expected` | Also appears as a **match**. Shop's choice: **hold** for release, or mark paid — either way notify the admin. |
| `no_reference` | `amount, currency, candidate_references[]` | Payment without a reference. Offer the candidate orders to the admin for a one-click assignment. |
| `bulk_payment` | `amount, currency, candidate_references[]` | One payment covers several orders → split across the candidates (admin confirms). |
| `no_reference_key` | `order_number` | Matched by order number but the order has no QR reference. If your platform always issues a reference, this cannot occur — otherwise flag it. |
| `low_confidence` | `reference, score, matched_aspects[], currency_mismatch?, payment_amount?, payment_currency?` | Tentatively matched to one order below the gate → **manual check**. On `currency_mismatch`, show `payment_currency` vs the order currency. |
| `checksum` | `batch_total` | A batch total does not add up → **hold the whole batch**, notify the admin. |

For every non-trivial warning, notify the admin (the service sends **no** e-mail — it does
not know shop addresses; notifying is the client's job).

### 5.3 Optional but recommended: a release limit + overpayment policy
- **Release limit.** A shop may want to confirm larger amounts by hand. Above the limit,
  don't auto-book — e-mail the admin a signed one-time confirm/reject link. At `0` every
  payment needs release; a very high value books everything automatically.
- **Overpayment policy.** Let the shop choose "hold and notify" vs "mark paid and notify".
- **Signed one-time links.** HMAC over `id|action` with your client secret, stored for a
  short TTL, single-use (clear siblings together). This is how the admin confirms without
  logging in, safely.

---

## 6. Data protection (keep this line)

- Drive from **open orders**; only matched credits come back. Everything else is a **count**.
- Don't read, store, or forward payer data (name/address/account) for non-matching credits.
- Normalise references identically on both sides: uppercase, keep only `A–Z0–9`.
- A shop's monthly bank data is its entire business — treat "unassigned" as a number, not a
  list. This stance is both a privacy requirement and a trust argument you can advertise.

---

## 7. Reference implementation

WooCommerce plugin, `inc/class-sqrip-avis.php` (GPL): a complete client — register,
callback, claim, per-type warning handling, single-delivery persistence, idempotency,
release limit, signed confirm links, and the order-driven privacy line. Read it as the
worked example next to this guide.

---

## 8. Getting access

Everything you need is public — the **Postman collection** carries the API base URL and the
exact request/response shapes:
[published docs](https://documenter.getpostman.com/view/57091140/2sBY4VLd8x) (read online or
fork it into your own workspace). The service (parsing, mailboxes, bank coverage, billing)
stays operated by sqrip — you integrate against it, you don't run it. For a partnership or
questions, reach out via [sqrip.ch](https://sqrip.ch).

**sqrip issues no integration-specific key.** Two credentials are in play, and neither is
handed out per integrator:

- **sqrip API key** — each shop's own key from its **sqrip.ch account (with credits)**, the
  same key it already uses for QR bills. Your client passes it as `sqrip_token` (Section 4.1);
  the service uses it both to gate access and to bill the shop per delivered match. Access is
  therefore gated by "the shop has a funded sqrip account", not by a key sqrip gives you.
- **callback token** (`token`) — a secret **your client generates itself**, once per shop, and
  reuses to authenticate the nudge callback and the claim. sqrip does not issue it.

---

## 9. License

- **Documentation** in this repository (this guide) — [CC BY 4.0](LICENSE): use and adapt
  freely, with attribution to netmex digital gmbh / sqrip.
- **Code and examples** added to this repository — [MIT](LICENSE-CODE).

© 2026 netmex digital gmbh. "sqrip" is a product of netmex digital gmbh. The reference
client (`class-sqrip-avis.php`) lives in the WooCommerce plugin and is GPL — that license
governs that file in its own repository.

---

*Questions or corrections? Open an issue. This guide is a work in progress and will grow
with reference clients and SDKs.*
