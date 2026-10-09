<div align="center">

# AI Customer Support & Voice Agent Automation

**Backend orchestration for an external AI voice platform, implemented as a 40-node n8n workflow**

An authenticated webhook that normalizes voice-agent tool calls, verifies protected requests, enforces PostgreSQL-backed idempotency, routes five customer-support actions, and returns one structured JSON response.

<br>

![n8n](https://img.shields.io/badge/n8n-Workflow-EA4B71?style=flat-square&logo=n8n&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Idempotency%20%26%20Logging-4169E1?style=flat-square&logo=postgresql&logoColor=white)
![Voice Agent](https://img.shields.io/badge/Voice%20Agent-Tool%20Calls-0969DA?style=flat-square)
![AI Automation](https://img.shields.io/badge/AI%20Automation-Grounded%20Answers-8957E5?style=flat-square)

![HMAC-SHA256](https://img.shields.io/badge/HMAC--SHA256-Verification-0969DA?style=flat-square)
![Idempotency](https://img.shields.io/badge/Idempotency-Owner%20Fenced-0969DA?style=flat-square)
![Human-in-the-Loop](https://img.shields.io/badge/Human--in--the--Loop-Escalation-0969DA?style=flat-square)
![REST APIs](https://img.shields.io/badge/REST%20APIs-Configurable-0969DA?style=flat-square)

</div>

---

## Overview

This repository contains the n8n workflow that serves as the backend orchestration layer between an external AI voice platform and a set of customer-support services. The voice platform calls the workflow's webhook whenever its agent invokes a support tool. The workflow validates the request, decides whether it is safe to run, executes the matching support action once per idempotency key, and returns a structured JSON result that the voice agent can speak or act on.

The project is complete and has been tested, with positive results from the tests executed. The validation scenarios used are preserved in [Testing and Validation](#testing-and-validation) as a repeatable reference for future deployments.

**What it receives.** An authenticated `POST` to the webhook path `voice-agent/support`. The normalizer accepts a canonical generic payload, a nested provider envelope, or a tool-call-list envelope, and reads the provider, session, call, tool-call, customer, message, and argument fields from them.

**What it processes.** Request normalization and validation, conditional HMAC verification for protected order lookups, request fingerprinting, an atomic PostgreSQL idempotency claim, intent routing, calls to configurable external HTTP services, LLM answer generation grounded in retrieved knowledge, evidence-gated persistence, and metadata logging.

**What it returns.** A single JSON response through one Respond to Webhook node, with `Content-Type: application/json; charset=utf-8`, `Cache-Control: no-store`, and an HTTP status code chosen per execution path.

The workflow is designed around a practical problem: voice agents retry, time out, and repeat tool calls, while some support actions (appointments, tickets, escalations) must not be duplicated and some data (order status) must not be exposed without verification.

> [!NOTE]
> This README documents the behavior implemented by the workflow. It does not state test counts, coverage figures, traffic volumes, latency results, or business outcomes.

## Workflow Screenshot

<img width="800" height="478" alt="1790416973137" src="https://github.com/user-attachments/assets/9b39fdeb-f528-4677-ad65-3002bdcb5014" />

<p align="center"><em>Workflow overview in n8n</em></p>


## Key Capabilities

| Capability | What the workflow implements |
|---|---|
| **Canonical request normalization** | Three input adapters (tool-call envelope, nested provider envelope, canonical generic) mapped to one internal schema. Control characters are stripped and field lengths are capped. |
| **Request validation** | Per-intent required-field checks, timestamp sanity checks, idempotency-key conflict detection, and a check that the external service configuration needed for the requested intent is present. |
| **Secure customer verification** | `get_order_status` fails closed unless a fresh, HMAC-SHA256 signed verification context matches the request. |
| **Request fingerprinting** | SHA-256 over a canonical JSON representation of the request, used to detect an idempotency key reused for a different request. |
| **PostgreSQL-backed idempotency** | Atomic claim with leases, owner tokens, fingerprint and intent matching, duplicate replay, and key-reuse detection. |
| **Stale execution handling** | Expired leases are reclaimed for read-only requests. Stale side-effecting requests are routed to fenced reconciliation from durable provisional evidence. |
| **Side-effect evidence gating** | A side-effecting result must be durably recorded as evidence before the idempotency row can become `DONE`. Uncertain outcomes never complete idempotency. |
| **Knowledge-grounded AI responses** | Retrieved knowledge is filtered to verified items, supplied to the LLM, and the generated answer is checked by deterministic rules against that reference context. Answers that fail are rejected and routed to human support. |
| **Customer intent routing** | Five supported actions plus an explicit unsupported-tool fallback. |
| **Appointment availability and booking** | Advisory availability lookup followed by booking with the canonical idempotency key and an optional reservation token. |
| **Support tickets and human escalation** | Dedicated configurable APIs, each called with the idempotency key and without blind workflow retries. |
| **Structured failure responses** | Every failure class is converted to the same JSON response contract with an error code, `retryable` flag, and `consistencyState`. |
| **Metadata-only operational logging** | A best-effort PostgreSQL log row per request that deliberately excludes conversation content. |

## Supported Customer Actions

The `toolName` supplied by the voice platform is lower-cased and mapped to one of five intents. Any other tool name resolves to `unknown` and is answered with an `UNSUPPORTED_TOOL` response (HTTP 400) that flags escalation as required.

| Intent | Accepted tool names | Purpose | Required inputs |
|---|---|---|---|
| `faq` | `faq`, `general_question`, `knowledge_base_question`, `knowledge_base`, `get_faq` | Answers a general question from verified retrieved knowledge using the configured LLM. | Customer message |
| `get_order_status` | `get_order_status`, `order_status` | Returns order status after secure verification, using deterministic logic and the order API. | Customer identity with phone or email, `orderId`, valid verification context |
| `book_appointment` | `book_appointment` | Checks availability, then books the appointment. | Customer identity, `preferredDate`, `preferredTime`, `timezone` |
| `create_support_ticket` | `create_support_ticket`, `support_ticket` | Creates a ticket in the configured ticketing API. | Customer identity, issue or message |
| `escalate_to_human` | `escalate_to_human`, `human_escalation` | Creates a human-support escalation. | Customer identity, reason or message |

All intents also require a session identifier (falling back to the call ID), a tool name, and a stable request identity (see [Idempotency](#idempotency)). Only the `faq` path uses an LLM; order lookups and side-effecting actions use deterministic workflow logic and their respective external APIs.

## Architecture

The repository contains the n8n orchestration layer. It integrates with an external voice platform on the upstream side and with configurable HTTP services (knowledge base, LLM, order, calendar, ticketing, and escalation) and PostgreSQL on the downstream side. Each service boundary is defined by the endpoint and response-field contracts documented in [External Service Integrations](#external-service-integrations).

| Layer | Responsibility | Key nodes |
|---|---|---|
| **Ingress** | Header-authenticated `POST` webhook; response deferred to a Respond to Webhook node. | Webhook - Voice Agent Input |
| **Normalization** | Adapter selection, field extraction, sanitization, validation issues, verification payload and fingerprint source construction. | Normalize & Validate Request |
| **Security** | Conditional HMAC computation, verification state derivation, constant-time signature comparison. | HMAC Verification Required?, Verify Customer HMAC Context, Finalize Security & Request Identity |
| **Request identity** | SHA-256 request fingerprint and canonical idempotency key derivation. | Hash Request Fingerprint, Finalize Security & Request Identity |
| **Idempotency** | Atomic claim, state interpretation and routing, duplicate replay, stale reconciliation. | Claim Idempotency Key, Interpret Idempotency Result, Route Idempotency State, Build Duplicate Response, Reconcile Stale Side-Effect Evidence |
| **Intent routing** | Switch on the canonical intent with a fallback output. | Route Customer Intent |
| **AI / knowledge** | Knowledge retrieval, verified-knowledge filtering, LLM generation, grounded response validation. | Search Knowledge Base through Validate Grounded AI Response |
| **Business actions** | Order lookup, calendar availability and booking, ticket creation, human escalation. | Get Order Status, Check Appointment Availability, Create Appointment, Create Support Ticket, Create Human Escalation |
| **Persistence** | Side-effect evidence, idempotency result, persistence interpretation and routing. | Persist Side-Effect Evidence, Side-Effect Evidence Gate, Persist Idempotency Result, Interpret Persistence Result, Route Persistence Result |
| **Observability** | Metadata-only request log. | Log Conversation |
| **Response** | Unified response builders and a single terminal webhook response. | Build Final Response & Log, failure builders, Return Voice Response |

<details>
<summary><strong>Node inventory (40 nodes)</strong></summary>

<br>

| Group | Count | Nodes |
|---|---|---|
| Ingress and normalization | 2 | Webhook - Voice Agent Input; Normalize & Validate Request |
| Security and request identity | 5 | HMAC Verification Required?; Verify Customer HMAC Context; Hash Request Fingerprint; Finalize Security & Request Identity; Request Valid? |
| Idempotency | 7 | Claim Idempotency Key; Interpret Idempotency Result; Route Idempotency State; Build Duplicate Response; Reconcile Stale Side-Effect Evidence; Build Stale Reconciliation Response; Build Idempotency Failure |
| Intent routing | 1 | Route Customer Intent |
| AI and knowledge | 5 | Search Knowledge Base; Prepare Verified Knowledge; Verified Knowledge Available?; Generate AI Response; Validate Grounded AI Response |
| Business actions | 7 | Get Order Status; Check Appointment Availability; Appointment Slot Available?; Create Appointment; Create Support Ticket; Create Human Escalation; Build Appointment Availability Outcome |
| Response and persistence | 9 | Build Final Response & Log; Persist Side-Effect Evidence; Side-Effect Evidence Gate; Build Side-Effect Evidence Failure; Persist Idempotency Result; Interpret Persistence Result; Route Persistence Result; Log Conversation; Return Voice Response |
| Failure builders | 4 | Build Validation Action Result; Build Fingerprint Failure Response; Build Security Processing Failure; Build Terminal Response Failure |

The workflow uses `executionOrder: v1` and is exported with `active: false`.

</details>

## Request Processing Stages

| Stage | Purpose |
|---|---|
| 01 | **Ingestion.** The webhook accepts an authenticated `POST` on `voice-agent/support` and defers the reply to the response node. |
| 02 | **Normalization.** Adapter selection, field extraction, sanitization, intent mapping, and generation of a request ID when none is supplied. |
| 03 | **Base validation.** Required fields per intent, timestamp checks, idempotency-key conflict check, and required-configuration check. |
| 04 | **Verification gate.** Only `get_order_status` is sent through HMAC computation; other intents follow their own configured validation requirements and skip it. |
| 05 | **Fingerprint.** SHA-256 over the canonical fingerprint source. |
| 06 | **Security and identity finalization.** Derives the verification state, the fingerprint (`fp-` prefix), and the canonical idempotency key, then computes `isValid`. |
| 07 | **Validity gate.** Invalid requests are answered by the validation builder without claiming an idempotency key. |
| 08 | **Idempotency claim.** One atomic PostgreSQL statement inserts, reclaims, or classifies the key. |
| 09 | **Idempotency routing.** Business execution, duplicate replay, active, stale reconciliation, key reuse, or error. |
| 10 | **Intent routing.** One of five action paths, or the unsupported-tool fallback. |
| 11 | **Action execution.** Knowledge retrieval and LLM answer, order lookup, availability and booking, ticket creation, or escalation. |
| 12 | **Response construction.** Upstream results are classified and converted to the response contract with a `consistencyState`. |
| 13 | **Evidence persistence and gate.** For side-effecting intents, the provisional result is recorded and checked before finalization. |
| 14 | **Idempotency persistence.** The row becomes `DONE` or `RETRYABLE_ERROR`, subject to owner, fingerprint, intent, and evidence fences. |
| 15 | **Persistence interpretation.** Persistence outcomes can downgrade a response to an explicit uncertain or unconfirmed state. |
| 16 | **Metadata logging.** One row is written to the request log, best-effort. |
| 17 | **Response.** One terminal JSON response with the selected HTTP status. |

## Upstream Request Contract

The voice platform integrates with the workflow through a defined request contract.

| Element | Contract |
|---|---|
| **Transport** | Authenticated `POST` to `voice-agent/support` using the configured webhook header credential. |
| **Input formats** | Canonical generic payload, nested provider envelope, or tool-call-list envelope. All are normalized to one internal schema. |
| **Request identity** | A stable `requestId` (generated when none is supplied), plus provider, session, call, and tool-call identifiers. |
| **Idempotency key** | An explicit `idempotencyKey` (body) or `x-idempotency-key` (header), or a stable call and tool-call identity from which the key is derived. See [Idempotency](#idempotency). |
| **Intent mapping** | `toolName` is lower-cased and mapped to one of five intents (see [Supported Customer Actions](#supported-customer-actions)). |
| **Protected verification context** | For `get_order_status`: verification reference, status, timestamp, and signature, supplied in the body or in the `x-verification-signature` and `x-verification-timestamp` headers. See [Protected Order-Status Verification](#protected-order-status-verification). |
| **Session identity** | A session identifier, falling back to the call ID. |

## AI Response Handling

AI is used only on the `faq` path, which implements a knowledge-grounded response pipeline: retrieve, filter to verified reference material, generate, and validate deterministically before returning.

| Step | Behavior |
|---|---|
| **Retrieval** | `POST` to the knowledge-base search endpoint with the customer message as `query` and `topK` of 5. Timeout 10 s; up to 2 attempts with a 1.5 s wait. |
| **Verified knowledge preparation** | Results are read from `results` or `documents`. An item is kept only if the response sets `sourceVerified: true` or the item sets `verified: true`. Content is sanitized and capped at 2,500 characters; at most 5 items are kept. |
| **Verified-knowledge gate** | If no verified knowledge is available, the LLM is not called and the request is routed to human support. |
| **Generation** | `POST` to the configured chat-completions endpoint with `temperature` 0.2. The system message treats customer and retrieved content as untrusted data, restricts answers to the supplied reference data, and directs the model to recommend human support when the data does not support an answer. The user message carries the question and the verified reference data as JSON. Timeout 10 s; up to 2 attempts. |
| **Grounded validation** | The answer is capped at 1,200 characters and checked against the retrieved reference context by the deterministic rules below. |

**Validation rules applied to the generated answer.** Validation uses lexical grounding checks against the verified reference context. These rules enforce that answers are supported by retrieved material and that unsupported, unsafe, or insufficiently grounded answers do not reach the caller.

| Check | Failure code |
|---|---|
| Empty answer | `AI_RESPONSE_EMPTY` |
| Credential-like assignments, instruction-override phrases, system or developer tags, or references to a system prompt or internal architecture | `AI_RESPONSE_UNSAFE_CONTENT` |
| High-risk claim wording (price, refund, policy, delivery, fee, warranty, cancellation, and similar) or currency or date patterns, with fewer than two shared significant words with the reference data | `AI_RESPONSE_UNGROUNDED_CLAIM` |
| A numeric value in the answer that does not appear in the reference text, when high-risk wording is present | `AI_RESPONSE_UNSUPPORTED_FACT` |
| A non-refusal answer sharing no significant words with the reference data | `AI_RESPONSE_UNGROUNDED` |
| A non-refusal answer with no verified source text | `AI_RESPONSE_NO_VERIFIED_SOURCE` |

A validation failure produces an unsuccessful, retryable response (HTTP 502) that flags human support as required. The validation strategy is lexical and rule-based; it is not a semantic fact-verification engine. Its purpose is to reject answers that are not grounded in verified reference material and to route those cases to human assistance.

**Knowledge-base and LLM integration.** The retrieval endpoint is `SUPPORT_KB_BASE_URL` plus `/v1/search`. The generation endpoint is `SUPPORT_LLM_BASE_URL` plus `/chat/completions`, with the model name taken from `SUPPORT_LLM_MODEL`. Both are configurable external services; the knowledge-base implementation and the model are provided by the deployment environment.

## Security Design

The security architecture combines authenticated ingress, signed verification for protected data, request identity controls, database-fenced state management, grounded AI output handling, and metadata-only logging. These controls reduce risk in the areas they cover; they are not a claim of universal protection against every possible attack.

| Mechanism | Purpose | Implementation |
|---|---|---|
| **Authenticated webhook** | Restricts who can invoke the workflow. | Webhook node uses header authentication through an n8n credential. |
| **HMAC-SHA256 verification** | Binds protected order access to a signed customer-verification context. | Crypto node computes a hex HMAC with `VOICE_VERIFICATION_HMAC_SECRET` over a versioned payload; applied to `get_order_status`. Other intents follow their own configured validation requirements. |
| **Context binding** | Prevents a signature for one context being reused in another. | The signed payload includes provider, session, call, tool call, intent, verification reference, verification timestamp and status, customer fields, and the order ID. |
| **Freshness check** | Limits the lifetime of a verification. | Age must be non-negative and no greater than `VERIFICATION_MAX_AGE_SECONDS` (default 300). |
| **Constant-time comparison** | Avoids timing differences when comparing signatures. | A length-padded XOR comparison over the supplied and computed signatures. A `sha256=` prefix on the supplied value is accepted. |
| **Fail-closed order access** | Never returns order data on an unverified request. | Any verification state other than `verified` invalidates the request. The response builder re-checks verification before accepting an order lookup. |
| **Request fingerprinting** | Detects reuse of a key for a different request. | SHA-256 over provider, intent, channel, customer, message, and normalized arguments. |
| **Idempotency-key conflict detection** | Rejects ambiguous caller input. | A body key and a header key that differ are rejected as a validation issue. |
| **Owner-token fencing** | Ensures only the execution that holds a row can finalize it. | Each execution carries a claim token; update statements require a matching owner token. |
| **PostgreSQL-backed state management** | Provides atomic, durable coordination of concurrent and repeated requests. | Single-statement claim with row locking and lease expiry. |
| **Durable side-effect evidence** | Prevents completion without durable evidence. | See [Side-Effect Safety](#side-effect-safety). |
| **Grounded AI answer validation** | Limits unsupported or unsafe generated content. | Verified-knowledge gate, system instructions that treat customer and retrieved text as data, and deterministic output screening before the answer is returned. |
| **Metadata-only logging** | Keeps conversation content out of the operational log. | See [Operational Logging](#operational-logging). |
| **Centralized configuration and secret handling** | Keeps secrets and endpoints out of the workflow JSON. | n8n variables and n8n credentials. |

### Protected Order-Status Verification

HMAC-SHA256 verification protects the order lookup path. For `get_order_status`, the caller supplies a trusted verification context (reference, status, timestamp, and signature, in the body or in the `x-verification-signature` and `x-verification-timestamp` headers). A valid signed context is a deliberate requirement of the order lookup path. The workflow derives one of these states:

| State | Condition |
|---|---|
| `missing_reference` | No verification reference. |
| `unverified` | Status is not `verified`. |
| `invalid_timestamp` | Timestamp absent or unparsable. |
| `future_timestamp` | Timestamp is in the future. |
| `expired` | Age exceeds the configured maximum. |
| `missing_signature` | No signature supplied. |
| `invalid_signature` | Supplied signature does not match the computed HMAC. |
| `verified` | All checks pass. |

Only `verified` allows the request to proceed. Every other state returns HTTP 403 with a specific error code. If the HMAC computation itself fails, the response is HTTP 503 with `VERIFICATION_BACKEND_UNAVAILABLE`.

The signing component (typically the voice platform or a service trusted by it) and this workflow share `VOICE_VERIFICATION_HMAC_SECRET` and the signed payload format below.

<details>
<summary><strong>Signed payload field order</strong></summary>

<br>

The signature is computed over the JSON string produced by the workflow with this fixed key order. A caller producing signatures must reproduce the same string, using the normalized (trimmed, sanitized) values, a lower-cased provider, status, and email, and the canonical intent name.

1. `version` (the literal `support-verification-v1`)
2. `provider`
3. `sessionId`
4. `callId`
5. `toolCallId`
6. `intent`
7. `verificationReference`
8. `verificationTimestamp` (as supplied)
9. `verificationStatus`
10. `customerId`
11. `customerName`
12. `customerPhone`
13. `customerEmail`
14. `protectedResourceId` (the order ID for `get_order_status`)

Missing values are represented as empty strings.

</details>

## Idempotency

Idempotency and side-effect safety are core engineering features of this workflow. Repeated, concurrent, and interrupted requests are controlled by a PostgreSQL-backed state machine, so the workflow's own control logic does not execute the same action twice for the same key.

The key is the caller-supplied `idempotencyKey` (body field or `x-idempotency-key` header) or, when absent, a key derived from stable request identifiers. Request data is first normalized to a canonical form, and a SHA-256 fingerprint of that canonical representation is stored with the claim.

| Intent type | Accepted key sources |
|---|---|
| Side-effecting (`book_appointment`, `create_support_ticket`, `escalate_to_human`) | Supplied key, or `provider:callId:toolCallId`. |
| Read-only (`faq`, `get_order_status`) | Supplied key, or `provider:callId:toolCallId`, or `provider:tool:toolCallId`, or `provider:request:requestId` when the request ID was supplied. |

Requests without an acceptable key are rejected during validation.

**Internal and external idempotency.** Internal idempotency is enforced by this workflow's PostgreSQL state machine (atomic claims, leases, owner-token fencing, replay, and reconciliation). External idempotency is part of the integration contract: the canonical key is sent to the calendar, ticket, and escalation services in the request body, and those services are expected to honor it as part of their API contract. The workflow does not assume that any remote provider guarantees idempotency beyond what that provider's own contract establishes, and it handles uncertain outcomes explicitly (see [Side-Effect Safety](#side-effect-safety)).

### Claim behavior

The claim is a single PostgreSQL statement. It inserts an `IN_PROGRESS` row with `ON CONFLICT DO NOTHING`; if the row exists, it locks it (`FOR UPDATE`) and either reclaims it or classifies it. The stored metadata includes an owner token, request fingerprint, intent, side-effecting flag, and lease length. The lease is `SUPPORT_IDEMPOTENCY_LEASE_SECONDS` (default 300), with a minimum of 30 seconds enforced in the query.

### State model

| State | Meaning | Outcome |
|---|---|---|
| `new` | Key claimed for the first time. | Business execution proceeds. |
| `reclaimed` | A read-only `IN_PROGRESS` row whose lease expired, with a matching fingerprint and intent, was taken over. | Business execution proceeds. |
| `reclaimed_retryable` | A `RETRYABLE_ERROR` row with matching fingerprint, intent, and side-effect flag was taken over. | Business execution proceeds. |
| `reclaimed_side_effect` | A side-effecting `IN_PROGRESS` row whose lease expired was taken over by this execution. | Routed to fenced stale reconciliation; the action is not re-run. |
| `duplicate_done` | The key is `DONE` with a matching fingerprint. | Stored response is replayed with action `idempotency_replay`. Missing replay metadata gives HTTP 503 `IDEMPOTENCY_METADATA_MISSING`; a stored non-terminal result gives HTTP 409 `IDEMPOTENCY_STORED_RETRYABLE_RESULT`; a missing stored result gives HTTP 503 `IDEMPOTENCY_RESULT_MISSING`. |
| `active` | The key is `IN_PROGRESS` and the lease has not expired. | HTTP 409 `REQUEST_IN_PROGRESS`, retryable. |
| `stale` | The lease has expired but the row could not be reclaimed by the rules above. | Routed to reconciliation, which returns an explicit uncertain response unless this execution owns the row and evidence supports a result. |
| `key_reuse` | Same key, different fingerprint or intent. | HTTP 409 `IDEMPOTENCY_KEY_REUSE`, not retryable. |
| `error` | Backend failure, unknown status, or a `RETRYABLE_ERROR` row that cannot be reclaimed. | HTTP 503 with `IDEMPOTENCY_BACKEND_FAILURE`, `IDEMPOTENCY_STATE_ERROR`, or `RETRYABLE_ERROR_NOT_RECLAIMED`-derived handling. |

### Stored statuses

| Status | Use |
|---|---|
| `IN_PROGRESS` | A live or leased claim. |
| `DONE` | Terminal result stored for replay. |
| `RETRYABLE_ERROR` | Transient, non-terminal result; reclaimable with the same key, fingerprint, and intent. |
| `ERROR` | Recognized by the claim query and treated as the `error` state. |

### Stale side-effect reconciliation

For a side-effecting request the workflow never re-runs the external call after a stale lease. The reconciliation statement locks the row only if the status is `IN_PROGRESS` and the owner token, fingerprint, intent, and side-effecting flag all match, and then evaluates the durable provisional evidence:

| Reconciliation state | Condition | Response |
|---|---|---|
| `RECONCILED` | Provisional response has `consistencyState: confirmed` and its recorded owner, fingerprint, and intent match. The row is moved to `DONE`. | The stored result is replayed with action `stale_reconciliation_replay`. |
| `UNCERTAIN_EVIDENCE` | Provisional evidence exists but is not confirmed. | HTTP 503 `STALE_SIDE_EFFECT_OUTCOME_UNCERTAIN`. |
| `NO_EVIDENCE` | The row is owned but holds no provisional evidence. | HTTP 409 `STALE_SIDE_EFFECT_REQUIRES_RECONCILIATION`, retryable. |
| `NOT_FOUND_OR_OWNER_CHANGED` | The row is no longer owned by this execution. | HTTP 503 `STALE_SIDE_EFFECT_OWNER_CHANGED`. |

All non-reconciled responses instruct the caller to reuse the same idempotency key and not to create the action again.

## Side-Effect Safety

Side-effecting intents are `book_appointment`, `create_support_ticket`, and `escalate_to_human`. The following controls work together to avoid duplicate or unconfirmed side effects.

| Control | Behavior |
|---|---|
| **Idempotency key propagation** | The canonical key is sent in the body of the availability, appointment, ticket, and escalation requests. |
| **No blind retries** | Appointment, ticket, and escalation `POST` nodes have workflow-level retry disabled. Only the knowledge-base, LLM, and order-lookup nodes retry (2 attempts, 1.5 s wait). |
| **Structured uncertain outcomes** | A timeout, network error, or 5xx from a side-effecting call is reported as `side_effect_outcome_uncertain` with `consistencyState: uncertain`, `retryable: true`, and codes such as `APPOINTMENT_OUTCOME_UNCERTAIN`, `TICKET_OUTCOME_UNCERTAIN`, or `ESCALATION_OUTCOME_UNCERTAIN`. These are never treated as success. |
| **Confirmation by provider reference** | Success requires a returned identifier (appointment, ticket, or escalation ID). A 2xx response without one is reported as an invalid response with `consistencyState: uncertain`. |
| **Owner-token fencing** | Evidence and result updates require `IN_PROGRESS` status and matching owner token, fingerprint, intent, and side-effecting flag. |
| **Provisional evidence** | Before finalization, the response is written to the idempotency row as provisional evidence. |
| **Evidence gate** | The gate passes when evidence is not required, when the request is not side-effecting and evidence was recorded, or when evidence was recorded and the response is `confirmed`. Otherwise the request is answered with a `SIDE_EFFECT_EVIDENCE_*` or `SIDE_EFFECT_OUTCOME_UNCERTAIN` error and does not proceed to the idempotency result write. |
| **Finalization fence** | For a side-effecting `DONE` write, the response must be `confirmed` and must equal the stored provisional response. |
| **Persistence failure handling** | If the final write fails or loses its fence for a side-effecting intent, the response is downgraded to `result_persistence_uncertain` (`*_RESULT_PERSISTENCE_UNCERTAIN`, HTTP 503). |
| **Retryable error states** | Retryable responses with a `confirmed` or `retryable` consistency state are stored as `RETRYABLE_ERROR` so the same key can be reclaimed. |

Together with the canonical key sent to each service, these controls give the workflow a consistent, auditable path for every side-effecting request: confirmed results are durably recorded and replayable, and any outcome that cannot be confirmed is reported explicitly as uncertain rather than as success.

## Appointment, Ticket, and Escalation Behavior

### Appointment booking

| Step | Behavior |
|---|---|
| Availability lookup | `POST` to `/v1/availability` with the idempotency key, preferred date and time, timezone, appointment type, and customer. Timeout 10 s, no retry. |
| Availability decision | A 2xx response with `available: true` proceeds to booking. Otherwise the outcome builder responds: `UPSTREAM_RATE_LIMITED` (429), timeout or server failure codes (retryable), `APPOINTMENT_AVAILABILITY_REJECTED` (4xx), `APPOINTMENT_UNAVAILABLE` (HTTP 409, with up to three alternatives from `alternatives`, `alternativeSlots`, or `slots`), or `APPOINTMENT_AVAILABILITY_RESPONSE_INVALID`. |
| Advisory availability | Availability is advisory; the booking response is authoritative. A slot conflict at booking time returns `APPOINTMENT_SLOT_TAKEN` (HTTP 409). |
| Booking | `POST` to `/v1/appointments` with the idempotency key, customer, and appointment arguments. A `reservationToken`, `holdToken`, `slotToken`, or `reservation_id` from the availability response is forwarded as `reservationToken` when present. |
| Confirmation | Success requires `appointmentId`, `bookingId`, or `confirmationId`. Without one the response is `APPOINTMENT_RESPONSE_INVALID` and the booking is not claimed. |

### Support ticket

`POST` to `/v1/tickets` with the idempotency key, customer, issue, category (default `general_support`), priority (default `normal`), session ID, and call ID. Success requires `ticketId` or `caseId`.

### Human escalation

`escalate_to_human` calls a dedicated escalation API (`/v1/escalations`) with the idempotency key, customer, reason (the issue text, or "Customer requested human support"), the latest message, the detected intent, session ID, and call ID. Success requires an `escalationId` (or `caseId` / `ticketId`) and returns `escalationRequired: true` and `escalationCreated: true`.

Many other failure responses set `escalationRequired: true` as a signal to the voice agent that human help is advisable. Only this path creates an escalation record; the flag alone does not.

## External Service Integrations

The workflow connects the voice platform to configurable customer-support services. All endpoints are configurable, and the workflow contains no vendor-specific hostnames. Base URLs are concatenated directly with the path, so they should not end in a slash. Service boundaries are defined by the endpoint patterns, request fields, and response fields documented here.

| Service | Environment variable | Endpoint pattern | Method | Purpose |
|---|---|---|---|---|
| Knowledge-base search | `SUPPORT_KB_BASE_URL` | `/v1/search` | `POST` | Retrieve reference content for FAQ answers. |
| LLM chat generation | `SUPPORT_LLM_BASE_URL`, `SUPPORT_LLM_MODEL` | `/chat/completions` | `POST` | Generate a grounded answer. |
| Order lookup | `SUPPORT_ORDER_BASE_URL` | `/v1/orders/{orderId}?verificationReference={reference}` | `GET` | Retrieve order status for verified requests. |
| Calendar availability | `SUPPORT_CALENDAR_BASE_URL` | `/v1/availability` | `POST` | Check requested appointment slot. |
| Appointment creation | `SUPPORT_CALENDAR_BASE_URL` | `/v1/appointments` | `POST` | Create the appointment. |
| Support-ticket creation | `SUPPORT_TICKET_BASE_URL` | `/v1/tickets` | `POST` | Create a support ticket. |
| Human escalation | `SUPPORT_ESCALATION_BASE_URL` | `/v1/escalations` | `POST` | Create a human-support escalation. |

**Authentication.** The knowledge-base, order, calendar, ticket, and escalation nodes use an n8n Header Auth credential. The LLM node uses an n8n credential of type `openAiApi`, with the request sent to the configured base URL.

**HTTP error handling.** Every HTTP node has a 10-second timeout and returns the full response with `neverError` enabled, so HTTP error statuses are classified by the workflow rather than raised by n8n. Upstream failures are mapped to the response contract as rate limit (429), auth (502), rejected request (502), server error (503), network error (502), or timeout (504).

**Response contracts.** The workflow expects the following fields from each service:

| Service | Fields the workflow reads |
|---|---|
| Knowledge base | `results` or `documents`; `sourceVerified` (response) or `verified` (item) |
| Order | Order status, and optionally delivery estimate, shipping state, and tracking number |
| Calendar availability | `available`; optional `alternatives`, `alternativeSlots`, or `slots`; optional `reservationToken`, `holdToken`, `slotToken`, or `reservation_id` |
| Appointment creation | `appointmentId`, `bookingId`, or `confirmationId`; slot conflict indication |
| Ticketing | `ticketId` or `caseId` |
| Escalation | `escalationId`, `caseId`, or `ticketId` |

**Scope.** This repository is the orchestration layer. The voice platform, knowledge-base implementation, LLM service, order backend, calendar backend, ticketing backend, and human-support backend are separate services reached through the endpoint contracts above.

## Persistence

PostgreSQL provides the persistence and consistency layer: idempotency claims, leases, provisional side-effect evidence, stored response results, and metadata-only request logging. The database schema is provisioned as part of deployment infrastructure, since the workflow JSON contains no `CREATE TABLE` statements. A PostgreSQL credential must be configured in n8n.

| Table | Used for | Columns referenced by the workflow |
|---|---|---|
| `voice_agent_idempotency` | Idempotency claims, leases, provisional evidence, and stored results. | `idempotency_key`, `status`, `response_json` (JSONB), `created_at`, `updated_at` |
| `voice_agent_request_log` | Metadata-only request log. | `request_id`, `session_id`, `call_id`, `tool_call_id`, `provider`, `channel`, `intent`, `action`, `success`, `error_code`, `escalation_required`, `escalation_created`, `event_timestamp`, `message_length` |

`idempotency_key` must be backed by a unique constraint or primary key, because the claim statement uses `ON CONFLICT (idempotency_key)`. This uniqueness is what makes the claim atomic.

<details>
<summary><strong>Internal keys stored in <code>response_json</code></strong></summary>

<br>

| Key | Purpose |
|---|---|
| `_idempotencyVersion` | Record format version (6). |
| `_ownerToken` | Execution that currently owns the row. |
| `_requestFingerprint` | Fingerprint of the original request. |
| `_intent`, `_sideEffecting` | Intent and side-effect flag used for fencing. |
| `_leaseSeconds` | Lease length recorded at claim time. |
| `_retryReclaimedAt`, `_reclaimedAt`, `_previousOwnerToken` | Reclaim bookkeeping. |
| `_provisionalResponse`, `_provisionalRecordedAt`, `_provisionalOutcomeState`, `_provisionalOwnerToken`, `_provisionalRequestFingerprint`, `_provisionalIntent`, `_provisionalSideEffecting` | Durable provisional evidence for side-effecting results. |
| `_persistenceMode`, `response` | Finalization mode and the stored response for replay. |

</details>

## Configuration

Configuration connects the orchestration layer to its operating environment. It is read from n8n variables (`$vars`) and n8n credentials; values are never stored in the workflow JSON. Variables are required only for the customer actions that use them.

| Variable | Purpose | Required for |
|---|---|---|
| `SUPPORT_KB_BASE_URL` | Base URL of the knowledge-base search service. | `faq` |
| `SUPPORT_LLM_BASE_URL` | Base URL of the chat-completions service. | `faq` |
| `SUPPORT_LLM_MODEL` | Model name sent in the generation request. | `faq` |
| `SUPPORT_ORDER_BASE_URL` | Base URL of the order service. | `get_order_status` |
| `VOICE_VERIFICATION_HMAC_SECRET` | Shared secret for verification signatures. | `get_order_status` |
| `SUPPORT_CALENDAR_BASE_URL` | Base URL of the calendar service. | `book_appointment` |
| `SUPPORT_TICKET_BASE_URL` | Base URL of the ticketing service. | `create_support_ticket` |
| `SUPPORT_ESCALATION_BASE_URL` | Base URL of the escalation service. | `escalate_to_human` |
| `VERIFICATION_MAX_AGE_SECONDS` | Maximum age of a verification timestamp. Optional; defaults to 300. | `get_order_status` |
| `SUPPORT_IDEMPOTENCY_LEASE_SECONDS` | Idempotency lease length. Optional; defaults to 300, minimum 30. | All intents |

Before any work is done, the workflow checks that the variables required by the requested intent are defined. If one is empty, the request is rejected with HTTP 503 `INTEGRATION_CONFIGURATION_ERROR` (not retryable).

**n8n credentials:** Header Auth for the webhook, Header Auth for the backend APIs, a credential for the LLM node (type `openAiApi`), and a PostgreSQL credential.

## Security Considerations

- Never commit secrets, API keys, passwords, real tokens, or production credentials to this repository.
- Configure secrets through n8n variables and n8n credentials.
- Review any exported workflow JSON before publishing it. Exports can contain credential references, identifiers, and instance-specific values.
- Use a verification secret that is shared only between the voice platform's signing component and this workflow.
- Keep customer message content out of persistent storage beyond the controls described in this document.

## Installation and Deployment

1. **Import the workflow.** In n8n, import `AI_Customer_Support_Voice_Agent_Workflow.json`. It imports inactive.
2. **Provision PostgreSQL.** Create `voice_agent_idempotency` and `voice_agent_request_log` with at least the columns listed under [Persistence](#persistence), including a unique constraint on `idempotency_key`.
3. **Create and link n8n credentials.** Create the credentials listed under [Configuration](#configuration) and re-link them on every node that references one. Exported credential references are specific to the originating instance.
4. **Set n8n variables.** Define the variables for the intents you intend to enable. The workflow reads them through `$vars`, so confirm that Variables are available in your n8n edition.
5. **Connect the external services.** Make each HTTP service available at the endpoint patterns listed under [External Service Integrations](#external-service-integrations), returning the response fields described in this document.
6. **Configure verification.** Share `VOICE_VERIFICATION_HMAC_SECRET` with the component that signs order-verification contexts, and match the signed payload described under [Protected Order-Status Verification](#protected-order-status-verification).
7. **Validate every intent.** Use the n8n test webhook and the scenarios under [Testing and Validation](#testing-and-validation).
8. **Activate.** Point the voice platform at the production webhook URL once the scenarios have been validated in your environment.

## Testing and Validation

The project is complete and has been tested; the tests executed returned positive results. The scenario matrix below is preserved as a repeatable validation reference for future deployments and environment changes. It covers supported actions, validation and security controls, idempotency, and side-effect safety.

Illustrative request shape (placeholder values):

```json
{
  "provider": "example-provider",
  "sessionId": "session-001",
  "callId": "call-001",
  "toolCallId": "tool-001",
  "toolName": "faq",
  "message": "What are your support hours?"
}
```

```bash
curl -X POST "https://<your-n8n-host>/webhook-test/voice-agent/support" \
  -H "Content-Type: application/json" \
  -H "<webhook-auth-header-name>: <webhook-auth-header-value>" \
  -d @request.json
```

The test webhook URL is shown on the Webhook node in n8n.

### Supported actions

| Scenario | Expected behavior |
|---|---|
| `faq` with verified knowledge | HTTP 200, `success: true`, `data.source: verified_knowledge`. |
| `faq` with no verified knowledge or an unavailable knowledge base | Unsuccessful response flagging human support (`NO_VERIFIED_KNOWLEDGE` or `KNOWLEDGE_BASE_UNAVAILABLE`). |
| `get_order_status` with a valid signed context | HTTP 200 with status, and optionally delivery estimate, shipping state, and tracking number. |
| `book_appointment` with an available slot | HTTP 200 with a confirmation ID. |
| `book_appointment` with an unavailable slot | HTTP 409 `APPOINTMENT_UNAVAILABLE` with up to three alternatives when supplied. |
| `book_appointment` with a slot conflict at booking time | HTTP 409 `APPOINTMENT_SLOT_TAKEN`. |
| `create_support_ticket` | HTTP 200 with a ticket ID. |
| `escalate_to_human` | HTTP 200 with `escalationCreated: true` and an escalation ID. |
| Unsupported tool name | HTTP 400 `UNSUPPORTED_TOOL`. |

### Validation and security

| Scenario | Expected behavior |
|---|---|
| Missing required fields | HTTP 400 `INVALID_REQUEST` with the missing items in `data.missingFields`. |
| Missing configuration variable for the requested intent | HTTP 503 `INTEGRATION_CONFIGURATION_ERROR`. |
| Order request with no, unverified, or expired verification | HTTP 403 with the matching verification error code. |
| Order request with an invalid signature | HTTP 403 `INVALID_VERIFICATION_SIGNATURE`. |
| Body and header idempotency keys that differ | HTTP 400 `INVALID_REQUEST`. |
| AI answer that fails grounded validation | HTTP 502 with the matching `AI_RESPONSE_*` code, human support flagged. |

### Idempotency and side effects

| Scenario | Expected behavior |
|---|---|
| Repeat of a completed request with the same key | Stored response replayed with `idempotencyState: duplicate_done`. |
| Same key with a different request body | HTTP 409 `IDEMPOTENCY_KEY_REUSE`. |
| Repeat while the first request is running, or concurrent requests with the same key | One execution proceeds; the other receives HTTP 409 `REQUEST_IN_PROGRESS`. |
| Stale side-effecting claim | Reconciliation response; the external action is not re-run. |
| Timeout from a side-effecting service | `consistencyState: uncertain`, `*_OUTCOME_UNCERTAIN`, HTTP 504 or 503. |
| 2xx from a side-effecting service without an identifier | `*_RESPONSE_INVALID` with `consistencyState: uncertain`. |
| PostgreSQL unavailable at claim time | HTTP 503 `IDEMPOTENCY_BACKEND_FAILURE`. |
| PostgreSQL failure when persisting a side-effecting result | HTTP 503 with `*_RESULT_PERSISTENCE_UNCERTAIN`. |
| Evidence persistence after a side-effecting action | Provisional evidence is present in the idempotency row before finalization. |
| Idempotency row after a successful request | Inspect `voice_agent_idempotency` and confirm the expected status transition (`DONE`, or `RETRYABLE_ERROR` for transient failures). |
| Any failure path | Response is a structured JSON body following the [Response Contract](#response-contract). |

## Failure Handling

Every failure class is converted to the [Response Contract](#response-contract), so the voice agent always receives a structured JSON body.

| Failure class | Handling |
|---|---|
| **Invalid request** | Validation builder; HTTP 400 `INVALID_REQUEST`, listing up to five missing items. |
| **Missing required fields** | Part of the invalid-request path. |
| **Integration configuration failure** | HTTP 503 `INTEGRATION_CONFIGURATION_ERROR`, not retryable. |
| **Verification failure** | HTTP 403 with a specific verification code; HTTP 503 `VERIFICATION_BACKEND_UNAVAILABLE` if the HMAC step errors. |
| **Fingerprint failure** | HTTP 503 `REQUEST_FINGERPRINT_FAILURE`, retryable. |
| **Security processing failure** | HTTP 503 `SECURITY_PROCESSING_FAILURE`, retryable. |
| **Idempotency failure** | Duplicate, active, key-reuse, and backend-error responses as described under [Idempotency](#idempotency). |
| **Stale side-effect reconciliation** | Replay of confirmed evidence, or an explicit uncertain response. |
| **Upstream service failure** | Classified as rate limit (429), auth (502), rejected request (502), server error (503), network error (502), or timeout (504). |
| **Persistence failure** | Side-effecting intents are downgraded to an uncertain state; read-only intents are marked `persistence_unconfirmed`. |
| **Terminal processing failure** | `RESPONSE_PROCESSING_FAILURE` for read-only intents; `RESULT_PROCESSING_UNCERTAIN` for side-effecting intents. HTTP 503. |

<details>
<summary><strong>Error code reference</strong></summary>

<br>

| Group | Codes |
|---|---|
| Validation | `INVALID_REQUEST`, `INTEGRATION_CONFIGURATION_ERROR`, `UNSUPPORTED_TOOL` |
| Verification | `VERIFICATION_REQUIRED`, `VERIFICATION_REFERENCE_REQUIRED`, `VERIFICATION_SIGNATURE_REQUIRED`, `INVALID_VERIFICATION_SIGNATURE`, `VERIFICATION_EXPIRED`, `INVALID_VERIFICATION_TIMESTAMP`, `VERIFICATION_BACKEND_UNAVAILABLE` |
| Request identity | `REQUEST_FINGERPRINT_FAILURE`, `SECURITY_PROCESSING_FAILURE` |
| Idempotency | `IDEMPOTENCY_KEY_REUSE`, `REQUEST_IN_PROGRESS`, `STALE_REQUEST_NOT_RECLAIMED`, `IDEMPOTENCY_STORED_RETRYABLE_RESULT`, `IDEMPOTENCY_METADATA_MISSING`, `IDEMPOTENCY_RESULT_MISSING`, `IDEMPOTENCY_STATE_ERROR`, `IDEMPOTENCY_BACKEND_FAILURE` |
| Reconciliation | `STALE_SIDE_EFFECT_REQUIRES_RECONCILIATION`, `STALE_SIDE_EFFECT_OUTCOME_UNCERTAIN`, `STALE_SIDE_EFFECT_OWNER_CHANGED`, `STALE_RECONCILIATION_BACKEND_FAILURE` |
| Knowledge and AI | `KNOWLEDGE_BASE_UNAVAILABLE`, `NO_VERIFIED_KNOWLEDGE`, `AI_PROVIDER_FAILURE`, `AI_RESPONSE_INVALID`, `AI_RESPONSE_EMPTY`, `AI_RESPONSE_UNSAFE_CONTENT`, `AI_RESPONSE_UNSUPPORTED_FACT`, `AI_RESPONSE_UNGROUNDED_CLAIM`, `AI_RESPONSE_UNGROUNDED`, `AI_RESPONSE_NO_VERIFIED_SOURCE` |
| Orders | `ORDER_NOT_FOUND`, `ORDER_LOOKUP_FAILED`, `ORDER_RESPONSE_INVALID` |
| Appointments | `APPOINTMENT_UNAVAILABLE`, `APPOINTMENT_SLOT_TAKEN`, `APPOINTMENT_AVAILABILITY_FAILED`, `APPOINTMENT_AVAILABILITY_TIMEOUT`, `APPOINTMENT_AVAILABILITY_REJECTED`, `APPOINTMENT_AVAILABILITY_RESPONSE_INVALID`, `APPOINTMENT_PROVIDER_FAILURE`, `APPOINTMENT_RESPONSE_INVALID`, `APPOINTMENT_OUTCOME_UNCERTAIN` |
| Tickets and escalation | `TICKET_CREATION_FAILED`, `TICKET_RESPONSE_INVALID`, `TICKET_OUTCOME_UNCERTAIN`, `ESCALATION_CREATION_FAILED`, `ESCALATION_RESPONSE_INVALID`, `ESCALATION_OUTCOME_UNCERTAIN` |
| Upstream | `UPSTREAM_RATE_LIMITED`, `UPSTREAM_AUTH_FAILURE`, `UPSTREAM_REQUEST_REJECTED`, `UPSTREAM_SERVER_FAILURE`, `UPSTREAM_NETWORK_FAILURE`, `SIDE_EFFECT_OUTCOME_UNCERTAIN` |
| Persistence | `SIDE_EFFECT_EVIDENCE_PERSISTENCE_UNCERTAIN`, `SIDE_EFFECT_EVIDENCE_OWNER_FENCE_LOST`, `APPOINTMENT_RESULT_PERSISTENCE_UNCERTAIN`, `TICKET_RESULT_PERSISTENCE_UNCERTAIN`, `ESCALATION_RESULT_PERSISTENCE_UNCERTAIN`, `SIDE_EFFECT_RESULT_PERSISTENCE_UNCERTAIN`, `RESULT_PERSISTENCE_UNCERTAIN`, `RESULT_PERSISTENCE_UNCONFIRMED`, `RETRYABLE_RESULT_PERSISTENCE_UNCONFIRMED` |
| Terminal | `RESPONSE_PROCESSING_FAILURE`, `RESULT_PROCESSING_UNCERTAIN` |

</details>

## Operational Logging

The workflow uses a metadata-focused operational logging design. `Log Conversation` writes one row per request to `voice_agent_request_log` before the response is returned. The node is configured to continue on error, so the logging step is best-effort and the webhook response is always delivered, even if a log write fails.

| Logged field | Notes |
|---|---|
| Request ID, session ID, call ID, tool-call ID | Identifiers from the request. |
| Provider, channel | Lower-cased provider name; channel defaults to `voice`. |
| Intent, action | Canonical intent and the action taken. |
| Success state, error code | Outcome of the request. |
| Escalation required, escalation created | Escalation flags from the response. |
| Event timestamp | From the request, or generated when absent. |
| Message length | Length of the response message, not its content. |

Conversation content is deliberately excluded. The log does not store the customer's message, the response text, customer identity fields, or order data, which keeps the operational record free of personal conversation content while still supporting request correlation by identifier, outcome analysis by intent and error code, and escalation tracking.

Operational visibility is provided by this PostgreSQL request log, together with the structured response fields (`errorCode`, `consistencyState`, `idempotencyState`, `verificationState`, `upstreamStatusCode`, `httpStatus`) and n8n's own execution records.

## Response Contract

Every execution path returns a JSON object with `schemaVersion: "1.0"`. Fields are always present, but their values and the contents of `data` vary by path.

| Field | Description |
|---|---|
| `schemaVersion` | Response format version. |
| `success` | `true` only for a confirmed successful action. |
| `requestId`, `sessionId`, `callId`, `toolCallId` | Request identifiers, `null` when unavailable. |
| `provider` | Lower-cased provider name, or `unknown`. |
| `intent` | Canonical intent, or `unknown`. |
| `action` | Action or stage that produced the response, for example `create_appointment` or `idempotency_replay`. |
| `message` | Text intended for the voice agent, sanitized and capped at 1,200 characters. |
| `data` | Path-specific payload such as alternatives, order details, provider references, or a reconciliation key. |
| `escalationRequired` | Whether human support is advisable. |
| `escalationCreated` | Whether an escalation record was created. |
| `escalationId`, `ticketId` | Provider references, `null` when not applicable. |
| `retryable` | Whether the caller may retry. |
| `errorCode` | Machine-readable code, `null` on success. |
| `timestamp` | Request timestamp. |
| `verificationState` | Verification state, or `not_required`. |
| `idempotencyState` | Idempotency state reported for the request. |
| `consistencyState` | `confirmed`, `retryable`, `uncertain`, or `persistence_unconfirmed`. |
| `upstreamStatusCode` | Upstream HTTP status when one was received, otherwise `null`. |
| `httpStatus` | The HTTP status used for the webhook response. |

## Project Scope

> [!IMPORTANT]
> This repository is the **n8n orchestration layer** for the AI Customer Support & Voice Agent Automation system.

It contains the 40-node workflow that connects an external voice platform to the support services described above. The voice platform, knowledge base, LLM service, order backend, calendar backend, ticketing backend, human-support backend, and PostgreSQL database are separate components reached through the endpoint, credential, and schema contracts documented in this README. This separation lets each deployment connect the workflow to its own services and infrastructure.

## Repository Structure

```text
AI-Customer-Support-Voice-Agent-Automation/
    README.md
    AI_Customer_Support_Voice_Agent_Workflow.json
    screenshots/
        workflow-overview.png
```
