# Level 1 — Critical · Plan entitlement matrix

> **ACSA level: 1 (Critical).** Read every line. Do not merge without
> manual review by a human. Agent-only changes are not accepted.

This file is the single source of truth for what each subscription plan
can and cannot do. Every feature gate, every limit, every billing
decision reads from it. The file is small and looks easy to edit; that
is exactly why it is Level 1.

## Why this is Level 1

- **Maintenance lifetime.** This file outlives every feature in the
  product. It is touched on every plan change, every pricing experiment,
  every new capability, every A/B rollout. The accumulated cost of one
  unclear or wrong line gets paid for years. This is the consequence
  axis that puts this file at Level 1 on its own.
- **Customer impact.** A typo or wrong number silently changes what
  every paying customer can do. Most edits look one-line-trivial and
  would bypass careful review unless the file is explicitly Level 1.
- **Blast radius.** Not in a single edit — but the matrix is read from
  dozens of call sites, so any change propagates immediately on next
  deploy.
- **Access.** None special. The file does not import the database, the
  user service, or anything external.
- **Revertability.** Trivial in version control. Difficult in customer
  experience — once thousands of users have been gated wrongly for an
  afternoon, support tickets, refunds, and trust loss are not reverted
  by a `git revert`.

This is the canonical "Level 1 by maintenance lifetime" case. Nothing
dramatic happens in a given PR. Everything dramatic happens when a small
mistake survives for years, or when the file accumulates exceptions and
special-cases until nobody can read it.

## What Level 1 treatment looks like for this file

- The shape of a `Plan` is defined once, in this file. No magic strings
  (`"free"`, `"pro"`) sprinkled across the codebase.
- No conditionals. The matrix is data, not logic. If branching is
  required, it lives in the caller, never here.
- Every entry is filled. New fields default explicitly per plan; there
  is no implicit "missing key means false".
- The file does not import anything beyond stdlib types. Adding a
  dependency is itself a Level 1 change.
- Edits to this file always require manual review, even when the diff
  is one line.

---

## Python

```python
# ACSA Level 1 - Critical
# Read every line. No merge without manual review.
# Rationale: source of truth for plan capability across the entire app.

from dataclasses import dataclass


@dataclass(frozen=True)
class Plan:
    max_notes: int
    max_shares_per_note: int
    can_export: bool
    ai_assist: bool


PLANS: dict[str, Plan] = {
    "free": Plan(max_notes=200,    max_shares_per_note=2,     can_export=False, ai_assist=False),
    "pro":  Plan(max_notes=10_000, max_shares_per_note=100,   can_export=True,  ai_assist=True),
    "team": Plan(max_notes=10_000, max_shares_per_note=10_000, can_export=True, ai_assist=True),
}


def plan_for(plan_id: str) -> Plan:
    # Unknown plans fail closed. Every reader is forced to handle this.
    if plan_id not in PLANS:
        raise ValueError(f"unknown plan: {plan_id}")
    return PLANS[plan_id]
```

## TypeScript

```ts
// ACSA Level 1 - Critical
// Read every line. No merge without manual review.
// Rationale: source of truth for plan capability across the entire app.

export type PlanId = "free" | "pro" | "team";

export type Plan = Readonly<{
  maxNotes: number;
  maxSharesPerNote: number;
  canExport: boolean;
  aiAssist: boolean;
}>;

export const PLANS: Readonly<Record<PlanId, Plan>> = {
  free: { maxNotes: 200,   maxSharesPerNote: 2,     canExport: false, aiAssist: false },
  pro:  { maxNotes: 10000, maxSharesPerNote: 100,   canExport: true,  aiAssist: true  },
  team: { maxNotes: 10000, maxSharesPerNote: 10000, canExport: true,  aiAssist: true  },
};

export function planFor(planId: string): Plan {
  // Unknown plans fail closed. Every reader is forced to handle this.
  if (!(planId in PLANS)) {
    throw new Error(`unknown plan: ${planId}`);
  }
  return PLANS[planId as PlanId];
}
```

## Go

```go
// ACSA Level 1 - Critical
// Read every line. No merge without manual review.
// Rationale: source of truth for plan capability across the entire app.

package plans

import "fmt"

type Plan struct {
	MaxNotes         int
	MaxSharesPerNote int
	CanExport        bool
	AIAssist         bool
}

var plans = map[string]Plan{
	"free": {MaxNotes: 200,   MaxSharesPerNote: 2,     CanExport: false, AIAssist: false},
	"pro":  {MaxNotes: 10000, MaxSharesPerNote: 100,   CanExport: true,  AIAssist: true},
	"team": {MaxNotes: 10000, MaxSharesPerNote: 10000, CanExport: true,  AIAssist: true},
}

func PlanFor(id string) (Plan, error) {
	p, ok := plans[id]
	if !ok {
		// Unknown plans fail closed. Every reader is forced to handle this.
		return Plan{}, fmt.Errorf("unknown plan: %s", id)
	}
	return p, nil
}
```

---

## Notice what this file does *not* do

- It does **not** branch on the user, the time, the tenant, or any A/B
  flag. Those decisions live one layer up.
- It does **not** persist anything. Plan changes happen by code review,
  not by writing to a database.
- It does **not** define what each capability *means*.
  `ai_assist = true` gates a feature; the feature itself lives elsewhere
  and stays at its own appropriate level.
- It does **not** carry pricing, currency, or anything customer-facing.
  Pricing belongs in a separate file with its own review path.
