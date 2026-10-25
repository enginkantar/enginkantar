# BizOS — Rules-Driven Multi-Agent Business Operating System
### Design Document v1.0 · Engin Kantar · Claude Impact Lab, Seattle

> **One-liner:** GStack gives software teams a constitution (CLAUDE.md) that every AI coding agent obeys. **BizOS gives a small business the same thing** — every department runs on an AI agent that loads its rules from a fixed, human-editable file. The owner writes rules once; the AI staff follows them forever.

---

## 1. The Problem (lived, not imagined)

Small business owners are the router of their own company. Every email, DM, invoice question, price request, review, and new-hire question lands on **one person**. Software exists for each task, but nothing *decides, routes, and answers* the way a trained employee would — because employees are trained on **the business's own rules**, and AI tools ship with none.

**Insight:** The missing piece is not a smarter model. It is a **place to write the rules down** in a form AI can obey — per department, versioned, owned by the business.

---

## 2. Core Concept — Rules-as-Files

Each business domain gets one plain-text rules file. The file **is** the employee handbook, the SOP, and the system prompt, all at once.

```
/business/
├── company.rules.md          # identity, tone, legal lines never to cross
├── accounting.rules.md       # invoicing, refunds, payment terms, tax notes
├── sales.rules.md            # pricing, discounts, quote limits, upsells
├── support.rules.md          # SLAs, refund policy, escalation triggers
├── media.rules.md            # brand voice, posting cadence, banned topics
├── hr.rules.md               # roles, shifts, onboarding, policies
└── web.rules.md              # public-safe subset (what the website AI may say)
```

### Rules file anatomy (every file, same skeleton)

```markdown
---
domain: accounting
owner: engin@company.com
version: 3            # bump on every edit — agents log which version they used
updated: 2026-10-24
escalate_to: "+1-206-XXX-XXXX"   # human fallback
---

## IDENTITY
You are the accounting department of <Company>. Formal, precise, never guess numbers.

## HARD RULES (never break — refuse and escalate instead)
- Never quote a refund above $200 without human approval.
- Never share supplier cost prices.
- Invoices are NET-30; late fee 1.5%/month after day 45.

## SOFT RULES (defaults — may adapt with judgment)
- Prefer offering store credit before cash refund.
- Round quotes to the nearest $5.

## KNOWLEDGE (stable facts the agent may state — live data NEVER lives here)
- Payment methods: card, ACH, Zelle.
- Tax: per our accountant's current WA guidance (reviewed quarterly — a
  perfectly obeyed wrong rule is still wrong, so every file carries a
  `review_by` date and flags itself when stale).
- Live facts (stock, prices, order status, balances) come from the systems
  of record via per-domain tools — rules files carry POLICY only.
- Shared policies (e.g. refund policy) live ONCE in company.rules.md;
  domain files reference it, so Support and Accounting can never diverge.

## ESCALATION TRIGGERS
- Customer mentions lawyer, chargeback, or fraud → human, immediately.
- Any amount > $500 → draft reply, hold for approval.
```

> **Enforcement note (v1.1):** HARD RULES are not enforced by the model *reading* them. They are compiled into a **Policy Gate** — deterministic code checks (amount caps, data-access scopes, send permissions) that run on every outbound action. **The model proposes; the gate disposes.** Rule text is the source of truth; code is the enforcer. A rule citation proves the agent *referenced* a rule — the gate proves the action *obeyed* it.

**Why files, not a database UI:** the owner edits Markdown (or dictates changes to the Orchestrator: *"from today, free shipping over $75"* → agent patches the file, bumps version, logs the diff). Git-style history = free audit trail. This is exactly the CLAUDE.md pattern, applied to business ops.

---

## 3. Architecture

```
            ┌──────────────────────────────────────────────┐
            │            CHANNEL ADAPTERS (in)             │
            │  Email · WhatsApp · Web form · DM · Phone    │
            └───────────────────┬──────────────────────────┘
                                ▼
                 ┌─────────────────────────────┐
                 │   UNIVERSAL WEBHOOK (ingress)│   POST /api/inbox
                 │   channel-agnostic envelope  │
                 └──────────────┬──────────────┘
                                ▼
                 ┌─────────────────────────────┐
                 │       ORCHESTRATOR AI        │  loads company.rules.md
                 │  classify → route → track    │  never answers domain Qs itself
                 └──┬────┬────┬────┬────┬──────┘
                    ▼    ▼    ▼    ▼    ▼
              ┌────────┐┌──────┐┌───────┐┌──────┐┌──────┐
              │ACCOUNT.││SALES ││SUPPORT││MEDIA ││ HR / │
              │ agent  ││agent ││ agent ││agent ││ACADEMY│
              └───┬────┘└──┬───┘└──┬────┘└──┬───┘└──┬───┘
                  │  each loads ONLY its own rules file │
                  └────┬──────┴──────┴────────┴────────┘
                       ▼
                 ┌─────────────────────────────┐
                 │  POLICY GATE (code, not AI)  │  hard limits enforced HERE:
                 │  amount caps·scopes·permissions│ model proposes, gate disposes
                 └──────────────┬──────────────┘
                                ▼
                 ┌─────────────────────────────┐
                 │  APPROVAL QUEUE (if required)│  approval binds to a draft
                 │                              │  version; any edit → re-approve
                 └──────────────┬──────────────┘
                                ▼
                 ┌─────────────────────────────┐
                 │   RESPONSE BUS (egress)      │  reply exits on the SAME
                 │                              │  channel it came from
                 └──────────────┬──────────────┘
                                ▼
                 ┌─────────────────────────────┐
                 │  AUDIT LOG (every step above)│
                 └─────────────────────────────┘

**Order is deliberate (v1.1):** draft → policy gate → approval (when required) → send. Nothing reaches a customer before the gate and, where tiered, the owner.
```

