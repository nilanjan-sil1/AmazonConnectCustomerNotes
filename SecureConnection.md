# Approach to Secure Connection from Amazon Connect Customer to Pindrop

The key architectural constraint is that Lambda never talks to Pindrop directly over the open internet — everything funnels through a single, tightly governed "secure gateway" hop (AWS PrivateLink to Pindrop's VPC endpoint service where available, or an allow-listed egress path via a shared network account). Here's how that maps onto AWS infrastructure:

<img width="1440" height="1120" alt="image" src="https://github.com/user-attachments/assets/efa4fd7a-1bc0-47e5-b6e8-c8dfbae9d852" />

A few things worth calling out on the security path, since that's the part unique to a protected account:

**Two ways to reach Pindrop without opening general internet access:**
1. **AWS PrivateLink** — if Pindrop exposes a VPC Endpoint Service (they do for enterprise/regulated deployments), you create an interface endpoint in the egress account and traffic never leaves the AWS backbone. This is the cleanest option — no NAT, no public IPs, no DNS exfiltration risk.
2. **Allow-listed egress fallback** — if PrivateLink isn't available for your Pindrop contract tier or region, route through a shared network/egress account via Transit Gateway, terminate at AWS Network Firewall with a strict domain allow-list (only Pindrop's specific API hostnames), then NAT Gateway out. All other outbound destinations are denied by default.

**Two distinct data paths, not one:**
- The **control path** (Lambda → Passport/Protect scoring requests/responses) is small, synchronous JSON over HTTPS.
- The **audio path** (KVS → Protect) is continuous media and is usually the harder integration — it needs a small bridging service (Lambda for short calls, or a long-running Fargate task for full-duration streaming) to consume KVS fragments and forward them, since Lambda has execution time limits.

**Governance:** put the VPC endpoint / firewall rule / PrivateLink connection in the shared egress account so a central security team owns and audits the one narrow path out — the workload account itself never gets broad internet egress.

One thing I couldn't verify from general knowledge: whether Pindrop currently offers a PrivateLink endpoint service in your specific AWS region/contract — that's worth confirming directly with your Pindrop account team, since it determines whether you need option 1 or option 2 above.

## What does the Lambda integration function do?

That confirms the exact mechanics — and it explains a key design constraint on this Lambda: it has to work within Connect's timeout model, not just call an API.

**Its core job: translate between Connect's contact flow and Pindrop's APIs**
The Lambda is the only thing in the flow that speaks HTTPS to the outside world. Everything Connect knows how to do is limited to its own block vocabulary — it can't natively call a third-party REST API, parse JSON responses, or handle OAuth token refresh. The Lambda exists purely to bridge that gap.

**Specific responsibilities:**

1. **Receive the invocation payload** — Connect automatically passes the full flow run data (ContactId, customer endpoint/ANI, any contact attributes set so far) to the function, plus whatever specific parameters you configure in `LambdaInvocationAttributes`.
2. **Retrieve Pindrop credentials** from Secrets Manager (the credential-security piece from earlier).
3. **Call Pindrop's API** — for the auth path, sending the ANI/phoneprint data to Passport; separately, kicking off or checking the status of the Protect scoring pipeline that's consuming the KVS audio.
4. **Parse and flatten the response** — Connect requires the Lambda's return value to be either a flat `STRING_MAP` (key/value pairs of strings) or nested `JSON`, set statically per block. Since Pindrop's raw API response likely has a nested structure, the Lambda's job is to shape that into whatever `ResponseValidation` type you configured, exposed to the flow under the `$.External` namespace.
5. **Respect Connect's tight timeout window** — this is the constraint that shapes the whole design: synchronous invocation caps out at **8 seconds max**, asynchronous at 60 seconds. Since Protect's real-time scoring is a streaming, ongoing process rather than an instant response, you can't just call it synchronously and wait — this is exactly why the earlier design used a **pending/loop/timeout pattern**: the Lambda kicks off the request and returns quickly, and the flow polls a `Check contact attributes` loop against a status attribute rather than blocking on one long call.
6. **Handle retries and errors gracefully** — Connect already retries throttled or 500-error Lambda invocations up to three times automatically, but the Lambda itself still needs application-level error handling for Pindrop-side failures (timeout, invalid response, service unavailable) so it returns a clean "auth unavailable" signal rather than crashing the block and dumping the caller onto a bare Error branch.
7. **Write results back as attributes** — since any external attribute is overwritten the moment you invoke a different Lambda function, this function's output must be immediately captured into user-defined attributes via a **Set contact attributes** block if anything downstream (screen pop, another Lambda call, risk branching) needs to reference it later.

