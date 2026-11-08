# 💳 PayGuard — AI-verified invoice payments

**Paste an untrusted invoice. PayGuard screens it for fraud, verifies every field against the source, cross-checks your purchase order — and only then creates a PayPal order for your approval.**

Built for the [PayPal AI Hackathon](https://paypalaihackathon.devpost.com) (Nov 2026). AI by [Clocked](https://clocked.rezolve.ai), payments by the **PayPal Orders API** (sandbox).

## The problem

Vendor email fraud is a billion-dollar problem: scammers send inflated invoices, change bank details, hide instruction overrides inside "portal sync notes", and pressure AP teams into paying. Every payment flow that treats an incoming invoice as trusted input is exposed.

PayGuard flips the model: **the invoice is hostile until the pipeline proves it safe, and a human still approves the money.**

## How it works

```
Invoice (untrusted) + Purchase Order (optional)
        │
        ▼
┌──────────────────────────────────────────────────────────┐
│  1 · Guard    /clocked/injection  → manipulation scan.   │
│        Flagged? Pipeline HALTS. No payment is prepared.  │
│  2 · Protect  /clocked/pii        → PII masked; only the │
│        masked copy continues downstream.                 │
│  3 · Decide   /clocked/decide     → fraud signals        │
│        (urgency, bank-change, secrecy, thread-hijack).   │
│  4 · Extract  /clocked/extract    → verbatim fields with │
│        character offsets.                                │
│  5 · Fill     /clocked/fill       → structured record,   │
│        one call (extract + score + yes_no fields).       │
│  6 · Verify   /clocked/grounding  → every extracted      │
│        value checked claim-by-claim against the source.  │
│  7 · Compare  /clocked/compare    → invoice vs PO, every │
│        conflict surfaced with paired evidence quotes.    │
└──────────────────────────────────────────────────────────┘
        │
        ▼
  Risk score 0–100 (server logic combining the outputs above)
        │
        ├─ pay_safe (0–24)  → PayPal order created for the EXACT
        │                      verified amount → human approves → capture
        ├─ review  (25–59)  → no order until a human confirms
        └─ block   (60–100) → no order, ever
```

## Meaningful use — delete an endpoint and the app demonstrably changes

- **Guard** removed → instruction overrides inside invoices would be processed instead of halting the run.
- **Protect** removed → raw PII (names, bank accounts) would flow into every downstream AI call.
- **Decide** removed → no fraud signals; the risk score would be blind to urgency pressure, bank-detail changes, secrecy claims, and thread hijacks.
- **Extract** removed → no verbatim amounts/identifiers, no verified values to pay against.
- **Fill** removed → no single-call structured record with urgency scoring and yes/no inference.
- **Verify** removed → hallucinated or ungrounded amounts would flow straight into the PayPal order.
- **Compare** removed → invoice-vs-PO conflicts (inflated totals, changed terms) would go unnoticed.
- **PayPal Orders API** — the point of the whole exercise: `create order → human approval → capture`, with the amount taken **only** from verified, grounded fields, sanitized server-side (`$7,960.00` → `7960.00`), refused if unparseable.

## Try it

Two samples ship with the app:

1. **Clean invoice + matching PO** → risk 0/100, all fields verified, PayPal order ready for approval.
2. **Fraud invoice** (inflated total, "we changed our bank account", hidden automation note) → Guard blocks the pipeline; no payment is ever prepared.

## Run it

```bash
git clone https://github.com/dualwarez/payguard && cd payguard
npm install
cp .env.example .env    # add CLOCKED_API_KEY, PAYPAL_CLIENT_ID, PAYPAL_CLIENT_SECRET
npm start               # http://localhost:3100
```

PayPal credentials come from the [developer dashboard](https://developer.paypal.com) (free sandbox). Clocked credentials from [clocked.rezolve.ai](https://clocked.rezolve.ai) (free tier includes 4,000 credits; a full verification run costs ~25).

## Architecture

```
browser (public/)            server (server.js)                 APIs
┌────────────────┐  POST /api/*  ┌────────────────────┐  Bearer  ┌────────────────────┐
│ vanilla JS SPA │ ────────────► │ Express            │ ───────► │ /clocked/* (7 AI   │
│ pipeline UI    │ ◄──────────── │ risk scoring       │ ◄─────── │ endpoints)         │
│ PayPal return  │  normalized   │ PayPal orders      │  OAuth   │ /v2/checkout/orders│
└────────────────┘               └────────────────────┘          └────────────────────┘
```

- **Both API keys stay server-side.** The browser talks only to our backend.
- The **amount paid is never taken from free text** — only from the verified record, parsed and bounded server-side (`$7,960.00` → `7960.00`, refused outside $0–100,000).
- **Human-in-the-loop is structural**: `review` and `block` verdicts produce no PayPal order at all. The AI prepares; the human approves; PayPal executes.
- Every Clocked response's `X-Clocked-Credits` header is accumulated and shown in the header — credit transparency is a feature.
- Vercel serverless entry: `api/index.js` wraps the Express app; `vercel.json` routes `/api/*` to it.

## Demo script (what the video shows)

1. Paste the fraud invoice → Guard halts the pipeline in seconds — no payment path exists.
2. Paste the clean invoice + PO → seven green steps, risk 0/100, every field badged "verified in source".
3. Click **Authorize payment with PayPal** → order created for exactly $796.00 → PayPal approval → capture. Verified, approved, paid.

## Team

Built by **dualwarez** (Dhananjay Singh) for the PayPal AI Hackathon. All code written during the challenge window — git history is the proof.
