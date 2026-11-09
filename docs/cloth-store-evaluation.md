# Cloth store pack: what was explored, and why it was stopped

**Date:** 7–10 November 2026
**Fork point:** `OrgoAI/bops` at `1e1beae` (Release v0.0.50)
**Status:** stopped. No code from this effort exists any more, here or anywhere else.

This file is a record for whoever picks the idea up again. It is deliberately
blunt about what was found, so the same ground does not have to be covered twice.

## What was being built

A cloth store pack on top of Bops: a seeded team of bots with built-in skills
and scheduled routines, for one shop that sells both over the counter and
online out of a single stock pool.

Three bots were designed:

| Bot | Role | Scope |
| --- | --- | --- |
| Ledger | Accountant | Daily close, payout reconciliation, supplier bills, sales tax, monthly close |
| Rack | Store Keeper | Stock health, channel reconciliation, markdown review, purchase orders, intake |
| Reach | Digital Marketing | Drops, posts, replies, ad reporting across Instagram, Facebook and TikTok |

Decisions taken along the way: the e-commerce platform was kept vendor-neutral;
accounting was Google Sheets only, with no QuickBooks or Xero; social access was
to go through each bot's own browser rather than Composio's connected apps.

Twenty skill bodies, three routine templates, two triage tags and a shop
owner's testing guide were written and reviewed. The accountant was finished
first and verified; the store keeper and marketing parts stayed behind an
export gate so they could not rot while they waited.

**None of that survives.** The work sat on a local branch that was never
pushed, and the clone was deleted on 10 November 2026 at the owner's request.
Everything below is what was learned, not what was kept.

## Why it was stopped

The pack's dependency on Orgo could not be removed for the part that mattered
most to the shop.

Bots in Bops do browser work on their own cloud computer, which is an Orgo VM.
Reading and writing Google Sheets does not need one: the `use_app` tool runs the
Composio SDK inside the server process (`lib/server/composio.ts`, the
`COMPOSIO_API_KEY` branch of the client). So the accountant and the store keeper,
both of which only ever touch spreadsheets and mail, can run with no Orgo
account at all.

Marketing cannot. Posting to Instagram, Facebook and TikTok through the bot's
own browser is browser work by definition, so it needs the VM. There is no way
around that short of replacing the whole approach with platform APIs, which was
explicitly rejected earlier in the design for being vendor-locked and
permission-heavy.

Weighed against one shop's bookkeeping, maintaining a self-hosted Bops install,
an OpenAI key and a Composio connection was judged more liability than the
automation was worth.

## Findings worth keeping

These were each verified against the code or a running server, not assumed.

### Orgo is not required for chat-only bots

The only hard gate is the sign-in screen. `components/app/bops-app.tsx` sends the
user to `SignIn` whenever `authStatus()` reports `needsSignIn`, and
`lib/server/orgo-sign-in.ts` only clears that for a self-hoster who has
`ORGO_API_KEY` set. Widening that one condition to accept a second environment
variable beside `BOPS_SELF_HOSTED=1` was enough to open the app with no Orgo
account whatsoever.

With that done, on a dev server with no Orgo credentials:

- `GET /api/auth/status` answered `needsSignIn: false`; with the flag turned off
  and nothing else changed, it answered `true`. The flag was the cause.
- The install opened seeded with the main bot and the accountant, each with its
  own chat.
- A message to the accountant reached the model and came back with
  `No OpenAI key is set: add OPENAI_API_KEY to .env.local.` — the only thing
  missing was the model key.

Everything else that wants Orgo disables itself rather than failing: the cloud
is off for a self-hoster, the relay returns early with no key, and the model is
called directly with `OPENAI_API_KEY`. Per-user paths fall back to `.data/` and
`~/.bops/` when nobody is signed in, which the comments in
`lib/server/user-paths.ts` state outright.

### Starter routines need no computer

This was the finding that changed the picture, and it was found late. Routines
have two execution modes. `lib/server/routines.ts` sends a routine with
`run: "chat"` to a single in-process turn of the bot with its tools and no
computer; anything else starts a task thread, which wants a VM.

Every routine made from a starter template is created with `run: "chat"` — see
the `createRoutine` call in `acceptStarters`, `lib/server/routine-suggestions.ts`.
`needsNoComputer` in `lib/types.ts` is what the rest of the app reads.

Confirmed live: a starter accepted for the accountant was stored with
`run: "chat"`, and running it on demand failed only for want of the OpenAI key
while creating no session at all. An earlier reading of the task-routing code in
`lib/server/sessions.ts` had suggested routines always go to the cloud. That
reading was wrong, because it skipped how starters are created.

### Starters can be scoped to one bot

Templates in `lib/routine-catalog.ts` carry no notion of which bot they are for,
so every template for an app is offered to whichever bot that app was connected
for. A pack that adds several bots and several templates for the same app will
offer all of them to one bot unless a scope is added. Keying the offer on
`Bot.role` worked, and held up live: with three templates added for Google
Sheets and one accountant in the install, only the accountant's was offered.

Scope on the role, not the bot id. Ids are derived from names and get a suffix
when they collide (`freeBotId` in `lib/server/bots.ts`), while a role set by a
pack stays put.

### Tool limits that bite

Two tools were misused in draft skill text before being checked against their
implementations:

- `draft_reply` (`lib/server/reply-drafts.ts`) takes `app` from an enum of Gmail,
  Slack and Discord only, and needs ids that came from a read in the same turn.
  It cannot draft a reply for an arbitrary app.
- `orbit_issue_open` (`lib/server/orbit.ts`) is strict, with every property
  required and two of them nullable.

Any pack whose skills name tools should check each signature before shipping
the prose.

### The skill word rules are enforced by a test

`scripts/test-builtin-skills.mjs` rejects snake_case outside known tool names,
dashes used as punctuation, protocol URLs, a handful of banned words, bodies
over sixty lines and descriptions outside 40–220 characters. It applies to
skill text only, not to code comments. `CONTRIBUTING.md` does not carry these
rules, so the test is the only statement of them.

### Test scripts break on a path with a space

Ninety-four scripts under `scripts/` take their root from
`new URL("..", import.meta.url).pathname` and hand it to `pathToFileURL`, which
encodes a path that is already encoded. Any checkout whose path contains a space
crashes them. Scripts that use `fileURLToPath` are fine. The cheap fix is to
keep the checkout somewhere without spaces; the real fix is to change the
ninety-four.

Separately, `scripts/test-bot-groups.mjs` must run without
`--conditions=react-server`, or React's named exports fail to resolve.

## If someone resumes this

1. Decide about marketing first, not last. If social posting is wanted, an Orgo
   account is part of the cost from day one. If it is not, the accountant and
   store keeper can run with an OpenAI key and a Composio key and nothing else.
2. Patch the sign-in gate behind its own environment variable and require
   `BOPS_SELF_HOSTED=1` beside it, so a hosted multi-user server can never be
   opened by setting one variable.
3. Set `role` on routine templates before adding a second bot that shares an app.
4. Write the skill prose last and review it by walking real numbers through it.
   Every defect that mattered in this effort was arithmetic or wording inside a
   skill body, and the test suite was green through all of them. Tests catch
   shape; they do not catch a sentence that says "counted less the sheet figure
   less those sales" when it means "counted less the difference".

## Licence

Upstream is FSL-1.1-ALv2: running a fork for one's own shop is fine, a competing
hosted product is not, and the licence converts to Apache 2.0 after two years.
