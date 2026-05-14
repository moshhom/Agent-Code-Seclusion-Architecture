# Level 1 — Critical · Billing Webhook Handler

> **ACSA level: 1 (Critical).** Read every line. Do not merge without
> manual review by a human. Agent-only changes are not accepted.

A tiny SaaS notes app needs a Stripe billing webhook to know when a
customer paid, cancelled, or had a charge fail. This file is the entry
point for those events.

## Why this is Level 1

- **Blast radius.** A bug here can leak entitlement to users who haven't
  paid, revoke access from users who have, double-charge, or accept
  forged events as real.
- **Access.** It holds the Stripe signing secret, writes to the
  billing state of every account, and is the only path that promotes
  a free user to a paid one.
- **Revertability.** Money moves. Refunds, accounting corrections, and
  customer-support fallout are all manual and expensive.
- **Customer impact.** A wrong response code or a missed event silently
  desyncs the source of truth (Stripe) from the app, and the user feels
  it the next time they try to use a paid feature.
- **Maintenance lifetime.** Billing stays in the codebase forever. It
  will be read, debugged, and modified for as long as the product is
  alive.

## What Level 1 treatment looks like for this file

- Signature is verified **before** any work. No parsing, no DB access,
  no logging of the body until the signature passes.
- Constant-time comparison. No early-exit on first byte mismatch.
- Idempotency: every event ID is checked against a store before it is
  processed. Stripe retries.
- The set of imports is narrow and audited. This file does not pull in
  the rest of the app — it reaches into a tiny billing module that owns
  the DB writes, and nothing else.
- Runs in its own process / service where possible, with credentials
  scoped only to the billing tables and the Stripe key.

---

## Python

```python
# ACSA Level 1 - Critical
# Read every line. No merge without manual review.
# Rationale: Stripe webhook entry; access to signing secret and billing state.

import hmac
import hashlib
import os
import time
from http import HTTPStatus

import stripe  # narrow, audited dependency

from billing import apply_event, already_processed, record_processed

STRIPE_WEBHOOK_SECRET = os.environ["STRIPE_WEBHOOK_SECRET"]
MAX_SKEW_SECONDS = 5 * 60


def handle_webhook(raw_body: bytes, signature_header: str) -> int:
    # 1. Verify signature first. Nothing else happens until this passes.
    try:
        event = stripe.Webhook.construct_event(
            payload=raw_body,
            sig_header=signature_header,
            secret=STRIPE_WEBHOOK_SECRET,
            tolerance=MAX_SKEW_SECONDS,
        )
    except (stripe.error.SignatureVerificationError, ValueError):
        return HTTPStatus.BAD_REQUEST

    # 2. Idempotency. Stripe retries; we must not double-apply.
    if already_processed(event["id"]):
        return HTTPStatus.OK

    # 3. Apply the event inside the billing module. This file does not
    #    touch the DB directly.
    apply_event(event)
    record_processed(event["id"])
    return HTTPStatus.OK
```

## TypeScript

```ts
// ACSA Level 1 - Critical
// Read every line. No merge without manual review.
// Rationale: Stripe webhook entry; access to signing secret and billing state.

import Stripe from "stripe"; // narrow, audited dependency
import { applyEvent, alreadyProcessed, recordProcessed } from "./billing";

const STRIPE_WEBHOOK_SECRET = process.env.STRIPE_WEBHOOK_SECRET!;
const MAX_SKEW_SECONDS = 5 * 60;

const stripe = new Stripe(process.env.STRIPE_API_KEY!, {
  apiVersion: "2024-06-20",
});

export async function handleWebhook(
  rawBody: Buffer,
  signatureHeader: string,
): Promise<number> {
  // 1. Verify signature first. Nothing else happens until this passes.
  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(
      rawBody,
      signatureHeader,
      STRIPE_WEBHOOK_SECRET,
      MAX_SKEW_SECONDS,
    );
  } catch {
    return 400;
  }

  // 2. Idempotency. Stripe retries; we must not double-apply.
  if (await alreadyProcessed(event.id)) {
    return 200;
  }

  // 3. Apply the event inside the billing module. This file does not
  //    touch the DB directly.
  await applyEvent(event);
  await recordProcessed(event.id);
  return 200;
}
```

## Go

```go
// ACSA Level 1 - Critical
// Read every line. No merge without manual review.
// Rationale: Stripe webhook entry; access to signing secret and billing state.

package billinghttp

import (
	"io"
	"net/http"
	"os"

	"github.com/stripe/stripe-go/v76/webhook" // narrow, audited dependency

	"example.com/notes/billing"
)

var (
	stripeWebhookSecret = os.Getenv("STRIPE_WEBHOOK_SECRET")
	maxSkewSeconds      = int64(5 * 60)
)

func HandleWebhook(w http.ResponseWriter, r *http.Request) {
	body, err := io.ReadAll(r.Body)
	if err != nil {
		w.WriteHeader(http.StatusBadRequest)
		return
	}

	// 1. Verify signature first. Nothing else happens until this passes.
	event, err := webhook.ConstructEventWithTolerance(
		body,
		r.Header.Get("Stripe-Signature"),
		stripeWebhookSecret,
		maxSkewSeconds,
	)
	if err != nil {
		w.WriteHeader(http.StatusBadRequest)
		return
	}

	// 2. Idempotency. Stripe retries; we must not double-apply.
	processed, err := billing.AlreadyProcessed(r.Context(), event.ID)
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		return
	}
	if processed {
		w.WriteHeader(http.StatusOK)
		return
	}

	// 3. Apply the event inside the billing module. This file does not
	//    touch the DB directly.
	if err := billing.ApplyEvent(r.Context(), event); err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		return
	}
	if err := billing.RecordProcessed(r.Context(), event.ID); err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		return
	}
	w.WriteHeader(http.StatusOK)
}
```

---

## Notice what this file does *not* do

- It does **not** parse the event before verifying the signature.
- It does **not** log the raw body or the signature header.
- It does **not** import the broader app (no router, no user service, no
  analytics). The blast radius of a compromise here is capped at what
  the `billing` module exposes.
- It does **not** branch on `event.type` here. The dispatch lives inside
  `billing.apply_event`, which is also Level 1 and has its own review
  path. Keeping the entry point tiny keeps the surface that has to be
  read every line short.