In short: it's not just an API proxy — it's an **adapter that reshapes Pindrop's async, potentially slow scoring process into the bounded, synchronous/asynchronous vocabulary Connect's flow engine understands**, while also owning credential retrieval and response-format compliance.

## How does Amazon Connect stream audio into KVS?

Good — this fills in the actual mechanics precisely, and a few of these details matter directly for your Pindrop integration.

**One KVS stream per active call, provisioned automatically**
When live media streaming is enabled and a Start media streaming block runs, Connect creates a **dedicated Kinesis Video Stream for that specific call**, provisioning new streams as needed to handle concurrent calls (subject to your account's KVS service quota — Support can raise this if you're doing high call volume). You never pre-create streams; Connect owns their lifecycle.

**Format specifics your Fargate bridge needs to know:**
- Audio is sent at **8 kHz sampling rate**, 16-bit, mono, signed PCM — this is telephony-grade audio, not high-fidelity, which matters for whether Pindrop's models expect that exact input format or need resampling.
- Streams use **KVS multi-track support**, keeping what the customer says and what the customer hears on **separate tracks** (`AUDIO_FROM_CUSTOMER` and `AUDIO_TO_CUSTOMER`) — this is convenient for Protect, since fraud scoring generally only needs the customer-side audio, not the agent/prompt audio mixed in.

**How your bridge actually reads it**
There's no push notification — your Fargate consumer calls the KVS `GetMedia` (or `GetMediaForFragmentList`) API against the `StreamARN` your Lambda captured, starting from the `StartFragmentNumber` contact attribute so it picks up exactly where streaming began rather than missing or duplicating audio. The **Kinesis Video Streams parser library** handles the low-level frame/fragment parsing so you don't hand-roll that.

**Security, since this is the audio path in your protected account:**
- Encrypted in transit via **TLS**, and at rest via **KMS** — encrypted before it's written to KVS storage, decrypted only on retrieval — consistent with the rest of your architecture using customer-managed keys.
- **Data retention is configurable per instance**, and importantly — if retention is set to 0, nothing is stored and your bridge must consume it live or lose it; you need retention > 0 if you want any replay capability (e.g. for debugging a Protect scoring dispute), balanced against how long you actually want raw caller audio sitting around given data-minimization requirements.

**One easy mistake to flag:** don't modify or manually manage the KVS streams Connect creates — AWS explicitly recommends against it, since Connect owns their lifecycle and unexpected behavior can result. Your bridge should be a pure **consumer** of the stream, never touching stream configuration itself.

## What does the Fargate audio bridge do for Pindrop Protect?

This piece is custom infrastructure you build — Pindrop doesn't provide it, and AWS doesn't provide it out of the box. It's the glue between "Connect has audio sitting in KVS" and "Protect needs a continuous audio feed to score." Here's what it actually has to do:

**1. Discover which stream to read**
It receives the `StreamARN` and `StartFragmentNumber` — either invoked directly by the contact-flow Lambda with those values passed in, or by polling/subscribing to a lightweight event (e.g. an SNS notification your Lambda publishes) so it doesn't need to be invoked synchronously from the flow itself.

