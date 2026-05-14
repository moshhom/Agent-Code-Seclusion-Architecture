# Level 1 — Critical · System-wide status banner

> **ACSA level: 1 (Critical).** Read every line. Do not merge without
> manual review by a human. Agent-only changes are not accepted.

A single function controls the banner that appears at the top of every
signed-in page of the app: scheduled maintenance, incident messaging,
new-feature announcements. When this function runs, the result reaches
every user on next page load.

## Why this is Level 1

- **Blast radius — on the user surface, not ours.** A wrong banner
  reaches every user simultaneously. "We're down for maintenance" pinned
  on a healthy app, an announcement scheduled for next Tuesday going out
  today, a stale incident banner left on for a week. Each is mass user
  confusion. This is the consequence axis that puts this file at Level 1
  on its own.
- **Customer impact.** Disorientation, support load, perceived
  unreliability — across every account at once.
- **Access.** Writes one key in the cache the UI reads from. No DB, no
  user records, no money, no secrets.
- **Maintenance lifetime.** Long. Every incident, every release
  announcement, every maintenance window touches this path.
- **Revertability.** Trivial in mechanics — call the function with
  `None` and the banner is gone. But minutes of every user staring at
  the wrong message are not refunded.

This is the canonical "blast radius on the user side, no harm to the
system" case. No data lost, no security boundary crossed, no money
moved. Every user simply sees the wrong message at the same time. The
session-token example sits one room over: blast radius on the security
boundary, where the failure first hits *us* and reaches users through
the breach. This one bypasses us and lands on users directly.

## What Level 1 treatment looks like for this file

- Inputs are validated explicitly: start before end, future-only end,
  non-empty trimmed message, capped length.
- An audit row is written before the cache write, so the change is
  traceable even if propagation fails.
- One key, one writer. Nothing else in the codebase writes to
  `global.status_banner`.
- The cache write is the only side effect. No mailers, no push
  notifications, no Slack — those live elsewhere with their own
  review path.
- Edits to this file always require manual review, regardless of how
  small the diff looks.

---

## Python

```python
# ACSA Level 1 - Critical
# Read every line. No merge without manual review.
# Rationale: single writer for the banner every signed-in page renders.

from dataclasses import dataclass
from datetime import datetime

from auditing import record_banner_change
from cache import cache_set

MAX_MESSAGE_LEN = 280


@dataclass(frozen=True)
class Banner:
    message: str
    starts_at: datetime
    ends_at: datetime


def set_status_banner(
    banner: Banner | None,
    actor: str,
    now: datetime,
) -> None:
    # 1. Validate. None is allowed and means "clear the banner".
    if banner is not None:
        msg = banner.message.strip()
        if not msg:
            raise ValueError("empty banner message")
        if len(msg) > MAX_MESSAGE_LEN:
            raise ValueError("banner message too long")
        if banner.starts_at > banner.ends_at:
            raise ValueError("banner window inverted")
        if banner.ends_at < now:
            raise ValueError("banner window already expired")

    # 2. Audit first so the change is traceable even if the cache write fails.
    record_banner_change(actor, banner, now)

    # 3. One key, one writer.
    cache_set("global.status_banner", banner)
```

## TypeScript

```ts
// ACSA Level 1 - Critical
// Read every line. No merge without manual review.
// Rationale: single writer for the banner every signed-in page renders.

import { recordBannerChange } from "./auditing";
import { cacheSet } from "./cache";

const MAX_MESSAGE_LEN = 280;

export type Banner = Readonly<{
  message: string;
  startsAt: Date;
  endsAt: Date;
}>;

export async function setStatusBanner(
  banner: Banner | null,
  actor: string,
  now: Date,
): Promise<void> {
  // 1. Validate. null is allowed and means "clear the banner".
  if (banner !== null) {
    const msg = banner.message.trim();
    if (msg.length === 0) throw new Error("empty banner message");
    if (msg.length > MAX_MESSAGE_LEN) throw new Error("banner message too long");
    if (banner.startsAt > banner.endsAt) throw new Error("banner window inverted");
    if (banner.endsAt < now) throw new Error("banner window already expired");
  }

  // 2. Audit first so the change is traceable even if the cache write fails.
  await recordBannerChange(actor, banner, now);

  // 3. One key, one writer.
  await cacheSet("global.status_banner", banner);
}
```

## Go

```go
// ACSA Level 1 - Critical
// Read every line. No merge without manual review.
// Rationale: single writer for the banner every signed-in page renders.

package status

import (
	"context"
	"errors"
	"strings"
	"time"

	"example.com/notes/auditing"
	"example.com/notes/cache"
)

const maxMessageLen = 280

type Banner struct {
	Message   string
	StartsAt  time.Time
	EndsAt    time.Time
}

func SetStatusBanner(
	ctx context.Context,
	banner *Banner,
	actor string,
	now time.Time,
) error {
	// 1. Validate. nil is allowed and means "clear the banner".
	if banner != nil {
		msg := strings.TrimSpace(banner.Message)
		switch {
		case msg == "":
			return errors.New("empty banner message")
		case len(msg) > maxMessageLen:
			return errors.New("banner message too long")
		case banner.StartsAt.After(banner.EndsAt):
			return errors.New("banner window inverted")
		case banner.EndsAt.Before(now):
			return errors.New("banner window already expired")
		}
	}

	// 2. Audit first so the change is traceable even if the cache write fails.
	if err := auditing.RecordBannerChange(ctx, actor, banner, now); err != nil {
		return err
	}

	// 3. One key, one writer.
	return cache.Set(ctx, "global.status_banner", banner)
}
```

---

## Notice what this file does *not* do

- It does **not** decide *whether* a banner should be shown. The policy
  (incident playbook, release-notes flow, ops checklist) lives upstream.
- It does **not** persist the banner anywhere durable. The cache key is
  intentionally ephemeral — the worst-case outage is that the banner
  clears itself, never that an old banner re-appears unexpectedly.
- It does **not** read or write user records. Nothing here knows who
  the users are; the banner is global by construction.
- It does **not** send anything. Email blasts, push notifications, and
  external announcements are separate Level 1 files with their own
  review paths — each of those *does* affect "us" (deliverability
  reputation, irreversible sends) and is treated accordingly.
