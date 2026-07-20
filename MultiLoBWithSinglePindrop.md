# Architecture where you have multiple Line of Business (LoB) integrations with a single Pindrop instance

This is a genuinely different architectural problem from what we've discussed so far — you're not integrating one Connect instance with Pindrop, you're integrating *three*, and the constraint is that the Pindrop-specific logic can't be duplicated (and drift) three times across LOBs. The right pattern is to extract a **central, channel-agnostic Pindrop orchestration service** that every LOB's Connect flow calls through a thin, in-account adapter — the same hub-and-spoke thinking we used for network egress, now applied one layer up to the integration logic itself.Notice each LOB's local Lambda is deliberately "dumb" — it only knows how to talk to Connect and forward a normalized request onward. All the Pindrop-specific knowledge (API shapes, credentials, response parsing, scoring thresholds) lives in exactly one place. Here's how that maps to account topology:A few design decisions that make this actually work in practice, rather than just look clean on paper:

**Why the local Lambda has to exist at all, not just call the hub directly**
Amazon Connect's "Invoke AWS Lambda function" block can only target functions in the same AWS account — there's no native cross-account Lambda selection in the contact flow designer. So each LOB needs at least a thin local Lambda as the Connect-facing surface; its entire job is to receive the Connect invocation, normalize it into a shared request shape, and forward it to the hub. It should contain essentially zero Pindrop-specific logic — no scoring thresholds, no response parsing — so that when Pindrop changes their API or you add a fourth LOB, nothing in this local function needs to change.

**How the cross-account call actually gets made securely**
Two realistic options, and given the security posture you established earlier, the PrivateLink-style approach is the better fit:
- **Private cross-account API Gateway**, fronting the hub's orchestrator, shared into each LOB's VPC via a VPC endpoint service (using AWS RAM or endpoint service allow-listing) — traffic stays on the AWS backbone, same principle as the Pindrop PrivateLink connection, just one层 earlier in the chain.
- **Direct cross-account Lambda invoke** via a resource-based policy on the hub's orchestrator function, with each LOB's trigger Lambda execution role granted `lambda:InvokeFunction` — simpler to stand up, but less architecturally clean than the API Gateway/PrivateLink option and harder to apply consistent request validation across LOBs.

**The "channel-agnostic" requirement drives the contract, not the transport**
This is the part worth being precise about: channel-agnostic doesn't mean the *network path* is generic — it means the orchestrator's **input/output contract** doesn't assume Amazon Connect specifics. It should accept something like `{ channel: "voice" | "chat" | ..., lobId, sessionId, callerIdentifier, audioStreamRef? }` and return `{ authStatus, riskScore, livenessScore, reasonCodes }` — a shape that would work identically if LOB B later moved off Connect entirely, or if a future channel (chat-based authentication, say) needed to reach the same Pindrop relationship without any voice component at all.

**Why the audio bridge needs cross-account IAM, not just cross-account networking**
Each LOB's KVS stream is a resource that lives *in that LOB's own account* — Connect creates it there, not in the hub. So the shared Fargate bridge in the hub account can't just reach it over the network; it needs an **IAM role in each LOB account** (three roles, one per LOB) with a trust policy allowing the hub's Fargate task role to assume it and call `kinesisvideo:GetMedia` scoped to that LOB's streams only — this is the standard cross-account resource-access pattern, and it's worth keeping those three roles' permissions strictly scoped so a compromise of the hub can't read audio from a LOB it wasn't currently servicing.

**One open decision I'd flag rather than assume:** whether all three LOBs share a single Pindrop contract/credential, or whether Pindrop issues separate client credentials per LOB (common if each LOB has a distinct fraud-consortium profile, billing relationship, or regulatory boundary — plausible for three genuinely separate lines of business). If it's the latter, the hub's Secrets Manager needs three scoped secrets rather than one, and the orchestrator selects the right credential based on the `lobId` in the incoming request — a small change to the contract, but worth deciding with your Pindrop account team before building the shared secret store as if there's only one relationship to manage.

## More elaborate box diagram

Good — let's pin these down concretely rather than leaving them as generic boxes.

**The two main components, and what each actually is:**

1. **The orchestrator** — the logic/control-plane layer. Not one service, but a small serverless stack: a **private API Gateway** as the entry point the three LOBs call into, backed by **AWS Step Functions** as the actual orchestrator, which invokes one or more **Lambda functions** to do the real work (call Passport, trigger the audio bridge, poll for Protect/Pulse completion, normalize the response, write results to DynamoDB).