**2. Continuously consume KVS**
Using the KVS parser library, it calls `GetMedia` against that stream ARN, reading fragments from the `AUDIO_FROM_CUSTOMER` track specifically (Protect generally only needs the caller's voice, not agent/system prompts) as they arrive — this has to run for the **duration of the call**, which is exactly why it's Fargate and not Lambda: Lambda's max execution time (15 minutes) and cold-start/connection-reuse behavior make it a poor fit for a long-lived, continuously-open streaming consumer, whereas a Fargate task can hold that connection open indefinitely.

**3. Reframe/repackage the audio for Pindrop's ingestion format**
KVS delivers PCM audio in Matroska-wrapped fragments via the parser library's frame objects — Pindrop's real-time ingestion API almost certainly expects a specific transport (commonly a WebSocket or gRPC stream of raw PCM/Opus chunks, format depending on their integration spec). The bridge's job is to strip out KVS's fragment/frame structure and re-chunk the raw audio into whatever wire format and cadence Pindrop's streaming endpoint expects — this is genuine integration work, since the two systems weren't designed to speak to each other directly.

**4. Forward audio through the secure egress path**
Rather than opening its own arbitrary outbound connection, the bridge sends the repackaged audio stream out through the same PrivateLink endpoint or firewall-allow-listed path used for the control-plane API calls — so audio traffic gets the identical governance and monitoring as everything else leaving the account.

**5. Relay Protect's scoring responses back**
Protect returns risk signals **as the call progresses**, not just once at the end — the bridge needs to receive these (likely a streaming response channel or periodic callback) and push updates back into the system your Lambda/contact flow reads from — typically by writing to a fast-access store (DynamoDB, or invoking an update via EventBridge) keyed on `ContactId`, since the contact flow's `Check contact attributes` polling loop needs somewhere current to read the `RiskScore` from.

**6. Handle the call's end cleanly**
When the `Stop media streaming` block runs or the call disconnects, the bridge needs to detect that the stream has closed, flush/finalize anything buffered, close its connection to Pindrop gracefully, and terminate the Fargate task rather than leaving it running and consuming resources indefinitely.

**Why this component tends to be the hardest part of the whole integration in practice:** everything else in this architecture is mostly configuration — flow blocks, IAM policies, firewall rules. This bridge is the one piece of genuine custom software, and it's stateful, long-running, and sits directly in the real-time path of a live phone call, so latency and reliability here directly affect call quality and how fast a risk score is available to the agent.

## How are Pindrop API credentials secured?

Credential handling here follows a fairly standard AWS pattern, with a couple of specifics that matter for Pindrop:

**Storage**
- Pindrop API keys/client secrets live in **AWS Secrets Manager**, not in Lambda environment variables, contact flow attributes, or code. Environment variables are visible to anyone with Lambda console/read access and don't get automatic rotation — Secrets Manager does.
- The secret is encrypted at rest with a **customer-managed KMS key** (not the default AWS-managed key), scoped so only the specific Lambda execution role(s) that call Pindrop can decrypt it. This gives you a CloudTrail record of every decrypt call.

**Retrieval**
- Lambda fetches the secret at cold start (or via a short-lived cache) using the Secrets Manager SDK call, authenticated by its IAM execution role — no long-lived credentials baked into the deployment package or CI/CD pipeline.
- The Lambda reaches Secrets Manager over a **VPC interface endpoint**, so that call also never leaves the AWS network, consistent with the rest of the architecture.

**Rotation**
- Secrets Manager's automatic rotation feature can call a rotation Lambda on a schedule (e.g. every 30–90 days) that requests a new key/secret from Pindrop's admin API (if they support programmatic rotation) or, at minimum, alerts your team to rotate manually and updates the stored value.
- Rotation should be tested against Pindrop's actual auth model — some vendors use static API keys, others use OAuth client-credentials flows with short-lived tokens. If it's OAuth, Lambda caches the access token (with its own short TTL) rather than re-authenticating on every call, and the long-lived client secret is what sits in Secrets Manager.

**Least privilege**
- IAM policy on the Lambda role should name the exact secret ARN — no wildcard `secretsmanager:GetSecretValue` on `*`.
- Separate secrets (and ideally separate IAM roles) for the Passport/Protect API credential vs. any Pulse export credential, so a compromise of one doesn't expose the other.

**Audit trail**
- CloudTrail logs every `GetSecretValue` call, and CloudWatch alarms can flag unusual access patterns (e.g. a spike in calls, or access from an unexpected principal) — useful given this credential is effectively a bridge to an external vendor.

If you want, I can sketch this credential-retrieval flow as its own small diagram, or draft the IAM policy JSON for the Lambda role.

## What gets stored in S3 for Pindrop Pulse?

I need to correct something significant from earlier in this conversation before answering, since it changes what the honest answer here actually is.

**Correction:** I've been describing Pindrop Pulse as a post-call analytics/dashboard product throughout this thread — that's wrong. Based on Pindrop's own current positioning, **Pulse is audio deepfake and liveness detection** — a probabilistic, largely real-time scoring capability that returns a liveness score (0–100) and liveness reason codes, similar in nature to Protect's fraud scoring rather than a batch analytics tool. It's sold as an add-on alongside Passport and/or Protect, not a standalone reporting layer.

That means the "Pulse gets a batch S3 export at end of call" pattern I described earlier isn't grounded in anything I can confirm — I was pattern-matching to a generic vendor-analytics assumption rather than Pindrop's actual product.

**What I can honestly say gets stored in S3 in this architecture:**
- **Call recordings** (from Amazon Connect's own recording feature) — the standard compliance artefact, encrypted with your KMS key, used for QA, dispute resolution, and regulatory retention requirements.
- **Contact Trace Records (CTRs)** — Connect's structured metadata about each contact (timestamps, queue, agent, disposition, and any contact attributes you chose to persist, including your `AuthStatus`, `RiskScore`, and now presumably a `LivenessScore` if you're using Pulse).
- Possibly a **consolidated audit log** you build yourself — joining the CTR with the Passport/Protect/Pulse scores at the time of the call, useful for after-the-fact fraud investigation or model-performance review, but this would be your own construction, not something Pindrop directly consumes from S3.

**What I don't have confirmed detail on:** whether Pindrop offers any product (a dashboard, a case-management tool, an analytics workspace) that ingests historical data from a customer's S3 bucket at all, and if so, under which product name and what format it expects. Given Pulse is apparently real-time scoring rather than an analytics layer, that batch-ingestion piece may not exist the way I assumed, or may live under a different part of Pindrop's platform (their case-management features were mentioned in older material, but I can't confirm current details).

