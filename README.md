# The Action Gateway

**Architecture reference: separating model proposals from credentialed execution.**

> Status: design document and JSON schema only. This repository does not contain a runnable gateway, credential store, approval service, or verified deployment. Do not treat it as an executable security control. The safeguards below are design requirements.

Every company doing real work with LLMs hits the same wall. The model is useful when it can read your data. It's valuable when it can *act* — update the CRM, send the follow-up, create the opportunity, dispatch the job. But nobody sane wants to hand an LLM a live API key to their system of record, and the security review that follows will stop the project cold.

Most teams resolve this one of two ways. They stay read-only and never capture the value. Or they quietly put credentials in an environment variable and hope.

There's a third option. It's structural, it's boring, and it works.

---

## The idea in one paragraph

Models don't get credentials. Models get to *ask*. They submit a structured action proposal — what they want to do, to which record, why, and how confident they are — to a gateway service. The gateway is the only thing in the system that holds credentials to downstream services. It validates every proposal against an allowlist, decides whether a human needs to approve it, routes those approvals to a person's phone, executes what's permitted, and writes every single decision to an append-only audit log.

The model is a proposer. The gateway is the authority. That separation is the whole thing.

---

## Why this is different from "just use tool calling"

Tool calling puts the credential inside the agent's runtime. Whatever the model can call, it can call — and prompt injection, a confused-deputy chain, or an over-eager retry loop all execute with your full authority. The blast radius is everything the key can touch.

Here the credential never enters the model's context or its runtime. A compromised or manipulated model can only emit a proposal, and a proposal is just JSON that gets validated by code the model never sees. A correctly implemented gateway can reject unauthorized proposals, but allowed harmful actions, compromised credentials, implementation bugs and authorization mistakes remain risks.

It also gives you three things tool calling alone doesn't:

- **Revocability per model.** Each model gets its own token. If one misbehaves, you kill that token without touching the others.
- **An audit trail that survives the conversation.** Every proposal — approved, rejected, or executed — is a durable row. The chat transcript is not your audit log.
- **A human in the loop, selectively.** Not on everything, which nobody will tolerate. On the specific action classes you decide are consequential.

---

## The action proposal contract

```json
{
  "action": "create_opportunity",
  "target": {
    "system": "crm",
    "location_id": "loc_abc123",
    "record_id": "contact_789"
  },
  "payload": {
    "name": "Quarterly service renewal",
    "value": 4800,
    "pipeline_stage": "quoted"
  },
  "reason": "Customer confirmed renewal intent on the call transcript from 2026-09-14.",
  "confidence": 0.86,
  "requires_approval": false,
  "proposed_by": "model-a",
  "idempotency_key": "conv_4412_opp_create_01"
}
```

Field notes, because the details are where this earns its keep:

| Field | Why it exists |
|---|---|
| `action` | Must match the allowlist exactly. Anything unrecognised is rejected, never interpreted. |
| `target` | Explicit. The model never sends a raw URL or a query — it names a system and a record. |
| `reason` | Free text, required. It goes in the audit log. When something goes wrong in three months, this is the field that tells you what the model thought it was doing. |
| `confidence` | The model's own estimate. The gateway uses it as *one* input to the approval decision, never as the sole gate — a confidently wrong model is the normal failure mode, not the exotic one. |
| `requires_approval` | The model's suggestion. **The gateway overrides it.** A model that sets this to `false` on a destructive action still gets held. Never let the proposer decide whether it needs supervision. |
| `idempotency_key` | Retries are guaranteed. Duplicate sends are guaranteed. This makes them harmless. |

---

## Action classes

Sort every action into one of four buckets and the approval policy writes itself.

**Read** — fetch a contact, list open jobs, pull a record. No approval. Log it anyway; you want the access trail.

**Safe write** — add a note, apply a tag, update a custom field. Reversible, low consequence. No approval, full logging.

**Outbound** — send an SMS, send an email, create an opportunity. These touch a customer and cannot be unsent. Approval required above a value threshold, or for any first contact.

**Destructive** — delete, merge, bulk-update, change a payment detail. Always approval. No exceptions, no confidence score high enough.

The list is deliberately short. If an action doesn't fit cleanly into one of these, it doesn't belong on the allowlist yet.

---

## Approval routing

An approval that takes a day is an approval nobody uses, and a system nobody uses gets bypassed. Route them to where the operator already is — phone, chat, whatever they check without thinking. The message needs exactly four things:

1. What the model wants to do, in one line of plain language
2. Which record it affects
3. The model's stated reason
4. Approve / Reject

Anything beyond that and people stop reading. Set a timeout — an approval nobody answers should expire as a rejection and log as one, not sit pending forever.

---

## The audit log

Append-only. One row per proposal, whatever the outcome.

```
id, timestamp, proposed_by, action, target_system, target_record,
payload_hash, reason, confidence, decision, decided_by,
executed_at, result_status, error
```

Two decisions that matter more than they look:

**Keep it out of your primary application database.** Separate store, separate credentials, ideally append-only at the storage layer. An audit log your application can rewrite is not an audit log.

**Hash the payload rather than storing it raw.** Payloads contain customer data, and your audit log will have a longer retention period and a wider read audience than you're planning for. Store the hash, store the shape, and keep the contents in the system that's already governed for it.

---

## Reference flow

```
  Model A ─┐
  Model B ─┼──▶  [ Gateway ]  ──▶  validate against allowlist
  Model C ─┘         │                     │
   (per-model        │                     ├─▶ rejected ──▶ audit log
    tokens, no       │                     │
    credentials)     │                     ├─▶ needs approval ──▶ operator's phone
                     │                     │          │
                     │                     │          ├─ approve ─┐
                     │                     │          └─ reject ──┼─▶ audit log
                     │                     │                      │
                     │                     └─▶ auto-approved ─────┤
                     │                                            ▼
                     │                                    [ credential store ]
                     │                                            │
                     └────────── audit log ◀──── execute ────────▶ CRM / billing /
                                                                   dispatch / etc.
```

---

## Implementation notes

A possible implementation uses self-hosted n8n with a separate audit store. Nothing here is n8n-specific — it's a Lambda, a small service, an API route. What matters is the properties, not the runtime:

- The gateway is the **only** holder of downstream credentials
- The allowlist is **code**, not configuration a model can influence
- `requires_approval` is decided by the **gateway**, never honoured from the proposal
- Every proposal is logged, including the rejections — especially the rejections
- Each proposer gets its **own** revocable token

---

## What this doesn't solve

Being straight about the limits, because a pattern oversold is a pattern nobody trusts:

- It doesn't stop a model proposing something dumb-but-allowed. It bounds the damage and makes it visible; it doesn't make the model smarter.
- Approval fatigue is real. Put too much in the approval bucket and people start approving without reading, which is worse than automation because now it's laundered through a human.
- The gateway becomes a single point of failure and a high-value target. It deserves the security posture you'd give a credential vault, because that's what it is.
- It adds latency. For anything interactive, the proposal round-trip is noticeable.

---

## Motivation

This design explores bounded AI actions in business workflows. The public artifact is the contract and design; implementing and verifying every property remains separate work.

*Cady Lalanne — CLD Technology. [LinkedIn](https://www.linkedin.com/in/cadylalanne/)*

*MIT licensed.*
