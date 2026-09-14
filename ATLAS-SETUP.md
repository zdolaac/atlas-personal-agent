# Atlas — your personal agent (Pluto-style build)

This is a customized copy of Ross Simeles' open-source `eve-agents` starter
(the same base he pointed people to as the entry point for building a
personal agent like his "Pluto"). It's a real Next.js/Eve app, not a mockup —
typecheck passes clean.

## What's already done
- Renamed the agent to **Atlas**, owner set to **Paul** (`agent/lib/owner.ts`)
- Fixed a bug where the memory container tag was hardcoded to the template
  author's name (`"micky"`) — it now derives from the owner name, so your
  memories won't accidentally share a namespace with the template's original
  deployment (`agent/lib/memory-store.ts`)
- Personalized `agent/instructions.md` — this is Atlas's actual system
  prompt, not a placeholder. Edit it directly to change tone, add context
  about Migrate AI, etc.
- `.env.local` scaffolded with identity vars pre-filled; secrets left blank
  for you to fill in

## Requirements on your machine
- **Node 24.x** — the repo's `engines` field enforces this strictly. I ran
  everything here on Node 22 (this sandbox's version) and it typechecked
  fine, but `npm install`/`npm run dev` may behave differently on 24 since
  that's what `eve` and `@agent-browser/eve` actually declare support for.
  Use `nvm install 24` if you don't already have it.

## Accounts you need to create (in rough priority order)

| Capability | Account | What it gives you |
|---|---|---|
| Core chat + threads | **Vercel** (free tier works for dev) | Hosting, AI Gateway (model billing routes through here — no separate Anthropic/OpenAI key needed) |
| Database | **Neon** (free tier) | Postgres for threads, reminders, webhooks, receipts |
| Long-term memory | **Supermemory** | Backs `remember`/`forget`/`search_memory` and the auto-profile injected each turn |
| App integrations | **Composio** | One MCP connection fronting Gmail, Calendar, Notion, Slack, GitHub, Linear, 1000+ apps — OAuth per app happens conversationally |
| File sharing | **Vercel Blob** | Backs `share_file` and the chat-created skill store |
| Push notifications | none — run `npx web-push generate-vapid-keys` locally | Browser push for proactive reminders/webhooks |
| Telegram (optional) | **BotFather** on Telegram | DM-only channel, allowlisted by your Telegram user id |

Browser control (`@agent-browser/eve`) needs no separate account — it runs
inside the Vercel sandbox and is already wired in `agent/sandbox.ts`.

Not included in this repo, despite being mentioned in the video: **AgentMail**
(agent's own email address) and **agent-card** (agent's own spend card). Those
are separate hosted products Ross wired into his own private tool, not part
of this open-source base. Worth evaluating once the core agent is running,
not before — no code here assumes them.

## Getting it running
```bash
cd eve-agents
npm install
cp apps/eve/.env.local apps/eve/.env.local   # already scaffolded — fill in real keys
npm run dev   # localhost:3000, wait for "[eve:dev] server listening"
```

## Real security work merged in from `neural-code`

The `neural-code` transcript (a from-scratch coding-agent build) drew a hard
line between two layers: a *policy* layer that decides what's worth asking a
human about, and an *enforcement* layer the agent can't argue its way around
even if the policy layer is fooled. Checking Atlas against that line found a
real gap and confirmed a real strength:

**Gap, now fixed:** Eve exposes an `approval` field on tools and MCP
connections (`always()` / `once()` / `never()`), but per Eve's own docs,
"when omitted, tool calls execute without approval." None of Atlas's tools
had it set — including `connections/composio.ts`, the single gateway to
1000+ external apps (Gmail, Calendar, Notion, Slack, GitHub, Linear). That's
structurally the same shape as the SerpApi incident: an agent taking a real
external action with nothing checking it first. Fixed by adding `approval`
to:
- `connections/composio.ts` — `always()`, blanket, since Composio's actual
  action set is discovered at runtime and can't be split into safe-read vs.
  risky-write from here
- `tools/create_webhook.ts`, `tools/share_file.ts` — `always()`: each call
  creates a new externally-reachable artifact (a URL anyone can hit)
- `tools/forget.ts`, `tools/delete_receipt.ts` — `always()`: irreversible
  deletions where the harm is silent until the missing data is needed
- `tools/delete_webhook.ts` — `once()`: shrinks attack surface rather than
  growing it, so lower stakes, but still worth one confirmation per session

Typechecked clean against Eve's actual compiled types — this isn't
speculative, it compiles.

**Already strong, confirmed rather than changed:** Eve's Vercel Sandbox
backend does *not* inherit the app process's environment variables unless
explicitly passed via `opts.env` — `agent/sandbox.ts` doesn't do that, so
`COMPOSIO_API_KEY` / `SUPERMEMORY_API_KEY` / `DATABASE_URL` are structurally
absent from the browser sandbox by default, the same way `neural-code`'s
kernel sandbox denies-by-default. Worth guarding, not fixing: if you (or a
future agent-written patch) ever add `env: process.env` to that sandbox's
create options to "make things easier," that quietly undoes this. A strict
network allow-list (the other half of `neural-code`'s sandbox) doesn't map
cleanly onto Atlas, though — the whole point of the browser tool is open
web access, so denying egress by default would break the feature it exists
for. Not adopting that piece was a deliberate choice, not an oversight.

## The foundation-machine layer (subagents)

Everything above is Atlas manufacturing its own tooling (`create_skill`) -
new skills for the *same* running instance. That's tier 1 of the
manufacturing analogy collapsed into tier 3: one machine that grinds its
own replacement parts. What was missing was a foundation machine that
stamps out genuinely *distinct* product-machine instances - separate
agents with their own restricted tools and no inherited access.

Eve has this as a real, compile-time-authored primitive:
`agent/subagents/<id>/` - its own `agent.ts`, `instructions.md`, and
`tools/`, discovered and compiled into a distinct graph node with **no
inheritance** from the parent (confirmed directly in Eve's own type
definitions: the same non-inheritance rule that governs per-subagent
sandboxes). The parent calls it through the `task` tool, using its
`description` as what the parent sees.

**Built and included:**
- `agent/subagents/researcher/` - the first real subagent. Read-only:
  one tool (memory search, reusing the parent's own `memoryStore` rather
  than a separate implementation), no write/delete/webhook/Composio
  access - not because it's told not to use them, but because those
  tools simply do not exist in its folder. Same structural guarantee as
  `neural-code`'s subagent isolation.
- `scripts/scaffold-subagent.ts` - the actual foundation-machine tool.
  Run it with an id and a description; it writes a new
  `agent/subagents/<id>/` scaffold with an empty `tools/` folder (empty
  is the deliberate safe default - you add exactly what that subagent
  needs, then it's a design decision, not a leftover).
  ```bash
  cd apps/eve
  npx tsx scripts/scaffold-subagent.ts pr-reviewer \
    --description "Reviews a diff against project conventions and returns a risk assessment." \
    --model "anthropic/claude-sonnet-5"
  ```

**Honest limit on verification:** I confirmed the file layout and API
match Eve's documented convention exactly, and `tsc` passes clean against
Eve's real exported types. I could **not** confirm Eve's actual
discovery/compiler pass picks up `agent/subagents/researcher/` into the
compiled graph - the `eve` CLI refuses to run below Node 24 (this sandbox
has 22), and a full `next build` failed here on an unrelated network
restriction (blocked Google Fonts fetch, not anything about the
subagent). Run `npm run build` on your own machine (Node 24) before
trusting that the researcher subagent is actually callable - this is
exactly the "verified against real behavior vs. assumed" distinction
worth checking before treating it as done.

**Deliberately not made dynamic:** Atlas cannot scaffold a new subagent
on its own initiative - `scaffold-subagent.ts` is a script you run, not
an Atlas tool. A new subagent changes what the whole system can do; that
belongs behind a human running a command and reviewing a diff before
redeploying, not behind an agent deciding to expand its own capability
graph.

## Remote machines (the "Mac Mini" pattern) and the "lavish" review board

Two things from the firstmate transcript, checked separately:

**Cross-machine dispatch - built, and it's not from firstmate's GitHub.**
firstmate's local-first-mate-dispatches-to-a-Mac-Mini-secondmate pattern
is built on SSH, tmux, and persistent git worktrees - specific to
spawning independent CLI-harness processes on a remote host. That
mechanism doesn't transplant onto Eve's serverless architecture, so
nothing was ported from firstmate's repo for this. Instead, Eve has its
own native primitive for exactly this - `defineRemoteAgent` - confirmed
against Eve's own bundled guide
(`node_modules/eve/docs/guides/remote-agents.md`), not assumed:

- `agent/subagents/mini.ts` - a real, typechecked remote-agent
  definition. Reads `MINI_AGENT_URL`/`MINI_AGENT_TOKEN` at runtime (not
  compiled in - the whole point is the target doesn't exist yet), and
  throws a clear error immediately if either is unset rather than
  failing silently.
- `scripts/scaffold-subagent.ts --remote-url <ENV_VAR_NAME>` - the
  generator now stamps out a remote-agent file, not just local ones.
  Dry-run confirmed it writes working output.

Architecturally this is arguably stronger than firstmate's version: eve's
remote dispatch is durable and callback-based at the framework level -
the parent turn parks without holding compute while the remote works,
and resumes on a posted-back callback - so a parent restart or redeploy
mid-dispatch doesn't lose the in-flight call the way killing an SSH
session would.

**What's still missing to actually use it:** something running on the
Mac Mini that speaks eve's `/eve/v1/session` protocol - i.e., an actual
second eve deployment out there (self-hosted, Docker or microsandbox
backend rather than Vercel's, since the Mini isn't a Vercel deployment).
`mini.ts` is the calling side; the receiving side doesn't exist yet.

**"Lavish" - not available from the GitHub you gave me, because it isn't
there.** I checked directly: firstmate's own repo doesn't contain
lavish's source. `bin/fm-procevent-lavish.sh` is just a thin adapter that
polls a separate, presumably proprietary `lavish-axi` CLI - the docs
call it "a presentation-only dependency," installed and versioned
independently, with firstmate falling back to plain text when it's
unavailable. There's nothing to port; the actual interactive review-board
tool isn't open source in the repo you linked. I didn't build a
substitute for this since it wasn't the confirmed part of the ask - happy
to design one (most naturally as a page in Atlas's own Next.js app,
rendering a subagent's proposed options with buttons that post feedback
back to that subagent's thread) if you want it.

## Webhook hardening (from *Mastering API Architecture*)

Cross-checked Atlas's actual webhook receiver (`agent/channels/hooks.ts`)
against a real API-architecture reference (Gough/Bryant/Auburn,
*Mastering API Architecture*, ch. 3: "Protect APIs from Overuse and
Abuse" and "API Lifecycle Management"). Confirmed real strengths already
in place - constant-time secret comparison, a single generic 404 for both
unknown-id and bad-secret (no probing signal), payload size capping - and
two genuine gaps the book names directly, now fixed:

- **No rate limiting.** Every valid POST woke the agent - a real model
  call - with zero throttling. A leaked secret or a sender's retry storm
  had no ceiling. Fixed: `MIN_SECONDS_BETWEEN_FIRES = 10` in
  `agent/lib/webhooks-db.ts` - a throttled fire is acked (200, so senders
  don't retry-storm harder) but never wakes the agent, and is counted
  separately (`throttled_count`) rather than silently absorbed into the
  same `fire_count`, so a webhook being hit too often is visible via
  `list_webhooks`, not hidden.
- **No expiry ("Retirement" in the book's API lifecycle).** A webhook
  lived until someone remembered to call `delete_webhook` - no built-in
  end date for something that only ever needed to exist for one event
  window. Fixed: `create_webhook` now takes an optional `expiresInHours`;
  an expired hook returns 410 Gone and deletes itself on the next hit
  (lazy cleanup, no cron needed). Omitting it preserves the old
  indefinite-lifetime behavior exactly - not a breaking change.

Schema changed (`throttled_count`, `expires_at` columns) via
`ALTER TABLE ... ADD COLUMN IF NOT EXISTS`, not a separate migration file
- matching how this single-table store already handled its own schema,
and safe to run against an existing deployment's data.

**Verification, and its real limit:** typechecked clean against Eve's
actual types. The throttle/expiry decision logic (`isThrottled`,
`isExpired`) was verified for real - six assertions against actual
`Date` math, not just read and trusted - confirming a hook fired 2
seconds ago is throttled and one fired 15 seconds ago isn't, an expiry
1 second in the past trips and one an hour out doesn't. What's *not*
verified: the actual deployed handler end-to-end, since that needs a
live Postgres connection and a real `npm run build` this sandbox doesn't
have - the same limit as every other Atlas change this session, named
plainly rather than glossed over.

## Memory search relevance floor (from *Hands-On Large Language Models*)

Checked Atlas's `search_memory` against Alammar & Grootendorst's dense-
retrieval chapter, which names this directly: "it's sometimes desirable
to have a max threshold of similarity score to filter out irrelevant
results." Confirmed the gap for real - `memoryStore.search()` already
gets a `similarity` score back from Supermemory (`searchMode: "hybrid"`)
but nothing was filtering on it; every result, however weak the match,
was returned to the model with a number attached that nothing acted on.

Fixed: `memoryStore.search()` takes an optional `minSimilarity`,
undefined by default (existing callers keep their exact current
behavior - not a breaking change). Both `search_memory.ts` (main tool)
and the `researcher` subagent's copy now pass `MIN_SIMILARITY = 0.15`.

**Stated honestly, not glossed over:** that `0.15` is an unvalidated
guess, not a tuned value - Supermemory's hybrid-search score scale and
real distribution were never measured against actual query/result pairs
from this deployment. Both tool files say this in a comment. If memory
search starts feeling like it's missing things that should have matched,
suspect this constant first, not the underlying search. Verified only
the filter logic itself for real (a small standalone script confirming
`>=` semantics and the undefined-means-unfiltered default) - not the
live Supermemory score distribution, which needs real usage to observe.

## What to decide before deploying live
- **Sub-agent workflows** (`agent/tools/workflow.ts`, capped at 10 fan-out) —
  useful later if you want Atlas coordinating multiple sub-tasks, no setup
  needed beyond what's already wired.
- **Skill authoring at runtime** — Atlas can already write/delete its own
  skills via chat (`create_skill`/`delete_skill` tools), stored in Vercel
  Blob. This is a much lighter mechanism than your Migrate AI `.agents/`
  file-based skill system — don't conflate the two; this is for Atlas's own
  personal routines, not Migrate AI's SKILL.md architecture.
- **Multi-thread / group-chat parity with Pluto** — threading, rename/pin/
  delete, and forking a thread from any message are already built into the
  web chat UI (`app/chat.tsx`). No extra work needed for that specific gap
  the video's Grok Bot comparison flagged.

## Honest gaps versus what the video showed
- No voice interface (the "Hey Ruth" demo) — this repo is text-chat only.
- iMessage: not present here; the video mentioned it as "coming soon" on
  Pluto specifically, not as a feature of this open-source base.
- Multi-agent orchestration ("Percy" as a distinct sub-bot) isn't a separate
  agent definition in this repo — it's one agent (Atlas) with the `workflow`
  tool for fan-out, not literally spinning up named child agents with their
  own identities. If you want that, it's a real extension, not something
  already wired.
