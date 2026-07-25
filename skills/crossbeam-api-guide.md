---
name: crossbeam-api-guide
description: >-
  Routes any "how do I get Crossbeam data into X" question to the right access
  method — REST API, Webhooks/Signals, the MCP server, or the in-app AI Chat —
  and points to the exact docs, scopes, and endpoints. Use whenever someone asks
  how to access Crossbeam data programmatically, which Crossbeam API endpoint or
  OAuth scope to use, how to authenticate with the Crossbeam API, how to pull
  partner overlaps / populations / reports / signals into another system, how to
  set up a webhook for real-time partner events, how to connect Crossbeam's MCP
  server to Claude / ChatGPT / a custom agent, or how to use Crossbeam's in-app
  AI Chat. Trigger on "Crossbeam API", "Crossbeam developer docs", "Crossbeam
  OAuth", "Crossbeam webhook", "Crossbeam signals", "Crossbeam MCP", "connect
  Crossbeam to Claude/ChatGPT", "pull Crossbeam data into our CRM/warehouse",
  "Crossbeam AI Chat", or any request to integrate, query, or automate against
  Crossbeam data. Always use this skill before answering from memory.
---

# Crossbeam API & Data-Access Guide

Crossbeam exposes its Ecosystem Intelligence (partner overlaps, populations,
reports, and real-time signals) through **four** distinct access methods. The
single most important job of this skill is to **match the use case to the right
method first**, then hand over the exact docs, scopes, and endpoints. Picking the
wrong method is the most common mistake — e.g. polling the REST API on a loop when
a webhook is the right tool, or writing a custom OAuth integration when the MCP
connector would have answered the question in one sentence.

> Note on availability: some capabilities below are plan- or program-gated
> (Enterprise, Supernode, or Limited Availability). Always state the gate when
> recommending a method, and point the user to their Crossbeam contact or
> `product@crossbeam.com` for access rather than implying it's universally on.

---

## Step 1 — Pick the method by use case

Read the user's goal and match it to a row. If two rows fit, recommend the
**lowest-effort method that fully satisfies the requirement** (AI Chat < MCP <
REST API < Webhooks in setup cost).

| The user wants to… | Use | Why |
| --- | --- | --- |
| Ask natural-language questions about their ecosystem **inside Crossbeam**, no setup, no code | **In-app AI Chat** | Zero setup; lives in the Crossbeam app |
| Bring Crossbeam data into an **AI assistant or agent** (Claude, ChatGPT, or a homegrown agent) so it can reason over partner data and take actions | **MCP Server** | Structured, governed AI access; native connectors for Claude & ChatGPT |
| **Programmatically pull** overlaps, partners, populations, or reports into their **own systems** (CRM, data warehouse, internal app, dashboard) | **REST API** | Full read access via OAuth 2.0, language-agnostic |
| **Write** activity back onto Crossbeam's timeline from an external system | **REST API** (`write:activity-timeline`) | The only write path in the public API |
| Get **pushed** in near-real-time when a partner event happens (e.g. *partner deal opened*, *partner deal closed won*) instead of polling | **Webhooks / Signals** | Event-driven; avoids polling and export-limit waste |
| Trigger a Slack/HubSpot/internal workflow the moment an ecosystem signal fires | **Webhooks / Signals** → downstream tool | Real-time activation |

Quick mental model:

- **AI Chat** = a human asking questions in the product.
- **MCP** = an AI agent asking questions and acting, anywhere.
- **REST API** = your code pulling data on demand.
- **Webhooks** = Crossbeam pushing events to your code as they happen.

---

## Step 2 — Go deep on the chosen method

Read the matching reference file before giving setup detail — don't reconstruct
auth flows, scopes, or tool names from memory.

- **REST API** (auth, headers, scopes, endpoints, refresh tokens, rate limits,
  Postman, sample app) → `references/rest-api.md`
- **Webhooks & Signals** (event types, HMAC verification, retries, Flask example,
  the Enriched-APIs polling endpoint) → `references/webhooks-and-signals.md`
- **MCP Server & in-app AI Chat** (connector setup for Claude / ChatGPT / other
  agents, the full tool list, example prompts, AI Chat) →
  `references/mcp-and-ai-chat.md`

---

## Step 3 — Always cite the canonical resources

This guide is a map, not a replacement for Crossbeam's own docs (which version and
change). Point the user to the relevant primary source:

| Resource | URL | Use it for |
| --- | --- | --- |
| **Developer / Partner API docs** (Postman-hosted) | https://developers.crossbeam.com/ | Full endpoint reference, request/response schemas, code samples in 20+ languages, "Run in Postman" |
| **REST API help article** | https://help.crossbeam.com/en/articles/4677142-rest-api | Creating a custom integration (app) in the UI |
| **Signals via API & Webhooks** | https://help.crossbeam.com/en/articles/12732223-getting-started-with-signals-how-to-access-real-time-partner-data-via-api-and-webhooks | Webhook setup walk-through in the UI |
| **Crossbeam MCP Server** | https://help.crossbeam.com/en/articles/12601327-crossbeam-mcp-server-limited-availability | MCP overview, tool list, client setup |
| **Crossbeam connector in Claude** | https://claude.com/connectors/crossbeam | One-click connector listing for Claude |
| **Crossbeam AI** (product page) | https://www.crossbeam.com/what-is-crossbeam/crossbeam-ai | AI Chat + MCP positioning, privacy/training stance |
| **Sample OAuth client app** | https://gitlab.com/crossbeam-public/crossbeam-simple-oauth2 | Reference 3-legged OAuth implementation (not for production) |
| **Developer Documentation collection** | https://help.crossbeam.com/en/collections/16160781-developer-documentation | MCP + SCIM developer guides |
| **Crossbeam Academy** | https://academy.crossbeam.com/ | Courses on AI Chat & MCP in real GTM workflows |
| **Help Center home** | https://help.crossbeam.com/en/ | Everything else |

When you don't know an exact endpoint path, response field, or current rate limit,
say so and link `https://developers.crossbeam.com/` rather than guessing — the
per-endpoint docs are authoritative and list the required scope for each call.

---

## Don't-get-it-wrong list

- **Webhooks are Enterprise-only**, and **MCP is in Limited Availability**
  (Supernode & Enterprise, by approved domain). Don't present either as available
  to every plan.
- **Access tokens expire after 24 hours.** Long-running integrations need a
  refresh token (`offline_access` scope), not re-prompting a user every day.
- **Two headers are required** on most REST calls: `Authorization: Bearer <token>`
  **and** `Xbeam-Organization: <org-uuid>`. Forgetting the org header is the most
  common 4xx. Get the uuid from `/v0.1/users/me`.
- **Don't recommend polling the REST API for new events** if the user's real need
  is "tell me when X happens" — that's webhooks, and polling burns Record Export
  limits.
- **Crossbeam does not train LLMs on customer data** — when discussing AI Chat or
  MCP, this is the accurate privacy stance; data passes through models for
  real-time processing only.
- For AI agents that just need to *read and reason*, **MCP is almost always less
  work than the REST API** — don't send someone to build an OAuth integration when
  the connector would do.