**Design lineage (worth saying in the pitch):** this is the IBM MQ + CICS pattern from core banking, rebuilt with AI. The trigger monitor became the Orchestrator; CICS transactions became specialist agents; the reply queue became the response bus. The pattern has run the world's banks for 40 years — we're giving it to a coffee shop.

---

## 4. The Channel-Agnostic Envelope

Every message, from any channel, is normalized into **one JSON shape** at ingress — and every reply leaves in the same shape. Adapters translate; the core never knows or cares which channel it serves.

```json
// INBOUND  → POST /api/inbox
{
  "message_id":  "uuid",
  "source":      "whatsapp | email | webform | instagram | phone",
  "reply_to":    "channel-native address (phone, email, thread id)",
  "customer":    { "name": "…", "known": true, "history_ref": "crm:123" },
  "content":     { "type": "text|image|file", "text": "…", "attachments": [] },
  "received_at": "ISO-8601"
}

// ORCHESTRATOR VERDICT (internal)
{
  "message_id": "uuid",
  "domain":     "accounting | sales | support | media | hr | unknown",
  "confidence": 0.93,
  "urgency":    "normal | high | critical",
  "route":      "auto_reply | draft_for_approval | human_now"
}

// OUTBOUND ← response bus
{
  "message_id":   "uuid",
  "in_reply_to":  "original message_id",
  "domain":       "accounting",
  "rules_version": 3,
  "action":       "sent | held_for_approval | escalated",
  "content":      { "type": "text", "text": "…" },
  "audit":        { "rule_citations": ["HARD-2", "KNOW-1"], "model": "…", "latency_ms": 1240 }
}
```

Key properties (v1.1):
- **Case model:** every message attaches to a persistent `case_id` (conversation + pending transactions). One request may spawn tasks across several domains ("order is late → cancel → or replace and quote the price difference" touches Support + Accounting + Sales), but **one Case Owner speaks to the customer**; specialists return results to it — no contradictory replies.
- **Real idempotency:** the channel-native source message id is stored UNIQUE in Postgres (Redis is only a fast cache, not the guarantee). **Transaction ids are separate from message ids**, and external calls (e.g. the refund API) carry the same idempotency key — a crash between "refund executed" and "result recorded" can never double-refund on retry. Failed jobs land in a durable queue with retry + dead-letter review.
- **Traceable:** `rules_version` + `rule_citations` + gate verdict on every action — the compliance story.
- **Channel-symmetric:** the reply exits where the question entered.

---

## 5. Agent Specifications

### 5.1 Orchestrator
- Loads `company.rules.md` only. **Routes; never answers domain questions.**
- Classification contract: must output the verdict JSON above; `confidence < 0.7` → asks ONE clarifying question on the same channel; still unclear → `human_now`.
- **Rules editing is a separate, privileged path (v1.1):** it can only be started from an authenticated owner session, and the customer-message pipeline has **zero write access** to rules. Flow: owner asks → a proposed diff is shown → owner approves → only then the version bumps and publishes. An agent that reads untrusted customer input must never be the agent that can rewrite company policy — this is the prompt-injection firewall.

### 5.2 Specialist agents (accounting / sales / support / media)
- System prompt = `company.rules.md` + **own** rules file + few-shot examples **tagged with the `rules_version` they were approved under** — examples expire automatically on version bump, so stale examples can never override new policy. (Context isolation: the sales agent physically cannot leak supplier costs it never sees.)
- Tool access is per-domain (accounting → invoice API read; media → post scheduler; support → order lookup).
- Every answer must cite rule IDs it relied on. No matching rule → answer is held as **draft** + flagged "rules gap" so the owner can add the missing rule. **The rulebook grows from real traffic.**

### 5.3 Web Agent (public-facing)
- Separate agent, loads **only** `web.rules.md` — a hand-curated public-safe subset (prices, hours, FAQ). Hard rule: if it's not in the file, say "let me connect you" and open a ticket through the same envelope.
- This is the security boundary: the internet-facing brain holds zero internal knowledge. **And it works both ways (v1.1):** when an internal ticket's result flows back to the web channel, it passes the same `web.rules.md` filter on the way out — a clean first prompt alone is not the boundary.

