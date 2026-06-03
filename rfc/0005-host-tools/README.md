---
authors:
  - "@shiju-nv"
state: review
---

# RFC 0005 - Host-Tools

## Summary

RFC 0005 adds `tools.local`, a sandbox-local origin for host tools: tools the agent can list and call, but that execute outside the sandbox. Agents use `POST http://tools.local/mcp`; OpenShell serves that broker-backed MCP path without sending backend routes to the sandbox.

In this RFC, the relay means the sandbox proxy and gateway working together. The broker means the OpenShell host-tools broker service behind `/mcp`. The broker publishes the signed catalog, implements MCP behavior, authorizes tools, executes host-side work, shapes results, and writes broker audit. Root `POST http://tools.local/` is reserved for future OpenShell JSON-RPC 2.0 methods and is closed in v1.

## Current State in Upstream Main

Written against `main` at `cd7024962905ea6895298381d6422e0cc82b4405`.

Main has the sandbox-proxy and gateway machinery this design reuses, but it has no host-tool forwarding path.

- `proto/openshell.proto` defines RPCs for sandbox creation, configuration, and teardown, provider configuration, `ForwardTcp`, `ConnectSupervisor`, and `RelayStream`. It does not define host-tool messages or an invocation RPC.
- `crates/openshell-sandbox/src/proxy.rs` is the ordinary agent egress path. It evaluates policy, binds process identity, applies SSRF checks, handles L7 rules, and special-cases sandbox-local `policy.local`.
- `crates/openshell-sandbox/src/policy_local.rs` shows the reusable pattern: a sandbox-local hostname handled by the proxy and backed by gateway-held state. It is policy-specific and must not carry host-tool calls.
- The existing split keeps platform state and callback authorization on the gateway side, while the sandbox handles local process identity, filesystem access, network egress, credential injection, logs, and agent execution. Host-tool authorization is introduced here as part of the broker-backed host-tools extension.

Host-tool forwarding is new. It reuses the sandbox-local-host pattern without putting host-tool behavior in `policy.local`, the proxy, or the gateway.

## Problem

Agents sometimes need tools that require access to the user's machine or network, but should not run inside the sandbox. This RFC calls those host tools. Examples are OS keychain lookups, desktop/session APIs, already-authenticated CLIs, VPN-only internal APIs, or loopback services on the host.

The v1 design is scoped to tools that need one of two host-owned capabilities:

- *Host-local authority or state*: the tool needs local OS/session state, such as an OS keychain, already-authenticated CLI or desktop/session API. Credentials and host-local state must never enter the sandbox; the agent should see only the shaped tool result.
- *Host network reachability*: the tool needs the user's host network context. The sandbox cannot reach it under ordinary network policy without either exposing the route to sandboxed code or pushing per-tool L7 rules outside any single policy owner.

Neither capability has a path in current main, as summarized above. The sandbox fails closed for both capabilities by default-deny, with no audit signal that distinguishes "unsupported" from "denied".

Copying the tool into the sandbox is not a general fallback. It can duplicate the user's installed toolchain, miss native libraries or plugins, lose access to host sockets, keychains, auth caches, VPN state, and desktop/session APIs, or run a different version than the user configured.

Direct sandbox access to those services would expose host execution paths, credentials, backend addresses, or user-local services to sandboxed code, and would reduce gateway audit to ordinary network traffic.

OpenShell needs a separate host-tool path with these constraints:

- The sandbox calls `http://tools.local/mcp` for host tools and never sees backend routes. Root `http://tools.local/` is closed in v1 and must not list, call, authorize, or shape host tools.
- The sandbox proxy forwards only to the OpenShell gateway, and the gateway must authenticate the callback before forwarding it.
- The OpenShell host-tools broker service makes MCP decisions and performs backend calls; sandbox-visible catalog and result text must not expose host routes or credentials.
- Operators can distinguish gateway local rejections, broker failures, and broker decisions in gateway audit records for calls that reach the gateway; proxy-side pre-gRPC rejections are visible through bounded proxy security logs and metrics.

## Goals

Reviewers should decide these five questions:

- Sandbox-visible `http://tools.local/mcp` discovery and invocation, plus root `http://tools.local/` JSON-RPC reservation.
- Gateway-to-broker HTTP contract.
- Where gateway transport validation ends and broker MCP and execution responsibilities begin.
- Audit records and error mapping for host-tool calls.
- Whether a future release should record catalog identity fields such as policy ID, digest, signer, and signing key ID from broker-reported `tools/list` metadata. V1 relays the response body opaquely and does not parse, verify, or recompute the signed catalog.

