# Synadia Agent Protocol for NATS

**Version:** 0.3.0
**Status:** Draft
**Date:** 2026-09-21

## 1. Introduction

This document defines a protocol for identifying, discovering, and communicating with AI agents over NATS. Agents registered per this protocol can be enumerated, inspected, and prompted using a uniform set of subjects and message shapes regardless of the underlying agent framework, language, or runtime.

Built on two NATS primitives:

- **Subject hierarchy** for addressing, routing, and wildcard discovery.
- **Micro services** (`@nats-io/services` and equivalents) for registration and discovery.

Anything NATS already provides is used as-is. The protocol adds only what is missing: a subject convention, one shared service name, a request envelope, a streaming response wrapper, and a liveness beacon.

Out of scope for v0.3:

- End-to-end encryption.
- The `attachments` endpoint (§2 reserves the verb; §5.5 sketches intent).
- JetStream-backed at-least-once streaming.

Strong sender identity is not out of scope: §13 defines it as an optional extension that leaves the wire behaviour of §1–§12 unchanged.

### 1.1 Conventions

Normative keywords - **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, **MAY** - follow RFC 2119 and RFC 8174.

"The protocol" is the wire contract defined here. "SDKs" are language-specific libraries built on top, specified separately in `sdk-contract.md`.

### 1.2 Version

This specification is version `0.3.0`. Agents declare the protocol version they implement in service metadata (§3.2). Compatibility rules are in §11.

v0.3 introduced two wire-breaking changes vs. v0.2:

- **Verb-first subject hierarchy** (§2): each protocol endpoint owns its own positional slot via a `verb` token (`prompt`, `hb`, `status`, `attachments`) directly after the `agents.` root. v0.2 placed each endpoint on a sub-subject of the agent root; v0.3 hoists the verb up so endpoints don't share namespace with each other.
- **`status` endpoint** (§8.7): a request/reply companion to the periodic heartbeat. Replies with the same JSON payload shape as a heartbeat, so callers can bootstrap their liveness tracker without waiting a full interval.

There is no v0.2 ↔ v0.3 compatibility shim. Mixed-version deployments are allowed — they simply don't see each other on discovery, because the prompt subjects don't overlap.

---

## 2. Subject hierarchy

Every agent instance occupies a subject tree of the form:

```
agents.{verb}.{agent}.{owner}.{name}
```

| Token    | Role                                                                                                                                                                   |
|----------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `agents` | Fixed prefix. Reserved for this protocol.                                                                                                                              |
| `verb`   | Endpoint discriminator. One of `prompt`, `hb`, `status`, `attachments` (see table below). Reserved for protocol use; agents MUST NOT use these tokens for any other purpose. |
| `agent`  | Identifier of the harness / runtime. SHOULD be (an abbreviation of) `metadata.agent` (§3.2). Conventional abbreviations: `cc` for `claude-code`, `oc` for `openclaw` — see Appendix C. |
| `owner`  | Operator or account owning the instance. SHOULD match `metadata.owner`.                                                                                                |
| `name`   | Instance name within `{agent}/{owner}`. Lives only in the subject — not echoed in metadata.                                                                            |

The protocol reserves four verbs and assigns each a default subject. The endpoint **name** (used in `$SRV.INFO`) MUST match the verb token; the subject is open to overrides via `$SRV.INFO`, but the channels shipped with this protocol use the defaults.

| Verb          | Default subject                                  | Purpose                                                                          | Fixed?                   |
|---------------|--------------------------------------------------|----------------------------------------------------------------------------------|--------------------------|
| `prompt`      | `agents.prompt.{agent}.{owner}.{name}`           | Required prompt endpoint (§5, §6).                                               | No — default only        |
| `hb`          | `agents.hb.{agent}.{owner}.{name}`               | Liveness beacon (§8). The verb is the abbreviation `hb` because heartbeat traffic dominates per-account subject volume. | **Yes** (protocol-fixed) |
| `status`      | `agents.status.{agent}.{owner}.{name}`           | On-demand status request/reply (§8.7).                                           | No — default only        |
| `attachments` | `agents.attachments.{agent}.{owner}.{name}`      | Reserved for the future chunked-upload endpoint (§5.5).                          | No — default only        |

What the protocol fixes for endpoints is the endpoint **name**, not its subject:

- An agent MUST register an endpoint named `prompt`.
- An agent MUST register an endpoint named `status` (§8.7).
- If the agent exposes the future artifact endpoint, it MUST be named `attachments`.

Callers therefore MUST learn endpoint subjects from `$SRV.INFO.agents` (§4); they MUST NOT construct endpoint subjects from identity alone. The heartbeat subject is the one exception — it is fixed so callers can subscribe to `agents.hb.*.*.*` without a lookup.

### 2.1 Prompt endpoint metadata

The `prompt` endpoint's registration (§3) MUST declare endpoint metadata:

```json
{
  "max_payload": "1MB",
  "attachments_ok": true
}
```

| Key              | Type    | Required | Description                                                                                                                                                                        |
|------------------|---------|----------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `max_payload`    | string  | Yes      | Maximum single-message request payload size. Format is a positive integer followed by `B`, `KB`, `MB`, or `GB` (e.g. `512KB`, `1MB`, `4MB`). Callers MUST enforce locally (§5.4).  |
| `attachments_ok` | boolean | Yes      | Whether the endpoint accepts JSON envelopes containing an `attachments` array. If `false`, callers MUST NOT send attachments; plain text and JSON-without-attachments remain valid. |

### 2.2 Naming rules