I'd recommend not building against my earlier "Pulse = Pindrop's analytics/dashboard product consuming S3 exports" description — instead, confirm directly with your Pindrop account team what liveness/deepfake detection (Pulse) actually expects as input (likely: another consumer of the same real-time audio stream your Fargate bridge already handles, alongside Protect) and whether any separate reporting/dashboard product exists and what it needs from you.

## What VPC interface endpoints are needed for this setup?

Good — this surfaces one detail that materially affects your timeline: **the KVS endpoint isn't self-service**.

**The full list you need in the protected account's VPC:**

| Endpoint | Service name pattern | Why |
|---|---|---|
| Secrets Manager | `com.amazonaws.<region>.secretsmanager` | Lambda and the Fargate bridge both retrieve Pindrop credentials |
| KMS | `com.amazonaws.<region>.kms` | Decrypting those secrets, and decrypting/encrypting S3 recordings, KVS audio |
| Kinesis Video Streams | `com.amazonaws.<region>.kinesisvideo` | Fargate bridge's `GetMedia` calls to read customer audio |
| STS | `com.amazonaws.<region>.sts` | Any cross-account role assumption — e.g. if the Fargate task or Lambda assumes a role in the shared egress account to reach the PrivateLink endpoint there |
| ECR API + ECR DKR | `com.amazonaws.<region>.ecr.api` / `.ecr.dkr` | Fargate needs these to pull its container image if the subnet has no NAT/internet route |
| CloudWatch Logs | `com.amazonaws.<region>.logs` | Fargate task logging, Lambda logging — without this, a fully private subnet can't ship logs anywhere |
| EventBridge (if used) | `com.amazonaws.<region>.events` | Only if your architecture uses EventBridge for the async "score is ready" callback pattern discussed earlier |

