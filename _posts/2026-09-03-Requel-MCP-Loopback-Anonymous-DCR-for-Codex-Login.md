---
layout: post
title: Requel MCP — loopback-restricted anonymous DCR so `codex mcp login` just works
---

The [Streamable HTTP writeup]({% post_url 2026-07-27-Requel-MCP-Server-From-Custom-JSON-RPC-to-Streamable-HTTP %})
ended with a nice story for most clients and one wart for Codex. Claude Code, VS Code/Copilot and
Claude Desktop could all reach Requel's MCP endpoint over OAuth, but Codex could only connect if I
pre-registered a client by hand and fed it the returned `client_id` — exactly the kind of manual
step `codex mcp login` is supposed to remove. This follow-up is about closing that gap
([Requel #238](https://github.com/rreganjr/Requel/issues/238)): a small, opt-in, loopback-restricted
**anonymous** Dynamic Client Registration path that lets `codex mcp login requel` complete with no
PAT and no pre-registration. It also covers the one client that still needs a local bridge —
Claude Desktop — and the little wrapper script that makes that bridge actually work.

Like the posts before it, I paired on this with Claude.

## Where the last post left Codex

Requel's authorization server (added in the [OAuth 2.1 work]({% post_url 2026-07-01-Adding-OAuth-2.1-to-Requel-MCP %}))
implements **gated** DCR: the OIDC client-registration endpoint at `POST /connect/register` is
always mounted, but Spring Authorization Server requires the caller to present an access token with
scope `client.create` (minted from a seeded `registrar` client). That's a sensible default —
anonymous registration on an internet-facing AS is an abuse magnet.

The catch is how interactive CLIs register. Claude Code will accept a pre-registered `client_id`
and a fixed callback port, so it slots into the gated flow: mint an initial access token, register a
client for its loopback callback, hand it the `client_id`. **Codex can't be handed a `client_id` at
all.** It performs anonymous DCR as a fixed part of its login sequence, so the gated endpoint
rejected it and Codex fell back to "connect with a static token," which is the PAT dance the whole
OAuth flow was meant to retire.

So the requirement was narrow: let a client self-register *without* a token, but only when doing so
can't meaningfully widen the attack surface — i.e. only for **loopback** clients, only when a
deployment opts in, and with every other policy (PKCE, consent, scope, public-client) still stamped
on the result.

## Two things were actually broken

Reading the code and then watching Codex on the wire, there were two distinct blockers, and the
second one surprised me.

**1. The registration POST was rejected.** Permitting the request at the security-filter layer isn't
enough on its own: Spring AS's `OidcClientRegistrationAuthenticationProvider` still demands a
principal carrying the `client.create` scope. So an anonymous `POST /connect/register` fails deep
inside the provider, not just at the filter chain. Any fix has to *fully handle* the anonymous case
itself rather than trying to wave it past the gate.

**2. Codex never even POSTed.** Before registering, Codex reads the RFC 8414
`oauth-authorization-server` discovery document to decide whether the server supports DCR — and
Spring AS advertises `registration_endpoint` only in the *OIDC* `openid-configuration` document, not
the RFC 8414 one. With the field missing from the doc Codex actually reads, it concluded "Dynamic
client registration not supported" and skipped registration entirely. This was the real discovery
blocker, and no amount of fixing the registration handler would have helped until the metadata
advertised the endpoint where Codex looks for it.

## The fix

The change is deliberately small and lives entirely in the authorization-server config. Nothing in
the resource server, the MCP transport, or the command gateway moved.

### A scoped filter that fully handles the anonymous-loopback case

The new `AnonymousLoopbackDcrFilter` is a `OncePerRequestFilter` on `POST /connect/register`. It
activates **only** when all three guards hold:

- the deployment opted in (`requel.oauth.dcr.allow-anonymous-loopback=true`),
- the request has **no** `Authorization` header (the anonymous case — anything with a bearer is the
  gated/initial-token path and passes straight through), and
- the request peer is loopback **and** every `redirect_uri` in the body is a loopback URI
  (`127.0.0.1`, `[::1]`, or `localhost`).

When it activates, the filter parses the RFC 7591 body, builds the `RegisteredClient`, saves it via
the `RegisteredClientRepository`, and writes the RFC 7591 `201` response itself. On anything that
doesn't qualify it just calls `chain.doFilter(...)` and the normal gated flow runs unchanged — so an
anonymous *non*-loopback caller still gets Spring AS's usual `401`, and a non-loopback `redirect_uri`
is rejected with `invalid_redirect_uri`. Choosing a filter that owns the whole response (rather than
synthesizing a scoped principal for Spring AS's own provider) keeps the change from coupling to Spring
AS internals, and it's auto-discovered because it sits at the same advertised URL — the client needs
no extra configuration.

### One policy, defined once

The important design rule: a gated registration and an anonymous one must produce **byte-identical
clients**. So the client-building logic was extracted out of the existing gated converter into a
shared, package-visible helper in `AuthorizationServerConfig`:

```java
// AuthorizationServerConfig.java — one policy, called by both the gated
// OIDC converter and the new anonymous-loopback filter.
static RegisteredClient buildLoopbackMcpClient(String clientName, List<String> redirectUris) {
    // validate every redirect_uri is loopback (127.0.0.1 / [::1] / localhost)
    // public client + PKCE required   (ClientAuthenticationMethod.NONE, requireProofKey(true))
    // authorization consent required  (requireAuthorizationConsent(true))
    // scope = "mcp"
    // 1h access token / 30d rotating refresh
    ...
}
```

The gated `DcrRegisteredClientConverter` now delegates to `buildLoopbackMcpClient(...)`, and so does
the anonymous filter. Whatever path a client comes in through, it ends up public/PKCE,
consent-required, scoped to `mcp`, with the same token lifetimes. There's exactly one place to reason
about registration policy.

### Advertise the endpoint where the client looks

Finally, `AuthorizationServerConfig` now advertises `registration_endpoint` in the RFC 8414
`oauth-authorization-server` metadata, gated on DCR being usable. That's the one-line change that let
Codex discover the endpoint and actually begin registering.

The property accessor `allowAnonymousLoopbackDcr()` reads
`requel.oauth.dcr.allow-anonymous-loopback` (default **false**) and is logged at startup alongside
the other OAuth flags. When it's off, the filter is never added to the chain and behavior is identical
to the gated-only build — the secure default is preserved. The property is independent of
`requel.oauth.dcr.enabled` (the anonymous path needs no registrar and no initial token), so the two
can be on together: gated DCR for non-loopback clients, anonymous DCR for loopback ones.

## Configuration

This is opt-in and intended for local/dev use, where the AS is bound to loopback anyway. In a dev
profile:

```properties
# Anonymous, loopback-only Dynamic Client Registration (Requel #238).
# Lets a client self-register over RFC 7591 with NO token, but only when its
# redirect URIs are loopback. This is what makes `codex mcp login` work with
# no PAT and no pre-registration. Default is false; production stays gated-only.
requel.oauth.dcr.allow-anonymous-loopback=true

# Gated DCR (the initial-access-token path) is independent and can stay on
# for non-loopback clients / the pre-registration recipe.
requel.oauth.dcr.enabled=true
requel.oauth.dcr.registrar-client-secret=dev-registrar-secret
```

Keep `allow-anonymous-loopback` **out** of any production profile: it opens (loopback-only,
consent-gated) self-registration, which you don't want on an internet-facing authorization server.
The default-false posture means simply not setting it restores the gated-only behavior.

## Using it — Codex over OAuth, now with zero setup

With the property on, the whole recipe is two commands. Make sure no `REQUEL_TOKEN` is exported, so
Codex authenticates via OAuth rather than falling back to a static bearer:

```bash
codex mcp add --transport http requel http://localhost:8080/api/mcp
codex mcp login requel
```

Codex reads the RFC 8414 metadata, discovers `/connect/register`, self-registers anonymously with its
loopback callback, then opens a browser for your login and a one-time consent. On success `requel`
shows connected and the tools appear. The client it registered is public/PKCE, scope `mcp`,
consent-required — the same policy the gated path stamps. No token is pasted anywhere.

Compare that to the previous post's Codex section, which needed a manual pre-registration step and a
copied `client_id`. That step is gone. And because the anonymous-loopback path applies to any
loopback client, recent Claude Code builds that self-register can use `claude mcp login requel` the
same way, instead of the pre-registered-`client_id` recipe. If a future client changes its callback
host or path such that the loopback check rejects it, read the exact `redirect_uri` from the first
login attempt and confirm it's a `127.0.0.1`/`localhost` URI.

## The Claude Desktop wrinkle — and the bridge script

One client still can't reach the endpoint directly: **Claude Desktop**. Its
`claude_desktop_config.json` only spawns local *stdio* servers, so it can't point at Requel's remote
Streamable HTTP endpoint at all. (The clean answer is to add it as a remote connector in Settings →
Connectors instead, which does OAuth in the browser.) But if you're wiring it up through the config
file, you need the community [`mcp-remote`](https://www.npmjs.com/package/mcp-remote) proxy to bridge
stdio to the Streamable endpoint — and that's where a subtle gotcha bites.

If you inline the proxy invocation directly in the JSON args, a GUI-spawned client pre-expands
`${VAR}` and mangles the quoting, which produced an empty `Authorization: Bearer ` header — the
credential silently dropped. The fix is to move the invocation into a real shell script the client
execs, so everything is read verbatim. The script also sources `nvm` (GUI shells often lack `node`
on `PATH`) and reads the token from a file outside the repo, so no secret ever lives in source:

```sh
#!/bin/sh
# Launches the Requel MCP stdio bridge for Claude Desktop.
#
# Claude Desktop only spawns local stdio servers, so it can't point at Requel's
# remote Streamable HTTP endpoint directly. This wrapper runs the community
# `mcp-remote` proxy, which bridges stdio <-> the Streamable HTTP MCP endpoint
# Requel serves at POST /api/mcp.
#
# Kept as a standalone script (not inlined in the JSON) because GUI-spawned
# clients pre-expand ${VAR} and mangle quoting in the config args, which
# produced an empty "Authorization: Bearer " header. A real shell reads
# everything verbatim: it sources nvm (GUI shells lack node on PATH), loads the
# PAT from an out-of-repo token file, and execs the proxy.

# Make node/npx available in a GUI-spawned shell (harmless if already on PATH).
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"

REQUEL_TOKEN_FILE="$HOME/.config/requel-tokens/local"
if [ ! -r "$REQUEL_TOKEN_FILE" ]; then
  echo "requel-mcp: token file not found: $REQUEL_TOKEN_FILE" >&2
  exit 1
fi
REQUEL_PAT="$(tr -d '\r\n' < "$REQUEL_TOKEN_FILE")"

exec npx -y mcp-remote http://localhost:8080/api/mcp \
  --header "Authorization: Bearer $REQUEL_PAT" \
  --header "X-Requel-Client: claude-desktop"
```

Then point the client at the script rather than at `npx` directly:

```json
{
  "mcpServers": {
    "requel": {
      "command": "/absolute/path/to/requel-mcp.sh"
    }
  }
}
```

The credential is a user-minted personal access token (`reqpat_…`), revocable from the Requel UI; the
gateway resolves it to the owning user on every request. The `X-Requel-Client` header is optional —
Requel records it for per-client audit attribution. Note this stdio bridge is now genuinely a
*fallback* for stdio-only clients: Codex graduated off it entirely with #238, connecting natively
over Streamable + OAuth, and the same script pattern works for any other stdio-only client (swap the
`X-Requel-Client` value) by defaulting `mcp-remote` to the Streamable endpoint with no transport
flag.

## Security stays put

None of this loosened the authorization story. The anonymous path is loopback-only and off by
default; it stamps the exact same policy as the gated path (public client, PKCE, `scope=mcp`), and
**consent is still required at the authorize hop** — the human backstop against a silently registered
client. Every tool call still arrives at `/api/mcp/**` behind the OAuth 2.1 resource-server chain,
the token subject still maps back to a Requel user, and the command gateway still enforces
per-stakeholder authorization on every write. A client registered this way can do nothing the user
who logged in and consented couldn't do in the browser, and every call it makes is audited. The only
thing #238 changed is how a loopback client *obtains* its client identity — not what it's allowed to
do with it.

## Wrap-up

The arc from the last post is short and satisfying: Streamable HTTP made the transport universal, and
#238 made the *login* universal too — for the price of one scoped filter, one shared policy helper,
and one line of discovery metadata, all behind an opt-in flag that stays off in production. Codex now
logs in with two commands and no token; the pre-registration recipe is still there for anyone who
wants gated DCR; and the one remaining stdio-only client has a small, secret-free wrapper script that
makes its bridge behave.

The full client recipes and the DCR verification runbook live in the Requel repo:
`doc/mcp_remote_connection.md`, `doc/238-loopback-anon-dcr-plan.md`, and the OAuth verification notes.