### 5.4 HR / Academy Agent (employee onboarding & Q&A)
- Same rules files, opposite audience: **answers employees, not customers.**
- New hire asks "how do refunds work?" → answers from `support.rules.md` + `accounting.rules.md`, citing rule IDs → the handbook that's never out of date, because it IS the live rulebook.
- Onboarding mode: generates a day-1 checklist and a 10-question quiz from the rules files; logs completion.
- Flywheel: when an employee asks something no rule covers → "rules gap" ticket → owner adds rule → next hire never asks again.

---

## 6. Human-in-the-Loop & Trust

| Tier | What happens | Example |
|---|---|---|
| `auto_reply` | Agent answers, logs | "What are your hours?" |
| `draft_for_approval` | Reply queued, owner taps ✓/✗ on phone | Refund $120 |
| `human_now` | Push notification, agent holds | "My lawyer will call" |

Owner's daily surface = **one approval queue + one audit feed.** The promise is not "AI replaces you"; it's **"you only see exceptions."**

**Approval SLA (v1.1):** every held item has a priority and a timeout. No owner response in 2h → the customer gets a polite hold message and the reminder escalates (push → SMS); critical items ring the phone. An unanswered queue never silently drops a customer.

**Honest expectation:** in week one, rule coverage is low, so the queue is busy — but approving a ready draft is ~10x faster than composing from scratch, and every rejection writes a rule, so the queue shrinks weekly. **Success metrics:** owner-minutes per day, time-to-resolution, error/reopen rate — not just the auto-reply percentage.

---

## 7. Hackathon MVP (buildable in one day)

**Cut to the bone — demo path:**
1. `/api/inbox` webhook (one endpoint, envelope above) — *2 hrs, pattern already exists in my notification service*
2. Orchestrator classify via Claude → route JSON — *1 hr*
3. TWO specialists only: **Sales** + **Support**, each with a real rules file written live on stage — *2 hrs*
4. **Academy agent** answering "I'm the new hire, how do refunds work?" from the same files — *1 hr, pure prompt*
5. Simple inbox UI (teammate) + approval queue — *parallel*
6. **Policy Gate stub:** one function enforcing the $200 refund cap in code — *30 min*
7. **Demo script:** send a WhatsApp-style message → watch it route → auto-answer with rule citation → EDIT the rules file live, resend, answer changes (*wow #1: edit-and-it-obeys*) → then ask the agent for a $500 refund → **the gate blocks it in code even though the model tried** (*wow #2: AI can be fooled, the gate can't*).

**Stack:** Node/TS or Go webhook core · Postgres (messages, audit) · Redis (idempotency) · Claude API (orchestrator + agents) · rules = plain .md in a repo.

---

## 8. Why This Wins

- **Lived problem:** built by a small-business owner drowning in 8 channels — I am the user persona.
- **Novel primitive:** not "another AI chatbot" — a **rules file format** businesses own, version, and audit. The CLAUDE.md idea, exported from software into Main Street.
- **Trust story judges ask about:** versioned rules + **code-enforced hard limits (Policy Gate)** + full audit trail = the honest promise is not "AI never forgets" but *"rules are versioned, critical limits are enforced in software, and every action is traceable."*
- **Grows itself:** every unanswerable question becomes a rule. The business gets more automated every week it runs.

---

## 9. The Trust Ladder — How the System Earns Autonomy

Owner feedback (✓/✗ on every answer) does **not** retrain the model. It drives two simple, auditable mechanisms instead:

**9.1 Graduated autonomy (the apprenticeship):**
- Day 1: every answer is `draft_for_approval` — like a new hire on probation.
- Promotion: when an intent class hits ≥95% approval over its last 50 answers, it graduates to `auto_reply` — but autonomy is granted per **narrow transaction type × risk tier**, never per department: answering opening hours and committing to a refund are different permissions, and money-moving actions stay behind the Policy Gate regardless of approval streaks.
- Demotion: a rejection on a low-risk class is a signal; a rejection on a money/legal class demotes immediately. Any rules-file or model change resets the evaluation window for affected classes.
- Deterministic thresholds, not ML — the owner can always see *why* something became automatic. The model's self-reported `confidence: 0.93` is a routing hint only, never treated as measured accuracy.

**9.2 Rejections become rules (the real learning):**
- When the owner edits/rejects an answer, the Orchestrator analyzes the diff and **proposes a rule patch**: *"You've corrected refund wording 3 times — add this to support.rules.md?"*
- One tap → rule added, version bumped, audit logged.
- The system learns, but what it learned lives as **readable text in the rules file** — not buried in model weights. Models forget; files don't.

**9.3 Few-shot bonus:** top-approved answers are injected into each agent's prompt as examples — the agent imitates its own best work, no training run required.

**Why not fine-tuning:** a small business generates hundreds of examples, not millions; fine-tuning is costly, drifts, and is unauditable — the opposite of the Rules-as-Files thesis.

*Pitch line: "Your AI staff starts as an intern, earns autonomy answer by answer, and every correction you make becomes a written rule it can never forget."*

---

*"Write your rules once. Your AI staff never forgets them."*