**Two that are gateway endpoints, not interface endpoints** (different mechanism, no ENI, no hourly cost, attached to route tables instead):
- **S3** — for call recordings, CTR export, and Fargate's image-layer pulls from ECR (which itself stores layers in S3)
- **DynamoDB** — only if you're using it as the fast-access store for the Lambda-polling pattern (writing risk scores where the contact flow's `Check contact attributes` loop can read them)

**The catch: Kinesis Video Streams' VPC endpoint requires AWS to allow-list your account first** — it's not something you can just create on demand like Secrets Manager or KMS. AWS's own documentation states it can take up to a week for them to add you to the service before you receive the actual endpoint service name to connect to. This is worth kicking off early in your project timeline, since it's a hard dependency for the entire audio-streaming path — nothing else in this architecture can proceed on the KVS side until that allow-listing completes.

**What's explicitly not on this list:** anything for Amazon Connect itself. Connect is invoked by the telephony network and orchestrates blocks server-side — your VPC resources don't need an interface endpoint to "reach" Connect; Connect reaches into your VPC (Lambda) or your VPC reaches out to AWS services (Secrets Manager, KMS, KVS), not the other way around.

One planning note: request the KVS allow-listing and file the service quota increase for concurrent KVS streams (mentioned earlier) in the same conversation with AWS Support, since both are prerequisites that sit outside your direct control and are easy to discover too late if you're building the rest of the infrastructure in parallel.

## What is a Transit Gateway and why is it used here?

A **Transit Gateway (TGW)** is AWS's managed network hub — think of it as a virtual router that sits between multiple VPCs (and on-premises networks, via VPN or Direct Connect) and lets them talk to each other without you having to build and maintain a mesh of individual peering connections. Instead of every VPC connecting directly to every other VPC, each one just connects to the Transit Gateway once, and the TGW routes traffic between them according to route tables you define.

**Why it shows up in this specific architecture:**

Without a Transit Gateway, your two accounts — the protected Connect workload account and the shared egress account — would need a direct **VPC peering connection** to reach each other. That works for a single pair of accounts, but it doesn't scale and doesn't match how most enterprises actually structure their networks:

1. **Centralizing egress is the whole point of the design.** The security goal here is "only one narrow, audited path leaves any workload account." If you have more than just this one Connect account — other application VPCs, other teams' workloads — you don't want each one building its own separate peering connection to the egress account and its own copy of the firewall rules. A Transit Gateway lets **every** spoke VPC in the organization attach once and share the same centrally-managed egress path, including the same Network Firewall domain allow-list and the same PrivateLink connection to Pindrop.

2. **It enforces routing discipline via route tables**, not implicit trust. You can attach the Connect VPC to the TGW but only give it a route table that permits traffic destined for the egress account's specific CIDR range — it can't accidentally (or maliciously) reach other spoke VPCs attached to the same TGW, because TGW route tables are per-attachment and don't have to be symmetric or fully meshed.

