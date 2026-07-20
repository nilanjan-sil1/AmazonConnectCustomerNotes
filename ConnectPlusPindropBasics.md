# Basics about Amazon Connect Contact Flows integrated with Pindrop Voice Biometrics and Protect Fraud Detection

Here's how three Pindrop products, namely Pindrop Passport, Pindrop Protect and Pindrop Pulse, typically slot into an Amazon Connect deployment, and how to reach Pindrop's cloud securely from a locked-down account.

**The three products play different roles in the call:**
- **Passport** does voice/phone-based authentication (caller enrollment + verification) — called synchronously early in the flow.
- **Protect** does real-time fraud/risk scoring off the live audio stream — runs continuously in parallel with the call.
- **Pulse** is post-call analytics/reporting — consumes call metadata and recordings asynchronously after the interaction ends.

<img width="1440" height="756" alt="image" src="https://github.com/user-attachments/assets/f1d18e26-af51-4150-83a5-26c5cd376708" />

## Tell me more about the Amazon Connect contact flow

The contact flow is where all the branching logic lives — it's the traffic controller that decides, in real time, what happens to the call based on what Passport and Protect report back. Walking through what each block actually does:

<img width="1440" height="1160" alt="image" src="https://github.com/user-attachments/assets/425f2019-f2ef-4bea-b5a2-8f98b692af11" />

**Call arrives** — the "Set logging behavior" and "Set voice" blocks typically run first, then the flow moves into media streaming setup. This is also where you'd set a whisper flow or initial DTMF/voice prompt if you want the caller to state a purpose before authentication starts.

**Start streaming, invoke Lambda** — two things happen close together: a **"Start media streaming"** block begins pushing audio to KVS, and an **"Invoke AWS Lambda function"** block calls your integration function, passing `$.ContactId`, `$.CustomerEndpoint.Address` (the ANI), and any DTMF-entered ID. The Lambda kicks off the Passport request and returns immediately with a "pending" or cached status — it doesn't block the caller waiting on Pindrop's response if that would add noticeable latency.

**Authentication check** — a **"Check contact attributes"** block branches on what Passport returned:
- *Pass* → skip straight to self-service or agent routing, no PIN/KBA needed.
- *Fail or no match* → route to a **step-up flow** (traditional PIN, OTP via SMS, or knowledge-based questions).
- *Pending* → a short **"Loop"** block with a wait, re-checking the attribute a few times before falling back to step-up, so the flow doesn't hang indefinitely on a slow API response.

**Risk scoring and branch** — similarly checks a `RiskScore` contact attribute set asynchronously by the Protect pipeline (via the Lambda callback from the audio bridge). Above a threshold, the flow can set a `FraudFlag` attribute, route to a specialized fraud-review queue, or force additional verification even if Passport already passed — since Protect can catch things like device/channel spoofing that voice-match alone wouldn't.

**Transfer to queue** — a standard **"Transfer to queue"** block, but the queue selection itself can be dynamic (a **"Check contact attributes"** branch choosing between a normal queue and a fraud/escalation queue based on the risk flag).

**Agent connects** — contact attributes (`AuthStatus`, `RiskScore`, `FraudFlag`) are visible to the agent's CCP, usually surfaced through a **screen pop** app (a Connect app integration or a Lambda-backed widget) so the agent sees "Caller verified, risk: low" or "Verification failed, risk: high — follow escalation procedure" without digging through logs.

One practical note: keep the Lambda invocations **non-blocking where possible** — Connect contact flows have execution time limits per block, so anything that might take Pindrop more than a second or two to respond should be designed with the pending/loop/timeout pattern shown above rather than a single synchronous call.

## What is the first block in a Connect contact flow for an inbound call?

Every Amazon Connect contact flow starts with an **Entry point** block — it's automatically placed on the canvas whenever you create a new flow and can't be deleted or moved. It has no configurable settings itself; it's just the fixed starting node that every other block chains from, and every connector path in the flow must eventually trace back to it before the flow can be published.

