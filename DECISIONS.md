# Engineering decisions — AI lead qualification pipeline

The load-bearing tradeoffs in this workflow, and where each would change under real
production load. The theme mirrors the sibling `tessera-ops-agent` repo: this is a
portfolio demo, so the calls favor something small, legible, and importable-in-one-
click over the heaviest available tool — with the honest limitations stated.

---

## 1. n8n (visual workflow) over a code-first service

**Decision.** The pipeline is an n8n workflow — a webhook, an HTTP request, an LLM
node, a Code node, an IF, and two destination nodes — not a Next.js route or a small
Python/Node service.

**Why.** The whole job is *glue between SaaS systems*: receive a webhook, call one
HTTP endpoint, call one LLM, append to Google Sheets, post to Slack. n8n gives all
of those as first-class nodes with managed OAuth credentials, retries, and a visual
canvas a non-engineer client can read and screenshot. Writing this as code would
mean hand-rolling Google Sheets and Slack auth, a web server, and deployment — more
moving parts for a flow whose value is the wiring, not any one step. The canvas is
also the deliverable: a prospect can *see* the qualification logic without reading
source. The one genuinely custom step (normalizing the LLM output) drops into a Code
node, so I don't lose code where code is actually warranted.

**When I'd do it differently.** Once the logic outgrows glue — many branches, shared
libraries, unit-tested business rules, complex retries/backoff, or a team that
reviews changes via pull requests — I'd move it to a versioned code service (the
webhook + an LLM SDK call) and keep n8n only for the parts that are still pure
integration. n8n workflows are JSON, but they diff and review far worse than code.

---

## 2. Standalone OpenAI node with structured JSON + an IF, not the AI Agent node

**Decision.** Qualification is a single OpenAI node (`@n8n/n8n-nodes-langchain.openAi`,
"Message a Model") with **Output Content as JSON** on, prompted to return
`{ score, tier, reasoning }`. A Code node normalizes it and a plain IF routes on
`score ≥ 8`. No AI Agent node, no tools, no memory.

**Why.** This is a one-shot classification with a fixed, known output shape — not an
agent that needs to plan, call tools, or loop. The AI Agent node brings an
agent-executor, tool-calling scaffolding, and a chat-model sub-node for a task that
is "read this, return three fields." A single completion with `response_format:
json_object` is cheaper, faster, fully deterministic in its *shape*, and trivial to
reason about: the downstream IF reads one number. Keeping the routing in a native IF
(rather than asking the model to "decide" and trusting free text) means the
hot/warm/cold branch is enforced by n8n, not by prose.

**Robust parsing is the catch, so it's handled explicitly.** JSON-mode still returns
a *string* the node parses, and models occasionally wander (a stray field, `score`
as a string, a missing tier). The Code node therefore parses defensively — accepts
an object *or* a string, coerces and clamps `score` to 1–10, and re-derives `tier`
from the score if the model's tier is missing or invalid. A bad response degrades to
`score: 0` (cold) instead of throwing mid-workflow. The IF uses **loose** type
validation so `"8"` still compares numerically. That normalizer is exactly the seam
I'd expand — with a JSON schema / structured-output guarantee — before trusting this
in production.

**When I'd do it differently.** If qualification needed to *act* — look up the
company in a CRM, enrich from an external data provider, or ask follow-up questions —
the AI Agent node (or a code agent) with real tools would earn its complexity. At
"score these three fields," it wouldn't.

---

## 3. Enrichment = one homepage fetch, scoped deliberately narrow

**Decision.** Enrichment is a single HTTP GET of the lead's own homepage, capped at
3000 characters into the prompt, with failures swallowed (`neverError` +
`onError: continueRegularOutput`).

**Why.** The goal is a cheap context signal — "does this company's own site look like
a real ICP fit" — not a data-mining operation. One GET of a page the lead themselves
volunteered is low-risk, fast, and needs no third-party scraping infrastructure. I
cap the HTML at 3k chars because homepages are mostly boilerplate and I'm paying per
token; the top of the page carries the signal. And I make the fetch non-fatal on
purpose: a lead who left `website` blank, or a site that's slow or blocks the
request, must still get qualified on their form fields — enrichment is an
enhancement, never a dependency. I frame this as "enrichment, not scraping"
everywhere because that distinction is real and clients ask about it: no headless
browser, no proxies, no crawling internal pages, no defeating rate limits.

**When I'd do it differently.** For richer firmographics (headcount, funding,
industry) I'd call a purpose-built enrichment API (Clearbit/Apollo-style) rather than
parse raw HTML — more reliable, structured, and legally cleaner than scraping. If I
did need on-page content at scale, I'd strip HTML to text server-side and respect
`robots`/rate limits, which the current single-GET approach sidesteps by staying tiny.

---

## 4. Webhook security — unguessable but unauthenticated (stated limitation)

**Decision.** The trigger is a plain n8n webhook at a hard-to-guess path
(`/webhook/tessera-lead-qualify`). It has **no authentication** on the request.

**Why.** For a demo this is the honest default: the URL is effectively a bearer
secret — unguessable, not enumerable — which is enough to keep casual traffic out.
Adding auth is a config choice, not a rewrite, so I keep the demo one import away from
working rather than shipping half-configured credentials. But I won't pretend an
unguessable URL is *access control*: anyone who learns the URL (a browser referrer, a
proxy log, a shared screenshot) can POST to it, and every POST spends an OpenAI call —
so the real backstop against abuse is a **spend cap on the OpenAI key**, the same
honest limit the sibling `tessera-ops-agent` relies on.

**When I'd do it differently.** In production I'd put the webhook behind **Header
Auth** — n8n's webhook node supports a credential that requires a shared-secret
header, so the form backend sends `X-Tessera-Signature: <secret>` and n8n rejects
anything without it — or an HMAC signature of the body, plus per-IP rate limiting at
the edge (Cloudflare) so a leaked URL can't be looped to run up the bill. I'd also
move off `onReceived` to a signed acknowledgement if the form needed proof of receipt.