3. **It's the standard AWS enterprise pattern for exactly this "protected account, no general internet access" scenario** — often called hub-and-spoke: a central network/egress account (the hub) holds the NAT Gateway, Network Firewall, and any PrivateLink endpoints; every workload account (a spoke) attaches to the TGW and routes all outbound traffic through the hub. It's the same mechanism whether you're consolidating egress for compliance, centralizing DNS resolution, or connecting to on-prem data centers via Direct Connect from one place instead of every account separately.

**Would you strictly need it for just this one Connect account?** No — for a single account talking to a single egress account, VPC peering alone would technically work and is simpler to set up. Transit Gateway earns its complexity when there's more than one spoke, or when the enterprise already has a hub-and-spoke network as their standard pattern (which is common in exactly the kind of "protected account with no default internet access" environment your original question described) — in that case, plugging Connect into the existing TGW is far less work than building a one-off peering exception for it. Given you mentioned this is a broader enterprise with an established protected-account posture, it's a safe bet the TGW hub already exists and Connect is just becoming one more spoke attached to it.

## How does AWS Network Firewall allow-list only Pindrop domains?

This is a **stateful domain list rule group** — one of Network Firewall's simplest but most useful rule types, since it doesn't require you to write raw Suricata syntax.

**How the matching actually works**
For HTTPS traffic (which is what your Lambda and Fargate bridge use to reach Pindrop), the firewall inspects the **Server Name Indication (SNI) field in the TLS handshake** — the hostname the client is trying to reach, sent in plaintext before encryption kicks in, even though the rest of the connection is encrypted. It doesn't decrypt your traffic to do this; SNI inspection happens at the handshake stage, before payload encryption is relevant. For plain HTTP, it inspects the `Host` header directly instead. Domain matching supports both explicit names (`api.pindrop.com`) and wildcards (`.pindrop.com`, which also covers all subdomains).

**Building the allow-list rule**
You create a stateful rule group of type `Domain list`, set `TargetTypes` to `["TLS_SNI", "HTTP_HOST"]` so both protocols are covered, list Pindrop's specific API hostnames under `Targets`, and set `GeneratedRulesType` to `ALLOWLIST`. Practically:

```json
{
  "RulesSource": {
    "RulesSourceList": {
      "Targets": [".pindrop.com"],
      "TargetTypes": ["TLS_SNI", "HTTP_HOST"],
      "GeneratedRulesType": "ALLOWLIST"
    }
  }
}
```

With `ALLOWLIST` mode, Network Firewall sends traffic only to the specified domains and drops everything else — you don't need a separate explicit deny-all rule; the allow-list behavior itself implies the default drop.

**A subtlety worth getting right: HOME_NET**
By default, domain list inspection only covers traffic originating from the VPC where the firewall is actually deployed. In this architecture, the firewall lives in the **shared egress account's VPC**, but the traffic you're inspecting originates from the **Connect workload account's VPC**, arriving via Transit Gateway. That means you need to explicitly set the `HOME_NET` rule variable to include the Connect VPC's CIDR range in addition to the egress VPC's own range — otherwise traffic transiting from the spoke account won't match your rules the way you expect, since it's arriving from an IP range outside the firewall's default inspection scope.

**Two operational notes:**
- **Rule ordering matters if you mix rule types.** AWS's guidance warns against combining `Reject`/`Alert` domain rule groups with `Allow` domain rule groups in a policy using default action ordering — the implicit drop from an allow-list rule group takes effect before reject/alert rules would fire, which can produce confusing results if you're layering additional inspection on top of the basic allow-list.
- **You still need the stateless layer configured to forward traffic to the stateful engine** — the stateless rules are a fast pre-filter; if they don't route matching traffic into the stateful engine, your domain rule group never gets evaluated at all. This is a common setup mistake per AWS's own troubleshooting guidance.

**Getting the exact hostnames right is the actual hard part** — this whole mechanism only works if you know precisely which domains Pindrop's Passport, Protect, and (if used) Pulse endpoints resolve to, including any regional or environment-specific subdomains (sandbox vs. production). That list needs to come from your Pindrop integration contact, not guessed, since an incomplete allow-list means legitimate API calls get silently dropped rather than erroring in an obvious way.

