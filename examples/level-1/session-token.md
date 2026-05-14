# Level 1 — Critical · Session token verifier

> **ACSA level: 1 (Critical).** Read every line. Do not merge without
> manual review by a human. Agent-only changes are not accepted.

Every request to a private endpoint passes through this function. It
takes a token from a cookie or header and returns the authenticated
user, or rejects the request. The file is pure logic over bytes — no
database, no network.

## Why this is Level 1

- **Blast radius.** A bug here grants the wrong identity to every
  downstream handler at once. Account takeover is the default failure
  mode. This is the consequence axis that puts this file at Level 1 on
  its own.
- **Access.** Holds the HMAC signing key.
- **Customer impact.** Reading or modifying any other user's data is on
  the table.
- **Revertability.** The file itself reverts cleanly — there are no DB
  writes here. The damage isn't as easy: tokens issued under a broken
  verifier may already have been used, so recovery means rotating the
  signing key and forcing every user to log in again.
- **Maintenance lifetime.** Auth code stays in the product forever.

This is the canonical "Level 1 by blast radius, no DB involvement" case.

## What Level 1 treatment looks like for this file

- Fails closed. Every error path returns "no user".
- Signature is verified with a constant-time comparison before the body
  is parsed.
- Nothing about the token is logged.
- Imports are stdlib + one well-known crypto primitive. No DB, no
  network, no third-party SDK.

---

## Python

```python
# ACSA Level 1 - Critical
# Read every line. No merge without manual review.
# Rationale: identity-issuing function for every authenticated request.

import base64
import hashlib
import hmac
import json
import os
from datetime import datetime

SIGNING_KEY = os.environ["SESSION_SIGNING_KEY"].encode()


def verify_token(token: str, now: datetime) -> str | None:
    # 1. Split. No meaning is read from either half yet.
    parts = token.split(".")
    if len(parts) != 2:
        return None
    try:
        body = base64.urlsafe_b64decode(parts[0] + "==")
        sig = base64.urlsafe_b64decode(parts[1] + "==")
    except ValueError:
        return None

    # 2. Constant-time signature check. Nothing downstream runs until this passes.
    expected = hmac.new(SIGNING_KEY, body, hashlib.sha256).digest()
    if not hmac.compare_digest(sig, expected):
        return None

    # 3. Only now is the body trusted enough to parse.
    payload = json.loads(body)
    if not isinstance(payload, dict):
        return None
    if int(payload.get("exp", 0)) < int(now.timestamp()):
        return None
    sub = payload.get("sub")
    if not isinstance(sub, str) or not sub:
        return None
    return sub
```

## TypeScript

```ts
// ACSA Level 1 - Critical
// Read every line. No merge without manual review.
// Rationale: identity-issuing function for every authenticated request.

import { createHmac, timingSafeEqual } from "crypto";

const SIGNING_KEY = Buffer.from(process.env.SESSION_SIGNING_KEY!, "utf8");

export function verifyToken(token: string, now: Date): string | null {
  // 1. Split. No meaning is read from either half yet.
  const parts = token.split(".");
  if (parts.length !== 2) return null;

  let body: Buffer;
  let sig: Buffer;
  try {
    body = Buffer.from(parts[0], "base64url");
    sig = Buffer.from(parts[1], "base64url");
  } catch {
    return null;
  }

  // 2. Constant-time signature check. Nothing downstream runs until this passes.
  const expected = createHmac("sha256", SIGNING_KEY).update(body).digest();
  if (sig.length !== expected.length || !timingSafeEqual(sig, expected)) {
    return null;
  }

  // 3. Only now is the body trusted enough to parse.
  let payload: unknown;
  try {
    payload = JSON.parse(body.toString("utf8"));
  } catch {
    return null;
  }
  if (typeof payload !== "object" || payload === null) return null;
  const p = payload as { sub?: unknown; exp?: unknown };
  if (typeof p.exp !== "number" || p.exp < Math.floor(now.getTime() / 1000)) {
    return null;
  }
  if (typeof p.sub !== "string" || p.sub.length === 0) return null;
  return p.sub;
}
```

## Go

```go
// ACSA Level 1 - Critical
// Read every line. No merge without manual review.
// Rationale: identity-issuing function for every authenticated request.

package session

import (
	"crypto/hmac"
	"crypto/sha256"
	"encoding/base64"
	"encoding/json"
	"os"
	"strings"
	"time"
)

var signingKey = []byte(os.Getenv("SESSION_SIGNING_KEY"))

type payload struct {
	Sub string `json:"sub"`
	Exp int64  `json:"exp"`
}

func VerifyToken(token string, now time.Time) (string, bool) {
	// 1. Split. No meaning is read from either half yet.
	parts := strings.Split(token, ".")
	if len(parts) != 2 {
		return "", false
	}
	body, err := base64.RawURLEncoding.DecodeString(parts[0])
	if err != nil {
		return "", false
	}
	sig, err := base64.RawURLEncoding.DecodeString(parts[1])
	if err != nil {
		return "", false
	}

	// 2. Constant-time signature check. Nothing downstream runs until this passes.
	mac := hmac.New(sha256.New, signingKey)
	mac.Write(body)
	if !hmac.Equal(sig, mac.Sum(nil)) {
		return "", false
	}

	// 3. Only now is the body trusted enough to parse.
	var p payload
	if err := json.Unmarshal(body, &p); err != nil {
		return "", false
	}
	if p.Exp < now.Unix() || p.Sub == "" {
		return "", false
	}
	return p.Sub, true
}
```

---

## Notice what this file does *not* do

- It does **not** look up the user in the database. The returned ID is
  trusted to be a valid identifier; whether that user still exists is
  the caller's problem.
- It does **not** log the token, the signature, or the payload.
- It does **not** accept a token whose signature it was unable to verify
  "for a good reason". The only successful path runs through the
  constant-time comparison.
- It does **not** *issue* tokens. The matching issuer is a sibling
  Level 1 file with its own review path.