For a typical inbound flow, the practical sequence right after the entry point is: Entry point, Set logging behavior (if configured), Set voice (if configured), Play prompt, and then whatever your integration-specific logic requires — in the Pindrop case, that's where you'd add **Set contact recording behavior** and then **Start media streaming** before invoking your Lambda.

Two extra things worth knowing for your setup:
- Amazon Connect ships with a sample inbound flow that's automatically assigned to the phone number you claim when you first set up flows, using a Check contact attributes block to route based on channel (voice, chat, or task) — that's a reasonable starting template to fork rather than building from a blank canvas.
- If you record calls (which you'll want for Pulse's post-call analytics), the compliance-minded pattern is a **Play prompt** informing the caller of recording, immediately followed by a **Set recording behavior** block, both right after the entry point and before any Pindrop Passport/Protect logic runs — so consent is established before biometric processing begins.

## How does the contact flow start KVS media streaming and invoke Lambda together?

They're not actually invoked in parallel — it's a two-block sequence with a specific order, because the Lambda needs stream metadata that only exists after streaming has started successfully.

**Step 1: Start media streaming block**
You place a **"Start media streaming"** block in the flow and configure it to capture audio from the customer, to the customer, or both. When it runs, Amazon Connect begins pushing audio to a Kinesis Video Stream and the block returns two outcomes:
- **Success** branch — streaming started; contact attributes are now populated with stream details.
- **Error** branch — streaming failed to start (e.g. live media streaming isn't enabled on the instance) — worth routing this somewhere sensible rather than silently continuing without audio.

Critically, Amazon Connect populates a set of **media-streaming contact attributes** as soon as the block succeeds — the ones your Lambda needs:
- `$.MediaStreams.Customer.Audio.StreamARN` — the Kinesis Video Stream ARN
- `$.MediaStreams.Customer.Audio.StartFragmentNumber` — identifies exactly where in the stream the customer audio begins
- `$.MediaStreams.Customer.Audio.StartTimestamp`

**Step 2: Invoke AWS Lambda function block, wired to the Success branch**
You connect the Success output of the streaming block directly into an **"Invoke AWS Lambda function"** block. In that block you select **"Media streams"** as the attribute type so the stream ARN and fragment number get passed into the Lambda's input event alongside the usual `ContactId` and customer endpoint. That's how your Fargate audio-bridge service (or a triggering Lambda that kicks it off) knows which Kinesis stream to open and where to start reading from — the `StartFragmentNumber` in particular lets a downstream consumer using `FRAGMENT_NUMBER` as its start selector begin reading audio from the exact point streaming began, rather than guessing or backfilling.

A couple of practical notes AWS's own docs flag:
- Streaming continues until an explicit **"Stop media streaming"** block runs — even if the contact is transferred to a different flow — so make sure every branch of your flow eventually hits a stop block, or you'll leak open streams.
- If this block is triggered during a **chat** contact, it routes down the Error branch automatically, since live media streaming only applies to voice.

So the accurate mental model isn't "start these two things simultaneously" — it's "start streaming, wait for success, then hand the resulting stream coordinates to Lambda" — which then triggers your Fargate bridge to actually begin consuming KVS and forwarding audio to Pindrop Protect.

## What are the possible outcomes of the Pindrop Passport authentication check in the contact flow?

Based on what's publicly documented, Pindrop doesn't expose a single fixed "outcome enum" the way you might see in an OAuth spec — Passport is fundamentally a **scoring engine**, not a binary yes/no service, and how you translate that score into contact flow branches is something you configure via policy thresholds. Here's what's actually confirmed vs. what's a design decision on your side:

**What Pindrop's model actually returns**
Passport combines several passive signals — phoneprint (carrier/device/network characteristics), voiceprint match against an enrolled sample, and (per Pindrop's own materials) keypress cadence analysis — into an authentication score that gives an IVR or agent the data necessary to make a decision. Their own documentation on the underlying biometric approach notes the decision can be binary — accept or reject — or it can produce a similarity score, where surpassing a predefined threshold authenticates the user. So the "outcome" is really: **a score, plus a policy engine that maps score ranges to a decision** — a powerful risk-decision engine creates a score, which is automated and governed by a flexible policy engine to build trust for genuine callers.

**What that policy engine typically resolves to, for the contact flow:**
- **High-confidence match** — score clears your configured threshold → treat as authenticated, skip PIN/KBA.
- **No match / below threshold** → route to step-up verification (PIN, OTP, knowledge-based questions) rather than an outright hang-up, since a failed passive match doesn't necessarily mean fraud — it can mean insufficient enrollment data or a noisy line.
- **Voice mismatch alert** — Pindrop has a specific feature for this: if a mismatched voice attempts to authenticate against an enrolled account, and accounts with multiple enrolled users can trigger an alert — that's a distinct signal from a generic "fail," useful for flagging potential account takeover rather than just an unenrolled caller.
- **Inconclusive / insufficient audio** — short calls, heavy background noise, or an unenrolled number won't produce a usable score at all; your contact flow needs a branch for "no score returned" separate from "score below threshold," since those imply different next steps (enroll now vs. step up).
- **Liveness/deepfake flag** — this technically comes from **Pulse**, not Passport, but often gets evaluated alongside it: Pulse delivers liveness scores to determine if a caller is a real person, which matters increasingly as synthetic voice attacks target passive voice auth.

**What I can't confirm from public sources:** the exact field names, status codes, or JSON schema Passport's API returns (e.g. whether it's literally `"outcome": "PASS"` or a numeric score you threshold yourself) — that level of detail lives in Pindrop's partner/developer documentation, which isn't public. I'd get that schema directly from your Pindrop integration contact before finalizing the Lambda's response-parsing logic, since designing your `Check contact attributes` branches around an assumed field name that turns out to be wrong is an easy way to end up with every call falling through to a default branch.

## How does the contact flow use the Pindrop Protect risk score to branch routing?

Good, this confirms the exact mechanics. Here's how the branching actually gets built:

**The block that does the work: Check contact attributes**
Once your Fargate audio bridge and Lambda write the risk score back as a contact attribute (e.g. a `RiskScore` attribute holding a numeric 0–100 value from Protect), you drop a **Check contact attributes** block right after it. This block supports comparisons of Equals, Is Greater Than, Is Less Than, Starts With, and Contains — and critically, you can add **multiple conditions to one block**, each producing its own output branch, evaluated in the order you list them.

**A typical two- or three-tier threshold setup:**
- Condition 1: `Is Greater Than 80` → high risk → route to a fraud-review queue or force step-up verification regardless of what Passport said
- Condition 2: `Is Greater Than 40` → medium risk → allow the call through but flag the agent's screen pop with a warning banner
- No match (default branch) → low risk → standard routing, no flag

Order matters here — Connect evaluates conditions top to bottom and takes the first match, so the higher threshold has to come first (checking `>80` before `>40`), otherwise a score of 90 would incorrectly satisfy the `>40` condition first if that one were listed above it.

**A few practical wrinkles worth planning for:**
- **Null/missing score** — if Protect's async scoring hasn't completed by the time this block runs (a real possibility given it's processing a live audio stream), the attribute won't exist yet. The docs note `Check contact attributes` can't natively distinguish "attribute doesn't exist" from a normal comparison — the standard workaround is to check the attribute against a value that couldn't naturally occur (like checking `Equals` a Lambda-set default of `-1`), or have your Lambda explicitly set a `RiskScoreStatus` attribute of `"pending"` / `"available"` / `"error"` that you branch on first, before checking the numeric value.
- **Combine with Passport's auth result**, not just Protect's score in isolation — you typically want a second `Check contact attributes` (or a Lambda that fuses both) so a caller can be flagged even if Passport passed them, since Protect's fraud signals (device/network anomalies) catch different attack patterns than a voiceprint mismatch would.
- **Attribute persistence across transfers** — contact attributes carry across the flow even through queue transfers, since they're shared by `InitialContactId`, so the risk score set early in the flow is still readable by the agent whisper flow or screen-pop Lambda later, without needing to re-fetch it.

## What contact attributes get passed when transferring to a queue?

Good — this confirms it clearly. It's worth separating two different things: what attributes exist *at* a queue transfer vs. what carries *through* it, since your Passport/Protect data falls into the second bucket.

**All contact attributes persist through the transfer — nothing is dropped**
There isn't a special "allow-list" of attributes that get forwarded when you hit a Transfer to queue block. Contact attributes are shared across all contacts with the same `InitialContactId` — meaning while carrying out transfers, a contact attribute updated earlier in the flow (like your `AuthStatus` or `RiskScore` from Pindrop) updates the attribute's value across both the inbound and transfer contact records. So anything your Lambda wrote as a user-defined attribute before the transfer block runs is still readable afterward — by the customer queue flow, the agent whisper flow, or a screen-pop Lambda.

**System attributes that become newly available once the queue is set:**
- `$.Queue.Name` — the queue's name
- `$.Queue.ARN` — the queue's ARN
- (in outbound whisper flows only) the queue's configured outbound caller ID number

**What's *not* automatically forwarded:**
- `$.StoredCustomerInput` (from a Store customer input block) is explicitly called out as **not included in contact records and not accessible in Lambda input** — if you captured something via DTMF/voice input that you want available downstream (e.g. after the transfer, or in an agent screen-pop Lambda), you have to explicitly copy it into a user-defined attribute with a **Set contact attributes** block first. This is an easy trap: if your Pindrop step-up flow uses Store customer input for a PIN, that raw value disappears unless you promote it.

**Practical implication for your integration:** right before the Transfer to queue block, it's worth adding a **Set contact attributes** block that consolidates everything you want the agent-facing screen pop to see — `AuthStatus`, `RiskScore`, `FraudFlag`, maybe a `PindropSessionId` for cross-referencing in Pulse later — into clearly named user-defined attributes. Don't rely on attributes set deep inside a Lambda's raw response object being readable later without that explicit copy step, and don't rely on Store customer input attributes surviving the transfer at all.

## What does the agent see in the CCP screen pop from this contact flow?

This clears up an important detail: the plain, out-of-the-box CCP doesn't render a screen pop at all — it's something you have to build.

**The base CCP has no built-in screen pop**
Screenpops aren't directly available in the CCP — to display caller contact attributes to agents, you have to use the **Amazon Connect Streams API** to embed the CCP into a custom web application (or use a CRM adapter, like the Salesforce CTI Adapter, that does this for you). So "what the agent sees" depends entirely on what your custom screen-pop app is built to render — Connect just makes the data available; the presentation layer is yours to design.

**What data is actually accessible to that custom app:**
Via Streams, your screen-pop code calls `contact.getAttributes()` when the contact connects, which returns all of the user-defined attributes you set during the flow — this is exactly where your `AuthStatus`, `RiskScore`, and `FraudFlag` attributes surface. You can also pull `contact.getConnections()` for endpoint info (the caller's number), and `agent.getName()` / `agent.getContacts()` for agent-side context.

**A realistic screen-pop layout for this integration would show:**
- Caller ID / customer name (if resolved via a CRM lookup keyed on ANI)
- **Passport auth result** — e.g. "Verified" / "Failed — stepped up via PIN" / "Not enrolled"
- **Protect risk score** — often rendered as a simple low/medium/high badge rather than a raw number, since agents need a fast visual cue, not a score to interpret
- **Fraud flag**, if set — typically styled distinctly (red banner) so it can't be missed, possibly with a note on which signal triggered it (voice mismatch vs. device/network anomaly)
- Any escalation guidance text your Lambda set as a user-defined attribute (e.g. "Follow fraud verification procedure B")

**Two implementation paths, in practice:**
1. **Native custom app** — build the screen pop yourself with Streams, full control over layout, hosted wherever you like.
2. **CRM-embedded** — if agents already work inside Salesforce or a similar CRM, a CTI adapter (which itself uses Streams under the hood) can push the same attributes into a panel inside that CRM instead of a separate window — this is usually the better experience since agents don't have to context-switch between two windows.

One thing worth deciding early: since `contact.getAttributes()` runs client-side in the agent's browser, make sure nothing overly sensitive (like a full risk-score breakdown or PII beyond what the agent needs) ends up in a contact attribute that a browser-based screen pop can read — the CCP attribute channel isn't the place for anything you wouldn't want visible in browser dev tools.
