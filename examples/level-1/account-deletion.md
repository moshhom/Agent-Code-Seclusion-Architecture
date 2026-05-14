# Level 1 — Critical · Account hard-delete

> **ACSA level: 1 (Critical).** Read every line. Do not merge without
> manual review by a human. Agent-only changes are not accepted.

When a user closes their account, or a privacy request is approved, this
function permanently removes their data from the application database.
It is the irreversible step at the end of the deletion pipeline.

## Why this is Level 1

- **Revertability.** None. Once a row is gone, it is gone. Restoring it
  means going back to a backup, identifying just this user's rows in a
  snapshot, and re-inserting them — a manual, error-prone process that
  may also re-create data the user explicitly asked to be removed. This
  is the consequence axis that puts this file at Level 1 on its own.
- **Access.** Holds a DB connection with `DELETE` rights over every
  user-scoped table.
- **Blast radius.** A bug that deletes the wrong user's rows is
  indistinguishable from a hostile data-loss incident.
- **Customer impact.** Total for the affected user.
- **Maintenance lifetime.** As long as the product handles personal data,
  which is forever in practice. New tables that store user-owned rows
  have to be added here by hand.

## What Level 1 treatment looks like for this file

- The caller's confirmation is re-verified against the target `user_id`.
  The fact that it was checked upstream is not enough.
- An audit row is written before any `DELETE`. If anything below fails,
  there is still a record that the attempt happened.
- All deletes run in a single transaction so the user is either entirely
  gone or entirely there. No half-deleted accounts.
- Tables are listed explicitly. No schema introspection, no "delete
  everywhere `user_id` appears". Adding a new user-owned table forces a
  Level 1 change here.

---

## Python

```python
# ACSA Level 1 - Critical
# Read every line. No merge without manual review.
# Rationale: irreversible DB deletion across user-owned tables.

from datetime import datetime

from auditing import record_deletion
from db import execute, transaction
from errors import AuthorizationError


def hard_delete_user(
    user_id: str,
    confirmation: DeletionConfirmation,
    now: datetime,
) -> None:
    # 1. Re-verify the confirmation. Upstream verification is not enough.
    if confirmation.user_id != user_id or confirmation.expires_at < now:
        raise AuthorizationError("invalid deletion confirmation")

    # 2. Audit first. If anything below fails, we still know we tried.
    record_deletion(user_id, confirmation.actor, now)

    # 3. Single transaction. All or nothing.
    with transaction():
        execute("DELETE FROM notes          WHERE user_id = %s", user_id)
        execute("DELETE FROM tags           WHERE user_id = %s", user_id)
        execute("DELETE FROM sessions       WHERE user_id = %s", user_id)
        execute("DELETE FROM billing_states WHERE user_id = %s", user_id)
        execute("DELETE FROM users          WHERE id      = %s", user_id)
```

## TypeScript

```ts
// ACSA Level 1 - Critical
// Read every line. No merge without manual review.
// Rationale: irreversible DB deletion across user-owned tables.

import { transaction } from "./db";
import { recordDeletion } from "./auditing";
import { AuthorizationError } from "./errors";

export async function hardDeleteUser(
  userId: string,
  confirmation: DeletionConfirmation,
  now: Date,
): Promise<void> {
  // 1. Re-verify the confirmation. Upstream verification is not enough.
  if (confirmation.userId !== userId || confirmation.expiresAt < now) {
    throw new AuthorizationError("invalid deletion confirmation");
  }

  // 2. Audit first. If anything below fails, we still know we tried.
  await recordDeletion(userId, confirmation.actor, now);

  // 3. Single transaction. All or nothing.
  await transaction(async (tx) => {
    await tx.execute("DELETE FROM notes          WHERE user_id = $1", [userId]);
    await tx.execute("DELETE FROM tags           WHERE user_id = $1", [userId]);
    await tx.execute("DELETE FROM sessions       WHERE user_id = $1", [userId]);
    await tx.execute("DELETE FROM billing_states WHERE user_id = $1", [userId]);
    await tx.execute("DELETE FROM users          WHERE id      = $1", [userId]);
  });
}
```

## Go

```go
// ACSA Level 1 - Critical
// Read every line. No merge without manual review.
// Rationale: irreversible DB deletion across user-owned tables.

package accounts

import (
	"context"
	"time"

	"example.com/notes/auditing"
	"example.com/notes/db"
	"example.com/notes/errors"
)

func HardDeleteUser(
	ctx context.Context,
	userID string,
	confirmation DeletionConfirmation,
	now time.Time,
) error {
	// 1. Re-verify the confirmation. Upstream verification is not enough.
	if confirmation.UserID != userID || confirmation.ExpiresAt.Before(now) {
		return errors.Authorization("invalid deletion confirmation")
	}

	// 2. Audit first. If anything below fails, we still know we tried.
	if err := auditing.RecordDeletion(ctx, userID, confirmation.Actor, now); err != nil {
		return err
	}

	// 3. Single transaction. All or nothing.
	return db.InTx(ctx, func(tx db.Tx) error {
		stmts := []string{
			"DELETE FROM notes          WHERE user_id = $1",
			"DELETE FROM tags           WHERE user_id = $1",
			"DELETE FROM sessions       WHERE user_id = $1",
			"DELETE FROM billing_states WHERE user_id = $1",
			"DELETE FROM users          WHERE id      = $1",
		}
		for _, stmt := range stmts {
			if _, err := tx.Exec(ctx, stmt, userID); err != nil {
				return err
			}
		}
		return nil
	})
}
```

---

## Notice what this file does *not* do

- It does **not** decide whether the user *should* be deleted. That
  decision lives upstream and ends with a `DeletionConfirmation`.
- It does **not** discover tables at runtime. Adding a new user-owned
  table requires editing this file.
- It does **not** soft-delete. Soft-delete is a separate file in the
  deletion pipeline; this is the final, hard step.
- It does **not** notify external services (Stripe, email provider, etc.).
  Those are sibling Level 1 files that each carry their own review path.