## Non-goals

This RFC does not change filesystem, process, network, or L7 sandbox policy. It also does not define the broker's policy language, approval product, storage model, signer trust store, key rotation, or backend implementation.

It rejects a general MCP server inside the sandbox, direct sandbox calls to the host-tool broker, host execution inside the gateway, and a second OpenShell JSON-RPC tool API that duplicates MCP `tools/list` or `tools/call`.

## Terms

### Host Tool

A tool listed in the broker's signed catalog and executed outside the sandbox. The broker may back it with a host-local API, credential helper, command wrapper, host-side MCP server, or remote API hidden from the sandbox.

### `tools.local`

A sandbox-local reserved hostname handled by the sandbox proxy. Root `http://tools.local/` is reserved for future OpenShell plain JSON-RPC 2.0 methods and is closed in v1. `http://tools.local/mcp` is an OpenShell HTTP endpoint backed by the host-tools broker.

### Sandbox Proxy

The in-sandbox process that already handles proxy traffic. This RFC adds `tools.local` routing and forwards accepted `/mcp` requests to the gateway with supervisor-provided sandbox context.

### Gateway

The server-side component behind the sandbox proxy. For host tools it authenticates sandbox context, adds trusted `_meta`, calls the broker, validates response framing, and writes gateway audit.

### Host-Tool Broker

The configured loopback HTTP service for the host-tools extension. It implements MCP behavior, authorizes host-tool calls, chooses backends, executes work, shapes output, and writes broker-side audit.

### Signed Tool Catalog

The broker verifies, canonicalizes, and stores the signed catalog behind `tools.local`. V1 does not parse broker `tools/list` responses to extract catalog identity fields.

## Design

Host-tool discovery and execution use one sandbox-to-gateway-to-broker route. This is reverse-proxy-like in the narrow sense that OpenShell forwards accepted `/mcp` JSON-RPC messages to one configured broker; it is not a byte-for-byte proxy because OpenShell validates the envelope, bounds body size, maps failures, and replaces sandbox-controlled `_meta` for trusted context.

```text
sandbox agent
  -> http://tools.local/mcp on sandbox proxy
  -> authenticated OpenShell gateway callback path
  -> OpenShell gateway

OpenShell gateway
  -> loopback host-tool broker over HTTP JSON-RPC
  -> broker MCP/backend implementation
  -> shaped result back through the gateway
```

## Protocol Surfaces

V1 uses MCP over JSON-RPC 2.0 at `POST http://tools.local/mcp` as the host-tool wire contract. It has three protocol boundaries:

- Sandbox to `tools.local`. The sandbox proxy reserves the `tools.local` host. Root JSON-RPC has no v1 methods and returns `method not found` for valid requests. The proxy serves `POST http://tools.local/mcp` as the broker-backed MCP-over-JSON-RPC 2.0 HTTP path, applies local admission checks, attaches sandbox context, and never exposes broker identity or backend routes to the sandbox.
- Proxy to gateway. New unary gRPC calls use the existing authenticated supervisor-to-gateway path. Sandbox identity comes from the gateway-minted JWT and same-sandbox-ID guard. Sandbox-supplied headers, `_meta`, and identity-shaped arguments do not cross as authority.
- Gateway to broker. The gateway sends HTTP JSON-RPC requests to the configured `default_broker`. It uses an OpenShell broker-client bearer token, forwards the JSON-RPC method and params for broker handling, and rejects non-JSON responses, server-originated JSON-RPC requests, invalid response shapes, and unsupported transport behavior.

OpenShell does not implement the MCP protocol. It does not create, store, or interpret MCP session IDs. It forwards valid single JSON-RPC objects on `/mcp`, including MCP methods such as `initialize`, `notifications/initialized`, `tools/list`, and `tools/call`, over the gateway-to-broker HTTP endpoint. MCP session state must live in the broker implementation and be represented in broker JSON-RPC state or broker response metadata, not in OpenShell routing or configuration.

## Invariant: Transparent `/mcp` Relay, Closed Root

In v1, sandbox policy does not authorize JSON-RPC method names on `tools.local`. The proxy and gateway may reject a request because the HTTP shape, `Origin`, `Accept`, size, JSON-RPC envelope, authenticated gateway callback, configured broker profile, broker HTTP response, or broker availability failed. They do not reject `/mcp` because the JSON-RPC method is `tools/list`, `tools/call`, or another MCP method, and they do not decide whether a specific tool name such as `get_weather` may run.

