---
layout: post
title: Adding OAuth 2.1 to Requel's MCP Server (and how I verified it)
---

Requel exposes an [MCP](https://modelcontextprotocol.io/) server so AI agents can read and write
project requirements through the same command gateway the UI uses — every call runs through
per-stakeholder authorization and auditing, so an agent only ever acts as the Requel user it's
authenticated as. Until now that authentication was a static bearer token: a login JWT or a personal
access token (PAT) pasted into the client config. That works for scripts, but interactive agent
clients — Claude Code, Cursor, Claude Desktop/Cowork — don't want a pasted token. On a `401` they
expect to *discover* the authorization server and drive an OAuth flow: log in, consent, done.

I added a spec-compliant OAuth 2.1 layer to the MCP endpoints. I built this pairing with
Claude (Anthropic's Cowork) — the planning, the five-slice implementation, the verification runbook,
and this write-up. What follows is what got built, the configuration, and — the part I care most
about sharing — exactly how I checked it, including connecting Claude Code end to end.

## The shape of the solution

Four pieces, backed by Requel's existing user store (no external identity provider):

- An **embedded authorization server** (Spring Authorization Server 1.5.x on Spring Boot 3.5) that
  does authorization-code + PKCE, issues JWT access tokens, and publishes
  [RFC 8414](https://www.rfc-editor.org/rfc/rfc8414) authorization-server metadata. Users log in with
  their real Requel credentials and approve a consent screen.
- `/api/mcp/**` as an **OAuth2 resource server** that validates those tokens and maps the token
  subject back to a Requel user, so the gateway's authorization runs unchanged.
- [RFC 9728](https://www.rfc-editor.org/rfc/rfc9728) **protected-resource metadata** plus a
  `WWW-Authenticate: Bearer resource_metadata="…"` header on `401`s, which is what lets a client
  auto-discover the AS.
- **Dynamic Client Registration** so an agent client can register itself instead of me hand-editing a
  client into the database.

The key integration seam was pleasantly small: Requel resolves the current user from the security
context by name, so any auth mechanism that sets an authentication whose name is a valid Requel
username makes the whole gateway "just work." The OAuth path only had to put the username in the
token's `sub` claim.

## Decisions worth calling out

- **One coarse `mcp` scope.** The token means "act as me through the MCP tools." The real limits stay
  in Requel's per-stakeholder authorization — the token never grants more than the user already has.
  Finer read/write scopes can come later if there's a reason.
- **Consent required for every client, no auto-approve.** Agent clients are third-party; the consent
  screen is where a human sees which client is connecting and what it's asking for. It's remembered
  per client, so it's a one-time prompt.
- **1-hour access tokens, 30-day rotating refresh tokens with reuse detection.** Short access tokens
  limit the blast radius of a leak; the long rotating refresh token means you're not logging in every
  few days (looking at you, every-7-day setup I won't name). If an old, already-rotated refresh token
  is ever replayed, the whole token family is revoked.
- **Gated, loopback-only DCR.** Registration requires a short-lived initial access token, and
  registered clients are forced to loopback redirect URIs (`127.0.0.1`/`localhost`), PKCE, consent,
  and the `mcp` scope. More on why "gated" below.

## The tricky part: filter-chain layering

The riskiest bit was Spring Security chain ordering, because `/api/mcp/**` is a subset of `/api/**`
and the more specific matcher has to win. It ended up as four chains, by `@Order`:

1. Authorization-server endpoints (`/oauth2/**`, discovery, `/connect/register`).
2. Interactive login + consent (`/login`, `/oauth2/consent`) — form login backed by the Requel user
   store.
3. The `/api/mcp/**` resource server (validates AS-issued tokens).
4. The existing stateless `/api/**` JWT chain.

The MCP chain keeps the **old auth working too**: a small bearer-token resolver inspects the JWS
header and routes RS256 (AS-issued) tokens to the resource server, while PATs (`reqpat_…`) and the
SPA's HS256 login JWT fall through to the existing filter. So OAuth is additive — nothing that worked
before broke.

## Configuration

The AS and DCR are off by default. To run it locally with a seeded dev client and the registrar
client for DCR:

```
java -jar modules/requel-app/target/requel-app-2.0.0-dev.jar \
  --spring.profiles.active=dev --server.port=8080 \
  --spring.datasource.username=root --spring.datasource.password=password \
  --requel.oauth.seed-dev-client=true \
  --requel.oauth.dcr.enabled=true \
  --requel.oauth.dcr.registrar-client-secret=registrar-secret
```

One production caveat I left as a follow-up: the AS signing key is generated at startup right now, so
tokens don't survive a restart. Fine for dev; needs a persistent keystore before it's real.

## Verifying it — the run script

Unit and integration tests cover the wiring, but OAuth is one of those features you don't believe
until you've watched a real token go end to end. I wrote a runbook and worked through it. The first
part is deterministic `curl`:

```
# RFC 8414 authorization-server metadata
curl -s http://localhost:8080/.well-known/oauth-authorization-server | jq .

# RFC 9728 protected-resource metadata
curl -s http://localhost:8080/.well-known/oauth-protected-resource | jq .

# The MCP 401 must advertise where to find the AS
curl -s -i http://localhost:8080/api/mcp/sse | grep -iE '^HTTP/|www-authenticate'
#   HTTP/1.1 401
#   WWW-Authenticate: Bearer resource_metadata="…/.well-known/oauth-protected-resource"
```

Then the full authorization-code + PKCE dance against the seeded dev client — generate a PKCE pair,
open the authorize URL in a browser, log in, consent, and exchange the code:

```
CV=$(openssl rand -base64 60 | tr -d '\n=+/' | cut -c1-64)
CC=$(printf '%s' "$CV" | openssl dgst -sha256 -binary | openssl base64 | tr '+/' '-_' | tr -d '=')
# open /oauth2/authorize?...&code_challenge=$CC&code_challenge_method=S256 in a browser,
# copy the ?code=… from the address bar, then:
curl -s -X POST http://localhost:8080/oauth2/token -d grant_type=authorization_code \
  -d client_id=requel-dev-client \
  -d "redirect_uri=http://127.0.0.1:8080/login/oauth2/code/requel-dev-client" \
  -d "code=$CODE" -d "code_verifier=$CV" | jq .
```

That returns a real JWT (`sub` = my username, `scope: ["mcp"]`), and using it as a bearer against
`/api/mcp/sse` gives the SSE handshake — proof the resource server accepts an AS-issued token and
runs as the right user.

### Connecting Claude Code — the payoff

This is the bit I actually wanted: an agent client connecting over OAuth with no pasted token. Claude
Code (in VS Code) does this, but with a wrinkle worth knowing.

Requel's DCR is **gated** — Spring Authorization Server implements *OIDC* Dynamic Client Registration,
whose `/connect/register` requires a short-lived initial access token (there's no anonymous
registration). Claude Code, left to its own devices, tries to register *anonymously*, which our
endpoint rejects. The fix: pre-register a client and hand Claude Code its `client_id` and a fixed
callback port. Claude Code supports exactly that (`--client-id` / `--callback-port`).

First, mint an initial access token from the registrar client and register a client whose redirect
URIs are Claude Code's loopback callback (register both host forms — the AS matches redirect URIs
exactly):

```
INIT=$(curl -s -u requel-registrar:registrar-secret -X POST http://localhost:8080/oauth2/token \
  -d grant_type=client_credentials -d scope=client.create | jq -r .access_token)

curl -s -X POST http://localhost:8080/connect/register \
  -H "Authorization: Bearer $INIT" -H 'Content-Type: application/json' \
  -d '{"client_name":"claude-code","redirect_uris":["http://127.0.0.1:8899/callback","http://localhost:8899/callback"],"grant_types":["authorization_code","refresh_token"]}' \
  | jq -r .client_id
```

Note there's no `scope` in that request — the AS validates a *requested* registration scope against
the initial token (which only carries `client.create`), so asking for `mcp` would fail. Our
registration converter stamps `scope=mcp` on the client anyway.

Then point Claude Code at the SSE endpoint with the pre-registered client and authenticate:

```
claude mcp add --transport sse requel http://localhost:8080/api/mcp/sse \
  --callback-port 8899 --client-id <client_id> --scope user
# then, in an interactive Claude Code session:
/mcp   ->  select "requel"  ->  Authenticate
```

That opens a browser, I log into Requel, approve consent, and Claude Code is connected — its tool
calls now run as me, through the gateway, with full auditing. No token in a config file anywhere.

Once it was connected I just started asking it things. Here's a lightly trimmed slice of that
session — this is the whole point, so I'll let it speak for itself:

```
❯ list my requel projects

  Called requel (ctrl+o to expand)

⏺ You're connected to Requel — 63 projects, though most are E2E test fixtures. The real ones:

  ┌─────┬──────────────────────┬────────────────────┬────────────────────────────────────────────────────┐
  │ ID  │         Name         │        Org         │ Content (goals/stories/actors/use-cases/scenarios) │
  ├─────┼──────────────────────┼────────────────────┼────────────────────────────────────────────────────┤
  │ 3   │ Imported Project     │ Harvard University │ 16 / 8 / 6 / 2 / 7 — richest project               │
  │ 5   │ Imported Project (1) │ Harvard University │ 12 / 8 / 4 / 2 / 3                                 │
  │ 1   │ Test Project         │ Test Org           │ 2 / 0 / 1 / 1 / 1                                  │
  └─────┴──────────────────────┴────────────────────┴────────────────────────────────────────────────────┘

  The other ~59 are auto-generated Playwright/E2E scaffolding (timestamp-suffixed, mostly empty) —
  your database is ~90% test debris that isn't being cleaned up between runs. Want me to drill into
  project 3 (the interesting one), or look at test-fixture cleanup?

❯ let's drill into project 3

  Called requel 4 times (ctrl+o to expand)

⏺ Project 3 — "Imported Project" (Harvard University, admin-owned) is Requel describing itself: a
  collaborative system for eliciting and analyzing requirements. It's been used as a scratchpad, so
  real thesis content (login / create-project stories with Rich, Theresa, and Ron) is interleaved
  with keyboard-mash test entries. 276 open issues, almost all auto-generated by the NLP analysis
  assistant — term-extraction suggestions, ambiguity warnings, spell-check misses.

  The one that stands out from the templated NLP noise is on Story 4, "Ron fails to log in":

  ▎ Ron is an actor in this story, but the story currently has no actors assigned. Of the project's
  ▎ existing actors, 'Interactive User' and 'Project User' are the human end-user candidates. Which
  ▎ actor should represent Ron, or should a new actor be created? See proposed positions.

  That's the AI assistant (not the legacy NLP one) doing real requirements reasoning — spotting an
  unlinked actor, surveying the existing candidates, and proposing IBIS positions to resolve it.

❯ yes let's look at story 4

  Called requel 2 times (ctrl+o to expand)

⏺ Story 4 shows both generations of the assistant architecture side by side on one entity.

  The legacy Stanford CoreNLP pipeline dumps raw linguistic analysis as notes — constituency parse
  trees, semantic roles, dependency graphs — and its term-extractor fires on every noun phrase ("a
  web browser" offered as a candidate actor), each with the same canned positions. Its thesaurus
  assistant suggests WordNet synonyms with no sense disambiguation: "message" → "hokum", "excursus".
  Nobody wants "the system displays a hokum." A good demo of why you moved toward LLM-based assistants.

  The AI assistant, on the same story, produced judgment instead of volume — a completeness review:

  ▎ Title vs. body mismatch: the title says 'fails to log in' but the body describes a prospective
  ▎ user with no account requesting one — there's no actual authentication failure. Security tension:
  ▎ it's linked to 'Secure data access' but the flow omits lockout / rate-limiting / input validation.

  ★ Insight ─────────────────────────────────────
  The positionType field is the architectural tell: AddActorPosition, AddGlossaryTermPosition, and
  friends are executable positions — resolving the issue by choosing one mutates the model (creates
  the actor, adds the glossary term). An IBIS position here isn't just discussion text, it's a
  command waiting to fire. That's the "positions that may automate a fix" idea, made concrete.
  ─────────────────────────────────────────────────
```

That whole exchange ran over the OAuth connection — nothing pasted anywhere, every call authorized as
me through the gateway and audited. That's the capability I wanted to be able to hand to an agent.

## The bugs verification caught (the fun part)

Three things the tests didn't, because they only show up with a running server and a real browser:

- **`@Value` resolved too early.** My config flags (`--requel.oauth.seed-dev-client=true` and friends)
  were silently ignored — the seeded clients never got created. The class contributes
  `SecurityFilterChain` beans and gets instantiated early enough that `@Value` placeholders fall back
  to their defaults. Reading the values from the `Environment` at point of use fixed it. This one cost
  the most head-scratching; a `@PostConstruct` that logged the resolved flags is what finally made it
  obvious.
- **`invalid_scope` on registration.** Requesting `scope: "mcp"` at DCR time fails because the initial
  token only holds `client.create`. Drop the scope from the request; the converter forces `mcp`.
- **The "404 with the code in the address bar."** The manual dev-client redirect URI has no handler, so
  after login you land on a 404 — which *is* the success case. The authorization code is in the
  browser's address bar, not on the page. I stared at that 404 for a minute before the penny dropped.

## What's next

The feature shipped as issue #83 on the `release/2.0` branch. Left as follow-ups: a persistent
signing key so tokens survive restarts, optionally unifying the Angular SPA's login onto OAuth, and
supporting an external identity provider for orgs that already run one — the resource-server
foundation makes both of those extensions rather than rewrites.

If you want the gory details, the design and slice-by-slice plan live in `doc/oauth_mcp_plan.md` and
the full verification runbook in `doc/83-oauth-verification.md`, both in the Requel repo.