2. **The cross-account audio bridge** — the media/streaming layer. This is an **Amazon ECS task running on AWS Fargate**: a long-running container that holds an open connection to a LOB's KVS stream and to Pindrop's real-time ingestion endpoint for the duration of a call.

**Is the orchestrator a "microservice"? Not in the classic always-on sense — and that's deliberate.**

You could build it as a persistent microservice on ECS/Fargate behind an internal load balancer, but **Step Functions + Lambda is the more idiomatic AWS choice here**, for reasons specific to this workload:
- The work is a **multi-step, mostly I/O-bound sequence** (call Passport → trigger audio scoring → poll for completion → aggregate → write result) — exactly the shape Step Functions is designed to coordinate, with built-in retry policies, timeout handling, and error branching per step rather than you hand-rolling that logic.
- Traffic across three LOB contact centers is **spiky, not constant** — serverless (pay-per-invocation) avoids paying for idle capacity between calls, unlike a standing microservice that's provisioned for peak load at all times.
- Step Functions gives you a **visual execution history per request** — genuinely valuable here since this whole hub exists to make fraud-relevant decisions; being able to pull up exactly which steps ran, in what order, with what inputs/outputs for any given `sessionId` is a real audit-trail asset your fraud/compliance team will eventually want.
- No servers or containers to patch, scale, or right-size for the orchestration layer itself — Fargate is reserved for the one component that genuinely needs a persistent connection (the audio bridge), not applied everywhere out of habit.

**Why the audio bridge specifically can't be serverless the same way:** as covered earlier, it needs to hold an open, continuous connection to KVS and to Pindrop for the full duration of a live call — that's fundamentally at odds with Lambda's execution-time ceiling, so Fargate (no server management, but a long-running container) is the right fit, while everything *around* it stays serverless.

## What does LOB A trigger Lambda actually send to the hub?

This is where the "channel-agnostic contract" from the diagram becomes concrete. The trigger Lambda's job is entirely translation: it takes whatever Connect handed it and reshapes it into the hub's generic schema — nothing Connect-specific should leak through to the orchestrator.