The `/mcp` relay is method-transparent after local admission checks. For example, a `tools/call` request with `params.name = "get_weather"` is forwarded to the broker unless an OpenShell transport or envelope check fails. The gateway may parse the method only for local relay hygiene, such as removing sandbox-supplied `_meta` and adding trusted `_meta` to `tools/call`. That parsing is not authorization.

Root `POST http://tools.local/` is different. It is reserved for future OpenShell JSON-RPC methods. V1 validates the envelope and returns JSON-RPC `method not found`; it never forwards root JSON-RPC to the broker and never exposes MCP `tools/list` or `tools/call` at root.

## Wire Contract

V1 keeps the OpenShell relay contract narrow.

Reviewers should read v1 as this fail-closed matrix:

- `POST http://tools.local/mcp` accepts one JSON-RPC object. Batch arrays and client-originated JSON-RPC responses are rejected before gRPC. The proxy applies the 65,536-byte sandbox-ingress cap before JSON dispatch; over-cap bodies return HTTP 413 and are recorded as proxy rejections.
- The proxy handles the sandbox-facing HTTP contract. `GET`, `DELETE`, and other HTTP methods return HTTP 405 with `Allow: POST`. Accepted JSON-RPC notifications are forwarded to the broker; if the broker accepts the HTTP request, the sandbox receives HTTP 202 with an empty body.
- The gateway requires an authenticated sandbox principal, enforces the selected profile's request limit, inflight limits, circuit breaker, and broker availability, removes sandbox-supplied `_meta`, and adds trusted OpenShell `_meta` only for `tools/call`.
- Gateway-to-broker HTTP is exactly `POST <base_url><rpc_path>` with `Authorization: Bearer <broker-token>`, `Content-Type: application/json`, and `Accept: application/json`. The gateway never forwards the sandbox JWT or sandbox-supplied headers.
- Broker responses to requests must be single JSON-RPC response objects with matching IDs. Server-originated requests, multiple responses, malformed JSON, timeout or size overruns, non-JSON bodies, and mismatched IDs fail closed as broker errors. Notification responses must have an empty body.
- Gateway failures become JSON-RPC errors with bounded `local_rejection_reason` or `broker_error_code` values. Broker execution failures remain MCP `CallToolResult` values with `isError: true`.
- The gateway never retries a timed-out `tools/call`. The broker defines retry and idempotency policy for side-effecting tools.

Catalog identity recording is not part of v1. The relay validates only the JSON-RPC response envelope needed for transport safety and does not parse the broker's `tools/list` result.

Returned tool content is text-only and must fit the v1 transport envelope. Broker-side result validation is part of the Machine-Readable Tool Contract below.

## Machine-Readable Tool Contract

`http://tools.local/mcp` exposes catalog-backed, machine-readable tools through the broker's MCP-over-JSON-RPC implementation, not terminal sessions or arbitrary host execution. The broker may use CLIs, local MCP servers, or remote APIs behind the broker, but every sandbox-visible tool is an MCP tool from the verified, signed catalog with stable JSON inputs and machine-readable results.

The broker must adapt implementation-specific output before it reaches OpenShell:

- command-backed tools run non-interactively and force JSON, a declared schema, or another machine-readable output mode before shaping the MCP result;
- successful calls return a valid MCP `CallToolResult`; when `outputSchema` is declared, `structuredContent` is required on success and must validate against that schema;
- selected-tool failures return `CallToolResult` with `isError: true` and a structured error envelope when the tool can produce one;
- malformed MCP messages, unknown tool names, gateway validation failures, transport failures, timeouts, invalid broker responses, and untrustworthy broker output stay JSON-RPC / gateway errors rather than tool results;
- tool descriptions, schema text, result text, and structured content are sandbox-visible and must not intentionally contain backend routes, credential names, credential values, approval references, broker-private policy labels, or other host-private implementation details.

## Authentication and Credential Boundaries

This RFC separates broker authentication from host-tool credentials. The broker-client bearer token proves to the broker that the HTTP client is the OpenShell gateway; it does not authorize any host tool against an upstream service and must not be reused as a tool credential. OpenShell may know that bearer token, but it must not know Vault, OS keyring, or CLI token-cache secrets used by individual tools.

Tool credentials stay behind the broker boundary. They are resolved by the broker, CLI, OS keychain, secret-manager lookup, or another operator-managed backend behind the broker. OpenShell never reads, stores, validates, forwards, logs, or exposes those values. The sandbox sees only the broker-shaped MCP tool catalog and tool results.

