# Agent Code Seclusion Architecture (ACSA)

> **Status: DRAFT — first scope.** Working notes, not a finished spec.
> Expect things to change.

## The problem

One of the hardest parts of working with AI agents is the **volume of
code**. How do you check thousands of generated lines, and actually trust
that what landed is good and safe?

A few practices that help, regardless of architecture:

- A **compact coding rules** markdown file that captures exactly how code
  should be written. Short, opinionated, kept in the repo. It matters more
  than people expect.
- **Audit agents.** Running a second agent over the diff helps a lot.
- **Scan code fast.** Train the eye; don't try to read everything
  word-for-word.
- **Tests, and lots of them** — including AI-written tests.
- For **critical code**, read every line.

ACSA starts from a point that runs through all of those practices, and
through AI-generated code in general: **not every line deserves the same
attention**, and pretending otherwise is how you either burn out or wave
through code you shouldn't.

## The idea

**Agent Code Seclusion Architecture (ACSA)** proposes *another* reason to
structure files and folders, on top of the usual ones (code responsibility,
business model, domain boundaries, etc.). This one takes AI agents into
account, and divides code into four trust levels.

### The four levels

| Level | Name             | What you do                                       |
| ----- | ---------------- | ------------------------------------------------- |
| 1     | Critical         | Read **every line**.                              |
| 2     | Very important   | Check the structure. Anything wrong → redo.       |
| 3     | Regular          | Check the overall structure.                     |
| 4     | Low impact       | Don't look at all.                                |

### How to decide the level

This is the part that gets misunderstood. **Complexity is not part of the
equation.** A gnarly, hard-to-read function can sit at level 4. A five-line
config-loader can sit at level 1.

What decides the level is **consequences**:

- What happens if this code doesn't work?
- What access does this code have? (secrets, prod data, money, identity,
  external network, file system, customer-facing surface...)
- How easy is it to revert?
- How hard will it hit customers if it breaks?
- How long will it need to be maintained? *(this one matters more than
  people think)*

The worse the blast radius, the less you trust it, the higher the level,
the more eyes on it.

## What it buys you

- **The volume stops being scary.** Thousands of lines of AI-generated code
  in a project no longer feels like a wall. You know which slices need
  attention and which don't, so you stay in control instead of drowning in
  diff.

- **Review starts from the blast radius, not the code.**
  - *Imports first.* What a file pulls in tells you what it can touch —
    network, filesystem, DB, secrets, payments, identity, other modules.
    Imports decide the level before the logic does.
  - *Close down doors.* Once you know what a file *should* be allowed to
    do, take away everything else: drop unused deps, narrow permissions,
    restrict env vars, hide it behind an interface that exposes only the
    operations it needs, run it where it can't reach what it shouldn't.
    Code that can't see your DB can't corrupt it. Code that can't read
    secrets can't leak them. The damage a file can do is bounded by what
    it can reach, not by whether it's correct — so a wrong, buggy, or even
    hallucinated low-trust file has nowhere to go.
  - *Reject per file, not per PR.* Because levels are tracked at the file
    (or folder) granularity, you don't have to accept or reject a PR as a
    block. Drop the one file you don't like, keep the rest, ask the agent
    to redo just that piece. No more "the PR is mostly fine, ship it."

- **You know what's disposable and what isn't.** Level-4 code can be
  regenerated, replaced, or deleted with almost no thought. Level-1 code is
  treated like load-bearing structure. That distinction has to be explicit
  — otherwise everything quietly drifts toward "important."

- **Complex logic stops being scary too.** If a tangled function lives
  inside a level-4 box with no doors open, it can just work. Complexity
  only costs you when it's wired into something that matters.

- **It pushes you toward separation — logical and physical.** Inside the
  codebase, levels become real boundaries: folder layout, narrow
  interfaces, restricted imports, file-level ownership. At runtime, the
  same boundaries get pushed out into different servers, processes, or
  sandboxes with different responsibilities and different access. The
  separation stops being a convention and starts being enforced — first
  by the structure of the repo, then by the system it runs on. That
  seclusion, on both sides, is where the name comes from.

## What this is *not*

ACSA is **not a replacement** for any other architecture. It's an add-on, a
second axis you overlay on top of whatever design you already use
(hexagonal, layered, DDD, modular monolith, whatever). The regular designs
are needed **more**, not less, when agents are writing code.

## Implementation

ACSA does not prescribe a project layout, a language, a CI setup, or a
runtime model. It is an add-on, so most of the implementation comes from
the other architectures already in use — layered, hexagonal,
ports-and-adapters, modular monolith, whatever the project runs on. The
work is in choosing where the level boundaries sit on top of that
structure, marking them in a way both agents and reviewers can see, and
enforcing them at review time and at runtime.

A pattern that falls out of this naturally: wrap a stretch of lower-trust
code in a frame of Level 1 code. The top of the frame validates and
narrows everything coming in — auth, schema, permissions, input
normalisation. The bottom of the frame validates and gates everything
going out — what is written to the database, what is sent to external
services, what is returned to the caller. In between, the middle can
hold anything that isn't Level 1 — graded as strictly or as loosely as
the work warrants — and stay safe, because anything it produces still
has to cross a Level 1 boundary before it touches anything real.

This is where the regular architectures earn their place. The top of the
frame is the controller, the request validator, the auth middleware, the
input port — names already supplied by whatever design the project uses.
The bottom is the repository, the service boundary, the output port, the
audit logger. ACSA does not invent those layers; it just leans on them,
marks them Level 1, and leaves the work between them looser. The cheapest
way to bring an existing module under ACSA is to leave the middle
untouched and put a frame around it.

### Examples

Small, file-sized examples that show what each level looks like in
practice. The same logic is shown in Python, TypeScript, and Go, with
the level marked three times: in the folder, in the file's header, and
in a banner inside each code block.

- **[Level 1 — Billing webhook handler](examples/level-1/billing-webhook.md).**
  Stripe webhook entry for a tiny notes SaaS. Signature verified first,
  idempotency enforced, dispatch handed off to a narrow billing module.
  Every line read.

More examples (Level 2, 3, and 4) will land here as they're written.

## Status

This repo is the working draft of the scope. It will change. Nothing here
is final.