**What Connect gives the trigger Lambda (the raw input)**
When the contact flow invokes it, Connect automatically passes the full flow run data, from which the Lambda pulls:
- `Details.ContactData.ContactId`
- `Details.ContactData.CustomerEndpoint.Address` (the ANI)
- `Details.ContactData.Attributes.MediaStreams.Customer.Audio.StreamARN` and `StartFragmentNumber` (from the earlier `Start media streaming` block)
- Any LOB-specific contact attributes already set earlier in the flow (account number, IVR menu path, etc., if the LOB's risk policy needs that context)

**What the trigger Lambda sends onward to the hub (the normalized payload)**
Something like:

```json
{
  "lobId": "lob-a",
  "channel": "voice",
  "sessionId": "<ContactId>",
  "callerIdentifier": "<CustomerEndpoint.Address>",
  "audioRef": {
    "provider": "kvs",
    "streamArn": "<StreamARN>",
    "startFragmentNumber": "<StartFragmentNumber>",
    "sourceAccountId": "<LOB A's account ID>"
  },
  "requestedChecks": ["auth", "risk", "liveness"],
  "callbackHint": {
    "resultStore": "dynamodb",
    "resultKey": "<ContactId>"
  }
}
```

A few deliberate choices in that shape, and why:

- **`sessionId` rather than `ContactId`** — the hub's contract shouldn't have a field literally named after a Connect concept, since a future chat-channel adapter wouldn't have a `ContactId` at all, just whatever session identifier its own platform uses. The trigger Lambda maps Connect's `ContactId` into the generic field; the orchestrator never needs to know it came from Connect specifically.
- **`sourceAccountId` inside `audioRef`** — this is the piece the hub's cross-account audio bridge actually needs to know *which* of the three per-LOB IAM roles to assume before calling `GetMedia`. Without this, the bridge has no way to know whether a given stream ARN belongs to LOB A, B, or C.
- **`requestedChecks`** — lets a LOB opt out of something (e.g. a LOB that hasn't licensed Pulse yet shouldn't have the orchestrator attempt a liveness check and fail) without the orchestrator needing separate code paths per LOB; it's just data-driven.
- **`callbackHint`** — since Protect/Pulse scoring is asynchronous relative to the trigger Lambda's own execution window (same 8-second synchronous / 60-second asynchronous ceiling we discussed earlier applies here too, now with an extra network hop added to the latency budget), the trigger Lambda needs to tell the orchestrator *where* to eventually write results the LOB's own polling loop can read — reusing the DynamoDB-backed pattern from the single-account design, just parameterized per LOB/contact rather than hardcoded.

**What deliberately does *not* go in this payload:** anything Pindrop-API-shaped — no Pindrop endpoint names, no auth-score thresholds, no field names from Pindrop's actual response schema. If any of that leaked into the trigger Lambda or the contract, you'd be back to duplicating Pindrop-specific knowledge across three LOBs, which defeats the entire reason for centralizing this in the first place. The trigger Lambda should be able to be generated from a template shared across all three LOBs with only `lobId` and account-specific IDs changing — if it needs LOB-specific business logic beyond that, that's a signal the boundary is in the wrong place.

## What does the request and response contract for the Pindrop orchestrator look like?

Building directly on the trigger Lambda's payload from the last answer — here's the full contract, covering both directions.

**Request contract (LOB → Hub orchestrator)**

```json
{
  "contractVersion": "1.0",
  "lobId": "lob-a",
  "channel": "voice",
  "sessionId": "<opaque session identifier>",
  "callerIdentifier": "<ANI, or channel-appropriate identifier>",
  "audioRef": {
    "provider": "kvs",
    "streamArn": "...",
    "startFragmentNumber": "...",
    "sourceAccountId": "..."
  },
  "requestedChecks": ["auth", "risk", "liveness"],
  "context": {
    "accountNumber": "...",
    "priorAuthHint": "returning_customer"
  },
  "callback": {
    "mode": "async_poll",
    "resultStore": "dynamodb",
    "resultKey": "<sessionId>"
  }
}
```

- **`contractVersion`** — non-negotiable from day one with three consumers. If the orchestrator ever needs a breaking change (a renamed field, a new required check), it can support v1 and v2 side by side while LOBs migrate on their own schedule, rather than a synchronized three-way deployment.
- **`audioRef` is optional** — a chat-channel adapter down the line would omit it entirely and only request `["risk"]` or whatever checks make sense without voice biometrics, which is exactly the point of keeping `requestedChecks` data-driven rather than assuming voice.
- **`context`** is a deliberately loose bag for LOB-specific signals that might influence a Pindrop policy decision (e.g. account tier) without polluting the top-level schema with fields only one LOB uses.
- **`callback.mode`** — supporting both `async_poll` (LOB polls its own DynamoDB table, matching the pattern used inside a single Connect flow) and potentially `webhook` later, without forcing every LOB onto the same integration style.

**Response contract (Hub orchestrator → LOB)**

Given the latency realities (Pindrop's real API round-trip, plus now a cross-account hop), the orchestrator almost certainly can't respond synchronously with final scores in most cases — so the *immediate* response and the *eventual* result are two different shapes.

Immediate response, returned right away:
```json
{
  "contractVersion": "1.0",
  "sessionId": "...",
  "accepted": true,
  "status": "processing",
  "pollAfterMs": 500
}
```

Eventual result, written to the shared result store (DynamoDB) under `resultKey`, which the LOB's own Connect flow polls for via its `Check contact attributes` loop:
```json
{
  "contractVersion": "1.0",
  "sessionId": "...",
  "status": "complete",
  "results": {
    "auth": {
      "outcome": "matched",
      "confidence": 87
    },
    "risk": {
      "score": 22,
      "band": "low"
    },
    "liveness": {
      "score": 95,
      "reasonCode": "no_anomalies_detected"
    }
  },
  "errors": [],
  "pindropRequestId": "..."
}
```

A few deliberate choices here worth calling out:

- **`status` as an explicit field, not inferred from presence/absence of data** — distinguishing `processing`, `complete`, `partial` (e.g. auth came back but liveness timed out), and `error` lets each LOB's flow branch cleanly rather than guessing from which fields happen to be populated. This maps directly onto the pending/loop/timeout pattern we designed for the single-LOB contact flow — the LOB's `Check contact attributes` loop is really polling this `status` field.
- **`band` alongside the raw numeric `score`** — screen pops and simple branching logic usually just need low/medium/high, and baking that translation into the orchestrator (using a single, centrally-governed threshold policy) is exactly why you centralized this in the first place. If each LOB translated raw scores into bands independently in their own trigger Lambda, you'd risk LOB A and LOB B applying inconsistent risk thresholds without anyone noticing — a real governance gap for something feeding fraud decisions.
- **`pindropRequestId`** — carried through for auditability, so if there's ever a dispute or an investigation, you can trace a specific LOB's contact record back to the exact Pindrop-side request/response, without exposing any Pindrop-specific field names to the LOB layer itself.
- **`errors` as a structured array, not a thrown exception** — since this crosses an account boundary and an async boundary, failures need to be data the LOB's flow can branch on ("liveness check failed, proceed with risk-only decision") rather than something that just breaks the polling loop.

**One governance question this contract design forces you to answer up front:** who owns changes to `requestedChecks` values, `band` thresholds, and `reasonCode` vocabularies — the central platform team running the hub, or can individual LOBs request customization? If LOBs have genuinely different risk appetites (a retail LOB tolerating more risk than a wealth-management LOB, say), you either need per-LOB threshold configuration *within* the orchestrator (keeping the contract identical but behavior tunable via `lobId`), or you accept that the contract technically allows drift and govern it via process instead. Baking `lobId`-keyed configuration into the orchestrator is the cleaner answer — it keeps the contract truly uniform while still letting each LOB's fraud team set their own bar.

## How does the cross-account audio bridge assume roles into each LOB account?

This is standard AWS cross-account access, but it's worth being precise about the moving pieces since there are three separate trust relationships to get right, one per LOB.

**The core mechanism: STS AssumeRole, one role per LOB account**

Each LOB account creates a dedicated IAM role — call it `PindropAudioBridgeRole` — that exists **in that LOB's account**, not the hub's. Its trust policy names the hub account's Fargate task role as the only principal allowed to assume it:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::<HUB_ACCOUNT_ID>:role/AudioBridgeFargateTaskRole"
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": { "sts:ExternalId": "lob-a-audio-bridge" }
    }
  }]
}
```

The role's **permissions policy** (also defined in the LOB account) is scoped tightly — only `kinesisvideo:GetMedia` and `kinesisvideo:GetDataEndpoint`, restricted to that LOB's own stream ARNs, ideally via a resource pattern rather than `*`:

```json
{
  "Effect": "Allow",
  "Action": ["kinesisvideo:GetMedia", "kinesisvideo:GetDataEndpoint"],
  "Resource": "arn:aws:kinesisvideo:eu-west-1:<LOB_A_ACCOUNT_ID>:stream/*"
}
```

**On the hub side**, the Fargate task role needs a permissions policy allowing it to assume *any* of the three LOB roles:
```json
{
  "Effect": "Allow",
  "Action": "sts:AssumeRole",
  "Resource": [
    "arn:aws:iam::<LOB_A_ACCOUNT_ID>:role/PindropAudioBridgeRole",
    "arn:aws:iam::<LOB_B_ACCOUNT_ID>:role/PindropAudioBridgeRole",
    "arn:aws:iam::<LOB_C_ACCOUNT_ID>:role/PindropAudioBridgeRole"
  ]
}
```

**The `ExternalId` condition is worth taking seriously, not treating as boilerplate** — it protects against the "confused deputy" problem: since the hub is a shared service acting on behalf of three different tenants, without an external ID, a bug or compromise causing the bridge to assume the wrong role for the wrong request would go undetected by IAM itself. Using a distinct external ID per LOB (which the orchestrator looks up based on the `sourceAccountId` field from the request contract we designed earlier) means the AssumeRole call itself fails fast if the bridge ever tries to use LOB A's external ID against LOB C's role.

**How this plays out at runtime, per the sequence diagram from earlier:**
1. Orchestrator receives the request, sees `audioRef.sourceAccountId` = LOB A's account ID.
2. Orchestrator tells the audio bridge which LOB it's servicing for this session.
3. Bridge calls `sts:AssumeRole` against LOB A's specific role ARN, with LOB A's external ID.
4. STS returns temporary credentials (typically 15-minute to 1-hour expiry — short-lived, not long-lived keys).
5. Bridge uses those temporary credentials to call `GetMedia` against the stream ARN, which only works because the assumed role's permissions are scoped to that account's streams.
6. When the call ends and the bridge is done, those credentials simply expire — nothing to revoke manually.

**A practical consequence for how you build the Fargate task itself:** since a single shared bridge might be servicing calls from multiple LOBs concurrently (three contact centers running simultaneously), the bridge can't hold one static set of assumed credentials — it needs to assume the right role **per active call/session**, cache each assumed-role credential set keyed by session, and let them expire naturally rather than trying to maintain one long-lived cross-account identity. This is a meaningfully different implementation shape than the single-LOB version of this bridge we discussed earlier, where there was only ever one account's stream to worry about.

**One thing worth deciding explicitly rather than defaulting into:** whether the LOB-side role and its trust policy are provisioned centrally (the platform team pushes a CloudFormation StackSet or Terraform module to each LOB account) or whether each LOB team creates and maintains their own copy by following a documented spec. Centralized provisioning keeps the three roles consistent and auditable from one place; self-service risks exactly the kind of drift the whole hub design was meant to avoid — a LOB tweaking their role's permissions slightly differently than the others, undetected, until it causes an access failure or (worse) an over-permissioned role nobody notices.

## Should the three LOBs share one Pindrop credential or have separate ones?

This is more of a business/contractual question than a purely technical one, so I'll lay out the considerations rather than assert a single right answer — but I do lean toward separate credentials, and here's why.

**The case for separate credentials per LOB (my recommendation)**

1. **Pindrop's fraud consortium and scoring almost certainly benefit from tenant isolation.** Protect's risk scoring cross-references a fraud consortium and builds behavioral baselines over time. If three genuinely separate lines of business share one Pindrop account/credential, their call volume and caller populations get commingled from Pindrop's perspective — a retail LOB's high call volume could dilute or skew risk modeling that a smaller wealth-management LOB depends on, and vice versa. Separate credentials typically map to separate Pindrop-side tenant configurations, keeping each LOB's model tuned to its own population.

2. **Blast radius on compromise.** If one credential is shared across all three, a leak or misuse affecting one LOB's traffic pattern (say, a bug in LOB C's trigger Lambda flooding Pindrop with malformed requests) can throttle or degrade service for LOB A and B too, since Pindrop's rate limits and abuse detection are almost certainly applied per-credential. Separate credentials contain that blast radius to the LOB that caused it.

3. **Billing and usage attribution.** Three separate lines of business likely have three separate budgets and cost centers. A shared credential makes it much harder to attribute Pindrop usage costs back to the LOB actually generating the call volume — you'd need to reconstruct that from your own logs rather than getting it natively from Pindrop's billing.

4. **Independent policy tuning.** Different LOBs plausibly want different risk thresholds, different enrollment policies, or different feature sets (one LOB might license Pulse, another might not, as I flagged in the earlier response contract). If Pindrop's policy engine is configured per-account/credential, separate credentials let each LOB's fraud team tune their own bar without needing the orchestrator to fake that separation in application logic.

5. **Regulatory and contractual boundaries.** If these are genuinely separate lines of business (possibly with different regulators, different data-handling obligations, or even different legal entities under one parent), a shared vendor credential can complicate audit and compliance stories — "which LOB's data flowed through this credential on this date" becomes a log-reconstruction exercise rather than something inherent to the access pattern itself.

**The case for one shared credential**

- **Simpler initial procurement and lower Pindrop contract complexity** — one relationship, one contract negotiation, one invoice, rather than three parallel vendor relationships to manage.
- **Marginally simpler secrets management** — one secret in Secrets Manager instead of three, though this is a small operational difference given you're already building a hub that's designed to route per-`lobId` anyway.
- Might make sense if the three LOBs are more like **product lines within one unified customer-facing brand** rather than truly separate businesses — e.g. if callers might plausibly interact with more than one LOB and a *unified* fraud/risk picture across LOBs is actually the goal, not a bug.

**What actually determines the right answer:** whether cross-LOB fraud visibility is a feature or a liability for your organization. If a fraudster who got flagged calling LOB A's contact center might plausibly target LOB B next, and you *want* Pindrop's consortium/history to catch that pattern across LOBs, a shared credential (or at least shared consortium visibility, which some vendors support even across separate credentials via account grouping) is actually valuable — you'd be deliberately trading tenant isolation for cross-LOB fraud correlation. If the three LOBs are functionally unrelated businesses that happen to share an AWS Organization, isolation is almost certainly the better default.

**My concrete recommendation:** default to **separate Pindrop credentials per LOB**, provisioned as three distinct secrets in the hub's Secrets Manager (`pindrop/lob-a`, `pindrop/lob-b`, `pindrop/lob-c`), with the orchestrator selecting the right one based on `lobId` from the request contract — this was actually already anticipated in the contract design a couple of turns back. Then raise the cross-LOB fraud-correlation question explicitly with your Pindrop account team — if it turns out to be valuable, ask whether Pindrop supports **linking separate credentialed accounts under one consortium umbrella**, which would give you both isolation *and* correlation, rather than forcing a single binary choice between them.