## Broker Profile

The host-tools gateway profile lives under `[openshell.gateway.host_tools]` in the RFC 0003 gateway configuration file. It selects the broker for the fixed sandbox path `POST http://tools.local/mcp`. Sandbox-facing paths are not profile fields.

`[openshell.gateway.host_tools.brokers]` is only a TOML map container. It has no fields of its own. Each child table, `[openshell.gateway.host_tools.brokers.<id>]`, defines one broker profile. V1 activates exactly one profile through `default_broker`; unselected profiles are parsed by serde but otherwise inert. The gateway does not read their token files, interpret their socket-activation setting, probe health, or route `/mcp` traffic to them.

```toml
[openshell.gateway.host_tools]
enabled = true
default_broker = "local"

[openshell.gateway.host_tools.brokers.local]
kind = "json_rpc_http"
base_url = "http://127.0.0.1:7901"
rpc_path = "/"
socket_activated = true
broker_client_auth_token_path = "/run/openshell/host-tools/broker-auth"
connect_timeout_ms = 1000
request_timeout_ms = 30000
max_request_body_bytes = 262144
max_response_body_bytes = 1048576
max_inflight_per_sandbox = 4
max_inflight_global = 64
circuit_breaker_threshold = 10
circuit_breaker_cooldown_ms = 30000
```

Field-to-env mapping follows RFC 0003 convention: uppercase TOML path with `.` and `_` joined by `_`, for example `OPENSHELL_GATEWAY_HOST_TOOLS_BROKERS_LOCAL_BASE_URL`. V1 needs CLI flags only for `default_broker`, `base_url`, and `request_timeout_ms`; other fields are TOML/env only.

The selected profile must pass these checks:

- `enabled` defaults to `false`.
- `kind` defaults to `json_rpc_http`, the only v1 transport kind.
- `base_url` must be a literal loopback HTTP origin with an explicit port: `http://127.0.0.1:<port>` or `http://[::1]:<port>`. Hostnames, DNS, wildcards, Unix sockets, HTTPS, remote HTTP, credentials, paths, queries, and fragments are rejected.
- `rpc_path` is the gateway-to-broker JSON-RPC endpoint path. It defaults to `/` and must be `/` or one non-empty segment of `[A-Za-z0-9._~-]+` after the leading `/`. `.`, `..`, query strings, fragments, percent-encoding, additional slashes, backslashes, empty segments, and normalization-dependent paths are rejected. This path is not the sandbox-facing root endpoint.
- Timeouts, byte limits, inflight limits, and circuit-breaker settings are HTTP-client controls for every broker request, regardless of JSON-RPC method. They are not `tools/list` or `tools/call` settings. Every timeout, byte count, and concurrency count must be a positive integer.
- Profile structs use `serde(deny_unknown_fields)`.

`socket_activated = true` is an explicit operator affirmation that the selected broker is expected to be managed outside the gateway process and reachable on the configured loopback endpoint. V1 does not accept systemd unit names or launchd labels in the broker profile and does not prove listener ownership at runtime. That proof is future hardening work.

`broker_client_auth_token_path` must be a regular file, mode `0600`, owned by the gateway UID. Contents are an ASCII bearer token: unpadded base64url, at least 43 characters, charset `[A-Za-z0-9_-]`, no whitespace, no `=` padding, and at most one trailing newline. The gateway reads this file once at startup and holds the token in memory. Token rotation requires coordinated gateway and broker restart.

The profile has four externally visible states:

- absent profile, `enabled = false`, or absent `default_broker`: host-tool forwarding is disabled while `tools.local` remains reserved;
- structurally invalid profile or invalid token file: gateway startup fails with a field-specific error and never logs token bytes;
- valid profile without `socket_activated = true`: gateway starts with host-tool forwarding disabled, logs warning evidence, and withholds the broker-auth token;

Profile changes require a gateway restart because RFC 0003 has no hot reload in v1. `/healthz` includes only OpenShell host-tool sub-status: enabled/disabled, default broker ID, configured broker IDs, active broker ID, circuit state, and last broker-hop HTTP result. It does not expose catalog identity in v1 because the relay does not parse broker `tools/list` responses.

### Deployment Topology

V1 supports one topology for each active broker: the gateway and broker run on the same host, and `base_url` resolves to that host's loopback interface. This is the only topology where the gateway's loopback interface is the user's host loopback interface, which the v1 scope requires.