## When should I use AWS PrivateLink vs NAT gateway egress to reach Pindrop?

The decision mostly comes down to whether Pindrop offers a VPC Endpoint Service for the specific products you're using (Passport/Protect scoring APIs), plus how much your organization values eliminating firewall/DNS-based controls as an attack surface. Here's the tradeoff:

**AWS PrivateLink — use this if Pindrop offers it for your contract tier/region**

*Advantages:*
- Traffic never touches the public internet, even in transit — it moves entirely over the AWS backbone via an ENI in your VPC, connected directly to Pindrop's endpoint service in their AWS account.
- No NAT Gateway, no public IPs, no DNS resolution to a public hostname at all — this eliminates a whole category of attack surface (DNS hijacking, BGP route leaks, man-in-the-middle on a NAT'd path) that firewall allow-listing can't fully close.
- Simpler security review: "this VPC endpoint connects only to this one specific service" is a much cleaner story for an auditor than "this NAT Gateway can reach anything the firewall doesn't explicitly block."
- No need to maintain a domain allow-list at all for this traffic — nothing to keep in sync if Pindrop adds/changes hostnames.

*Limitations:*
- Entirely dependent on Pindrop actually offering this for the products and account tier you're on — not all vendors expose PrivateLink for every service, and it's common for it to be reserved for enterprise/regulated-industry contracts.
- You're one endpoint per product/region — if Passport, Protect, and Pulse are on different endpoint services (or different regions), you may need multiple PrivateLink connections rather than one.
- Some vendors' PrivateLink offerings only cover *some* of their API surface (e.g. the real-time scoring path but not an admin/management API), so you may still need a secondary egress path for anything outside that scope.

**NAT Gateway + Network Firewall allow-list — use this as the fallback, or when you need broader reach**

*Advantages:*
- Works regardless of what Pindrop supports — it's just outbound HTTPS to specific hostnames, so it's available immediately without waiting on a vendor capability.
- More flexible if you need to reach multiple Pindrop endpoints across different services, environments (sandbox vs. production), or if Pindrop's architecture changes hostnames over time — you're just updating a domain list, not renegotiating a private connection.
- Necessary anyway if there's *any* other legitimate external destination your workload account needs (unlikely to be zero, even in a locked-down environment) — you'll likely need this egress path to exist regardless, so the marginal cost of also routing Pindrop traffic through it can be near zero.

*Limitations:*
- Relies on SNI/Host-header inspection as the security boundary — a real control, but a shallower one than "this connection cannot possibly reach anywhere except this named endpoint," since it's evaluated per-connection rather than architecturally impossible to route elsewhere.
- Ongoing maintenance burden: every time Pindrop changes or adds a hostname, someone has to update the firewall rule group, or calls start failing silently.
- NAT Gateway and Network Firewall both carry hourly and data-processing charges that scale with traffic volume — for a high-call-volume contact center, continuous audio-adjacent metadata and scoring traffic can add up, whereas PrivateLink's pricing model (per-endpoint-hour plus data processed) is often more predictable for this kind of steady, well-defined traffic pattern. I'd size this against your actual expected volume rather than assume either is cheaper — get current pricing for both from AWS's pricing pages rather than relying on my estimate here, since these rates do change.

**My practical recommendation for your situation:** given you described a genuinely protected account with essentially no external access by default, PrivateLink is the better architectural fit *if* Pindrop supports it — it lets you avoid standing up a NAT Gateway and firewall allow-list at all for this specific integration, keeping the account's "no internet access" posture true in the strongest sense rather than "no internet access, except this one curated exception." I'd treat NAT + firewall as the fallback you build only if Pindrop confirms PrivateLink isn't available for the products/region you need — which is exactly the confirmation I flagged as an open question back when we first discussed this architecture.