Subject tokens MUST conform to NATS subject naming rules (https://docs.nats.io/nats-concepts/subjects#characters-allowed-and-recommended-for-subject-names).

- Tokens MUST NOT begin with `$`.
- Tokens SHOULD use only `a`–`z`, `0`–`9`, `-`, `_`.
- Each token SHOULD be 1–63 characters; fully qualified subjects SHOULD stay under 256 characters.

Agent identifiers are not centrally registered. Collisions are the deployer's responsibility. Appendix C lists identifiers in common use and their conventional subject abbreviations.

### 2.3 Examples

```
agents.prompt.claude-code.aconnolly.synadia-com-2   # prompt for claude-code, session "synadia-com-2"
agents.prompt.cc.aconnolly.synadia-com-2            # same, using the "cc" abbreviation
agents.hb.cc.aconnolly.synadia-com-2                # heartbeat for the same instance
agents.status.cc.aconnolly.synadia-com-2            # status request/reply for the same instance
agents.prompt.openclaw.rene.default                 # OpenClaw, long-running, session-less (name "default")
agents.prompt.pi.mario.workspace-1                  # pi, session on workspace "workspace-1"
agents.prompt.hermes.ops.summarizer                 # Hermes instance "summarizer"
```

### 2.4 Wildcard discovery

```
agents.>                             # every protocol subject on the system
agents.prompt.>                      # every prompt endpoint
agents.hb.>                          # every heartbeat (equivalently: agents.hb.*.*.*)
agents.status.>                      # every status endpoint
agents.prompt.claude-code.>          # every claude-code prompt endpoint (full form)
agents.prompt.cc.>                   # every claude-code prompt endpoint (abbreviated form)
agents.prompt.*.aconnolly.>          # every prompt endpoint owned by aconnolly
agents.prompt.*.*.summarizer         # every prompt endpoint with name "summarizer"
agents.hb.*.*.*                      # every heartbeat beacon (protocol-fixed subject)
```

Each verb owns its own positional slot, so `agents.prompt.>` and `agents.hb.>` partition the namespace cleanly — heartbeats never compete with prompts under a wildcard subscription.

Full-form and abbreviated subject tokens do not cross-match under a NATS wildcard. A deployment SHOULD commit to one convention per `metadata.agent` value.

---

## 3. Service registration

Every agent MUST register as a NATS micro service using `@nats-io/services` or equivalent. Registration is the authoritative source for endpoint capability metadata and for enumeration via `$SRV.PING` / `$SRV.INFO`.

### 3.1 Required service fields

| Field         | Value                                                                                                                                                       |
|---------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `name`        | MUST be `agents`. Acts as the discovery filter that separates compliant agents from other NATS micro services.                                               |
| `version`     | Semver of the harness implementation (not the protocol). Example: `1.4.0`.                                                                                  |
| `description` | Human-readable description surfaced by `nats micro list` / `nats micro info`.                                                                               |
| `metadata`    | Object. See §3.2.                                                                                                                                           |

### 3.2 Required service metadata

The service `metadata` object MUST include:

```json
{
  "agent": "claude-code",
  "owner": "aconnolly",
  "session": "synadia-com-2",
  "protocol_version": "0.3"
}
```

| Key                | Type   | Required             | Description                                                                                                                                                 |
|--------------------|--------|----------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `agent`            | string | Yes                  | Canonical harness identifier (e.g. `claude-code`). The 3rd subject token MAY be an abbreviation of this value (§2).                                         |
| `owner`            | string | Yes                  | Operator / account. Matches the 4th subject token.                                                                                                          |
| `session`          | string | When session-aware   | Harness-specific session label. MUST be set for session-aware harnesses (`claude-code`, `pi`, `hermes`); MAY be omitted or set to `"default"` for session-less harnesses (`openclaw`). |
| `protocol_version` | string | Yes                  | Protocol version implemented. MUST match a MAJOR.MINOR value from §11.3.                                                                                    |

The instance name is not echoed in metadata — callers read it from the 5th token of any endpoint subject (§2).

Additional metadata keys MAY be included and MUST be preserved by tools that relay service info.

### 3.3 Queue group on the `prompt` endpoint

The `prompt` endpoint MUST be registered with a NATS queue group of `"agents"`. This:

- Enables load-balancing across same-subject instances (§3.4) without per-deployment coordination.
- Makes the queue group discoverable at runtime - the micro service framework reports it in `$SRV.INFO` as `endpoints[].queue_group`.

In `@nats-io/services` (TypeScript) this is `addEndpoint("prompt", { subject, handler, queue: "agents", metadata })`; equivalent mechanisms exist in other clients. Do NOT rely on the framework's default queue group - that value differs between implementations, which breaks interoperability across mixed-SDK deployments.

The heartbeat subject (§8) has no queue group - it is pub/sub.

### 3.4 Multiple instances

Multiple physical instances of the same logical agent MAY register simultaneously. They share `agent`/`owner`/`name` identity, which yields identical endpoint subjects, and the `"agents"` queue group on the `prompt` endpoint (§3.3) causes the NATS micro service framework to load-balance requests across them.

Each instance is distinguished by the service `id` (per-instance, framework-assigned). This id also appears in heartbeat payloads as `instance_id` (§8.3).

Different instances of the same logical agent SHOULD expose identical endpoints and metadata.

---

## 4. Discovery

The protocol defines exactly **two stable subjects** for discovery:

| Purpose | Subject                          | Semantics                                                 |
|---------|----------------------------------|-----------------------------------------------------------|
| General | `$SRV.PING.agents`               | Every compliant agent instance responds once.             |
| Direct  | `$SRV.INFO.agents.{instance_id}` | One specific instance responds with full service info.    |

Discovery returns each instance's endpoints with their subjects and metadata. Callers MUST use the `subject` field from the discovery record to address an endpoint - endpoint subjects are not protocol-fixed (§2) and cannot be reliably constructed from identity alone.

### 4.1 General discovery

```shell
nats req '$SRV.PING.agents' '' --replies=0 --timeout=2s
nats req '$SRV.INFO.agents' '' --replies=0 --timeout=2s
```

`$SRV.PING` returns ping-level records; `$SRV.INFO` returns full records including endpoint capability metadata. Callers typically use the latter.

### 4.2 Direct lookup

Callers with a known `instance_id`:

```shell
nats req '$SRV.INFO.agents.VMKS6MHK71PCPWGY38A7N5' '' --timeout=2s
```

Callers with a known identity tuple but no `instance_id` run general discovery and filter client-side on `metadata.agent`, `metadata.owner`, and the 4th token of the endpoint subject.

### 4.3 Using a discovery record

From a service info record, a caller:

1. Reads `metadata.agent`, `metadata.owner`, optional `metadata.session`, and `metadata.protocol_version`.
2. Locates the `prompt` endpoint by `name == "prompt"` and reads its `subject` and its metadata (`max_payload`, `attachments_ok`).
3. Derives the instance name from the 4th token of the endpoint's `subject` when the subject follows the default pattern (§2); otherwise the instance name is taken from the agent's `metadata.session` or is opaque to the caller.
4. Enforces §5.4 validation using the endpoint metadata.
5. Publishes requests to the endpoint's `subject` verbatim - no construction.

---

## 5. Request

A request is a single NATS message sent by a caller to the agent's `prompt` endpoint subject, as learned from discovery (§4.3). Request-side streaming is deferred to the future `attachments` endpoint (§5.5).

### 5.1 Envelope shape

A request payload is either:

- **Plain UTF-8 text** - shorthand for an envelope with only the `prompt` field. Enables `nats req` use without constructing JSON.
- **JSON envelope** - an object with at minimum a `prompt` field:

```json
{
  "prompt": "summarize the attached report",
  "attachments": [
    { "filename": "report.pdf", "content": "<base64>" }
  ]
}
```

| Field         | Type     | Required | Description                                                                                                              |
|---------------|----------|----------|--------------------------------------------------------------------------------------------------------------------------|
| `prompt`      | string   | Yes      | UTF-8 prompt text. MUST be non-empty.                                                                                    |
| `attachments` | object[] | No       | Zero or more attachment objects (§5.2). Agents with `attachments_ok: false` MUST reject a non-empty array with status `400` (§9). |

Additional top-level fields MAY appear; see §5.6.

### 5.2 Attachments

```json
{ "filename": "report.pdf", "content": "<base64>" }
```

| Field      | Type   | Required | Description                                                                                                                 |
|------------|--------|----------|-----------------------------------------------------------------------------------------------------------------------------|
| `filename` | string | Yes      | Authoritative file name. Agents interpret the bytes by extension or content sniff.                                          |
| `content`  | string | Yes      | Standard-alphabet, padded base64 (RFC 4648 §4). MUST NOT use URL-safe encoding and MUST NOT contain whitespace.             |

No MIME type is carried.

### 5.3 Discrimination rule

On receive, the agent:

1. Skips leading UTF-8 whitespace (`0x09`, `0x0A`, `0x0D`, `0x20`).
2. If the next byte is `{`, parses the remainder as JSON. If parsing fails, or the parsed object has no `prompt` string field, responds with status `400` (§9).
3. Otherwise, treats the original payload as UTF-8 text and promotes it to `{"prompt": <payload>}`.

A zero-byte request payload is invalid and MUST be rejected with status `400`.

### 5.4 Client-side validation

Before publishing, the caller MUST enforce the `prompt` endpoint's capability metadata (§2.1):

- If any attachment is present and `attachments_ok` is `false`, the caller MUST fail locally without publishing.
- The caller MUST compute the final encoded payload byte size and fail locally if it exceeds `max_payload`.

These local checks spare a round trip and agent-side resources. Agents MAY additionally enforce server-side and respond with `400`.

### 5.5 Future direction: artifact endpoint (≥ 0.3)

A future revision will define the `attachments` endpoint at `agents.attachments.{agent}.{owner}.{name}` (v0.3 verb-first) for large-file upload:

- Separate endpoint from `prompt`, with its own wire contract and request-side streaming (chunked uploads).
- Uploaded files are staged in a temp directory accessible to the agent and injected by reference into the next `prompt` request's context.
- Likely backed by JetStream Object Store.

Precise wire format, chunking, lifetime, and reference-handoff semantics are deferred. v0.1 implementations SHOULD structure attachment handling so that adding the `attachments`-endpoint code path is additive, not a rewrite.

### 5.6 Unknown fields

Envelope decoders MUST tolerate unknown top-level fields and unknown fields inside attachment objects without error. They MUST preserve such fields when relaying.

---

## 6. Response streaming

The `prompt` endpoint responds by publishing a sequence of chunks to the caller's reply subject, terminated by an empty-payload message.

### 6.1 Pattern

```
Caller                                    Agent
   |                                        |
   | --- request (reply=_INBOX.abc) ------▶ |
   |                                        |
   | ◀--  chunk 1 (to _INBOX.abc) --------- |
   | ◀--  chunk 2 (to _INBOX.abc) --------- |
   |      ...                               |
   | ◀--  terminator (to _INBOX.abc) -------|
```

### 6.2 Chunk wrapper

Every non-terminating chunk is a typed JSON object:

```json
{ "type": "<type>", "data": <value> }
```

| Field  | Type          | Required | Description                                                                           |
|--------|---------------|----------|---------------------------------------------------------------------------------------|
| `type` | string        | Yes      | Chunk discriminator. v0.1 defines `response`, `status`, `query` (§6.3–6.4, §7).       |
| `data` | string/object | Yes      | Chunk payload. Shape is determined by `type`.                                         |

Plain-text shorthand is **not** accepted on the response side: every non-terminating chunk MUST be a JSON object with a `type` discriminator.

Unknown chunk types MUST be silently ignored by callers, which continue consuming the stream (§6.6). Error responses use NATS micro service error headers instead of a typed chunk (§9).

### 6.3 `response` chunks

Content from the agent. `data` is either a string (the response text) or an object:

```json
{ "type": "response", "data": "Hello, world." }
```

```json
{ "type": "response", "data": { "text": "Hello, world.", "attachments": [ ... ] } }
```

| Field         | Type     | Required | Description                                                                                            |
|---------------|----------|----------|--------------------------------------------------------------------------------------------------------|
| `text`        | string   | Yes*     | Response text. When `data` is a bare string, that string IS the text.                                  |
| `attachments` | object[] | No       | Optional attachments from the agent; shape per §5.2. Callers MAY ignore.                               |

(*) When `data` is an object, `text` is required. When `data` is a string, the object form does not apply.

Callers MUST accept both the string and object forms transparently. Multiple `response` chunks MAY be emitted; callers concatenate `text` values in publication order.

### 6.4 `status` chunks

Lifecycle signal emitted during long-running work. `data` is a string status token:

```json
{ "type": "status", "data": "ack" }
```

v0.1 defines one status value:

| Value | Meaning                                                                                                              |
|-------|----------------------------------------------------------------------------------------------------------------------|
| `ack` | Acknowledges request acceptance. Resets the caller's inactivity timeout (§6.6).                                       |

Agents MUST emit exactly one `ack` status chunk as the **first message** on the reply subject, before any `response` or `query` chunks and before any work that introduces observable latency (e.g. a model call). The ack confirms the request was received and resets the caller's inactivity timeout (§6.6) ahead of any warm-up gap. This also makes the stream observable from generic NATS tooling (e.g. `nats req --wait-for-empty`) which would otherwise time out on the warm-up gap.

Callers MUST silently ignore unrecognized status values.

The terminal `done` is not a `status` chunk. The empty-payload terminator (§6.5) IS the done signal; SDKs MAY surface it to applications as a `status: done` event.

### 6.5 Stream termination

Every response stream MUST end with a **zero-byte body message carrying no NATS headers**. This is the uniform end-of-stream signal for all streams - successful or errored.

Agents MUST NOT publish further messages on the reply subject after the terminator.

**Successful completion.** Zero or more content / status / query chunks, then the empty terminator.

**Error completion.** Zero or more chunks, then a message carrying `Nats-Service-Error-Code` / `Nats-Service-Error` headers (with optional JSON body per §9.1), then the empty terminator. The error-headered message is not itself the terminator.

A stream consisting of a single `response` chunk followed by the empty terminator is valid and common.

### 6.6 Ordering, delivery, forward compatibility

Chunks are delivered in publication order. NATS core messaging is at-most-once; individual chunks - including the terminator - MAY be lost silently.

To prevent indefinite hangs on a lost terminator, callers MUST apply a per-stream inactivity timeout. Recommended default: **60 seconds since the last observed chunk**. On timeout, the caller treats the stream as terminated with a transport error.

Forward-compat rules for callers:

- MUST silently ignore chunks whose `type` is unrecognized.
- MUST tolerate unknown fields in `data` objects and preserve them when relaying.

Agents that require at-least-once delivery SHOULD wait for a future JetStream-backed pattern. v0.1 operates on core NATS only.

### 6.7 Cancellation

The protocol defines no cancellation signal. NATS subject delivery is interest-based: if a caller drops its subscription on the reply subject, subsequent chunks are discarded by the NATS server at no cost.

- Callers cancel by dropping the reply subscription. No wire signal is sent.
- Agents are not notified of cancellation and continue until natural completion. An agent detects an abandoned caller only if it issues a mid-stream query (§7) and the reply times out.
- Agents producing expensive long-running work SHOULD use a mid-stream query as a liveness check if cancellation semantics matter.

---

## 7. Mid-stream queries

An agent MAY pause its response stream to ask the caller a question - a permission prompt, a clarification, a menu selection. The response stream remains open; the caller publishes one reply to a fresh subject supplied by the agent; the agent resumes emitting chunks.

### 7.1 Query chunk

```json
{
  "type": "query",
  "data": {
    "id": "a8f1c2e4-9b63-4d7e-aaaa-112233445566",
    "reply_subject": "_INBOX.Xj7k9Q2pA",
    "prompt": "Confirm deletion of 200 files? (yes/no)"
  }
}
```

| Field           | Type     | Required | Description                                                                                      |
|-----------------|----------|----------|--------------------------------------------------------------------------------------------------|
| `id`            | string   | Yes      | Opaque correlation identifier. SHOULD be a UUID.                                                 |
| `reply_subject` | string   | Yes      | A fresh NATS subject (typically `_INBOX.xxx`) on which the agent expects exactly one reply.      |
| `prompt`        | string   | Yes      | The question text presented to the caller.                                                       |
| `attachments`   | object[] | No       | Optional attachments; same shape as §5.2.                                                         |

### 7.2 Reply

The caller publishes exactly one message to `reply_subject`. Payload follows §5.1 - plain UTF-8 text, or a JSON envelope with `prompt` + optional `attachments`. No acknowledgment is defined.

```shell
nats pub _INBOX.Xj7k9Q2pA "yes"
```

### 7.3 Lifecycle

- The agent chooses its own reply timeout. The protocol does not mandate a value.
- If the caller does not reply within the timeout, the agent MAY either (a) terminate the stream with an error per §9, or (b) proceed with a harness-defined default and continue emitting chunks. In case (b), the caller receives no signal.
- Multiple query chunks MAY be in flight concurrently within a single response stream. Each MUST use a distinct `reply_subject`.
- Query chunks do not terminate the stream. The §6.5 terminator rules still apply.

---

## 8. Heartbeat

Agents MUST publish a periodic heartbeat so callers can track liveness without polling. Heartbeats are pub/sub (fire-and-forget); there is no reply.

### 8.1 Subject

```
agents.hb.{agent}.{owner}.{name}
```

The verb token is the abbreviation `hb` (not `heartbeat`) because heartbeat traffic dominates per-account subject volume — every agent instance emits at the configured cadence regardless of caller activity. The shorter token measurably reduces wire overhead at scale without sacrificing readability.

Each instance publishes its own heartbeats to this subject. Callers distinguish instances by the `instance_id` field in the payload.

### 8.2 Cadence

Each instance chooses its own interval, configurable at SDK construction time. Recommended default: **30 seconds**. Values below 1 second SHOULD NOT be used on shared infrastructure.

The interval is carried in the payload (`interval_s`). Recommended offline threshold: **3 × interval_s since last observed heartbeat**, applied per `instance_id`.

Agents SHOULD begin publishing heartbeats only after service registration is complete, so that callers discovering the agent via `$SRV.INFO` find its metadata.

### 8.3 Payload

```json
{
  "agent": "claude-code",
  "owner": "aconnolly",
  "session": "synadia-com-2",
  "instance_id": "VMKS6MHK71PCPWGY38A7N5",
  "ts": "2026-04-21T14:23:01Z",
  "interval_s": 30
}
```

| Field         | Type   | Required           | Description                                                                                                           |
|---------------|--------|--------------------|-----------------------------------------------------------------------------------------------------------------------|
| `agent`       | string | Yes                | Matches `metadata.agent`.                                                                                              |
| `owner`       | string | Yes                | Matches `metadata.owner`.                                                                                              |
| `session`     | string | When session-aware | Matches `metadata.session`. Present iff the metadata field is set.                                                     |
| `instance_id` | string | Yes                | The micro service framework's per-instance identifier. Matches the service `id`.                                       |
| `ts`          | string | Yes                | UTC ISO 8601 timestamp of publication.                                                                                 |
| `interval_s`  | number | Yes                | This instance's cadence in seconds. > 0; recommended ≥ 1.                                                              |

The instance name is not duplicated into the payload — receivers extract it from the 5th token of the heartbeat's subject.

Callers MUST tolerate additional unknown fields.

### 8.4 On-demand reachability

For point-in-time reachability, callers SHOULD use the micro service ping instead of waiting for the next heartbeat:

```shell
nats req '$SRV.PING.agents' '' --replies=0 --timeout=2s
```

Callers correlate ping responses with heartbeats via `instance_id`.

### 8.5 Subscribe-before-discover

To avoid a race between enumeration and the first heartbeat, callers SHOULD subscribe to the heartbeat wildcard (`agents.hb.*.*.*`, scoped as narrowly as needed) **before** sending their first `$SRV.PING.agents`.

### 8.6 Shutdown

v0.3 defines no "going away" signal. Callers detect shutdown via the missed-beats threshold (§8.2).

### 8.7 Status endpoint (request/reply)

In addition to the periodic pub/sub heartbeat (§8.1–§8.3), every agent MUST expose a `status` endpoint:

| Aspect           | Value                                                                    |
|------------------|--------------------------------------------------------------------------|
| Endpoint name    | `status`                                                                 |
| Default subject  | `agents.status.{agent}.{owner}.{name}`                                   |
| Queue group      | `"agents"` (same as `prompt`; see §3.3 — load-balances across same-identity instances) |
| Request body     | Reserved. Agents MUST currently ignore the request body. Future revisions MAY define a request schema. |
| Reply body       | A §8.3 heartbeat-shaped JSON payload, freshly built per request.         |

The reply payload uses **exactly the §8.3 schema** — same `agent`, `owner`, optional `session`, `instance_id`, `ts`, `interval_s` fields. Receivers MAY treat a status reply as if it were a just-arrived heartbeat: feed it into the same liveness tracker, key on `instance_id`, etc.

Two motivating use cases:

1. **Bootstrap a tracker without waiting a full heartbeat interval.** A caller that just connected and wants point-in-time liveness for a known instance can `nats req agents.status.{agent}.{owner}.{name} ''` instead of waiting up to `interval_s` for the next pub/sub beat to arrive.
2. **Future agent-state queries.** Subsequent revisions MAY extend the reply payload with richer agent metadata (cost so far, queue depth, capability hints, ...). Sourcing the reply from the same builder used for periodic heartbeats keeps the two streams in lockstep.

#### 8.7.1 Implementation notes

- Agents SHOULD source the reply payload from the same builder used to construct periodic heartbeats so the two emit identical values for fields they share.
- A `respond` failure (broker dropped, request reply already torn down) is best-effort — agents MAY swallow it and continue serving.
- An exception thrown while constructing the payload MUST be returned as a `Nats-Service-Error-Code: 500` response (see §9), not propagated into the framework's default handling.

---

## 9. Errors

Errors are reported using the NATS micro service error response mechanism.

### 9.1 Wire shape

An error response carries two headers set by `respondError`:

- `Nats-Service-Error-Code` - numeric status code as a string (e.g. `"429"`).
- `Nats-Service-Error` - short human-readable description.

The body MAY be empty, or MAY carry a JSON object with richer context:

```json
{
  "error": "rate_limited",
  "message": "Too many concurrent requests for this agent instance",
  "retry_after_s": 30
}
```

| Field     | Type   | Required              | Description                                                                                     |
|-----------|--------|-----------------------|-------------------------------------------------------------------------------------------------|
| `error`   | string | Yes (if body is JSON) | Stable machine-readable error code. Lowercase snake_case recommended.                           |
| `message` | string | No                    | Human-readable detail. If absent, callers fall back to the `Nats-Service-Error` header.         |
| (other)   | any    | No                    | Additional fields MAY be included. Callers MUST tolerate unknown fields.                        |

If the body is empty or not valid JSON, callers MUST use the `Nats-Service-Error` header as the description.

### 9.2 Status code taxonomy

| Code | Error class                                                                                                                                         |
|------|-----------------------------------------------------------------------------------------------------------------------------------------------------|
| 400  | Malformed request: invalid envelope, empty payload, invalid base64, attachments sent when `attachments_ok=false`, request exceeds `max_payload`.   |
| 401  | Authentication required. (NATS usually enforces at connect time.)                                                                                   |
| 403  | Forbidden: caller authenticated but not authorized.                                                                                                 |
| 404  | Not found.                                                                                                                                          |
| 409  | Conflict: request conflicts with current agent state.                                                                                               |
| 429  | Rate limited.                                                                                                                                       |
| 500  | Internal error.                                                                                                                                     |

Agents MUST use codes from this table. Future revisions will add new codes but not reuse existing ones with new meanings.

### 9.3 Errors during a stream

An error mid-stream is an error-headered message (with optional JSON body per §9.1) published to the reply subject, followed by the empty terminator (§6.5). This applies whether the error occurs before any content or partway through. Error-terminated streams always end with two messages: error-headered, then empty terminator.

Agents MUST NOT publish further messages on the reply subject after the terminator.

---

## 10. Security

### 10.1 Authentication and authorization

Authentication and authorization are delegated to NATS server configuration. Agents inherit the security boundaries of the NATS accounts, users, and permissions they connect with.

- Reachability between a caller and an agent is determined entirely by NATS subject permissions.
- The protocol defines no pairing, allowlisting, or handshake.
- Deployments SHOULD use NATS accounts and subject permissions to isolate agents by tenant or environment.

E2E encryption is deferred to a future revision.

Per-message sender identity is an optional extension (§13): a signed `Agent-Sender` header lets a receiver verify which NATS user sent a request, in every topology. It identifies; which verified senders to accept remains the deployment's decision.

### 10.2 Credential management

Agents and callers SHOULD support NATS CLI contexts (`~/.config/nats/context/<name>.json`) for connection configuration.

---

## 11. Versioning

The protocol version an agent implements is declared in `metadata.protocol_version` (§3.2).

### 11.1 Version string format

MAJOR.MINOR strings (e.g. `"0.1"`, `"1.0"`). Patch/pre-release qualifiers MAY be present but have no compatibility meaning - callers MUST compare only the MAJOR.MINOR prefix.

### 11.2 Compatibility rules

- **Same MAJOR.MINOR**: full interoperability.
- **Same MAJOR, different MINOR**: callers SHOULD treat the agent as compatible for the caller's MINOR feature set and rely on forward-compat escape hatches (§5.6 unknown fields, §6.6 unknown chunk types).
- **Different MAJOR**: no interoperability guarantee.

The 0.x line is explicitly unstable. MINOR bumps within 0.x MAY break compatibility; callers SHOULD pin to an exact MAJOR.MINOR until 1.0.

### 11.3 Known versions

| Version | Status     | Notes          |
|---------|------------|----------------|
| `0.3`   | Draft      | This document. Verb-first subject hierarchy (§2 — `agents.{verb}.{a}.{o}.{n}`); heartbeat moves to `agents.hb.{a}.{o}.{n}` (§8.1); new `status` request/reply endpoint (§8.7) replies with the §8.3 heartbeat shape. Not wire-compatible with 0.2 — the prompt subject changed. |
| `0.2`   | Superseded | Subject hierarchy was `agents.{a}.{o}.{n}` (4 tokens, no verb); heartbeat at `agents.{a}.{o}.{n}.heartbeat`; no `status` endpoint. |
| `0.1`   | Superseded | Earliest draft — service name was `Synadia Agents`, no required queue group on the `prompt` endpoint. Not wire-compatible with 0.2. |

---

## 12. Implementation checklist

An **agent** is compliant with protocol `0.3` when it:

- Registers as a NATS micro service with `name = "agents"`.
- Declares `metadata.agent`, `metadata.owner`, `metadata.protocol_version = "0.3"`; adds `metadata.session` when session-aware.
- Registers an endpoint named `prompt` with queue group `"agents"` (§3.3) and endpoint metadata `max_payload` and `attachments_ok`. The endpoint's `subject` is agent-chosen; the recommended default (used by channel plugins) is `agents.prompt.{agent}.{owner}.{name}` (§2 verb-first).
- Registers an endpoint named `status` with queue group `"agents"` (§8.7). Default subject `agents.status.{agent}.{owner}.{name}`. Replies with a §8.3 heartbeat-shaped payload.
- On the `prompt` endpoint:
  - Accepts both JSON envelopes and the plain-text shorthand (§5).
  - Rejects malformed envelopes, empty payloads, invalid base64, oversize requests, and attachments-when-`attachments_ok=false` with status `400`.
  - Tolerates and preserves unknown envelope fields.
  - Emits an `{type:"status",data:"ack"}` chunk as the first message on the reply subject, before any `response`/`query` chunk and before any latency-inducing work (§6.4).
  - Emits response streams per §6: typed `{type, data}` chunks in publication order, terminated by a zero-byte headerless message. Errors precede the terminator with error headers.
- Publishes heartbeats on `agents.hb.{agent}.{owner}.{name}` at its configured cadence with all §8.3 fields.
- Responds to `$SRV.PING.agents` and `$SRV.INFO.agents` via the micro service framework.
- If it issues mid-stream queries: conforms to §7.
- Uses `respondError` per §9 for errors; `Nats-Service-Error-Code` is set from the §9.2 taxonomy.

Optional: an **agent that implements §13** (sender identity) additionally:

- Declares `min_sender_trust` (`any` or `signed`) in the `prompt` endpoint metadata; declares nothing on `status` (§13.9).
- Adds `user_nkey`, `account`, and `id_sig` to the service metadata (§13.10).
- Classifies every `prompt` and `status` request as verified, claimed, or absent before the application sees it. Rejects a malformed `Agent-Sender` with `400` and a failing signature with `401`, whatever `min_sender_trust` says (§13.7).
- Accepts `sub` only in the forms of §13.7.1; enforces the replay window and a nonce set.
- On a `signed` endpoint, rejects a request that is not verified with `401`. Answers `403` when it refuses a verified sender. Never rejects a `status` request on identity grounds (§13.9).
- Grants nothing on a claimed identity, or on `account` alone (§13.1, §13.7.2).
- Signs every heartbeat it publishes when it holds its seed. Never signs a reply (§13.11).

A **caller** is compliant when it:

- Performs discovery only via `$SRV.PING.agents` and `$SRV.INFO.agents[.{instance_id}]`.
- Reads each instance's `metadata.protocol_version` and applies §11.2 compatibility rules.
- Locates the `prompt` endpoint by `endpoints[].name == "prompt"`, reads its metadata, and enforces `max_payload` / `attachments_ok` locally before publishing (§5.4).
- Publishes to the `prompt` endpoint's `subject` as reported by `$SRV.INFO.agents`. MUST NOT construct the subject from identity alone.
- Subscribes to an appropriate heartbeat wildcard **before** initial `$SRV.PING.agents`.
- Applies a per-stream inactivity timeout (§6.6).
- Inspects NATS headers (`Nats-Service-Error-Code`) on every received message before interpreting the body.
- Treats a zero-byte body with no NATS headers as stream termination (§6.5).
- Silently ignores unknown chunk types, unknown endpoint names, and unknown metadata keys.
- Preserves unknown metadata fields when relaying.
- Tracks liveness per `instance_id` (§8.1).

Optional: a **caller that implements §13** (sender identity) additionally:

- Learns its agent ID once per connection: from the credentials JWT when it holds one, otherwise from `$SYS.REQ.USER.INFO`. Sends no `Agent-Sender` when it has no identity (§13.4).
- Attaches `Agent-Sender` to every `prompt` and `status` request. Signs it when it holds the seed, with a fresh nonce and `sub` per §13.6.2.
- Includes the header bytes in the local `max_payload` check (§13.5.1).
- Reads `min_sender_trust` before sending. Fails locally when the endpoint requires `signed` and it cannot sign; treats an unknown value as `signed` (§13.9).
- Builds and parses agent IDs only in the canonical text form, checking both halves (§13.3).
- Treats registration identity metadata as a claim unless `id_sig` verifies (§13.10).

---

## 13. Sender identity (optional extension)

This chapter defines an optional extension. A caller proves which NATS user sent a request; an agent proves which NATS user registered it and published its heartbeats. The extension is wire-compatible with §1–§12: an agent or caller that does not implement it behaves exactly as those sections define, and interoperates with one that does (§13.13). `protocol_version` stays `"0.3"`.

The subject hierarchy (§2) names only the receiver; no token names the caller. The sender identity therefore travels with the message, in a header.

### 13.1 Trust classes

The extension trusts only what the receiver can verify on the message itself. One mechanism meets that bar: a signature by the sender's user NKEY (§13.6). Everything else is a claim.

| Class    | Evidence                                                                 | Meaning                                                   |
|----------|--------------------------------------------------------------------------|-----------------------------------------------------------|
| verified | `Agent-Sender` header with a valid signature (§13.7)                     | The sender holds the seed for `user`.                     |
| claimed  | `Agent-Sender` header without `sig`; any `Nats-Request-Info` header      | Display only: logs, UI, conversation threading.           |
| absent   | No `Agent-Sender` header, or one with an unknown `v`                     | The message has no sender.                                |

A receiver MUST NOT grant anything on a claimed identity.

`Nats-Request-Info` is always a claim. The server stamps it only when a request crosses an account boundary through a service import with `share: true`. Within one account the server forwards a client-written `Nats-Request-Info` unchanged. Both arrive as identical bytes, so a receiver cannot tell a server stamp from a forgery.

### 13.2 Agent ID

The agent ID is the pair `(account, user)` that NATS gives every authenticated connection:

| Half      | Content                                                                                          |
|-----------|--------------------------------------------------------------------------------------------------|
| `account` | The account public NKEY (`A…`). On a server without operator mode, the account name (§13.3.1).  |
| `user`    | The user public NKEY (`U…`).                                                                    |

The protocol invents no identity of its own. Two agents that share a NATS user share one agent ID, and nothing downstream can tell them apart; each agent SHOULD connect as its own NATS user. Several instances of one logical agent (§3.4) on one user share one agent ID, which is consistent: they are one agent to the caller.

`owner` (§3.2) is a label naming the operator; `account` is the NATS account. They are unrelated, and neither replaces the other.

#### 13.2.1 What is secret

An NKEY is an ed25519 key pair in NATS text encoding. The design rests on keeping its halves apart:

| Half       | Looks like                  | Held by                | Role                                                                                              |
|------------|-----------------------------|------------------------|---------------------------------------------------------------------------------------------------|
| Seed       | `SU…` for a user            | the agent, no one else | Signs. Authenticates the connection and produces every signature in this chapter. **The secret.** |
| Public key | `U…` user, `A…` account     | anyone                 | Verifies signatures and names the identity. Safe to publish; useless for forging.                 |

A credentials file bundles the user JWT (public: the server's signed statement of who the user is) with the seed; the seed makes the file sensitive. Only the seed holder can sign. Rotating a seed produces a new public key and therefore a new agent ID; reissuing a JWT does not.

### 13.3 Canonical text form

Every place an agent ID is carried as text — a subject, a KV key, a JSON field, a stored record, a log line — MUST use this form and no other:

```
{account}.{user}
```

| Token     | Content                                                                    | Length       |
|-----------|----------------------------------------------------------------------------|--------------|
| `account` | The account as the server reports it (§13.4)                               | 56 for an NKEY |
| `.`       | One separator                                                              | 1            |
| `user`    | The user public NKEY: `U` and 55 base32 characters (`A`–`Z`, `2`–`7`)      | 56           |

- Upper case exactly as NATS issues the NKEY. Implementations MUST NOT change case.
- No whitespace, no quotes, no other separator.
- Two agent IDs are equal if and only if their text forms are byte-equal.
- On an operator-mode server (Synadia Cloud, and every `nsc`-managed deployment), `account` is the account public NKEY: `A` and 55 base32 characters. The form is then 113 characters.

A parser MUST accept exactly the strings matching

```
^(A[A-Z2-7]{55}|[A-Za-z0-9_-]+|\$G)\.U[A-Z2-7]{55}$
```

and passing the NKEY check. The regex is the shape check. The NKEY check (prefix byte and CRC-16) MUST run on `user`, and on `account` whenever `account` starts with `A` and is 56 characters long. The name branch of the regex also matches the NKEY shape; the NKEY check, not the regex, tells an NKEY account from a name.

Neither token may be empty. There is no zero agent ID: two zero IDs would compare equal, and two anonymous senders are not the same sender.

#### 13.3.1 Accounts without an NKEY

On a server without operator mode (accounts from the configuration file, no JWTs) an account has no NKEY. The server reports the configured account name; a server with no accounts reports the global account `$G`. `account` is then that name. The name form exists only for such servers, so that an NKEY user on a self-run server still has a verifiable identity.

The name MUST be one token matching `[A-Za-z0-9_-]+`, or the literal `$G`. A server that reports any other account name (one containing `.`, `*`, `>`, whitespace, or any other character) gives the connection no representable identity: it runs the protocol without identity (§13.4).

One user, three servers:

```
AABYLMBR6Q2CDXTLGRQCFA2GP76BGCDF7NZF2OVHH4RQ7L3Y3TZWJDRL.UAWW24XPLGOX3R3JF4OZEZZ6RUXMB55DSWJCEFFSUDFBCKJD4MSCMQYI   # operator mode
ACME.UAWW24XPLGOX3R3JF4OZEZZ6RUXMB55DSWJCEFFSUDFBCKJD4MSCMQYI                                                        # account named ACME
$G.UAWW24XPLGOX3R3JF4OZEZZ6RUXMB55DSWJCEFFSUDFBCKJD4MSCMQYI                                                          # no accounts
```

`$G` names a different account on every such server. An agent ID built from it is valid text, but not portable.

#### 13.3.2 In subjects and KV stores

The form is two subject tokens, each within the 63-character limit of §2.2. It drops into a subject unchanged and comes back out as that subject's last two tokens. With `svc` standing for any prefix a deployment owns, `svc.{account}.>` matches every agent of one account and `svc.*.{user}` matches one user in any account. The protocol itself defines no subject that carries an agent ID; its subjects stay as §2 defines them.

NKEY tokens are an exception to the lower-case recommendation of §2.2: they are upper case as issued. `$G` is a legal NATS token, not a §2.2-conformant one, and appears only on a server with no accounts.

A KV key allows `[-/_=.a-zA-Z0-9]` and uses `.` as its separator, so the NKEY form and the name form are valid KV key names unchanged. `$G` is not.

#### 13.3.3 Parse fixtures

A conforming parser gives these results. Each key is a real NKEY. The same cases, machine-readable, are in [`test-fixtures/identity/agent-id-fixtures.json`](https://github.com/synadia-ai/synadia-agents/blob/main/test-fixtures/identity/agent-id-fixtures.json).

| Input | Result |
|-------|--------|
| `AABYLMBR6Q2CDXTLGRQCFA2GP76BGCDF7NZF2OVHH4RQ7L3Y3TZWJDRL.UAWW24XPLGOX3R3JF4OZEZZ6RUXMB55DSWJCEFFSUDFBCKJD4MSCMQYI` | valid, NKEY account |
| `ACME.UAWW24XPLGOX3R3JF4OZEZZ6RUXMB55DSWJCEFFSUDFBCKJD4MSCMQYI` | valid, account name |
| `$G.UAWW24XPLGOX3R3JF4OZEZZ6RUXMB55DSWJCEFFSUDFBCKJD4MSCMQYI` | valid, global account |
| `UAWW24XPLGOX3R3JF4OZEZZ6RUXMB55DSWJCEFFSUDFBCKJD4MSCMQYI.AABYLMBR6Q2CDXTLGRQCFA2GP76BGCDF7NZF2OVHH4RQ7L3Y3TZWJDRL` | invalid: tokens swapped |
| `AABYLMBR6Q2CDXTLGRQCFA2GP76BGCDF7NZF2OVHH4RQ7L3Y3TZWJDRL:UAWW24XPLGOX3R3JF4OZEZZ6RUXMB55DSWJCEFFSUDFBCKJD4MSCMQYI` | invalid: separator |
| `aabylmbr6q2cdxtlgrqcfa2gp76bgcdf7nzf2ovhh4rq7l3y3tzwjdrl.uaww24xplgox3r3jf4ozezz6ruxmb55dswjceffsudfbckjd4mscmqyi` | invalid: case changed |
| `AABYLMBR6Q2CDXTLGRQCFA2GP76BGCDF7NZF2OVHH4RQ7L3Y3TZWJDRL.UAWW24XPLGOX3R3JF4OZEZZ6RUXMB55DSWJCEFFSUDFBCKJD4MSCMQY` | invalid: user NKEY truncated |
| `AABYLMBR6Q2CDXTLGRQCFA2GP76BGCDF7NZF2OVHH4RQ7L3Y3TZWJDRL.UAWW24XPLGOX3R3JF4OZEZZ6RUXMB55DSWJCEFFSUDFBCKJD4MSCMQYI.extra` | invalid: three tokens |
| `UAWW24XPLGOX3R3JF4OZEZZ6RUXMB55DSWJCEFFSUDFBCKJD4MSCMQYI` | invalid: a user NKEY alone is not an agent ID |
| `acme corp.UAWW24XPLGOX3R3JF4OZEZZ6RUXMB55DSWJCEFFSUDFBCKJD4MSCMQYI` | invalid: account name with a space |
| `AABYLMBR6Q2CDXTLGRQCFA2GP76BGCDF7NZF2OVHH4RQ7L3Y3TZWJDRM.UAWW24XPLGOX3R3JF4OZEZZ6RUXMB55DSWJCEFFSUDFBCKJD4MSCMQYI` | invalid: account has the NKEY shape but fails the CRC |

### 13.4 A connection's own identity

A party learns its own agent ID once per connection:

1. From the user JWT, when the connection uses a credentials file. `sub` is the user public NKEY; `iss` is the account public NKEY, or `nats.issuer_account` when a signing key issued the JWT.
2. Otherwise from `$SYS.REQ.USER.INFO`, which returns the connection's own `user` and `account` to any connected user.

With a JWT in hand the implementation SHOULD NOT ask the server. `$SYS.REQ.USER.INFO` is an ordinary subject: a same-account user with default permissions can subscribe to it, see the request, and answer before the server does. The JWT cannot be raced. An implementation that consults both sources MUST find the same pair, and otherwise treats the identity as unknown. Credentials files exist only in operator mode, so the JWT source never applies to a configuration-file server.

| Connection                                                     | Server reply                                          | Identity   |
|----------------------------------------------------------------|-------------------------------------------------------|------------|
| Operator mode, with credentials                                | `user` = `U…`, `account` = `A…`                       | `A….U…`    |
| Configuration-file server, NKEY user, named account            | `user` = `U…`, `account` = the name                   | `{name}.U…` |
| Configuration-file server, NKEY user, no accounts              | `user` = `U…`, `account` = `$G`                       | `$G.U…`    |
| Configuration-file server, NKEY user, name outside `[A-Za-z0-9_-]+` | `user` = `U…`, `account` = the name              | none       |
| No authentication                                              | `user` empty                                          | none       |
| Password or token user                                         | `user` = the user name, or `[REDACTED]` for a token   | none       |
| No reply within the timeout (default 2 s)                      | —                                                     | unknown    |

*None* means the server answered and the connection has no NKEY user; the remedy is an NKEY user or a credentials file. *Unknown* means the implementation does not know: the server did not answer, or a permission blocks the request. It MUST NOT guess, and MUST NOT keep the failure for the connection's lifetime; it retries after a short interval. A permission that blocks `$SYS.REQ.USER.INFO` yields no reply but an asynchronous `-ERR 'Permissions Violation for Publish …'`; an implementation that observes it SHOULD treat the identity as unknown at once rather than wait for the timeout.

A connection whose identity is none or unknown:

- Sends no `Agent-Sender` header — never one with empty fields, never an invented name.
- Fails a send locally, before publishing, when the target endpoint declares `min_sender_trust: signed` (§13.9).

A verified identity on a self-run server therefore needs one thing: an NKEY user in the server configuration, and its seed at the client. Every other local setup runs the protocol without identity, which every endpoint with `min_sender_trust: any` accepts.

### 13.5 The `Agent-Sender` header

A caller that implements this chapter and has an identity (§13.4) MUST set one `Agent-Sender` header on every `prompt` and `status` request:

```
Agent-Sender: {"v":1,"account":"A…","user":"U…","name":"claude-code","sub":"agents.prompt.claude-code.aconnolly.synadia-com-2","ts":"2026-04-28T14:23:01Z","nonce":"<NUID>","sig":"<base64url>"}
```

| Field     | Required     | Description                                                                                                                                  |
|-----------|--------------|----------------------------------------------------------------------------------------------------------------------------------------------|
| `v`       | Yes          | Header format version. This document defines the JSON number `1`.                                                                            |
| `account` | Yes          | The caller's account (§13.3).                                                                                                                |
| `user`    | Yes          | The caller's user public NKEY.                                                                                                               |
| `name`    | No           | Display name. Never identity. Outside the signature by design: a relay may rewrite it, and nothing may depend on it.                         |
| `sub`     | When signed  | The subject the caller publishes to, byte for byte, with the one exception of §13.6.2. The `subject` line of the signed input.               |
| `ts`      | When signed  | Request time, RFC 3339 UTC. Signers SHOULD emit second precision with the `Z` suffix; verifiers MUST accept fractional seconds.               |
| `nonce`   | When signed  | Unique per request from this `user`: 1–64 characters from `[A-Za-z0-9_-]`. A NUID (22 characters) is RECOMMENDED.                           |
| `sig`     | When signed  | Signature (§13.6), base64url without padding.                                                                                                |

`account` and `user` together are the agent ID (§13.2). A header without `sig` is a claim; a caller MAY send one when it holds no seed.

#### 13.5.1 Wire form

- The header name is `Agent-Sender`, matched case-sensitively.
- The value is compact JSON on one line: no line breaks, no whitespace between tokens. NATS header values cannot carry CR or LF.
- Unknown fields are ignored. A receiver treats a header with an unknown `v` as absent.
- A header is **malformed** when it is not valid JSON; lacks `v`, `account`, or `user`; carries a field of the wrong shape; or carries `sig` without all of `sub`, `ts`, and `nonce`. `v` is the number `1`, not the string `"1"`. A receiver rejects a malformed header with `400` (§9.2).
- The header counts toward the server's `max_payload`, which covers headers and payload together. Callers MUST include the header bytes in the local check of §5.4. A signed header block is about 400 bytes with an NKEY account (Appendix B.13).

### 13.6 Signing (caller side)

#### 13.6.1 Signed input

The caller signs with its user seed. The signed input is this byte string:

```
AGENT-SENDER-V1\n{account}\n{user}\n{subject}\n{ts}\n{nonce}\n{sha256(payload) lowercase hex}\n
```

- `\n` is one LF byte (`0x0A`). Every line, the last included, ends with one.
- `account`, `user`, `ts`, and `nonce` are the header's values, byte for byte. `subject` is the header's `sub`.
- `payload` is the raw message payload. An empty payload, for example a `status` request, hashes to the SHA-256 of zero bytes (`e3b0c442…b855`).

The signature is ed25519 over those bytes — the NKEY `Sign` operation — encoded base64url without padding. The signed input is never sent: the caller transmits only `sig`, and the receiver rebuilds the input from the header fields and the message it received.

The first line is a fixed tag. It separates this signature from every other signature the same seed produces (an `id_sig`, §13.10, never passes as a `sig`, nor the reverse), and it names the format, so a `V2` can coexist with `V1`.

| Bound               | Blocks                                                          |
|---------------------|-----------------------------------------------------------------|
| `account`, `user`   | A relay swapping the agent ID under a valid signature.          |
| `subject`           | Transplanting the header onto a request to a different agent.   |
| payload hash        | Reuse of the header on a different payload.                     |
| `ts`, `nonce`       | Replay of the identical request.                                |

Signing needs no knowledge of how the receiver is deployed — same account, cross account, relayed — only of the caller's own account's imports (§13.6.2).

#### 13.6.2 The signed subject

`sub` travels in the header because the receiver does not always see the subject the caller published to. The caller signs the subject it publishes to, with one exception:

- **An import renamed by the caller's own account.** When the caller's account imported the service under a different local name (`to` in a server configuration file, `local_subject` in an account JWT), the caller signs the exporter's subject: the subject its import names, which is also what `$SRV.INFO` reports. The receiver cannot know the local names its importers chose; the caller's own account configured the rename, and the caller must use the local name to publish at all. At the receiver `sub` then equals the arrival subject.

Two cases need nothing from the signer beyond signing what it publishes:

- **An export that inserts the caller's account** (`account_token_position`). The server inserts the caller's account at that position on the way in; the receiver removes it before comparing (§13.7.1 (b)). A `to` that merely drops that token is not a rename in the sense above.
- **A subject the signer also publishes for its own account.** When a subject has consumers in the signer's own account and another account imports it under a different name, the signer cannot sign the importer's name without failing every verifier at home. The heartbeat (§13.11) is the case. The receiver accepts it by §13.7.1 (c).

### 13.7 Verification (receiver side)

A receiver that implements this chapter classifies each request before the application sees it:

1. No `Agent-Sender` header, or an unknown `v`: **absent**. The application sees an absent sender, never an empty ID, and no lookup runs on it: an empty key must not match anything.
2. A malformed header (§13.5.1): reject with `400`.
3. A header with `sig`: rebuild the signed input from the header's `account`, `user`, `sub`, `ts`, and `nonce` and the SHA-256 of the received payload, and verify `sig` against `user`. Then check:
   - `sub` is acceptable for the arrival subject (§13.7.1).
   - `ts` lies within the replay window (default: 30 s skew).
   - `nonce` has not been seen for this `user` within the window. The nonce set is keyed by `(user, nonce)`.

   If all pass, the request is **verified**: the sender holds the seed for `user`, and `(account, user)` is the pair it signed. If any fails, reject with `401`. A failing signature is evidence of tampering or replay, never a weaker claim; it is rejected whatever the endpoint's `min_sender_trust`. An implementation MAY run the cheap checks before the signature; the outcome is the same.
4. A well-formed header without `sig`: **claimed**.

Rejections follow §9: an error-headered message, then the terminator.

| Code  | When                                                                                                                        |
|-------|-----------------------------------------------------------------------------------------------------------------------------|
| `400` | Malformed `Agent-Sender` header.                                                                                             |
| `401` | Signature required (§13.9) but absent; signature present and failing, in every mode; a refused sender that is claimed or absent — nothing was authenticated. |
| `403` | Sender verified, but not accepted by the receiver (§13.7.3).                                                                |

The nonce set lives in the receiving instance. Instances of one logical agent behind the `agents` queue group (§3.4) do not share it, so a replay that lands on another instance within the window passes the nonce check; the `ts` window bounds the exposure. A shared nonce store is a deployment option, not a protocol requirement.

#### 13.7.1 Acceptable `sub`

`sub` is always compared with the arrival subject of this message, never with a pattern: a pattern would let any agent holding a fresh signed header re-present it to a sibling under the same pattern. Exactly three forms are acceptable:

- **(a) Equal.** `sub` equals the arrival subject. This covers the direct case and the import renamed by the caller's own account (§13.6.2).
- **(b) Inserted account token.** The receiver is configured with an `account_token_position`, and `sub` equals the arrival subject with the token at that position removed. Behind such an export a caller may sign either the local name it publishes to or the token-bearing subject its import names; both verify, by (b) and (a).
- **(c) Renamed by the receiver's own import.** For a subject the signer also publishes for its own account (§13.6.2), where the receiver's account imported it under a different name. The receiver rebuilds, from the arrival subject and its own import, the one subject the message carried in the signer's account, and `sub` MUST equal it. This is an equality with a subject the receiver computes, never a match on a shape. Trailing tokens do not identify a subject: a rule on the tail alone would accept a signature the same agent made over another subject of the same shape — for a heartbeat on `agents.hb.{agent}.{owner}.{name}`, one over `agents.prompt.{agent}.{owner}.{name}`.

Whenever an `account_token_position` is configured, the token at that position of the arrival subject MUST equal the header's `account`, in every form. A position beyond the arrival subject's token count fails the check. The inserted token is a server stamp only on an endpoint no user of the receiver's own account can publish to; on an open endpoint a same-account user can publish the full subject, and a matching `account`, themselves.

#### 13.7.2 What the signature proves about `account`

The seed for `user` proves `user`. The same seed signs `account`, so `account` is the seed holder's word, bound to the signature: a relay cannot alter it; the sender could lie. `$SYS.REQ.USER.INFO` does not help the receiver: it describes the receiver's own connection, never the sender of a received message.

**The verified identity is `user`.** The protocol grants nothing on `account`. A user NKEY is a random ed25519 public key, so `user` alone makes the agent ID unique; `account` adds tenancy context and display value, never uniqueness. A forged `account` therefore gains nothing by itself: nothing in the protocol grants on it, and whoever does grant on it has verified it elsewhere.

A receiver that needs `account` verified uses a source outside the header: a server stamp where the deployment produces one — the token an `account_token_position` export inserts, or `Nats-Request-Info` — on an endpoint the deployment has closed to publishers in the receiver's own account, so that every request crossed an import; or a lookup in a registry it trusts. Such attestation is a deployment matter, outside the trust classes of §13.1.

#### 13.7.3 Acceptance

Whether a receiver accepts a verified sender is authorization, which the protocol leaves with the deployment (§10.1). This chapter defines how to verify a key, not which to accept. A receiver that refuses a verified sender answers `403`; one that refuses a claimed or absent sender answers `401`.

### 13.8 Stored messages

The header is not limited to live requests. JetStream stores client-set headers verbatim, so a signed `Agent-Sender` on a message published into a stream survives storage, replay, and redelivery. A consumer verifies it from the stored subject, payload, and header, later and offline: per-record, verifiable authorship. The server-injected `Nats-Request-Info` does not reach a stream; the signed header is the only authorship a stream consumer can verify.

Verification of a stored message differs from §13.7 in two ways:

- **Authorship only.** The consumer verifies the signature and `sub`, and skips the `ts` window and the nonce set: redelivery and replay show the same record again by design. A valid signature proves who signed the content. It does not prove that the record is unique, or that the signer published it into this stream: anyone with publish permission on the stream subject can store a copy, and the copy verifies. Consumers deduplicate on `(user, nonce)`. Publishers SHOULD set `Nats-Msg-Id` to the nonce, so the stream's own duplicate window helps. The stream sequence proves storage order, nothing more.
- **The stored subject is the arrival subject.** §13.7.1 applies with the stored subject, under the same configuration as live verification. A stream fed through an import renamed by the caller's account stores the exporter's subject, which the caller signed. A stream behind an export that inserts the account token stores the token-bearing subject, and the consumer removes the token by position. A stream `subject_transform`, or a mirror that transforms subjects, breaks the link: a stream whose records carry `Agent-Sender` MUST NOT transform subjects on the way in.

### 13.9 Declaring the requirement

An agent that implements this chapter MUST declare what its `prompt` endpoint requires of the sender, in that endpoint's metadata (§2.1), next to `max_payload` and `attachments_ok`:

```json
{
  "max_payload": "1MB",
  "attachments_ok": true,
  "min_sender_trust": "signed"
}
```

| Value    | The endpoint serves a request when                                                               |
|----------|--------------------------------------------------------------------------------------------------|
| `any`    | Always. The default, and what an endpoint without the field implies.                            |
| `signed` | The request is verified (§13.7). The receiver may still refuse the sender (`403`).              |

An endpoint that declares `signed` rejects a request that is not verified with `401`.

Callers read the requirement from `$SRV.INFO.agents` before sending, so an unsigned request fails predictably. A caller that reads an unknown value MUST treat it as `signed`, the strictest level it can satisfy on its own. There is no wire value for acceptance: the only thing a caller can act on is whether to sign.

The `status` endpoint (§8.7) declares nothing and is always answerable: a liveness probe MUST NOT depend on the prober's credentials. The caller still attaches `Agent-Sender` to a `status` request, and the receiver classifies it (§13.7) so the agent knows who probed it. A receiver MUST NOT reject a `status` request on identity grounds: a failing classification is logged and the reply is sent anyway.

### 13.10 Registration

An agent that implements this chapter adds its agent ID to the service metadata (§3.2), next to `agent`, `owner`, and `protocol_version`:

```json
{
  "user_nkey": "U…",
  "account": "A…",
  "id_sig": "<base64url>"
}
```

| Field       | Type   | Description                                                                 |
|-------------|--------|-----------------------------------------------------------------------------|
| `user_nkey` | string | The agent's user public NKEY.                                               |
| `account`   | string | The agent's account (§13.3).                                                |
| `id_sig`    | string | Signature over the identity fields (below), base64url without padding.      |

They are the pair the agent writes into every `Agent-Sender` header it sends.

Service metadata is self-declared. Without a signature any instance could register a foreign NKEY, capture reverse lookups for it, and misdirect callers to its own prompt subject. `id_sig` makes the claim verifiable. It is an ed25519 signature by the agent's user seed over:

```
AGENT-ID-V1\n{user_nkey}\n{account}\n{agent}\n{owner}\n{prompt_subject}\n
```

As with `Agent-Sender`, the input is never sent. A verifier rebuilds it from the `user_nkey`, `account`, `agent`, and `owner` metadata values and the `subject` of the instance's `prompt` endpoint, all read from the same `$SRV.INFO` record, and verifies `id_sig` against `user_nkey`. `prompt_subject` is the advertised subject, not one derived from the instance name: the agent chooses its subject (§2), and the subject is what a reverse lookup returns.

A valid `id_sig` proves the registrant holds the seed for `user_nkey` and vouches for that prompt subject. Metadata without `id_sig`, or with a failing one, is a claim.

**Reverse lookup.** A receiver maps a verified sender back to a protocol address by indexing `$SRV.INFO.agents` records by `(account, user_nkey)` and keeping only instances whose `id_sig` verifies. No verified instance means the sender is not a reachable agent: a human, a plain service, or an agent that is offline. Both proofs chain to the same seed, so they cannot disagree. Discovery is account-local (§4): the lookup sees only agents whose `$SRV` subjects the receiver's account can reach. Implementations SHOULD cache the index for a short time rather than enumerate per message. The lookup identifies; it never authorizes.

**What publication gives others.** Only public halves are published. A reader of `$SRV.INFO` can verify the agent's signatures, which is the point. They can claim the pair in an unsigned `Agent-Sender`, which lands in the claimed class. They can republish the agent's `id_sig`, which verifies only over the agent's own fields and prompt subject, and so re-announces the agent's registration and nothing else. Producing a `sig` or an `id_sig` that verifies, or connecting as the agent, needs the seed. A captured signed header is bound to its subject, payload, timestamp, and nonce.

### 13.11 Signed heartbeat

An agent that implements this chapter and holds its seed MUST attach `Agent-Sender` to every heartbeat it publishes (§8):

- `sub`: the heartbeat subject as published.
- `ts`: the heartbeat's own `ts` field (§8.3).
- `nonce`: fresh for every heartbeat.
- `sig`: over the heartbeat payload bytes as published, by the seed that signs `id_sig`.

Without a seed it publishes heartbeats without the header, as in 0.3. The payload does not change, so a subscriber that does not implement this chapter is unaffected. A signed heartbeat makes an agent's liveness attributable on its own, including to a subscriber in another account.

A subscriber that verifies heartbeats applies §13.7 to each. Where its own account imported the heartbeat subject under another name, `sub` is acceptable by §13.7.1 (c). There is no reply to carry an error: a heartbeat that fails verification is discarded, never downgraded to a claim.

An agent MUST NOT sign a reply. A `status` reply (§8.7) carries a heartbeat-shaped payload, but it arrives on the requester's inbox, where no signed subject could verify.

### 13.12 Security considerations

- **Trusted server.** The NATS connect handshake signs a server-chosen nonce with the same seed that signs `Agent-Sender`. A server the client should not trust — a hostile one, or a man in the middle on a connection without TLS — can present an `AGENT-SENDER-V1` input of its choice as that nonce and obtain a signature valid for the replay window against any agent. No implementation can prevent this; it is the precondition every NATS credential already has. Identity is meaningful only over TLS to a server whose certificate the client verifies.
- **Responder identity.** This chapter authenticates the caller to the receiver, not the responder to the caller. Any connection in the receiver's account with the right permissions can join the `agents` queue group on a prompt subject and answer in the agent's place; `id_sig` proves who registered, not who answered a given request. Account isolation and subject permissions remain the defense.
- **Confidentiality.** Identity only. Payload confidentiality stays with TLS and account isolation.
- **Display (informative).** Whatever shows an identity to a human should show its trust class, `verified` or `claimed`, next to it, so that a claim never reads as proof.

### 13.13 Relation to 0.3

Identity is additive:

- A caller with identity sending to an agent without it: the agent ignores the unknown header and serves the request.
- A caller without identity sending to an agent with it: the `prompt` endpoint declares `any` by default and serves the request with an absent sender. Only an endpoint that declares `signed` excludes callers without identity, and it says so in `$SRV.INFO`.
- The new metadata fields are additional metadata, which §3.2 already permits and requires relays to preserve.
- No subject, envelope, chunk, or heartbeat payload changes. §9.2 gains no codes.

The extension carries its own versions — `v` in the header, and the tags `AGENT-SENDER-V1` and `AGENT-ID-V1` — so it is versioned independently of the protocol. `protocol_version` stays `"0.3"`. Support is detectable without a version number: an agent implements this chapter if and only if its `prompt` endpoint metadata carries `min_sender_trust`; a caller implements it if and only if it sends `Agent-Sender`.

---

## Appendix A: Subject quick reference

```
# Identity and subjects (per §2 — verb-first, v0.3)
agents.prompt.{agent}.{owner}.{name}            # default `prompt` endpoint subject (§5, §6)
agents.hb.{agent}.{owner}.{name}                # liveness beacon (§8.1, protocol-fixed subject)
agents.status.{agent}.{owner}.{name}            # default `status` endpoint subject (§8.7)
agents.attachments.{agent}.{owner}.{name}       # default subject for future `attachments` endpoint (§5.5)

# Endpoint subjects are agent-chosen — the entries above are channel-plugin defaults, not
# mandatory. Callers learn actual endpoint subjects from $SRV.INFO.agents.

# Wildcards
agents.>                                        # all protocol traffic
agents.prompt.>                                 # all prompt endpoints
agents.hb.>                                     # all heartbeats (= agents.hb.*.*.*; protocol-fixed)
agents.status.>                                 # all status endpoints
agents.prompt.{agent}.>                         # all prompt endpoints on a harness
agents.prompt.*.{owner}.>                       # all prompt endpoints for an owner
agents.prompt.*.*.{name}                        # all prompt endpoints with this instance name

# Discovery — the only two stable subjects the protocol requires callers to know
$SRV.PING.agents                                # enumerate compliant agents (multi-response)
$SRV.INFO.agents                                # full service info per instance (multi-response)
$SRV.INFO.agents.{instance_id}                  # full service info for a specific instance
```

---

## Appendix B: Byte-level wire examples

JSON is shown formatted for readability; the wire uses compact UTF-8 encoding.

### B.1 Plain-text request

Published to `agents.prompt.claude-code.aconnolly.synadia-com-2` (the channel-plugin default subject for the `prompt` endpoint, v0.3 verb-first — the actual subject comes from `$SRV.INFO`):

```
summarize the attached report
```

Parsed as:

```json
{ "prompt": "summarize the attached report" }
```

### B.2 JSON request (text only)

```json
{"prompt":"summarize the attached report"}
```

### B.3 JSON request (text + attachment)

Valid only when the endpoint's `attachments_ok` metadata is `true`.

```json
{"prompt":"summarize","attachments":[{"filename":"report.pdf","content":"JVBERi0xLjQKJe..."}]}
```

### B.4 Response chunk (string `data`)

```json
{"type":"response","data":"Hello, world."}
```

### B.5 Response chunk (object `data`)

```json
{"type":"response","data":{"text":"Hello, world."}}
```

### B.6 Status chunk - `ack`

```json
{"type":"status","data":"ack"}
```

The mandatory first chunk on every response stream (§6.4). Confirms request acceptance and resets the caller's inactivity timeout (§6.6). The stream continues with `response` chunks.

### B.7 Query chunk

```json
{"type":"query","data":{"id":"a8f1c2e4-9b63-4d7e-aaaa-112233445566","reply_subject":"_INBOX.Xj7k9Q2pA","prompt":"Confirm? (yes/no)"}}
```

### B.8 Query reply (plain-text shorthand)

Published to the query's `reply_subject`:

```
yes
```

### B.9 Empty-payload terminator

A NATS message with:

- Zero-byte body.
- No NATS headers.

Published to the reply subject as the final message of every stream - successful or errored.

### B.10 Error signal + terminator

Error-terminated streams end with two messages.

**Message 1** - error signal:

Headers:
```
Nats-Service-Error-Code: 429
Nats-Service-Error: rate limited
```

Body (optional, per §9.1):
```json
{"error":"rate_limited","message":"Too many concurrent requests","retry_after_s":30}
```

**Message 2** - the empty terminator (B.9).

### B.11 Heartbeat

Published to `agents.hb.claude-code.aconnolly.synadia-com-2` (v0.3 verb-first):

```json
{"agent":"claude-code","owner":"aconnolly","session":"synadia-com-2","instance_id":"VMKS6MHK71PCPWGY38A7N5","ts":"2026-04-28T14:23:01Z","interval_s":30}
```

### B.11a Status request / reply (v0.3 §8.7)

Request — empty body to `agents.status.claude-code.aconnolly.synadia-com-2`:

```
(empty)
```

Reply — same JSON shape as a heartbeat (B.11), freshly built per request:

```json
{"agent":"claude-code","owner":"aconnolly","session":"synadia-com-2","instance_id":"VMKS6MHK71PCPWGY38A7N5","ts":"2026-04-28T14:23:01Z","interval_s":30}
```

### B.12 Service info response

Returned by `$SRV.INFO.agents` (one response per instance):

```json
{
  "name": "agents",
  "id": "VMKS6MHK71PCPWGY38A7N5",
  "version": "0.3.0",
  "description": "Claude Code — synadia-com-2",
  "metadata": {
    "agent": "claude-code",
    "owner": "aconnolly",
    "session": "synadia-com-2",
    "protocol_version": "0.3"
  },
  "endpoints": [
    {
      "name": "prompt",
      "subject": "agents.prompt.claude-code.aconnolly.synadia-com-2",
      "queue_group": "agents",
      "metadata": {
        "max_payload": "1MB",
        "attachments_ok": true
      }
    },
    {
      "name": "status",
      "subject": "agents.status.claude-code.aconnolly.synadia-com-2",
      "queue_group": "agents"
    }
  ]
}
```

### B.13 Signed request (§13)

Known-answer vector `operator-account-signed` from the SDKs' shared fixtures, [`test-fixtures/identity/sender-vectors.json`](https://github.com/synadia-ai/synadia-agents/blob/main/test-fixtures/identity/sender-vectors.json). The seed is a throwaway test key, published so the signature can be reproduced. Never use it anywhere else.

Inputs:

```
seed      SUAEJ6GDK6FSSB54LD45Q7W25AW7NUT7MVLBABIR5MIPFUTBW7ZNPK2KYE   (test only)
account   AABYLMBR6Q2CDXTLGRQCFA2GP76BGCDF7NZF2OVHH4RQ7L3Y3TZWJDRL
user      UCDUW5V44EBDBIK2FL4CTNDBQFNGBEZVJHSZGQVKRHASN4AV4IWPB5NT
subject   agents.prompt.demo-agent.alice.example
payload   {"prompt":"hello"}                                           (18 bytes)
ts        2026-08-28T12:00:00Z
nonce     nonce-operator-account-signed
```

Agent ID (§13.3, 113 characters):

```
AABYLMBR6Q2CDXTLGRQCFA2GP76BGCDF7NZF2OVHH4RQ7L3Y3TZWJDRL.UCDUW5V44EBDBIK2FL4CTNDBQFNGBEZVJHSZGQVKRHASN4AV4IWPB5NT
```

Signed input (§13.6.1): seven lines, each ending in one LF. The last line is the SHA-256 of the 18 payload bytes. Never sent.

```
AGENT-SENDER-V1
AABYLMBR6Q2CDXTLGRQCFA2GP76BGCDF7NZF2OVHH4RQ7L3Y3TZWJDRL
UCDUW5V44EBDBIK2FL4CTNDBQFNGBEZVJHSZGQVKRHASN4AV4IWPB5NT
agents.prompt.demo-agent.alice.example
2026-08-28T12:00:00Z
nonce-operator-account-signed
8a44725210b9dcd4fefd9f0eca07b70ae45e69274a3105fb25eb426a2cf8bbf4
```

`sig` — ed25519 over the signed input, base64url without padding (64 bytes, 86 characters):

```
EPGc0tNF0ZQeBqXUduwUp3zdi33k1AKIXOX-PUPSeY9QtS4V3d2EZLq-fHuPuYkw6Cc3HdVn_jVI58Y_3YBZBw
```

Published to `agents.prompt.demo-agent.alice.example` — header block, then payload. Header lines end in CRLF.

```
NATS/1.0
Agent-Sender: {"v":1,"account":"AABYLMBR6Q2CDXTLGRQCFA2GP76BGCDF7NZF2OVHH4RQ7L3Y3TZWJDRL","user":"UCDUW5V44EBDBIK2FL4CTNDBQFNGBEZVJHSZGQVKRHASN4AV4IWPB5NT","name":"claude-code","sub":"agents.prompt.demo-agent.alice.example","ts":"2026-08-28T12:00:00Z","nonce":"nonce-operator-account-signed","sig":"EPGc0tNF0ZQeBqXUduwUp3zdi33k1AKIXOX-PUPSeY9QtS4V3d2EZLq-fHuPuYkw6Cc3HdVn_jVI58Y_3YBZBw"}

{"prompt":"hello"}
```

The header value is 373 bytes; the header block (`NATS/1.0␍␊Agent-Sender: …␍␊␍␊`) is 401 bytes, which count toward `max_payload` together with the 18 payload bytes (§13.5.1). `name` is outside the signature.

At the receiver the arrival subject equals `sub` (§13.7.1 (a)). The receiver rebuilds the seven lines from the header fields and the SHA-256 of the received payload, and verifies `sig` against `user`.

---

## Appendix C: Known agent identifiers (informative)

Informative, not normative. Agent identifiers are not centrally registered (§2.2); the list below captures values in common use so new implementations can pick a non-colliding identifier without coordination.

| `metadata.agent` | Subject abbreviation | Harness / product            | Session semantics                                                                                          |
|------------------|----------------------|------------------------------|------------------------------------------------------------------------------------------------------------|
| `claude-code`    | `cc`                 | Anthropic's Claude Code CLI  | Session-aware. Each session SHOULD be its own registration; `metadata.session` carries the label.           |
| `openclaw`       | `oc`                 | OpenClaw agent runtime       | Session-less. `metadata.session` MAY be omitted or set to `"default"`. Instance name often `default`.       |
| `pi`             | `pi`                 | `pi` agent harness           | Session-aware. Same convention as `claude-code`.                                                            |
| `hermes`         | `hermes`             | Hermes agent harness         | Session-aware. Same convention as `claude-code`.                                                            |
| `dspy`           | `dspy`               | DSPy / ax-llm ReAct agent    | Session-aware. Same convention as `claude-code`.                                                            |

Additions to this table are non-normative; deployers SHOULD coordinate before claiming a new identifier to avoid collisions.