V1 excludes these topologies until a future RFC defines a non-loopback transport:

- *Containerized gateway*: container loopback is the container's network namespace. A containerized gateway must run on the host network or leave the profile unset.
- *Remote or multi-tenant gateway*: loopback brokers are host-local and cannot safely serve multiple users.
- *VM-isolated broker*: separate-VM brokers are not loopback to the gateway.

Service lifetime is outside the gateway in v1. Operators should run the broker under a service manager, such as paired systemd `.socket` + `.service` units on Linux or launchd `Sockets` on macOS, but the v1 gateway does not inspect the service manager. The gateway trusts only the configured loopback origin and broker bearer token.

## Future Direction: `tools.local` JSON-RPC

This RFC reserves `tools.local` for OpenShell local admission APIs and broker-backed host-tool MCP, not as a general OpenShell API namespace.

Future RFCs can add OpenShell JSON-RPC methods at the root endpoint without changing the agent-facing MCP URL. Examples include broker health, configured-broker diagnostics, and local gateway state inspection. Root methods must not expose broker catalog contents, duplicate MCP `tools/list` or `tools/call`, or provide per-request broker selection unless a future RFC defines routing, authorization, and audit rules.

Future OpenShell policy can match JSON-RPC method fields on `tools.local`, including `method = "tools/call"` and `params.name` for selected tool names. That is not part of v1. A future policy design must define rule precedence, audit records, denial error mapping, interaction with broker authorization, and whether policy runs in the sandbox proxy, gateway, or both. Until that exists, `/mcp` remains method-transparent after OpenShell transport and envelope checks.

Future route management for `tools.local` could become a small local gateway model, similar in spirit to Gateway API: declarative routes, per-route admission policy, health reporting, and broker selection. V1 intentionally hardcodes only two sandbox-facing routes, root `/` and `/mcp`, so reviewers can audit the security boundary before adding route programmability.

The `openshell.gateway.host_tools.brokers.<id>` layout is the extension point for multiple broker implementations. New broker kinds can be added under the same namespace, such as a different local JSON-RPC transport or a local process broker. Non-loopback broker transport requires a separate RFC that defines authentication, trust model, health checks, and failure mapping before it can be exposed through `tools.local`.

## Implementation Plan

1. Add `tools.local` as a sandbox-local reserved origin in the sandbox proxy, with closed root `POST http://tools.local/` and broker-backed `POST http://tools.local/mcp`.
2. Add gateway RPCs for forwarding accepted `/mcp` host-tool traffic on the existing authenticated supervisor-to-gateway path.
3. Add the `openshell.gateway.host_tools` namespace, named broker profiles, validation, startup health state, and broker-auth bearer-token handling.
4. Implement the gateway-to-broker HTTP client for the selected `default_broker`, including generic request/response limits, circuit-breaker behavior, and error mapping.
5. Add the `socket_activated = true` operator affirmation gate before the gateway reads the broker-auth token. Runtime service-manager listener-ownership proof is future hardening work.
6. Add tests for disabled/fail-closed behavior, broker-auth versus tool-credential separation, root `method not found`, root not forwarding to the broker, malformed root JSON-RPC input, malformed `/mcp` input, broker errors, audit fields, and proxy-side rejection before gateway RPC.

## Risks

- **Gateway scope creep.** If backend policy, approval logic, credential handling, MCP rules, or command execution move into the proxy or gateway, the RFC has failed.
- **MCP scope creep.** `http://tools.local/mcp` lets agents use MCP; it does not make the proxy or gateway full MCP implementations.
- **Root JSON-RPC becoming a second tool API.** Root must not duplicate MCP `tools/list`, MCP `tools/call`, broker catalog rules, or host-tool execution.

## Alternatives

### Gateway Runs Host Tools

The gateway validates policy and calls host-side MCP servers or command runners directly. Choose this only if OpenShell intentionally moves host-tool policy, backend routing, approval state, output shaping, and execution audit into the gateway.

### Provider Calls Host Tools Directly

The provider receives a remote MCP or tool-server URL. That exposes host routes outside the OpenShell relay, weakens audit, bypasses the sandbox proxy, and needs a separate RFC covering provider-facing schema generation, callback authentication, result adaptation, and audit correlation.

### Sandbox Calls Host APIs Directly

The sandbox holds scoped credentials and calls APIs. This puts credentials and route details in the sandbox; tools with no hidden host route, no broker credential, and a clean fit for ordinary network policy are not host tools and do not need `tools.local`.
