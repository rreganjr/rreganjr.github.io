---
layout: post
title: Requel's MCP Server — from a hand-rolled JSON-RPC controller to Streamable HTTP
---

Requel now ships a full [MCP](https://modelcontextprotocol.io/) server: an AI agent — Codex, Claude
Code, VS Code/Copilot, Claude Desktop — can connect and read *and write* project requirements
directly. It reads projects, goals, stories, actors, scenarios and the IBIS discussion layer, and it
can create and edit them. Crucially it does all of that through the **same command gateway the UI
uses**, so every call runs through per-stakeholder authorization and auditing, and the agent only
ever acts as the Requel user it authenticated as.

That feature landed across a run of tickets — the command gateway (#69), MCP cleanup and tool
naming (#82), OAuth 2.1 (#83/#81), the CLI and remote connector front-ends (#70/#100),
catalog-generated tools (#104), and finally the transport migration (#98). With #98 merged, the MCP
work for `release/2.0` is done. This post is the retrospective: why the server exists, how it's
built, and — the part I find most interesting — how its transport moved through three designs
(hand-rolled → Spring AI → Streamable HTTP) and why the last one was the right place to land.

Like the [OAuth writeup]({% post_url 2026-07-01-Adding-OAuth-2.1-to-Requel-MCP %}) before it, I built
this pairing with Claude.

## Why an MCP server at all

Requel has always had *assistants* — background agents that analyze the requirements model and attach
annotations: notes, issues, and IBIS positions, some of which can automate a fix. The original ones
were NLP-based (Stanford CoreNLP, WordNet). The direction of travel is LLM-based assistants and,
beyond that, letting **external** AI agents build and refine a Requel model directly.

MCP is the clean way to expose that. Instead of writing a bespoke integration per agent, Requel
speaks one open protocol and any MCP-capable client can drive it. But the requirement I cared about
most was that opening the model to agents must not open a hole in the authorization model. Requel's
UI already routes every mutation through a command gateway with per-stakeholder permission checks and
an audit trail. The MCP server had to sit on that same seam — not beside it. An agent connected as a
user with `Goal[Edit]` can edit goals; it cannot do anything that user couldn't do in the browser,
and every call it makes is recorded.

## The shape of the solution

Three layers, each a Maven module, plus the auth work:

- **`gateway-api`** — a persistence-ignorant façade over Requel's commands and queries.
  `CommandGateway` / `QueryGateway` are the interfaces; `GatewayCommandCatalog` is the single source
  of truth for the exposed write surface, describing each command as a `CommandDescriptor`
  (command type, input DTO, title, whether it's a write, authorization hint) with a JSON
  `CommandInputSchema` derived from the command's registered input. A `CommandPolicy` (allow/deny)
  gates which commands are reachable at all.
- **`service-impl`** — the in-process implementations (`InProcessCommandGateway` /
  `InProcessQueryGateway`) that wrap the existing `ApiCommandFactory` + `CommandHandler` chain, so the
  gateway enforces the allow/deny policy and the same per-stakeholder authorization the CQRS API runs.
- **`mcp-server`** — the MCP tool surface. `McpReadService` routes reads to the `QueryGateway`;
  `McpWriteService` routes writes to the `CommandGateway`. Reads and writes are exposed as MCP tools;
  the transport is provided by Spring AI (more on that below). `McpCallAuditor` records every tool
  call as an `McpCallAudit` row, on top of the command-level audit the gateway already produces.

Two front-ends besides the agents sit on the same `gateway-api`: a REST-backed
`gateway-rest-client`, and `requel-cli`, a picocli command-line tool (#70/#103). Because they all
consume the same catalog, they can't drift out of sync with what the server actually permits.

### The tools are generated from the catalog

The write tools aren't hand-listed. `McpWriteService` exposes a generic `runCommand` (any allowlisted
command type + JSON input) **plus one typed tool per command in `GatewayCommandCatalog`** — each named
after its command type (`EditGoal`, `EditStory`, …) with a JSON schema derived from the command's
registered input DTO. The same catalog drives the CLI's typed subcommands and the allow/deny policy
(#104). So the MCP tool list, the CLI, and the policy are always the same set — adding a command to
the catalog surfaces it everywhere at once, and a tool can never claim an operation the gateway
wouldn't actually allow.

Writes are opt-in behind `requel.gateway.write.enabled`. The app ships with writes enabled; set the
flag to `false` to run the server read-only, in which case the write tools are omitted from
`tools/list` entirely and any direct call is rejected. Reads are unaffected either way. This is a
coarse on/off switch — the real limits are still the per-stakeholder checks inside the gateway.

## The transport journey: custom → Spring → Streamable HTTP

The most interesting evolution wasn't the tools; it was how they got carried over the wire. It went
through three designs, and each stage was the right call *at the time* given what was known.

### 1. Hand-rolled JSON-RPC (where it started)

The first server was a bespoke JSON-RPC dispatcher over a single Spring MVC endpoint:

- `McpJsonRpcController` — `POST /api/mcp`, the HTTP transport.
- `McpJsonRpcHandler` — method dispatch (`initialize`, `tools/list`, `tools/call`, `resources/*`) and
  audit.
- `McpJsonRpcRequest` / `McpJsonRpcResponse` / `McpJsonRpcError` — the JSON-RPC envelopes.
- Plus the error-mapping exceptions and the descriptor/content DTOs.

That's ~270 lines of protocol and transport boilerplate wrapped around the part that actually
mattered — `McpReadService`, the tools and their JSON schemas.

Hand-rolling was reasonable, for one concrete reason: **the permission model was the open question,
and mounting under `/api/mcp` solved it for free.** Because the endpoint was a normal HTTP request
behind Requel's existing JWT security chain, every tool call inherited the authenticated user and the
`SecurityContext`, and the gateway naturally scoped to "projects visible to the current user." No new
identity plumbing. When the unknown is "how do agents authenticate," a plain HTTP `@PostMapping` is
the cheapest way to reuse the auth you already have.

The cost was that Requel owned every line of the protocol: the handshake, envelope handling, error
codes, and hand-built `Map.of(...)` schema literals — none of which is Requel's problem to solve.

### 2. Spring AI's MCP server starter

Once Spring AI matured (and Requel was on Spring Boot 3.5 / Spring AI 1.1.7 after the provider-port
work in #77), the protocol boilerplate became a framework's job. The
`spring-ai-starter-mcp-server-webmvc` starter owns the handshake, the `tools/list` / `tools/call`
envelope handling, schema advertisement, and — importantly — the transport. Requel's tools plug in as
a `ToolCallbackProvider`:

```java
@Bean
public ToolCallbackProvider requelToolCallbackProvider(McpReadService toolService,
        McpCallAuditor auditor, ObjectMapper objectMapper) {
    List<ToolCallback> callbacks = new ArrayList<>();
    for (McpToolDescriptor d : toolService.listTools()) {
        callbacks.add(new RequelMcpToolCallback(d, toolService, auditor, objectMapper));
    }
    return ToolCallbackProvider.from(callbacks);
}
```

`RequelMcpToolCallback` adapts one Requel tool descriptor to Spring AI's `ToolCallback` SPI —
building a `ToolDefinition` from the descriptor's name, description, and input schema, and delegating
execution back to `McpReadService.callTool(...)`, which already routes reads to the `QueryGateway`
and writes to the `CommandGateway`. The key design decision here: **transport choice must not leak
into business logic.** One tool implementation sits behind the transport regardless of what the
transport is, and auditing is preserved by recording the `McpCallAudit` row inside the callback.

The starter's WebMVC flavour served the **legacy HTTP+SSE** transport. That was fine for Claude Code
and VS Code, but it created a wrinkle: Spring AI's SSE endpoints and the hand-rolled controller both
wanted space on `/api/mcp`. #82 resolved that collision by *keeping* the controller for `POST
/api/mcp` and parking SSE alongside it — a deliberate stop-gap. So for a while two transports
coexisted, and interactive clients reached SSE through the community `mcp-remote` stdio bridge, often
with a pasted PAT.

### 3. Streamable HTTP only (#98) — and why it's the right endpoint

The final move was to drop SSE, delete the hand-rolled JSON-RPC controller entirely, and let Spring
AI's **Streamable HTTP** transport own `POST /api/mcp`:

```properties
spring.ai.mcp.server.protocol=STREAMABLE
spring.ai.mcp.server.streamable-http.mcp-endpoint=/api/mcp
# (the sse-endpoint / sse-message-endpoint properties were removed)
```

Why this is the best choice, not just the newest:

- **It's the spec's forward direction.** HTTP+SSE has been deprecated since the 2025-03-26 MCP spec.
  Streamable HTTP is a single endpoint that upgrades to a stream only when a call needs it — simpler
  than the SSE two-endpoint dance.
- **It's the only transport some clients speak.** Codex speaks remote MCP over Streamable HTTP
  *only*. Under SSE it couldn't connect natively at all; under Streamable it connects directly over
  OAuth.
- **Every client Requel supports already speaks it natively.** Claude Code, VS Code/Copilot, Claude
  Desktop, and MCP Inspector all probe Streamable first. So dropping SSE cost nothing — no
  currently-supported client is SSE-only — and it let the `mcp-remote` bridge and the pasted PAT
  disappear for interactive use. The bridge is now only needed for pure-stdio clients that can't do
  remote HTTP at all.
- **It forced the #82 collision resolved in the clean direction.** Retiring the controller frees
  `POST /api/mcp` for Streamable to own outright. The Spring AI `ToolCallback` path becomes the
  *single* transport behind the shared `McpReadService` / `McpWriteService` logic, and auditing stays
  exactly as it was because the live path already records via `RequelMcpToolCallback` — the only thing
  deleted was the JSON-RPC-specific audit overload.

The arc, in one line: hand-rolling bought a per-user auth model cheaply while that was the unknown;
Spring AI took back the protocol boilerplate once the framework was ready; Streamable HTTP dropped the
last bespoke piece and matched where the whole MCP ecosystem is going. Each step removed code Requel
never needed to own.

One thing that went away with the controller: four `requel://` MCP *resources*. They were served only
by the hand-rolled controller and each duplicated an existing tool (`listProjects`, `getGlossary`,
`getProjectTree`, …), so no query became unreachable. If a client ever needs the resource affordance,
they can come back as Spring AI resource specifications.

## Security stays put

None of this changed the authorization story. `/api/mcp/**` sits behind a dedicated OAuth 2.1
resource-server chain (#83) that validates authorization-server-issued tokens *and* still accepts
personal access tokens and the SPA login JWT. The token subject maps back to a Requel user, so the
gateway's per-stakeholder authorization runs unchanged. Authentication is Spring Security's job;
per-stakeholder authorization is the gateway's; Spring AI only supplies transport and tool
registration. Tool names are bare (`EditGoal`, not `requel.EditGoal`) because MCP tool names must
match `^[a-zA-Z0-9_-]{1,64}$` — spec-compliant clients reject dots.

## Using it

The server runs at `POST /api/mcp` on your local Requel (e.g. `http://localhost:8080`). Here are the
four ways I connect.

### Codex over OAuth (the #98 target — native Streamable, no bridge, no PAT)

Codex only does remote MCP over Streamable HTTP, which is exactly what Requel now serves:

```bash
codex mcp add --transport http requel http://localhost:8080/api/mcp
codex mcp login requel
```

Requel's Dynamic Client Registration is *gated* (Spring Authorization Server's OIDC DCR needs an
initial access token — no anonymous registration), and Codex tries to register anonymously. The fix
is to pre-register a client whose `redirect_uris` match Codex's loopback callback (read the exact URI
from a first login attempt), then point Codex at the returned `client_id`. After that, `codex mcp
login requel` opens a browser, you log into Requel and consent, and the tool calls run as you.

### Claude Code over OAuth (confirmed recipe)

Claude Code accepts a pre-registered `client_id` and a fixed callback port, so it works with the
gated DCR without anonymous registration. Mint an initial access token, register a client for Claude
Code's loopback callback (register both `127.0.0.1` and `localhost` forms — the AS matches
`redirect_uri` exactly), then:

```bash
claude mcp add --transport http requel http://localhost:8080/api/mcp \
  --callback-port 8899 --client-id <client_id> --scope user
# then in an interactive session: /mcp -> select "requel" -> Authenticate
```

A browser opens for Requel login + consent; on success the tools appear — nothing pasted anywhere.

### PAT — headless / CI / quick test

The MCP endpoint takes a bearer credential like any `/api/**` call, so a personal access token is the
simplest path for scripts. Log in once to get a short-lived JWT, mint a durable PAT with it, and use
the `reqpat_…` value as the bearer:

```bash
JWT=$(curl -s -X POST http://localhost:8080/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"you","password":"…"}' | jq -r .token)

curl -s -X POST http://localhost:8080/api/auth/tokens \
  -H "Authorization: Bearer $JWT" -H 'Content-Type: application/json' \
  -d '{"name":"ci","expiresInDays":365}'
#   -> {"token":"reqpat_…", ...}   (shown once)
```

For VS Code/Copilot, drop that bearer into `.vscode/mcp.json` with `"type": "http"` pointed at
`http://localhost:8080/api/mcp`. PATs are listable (`GET /api/auth/tokens`) and revocable
(`DELETE /api/auth/tokens/{id}`). The client acts as that user — scope it to what the assistant
should do, not an admin.

### requel-cli — the same gateway from a shell

`requel-cli` is a front-end over the very same `gateway-api`, so the command-line experience mirrors
the agent's:

```bash
requel --url http://localhost:8080 login --oauth        # or: login --token reqpat_…
requel --url http://localhost:8080 commands             # discover the write catalog
requel --url http://localhost:8080 projects             # reads over the QueryGateway
requel --url http://localhost:8080 project "<name>" --tree

# a real write, two equivalent ways:
requel --url http://localhost:8080 run EditGoal --input '{"projectName":"<name>","name":"CLI goal"}'
requel --url http://localhost:8080 edit-goal --project-name "<name>" --name "CLI goal"
```

The typed `edit-goal` subcommand (with `--project-name`, `--name`, … flags) is generated at startup
from the same catalog schema the MCP tools use (#103). The write runs as the logged-in user with the
allow/deny policy and per-stakeholder authorization enforced server-side — identical to what an agent
gets over MCP.

## What's next

With #98 in, the MCP server is Streamable-only, catalog-generated, OAuth- and PAT-authenticated, and
audited end to end. The natural follow-ups are on the OAuth side (a persistent signing key so tokens
survive restarts, optional external IdP support) rather than the transport, which is now settled.

The design notes live in the Requel repo: `doc/98-streamable-http-plan.md` for the migration,
`doc/port-tospring-boot-ai.md` for the transport comparison that motivated it, and
`doc/mcp_remote_connection.md` for the full client recipes.
