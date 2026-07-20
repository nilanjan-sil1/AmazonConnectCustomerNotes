# Basics of Pindrop

## How does Pindrop Passport voice authentication work?

Passport combines three passive signals, and the "passive" part is the key design principle — it authenticates while the caller just talks normally, without asking them to repeat a passphrase or answer security questions.

**The three technologies it fuses:**

**1. Deep Voice™ — the voice biometric engine**
This is Pindrop's neural network speaker-recognition system, described as the first end-to-end deep-learning speaker recognition model — it analyzes short utterances of the caller's actual speech directly from the raw audio, rather than relying on hand-engineered speech features the way older systems did. It filters out background noise and focuses on vocal characteristics unique to the speaker — pitch, cadence, spectral frequencies, pronunciation patterns.

**2. Phoneprinting™ — device and network fingerprinting**
This runs on the call's full audio signal (not the caller's voice specifically) and analyzes over 1,300 features to build a profile of the calling device, carrier, and geolocation — detecting anomalies that suggest the call isn't coming from where or what it claims. Notably, this can authenticate passively within the first 10 seconds of a call, without requiring any voice or keypress input at all — it's working before the caller has even said anything meaningful.

**3. Toneprinting™ — behavioral/keypress analysis**
Analyzes tone-based signals and metadata from every DTMF keypress the caller makes, adding a behavioral fingerprint layered on top of the voice and device signals.

**The enrollment → verification lifecycle:**
1. **Enrollment** — a voice profile is created during a caller's interaction (Pindrop markets this as happening "in seconds," without static phrases or PINs — either actively, where the user speaks a set passphrase, or passively, built up from natural conversation during regular calls).
2. **Verification** — on a later call, the newly captured voice is compared against the stored profile, generating a **similarity score**. If it clears a predefined threshold, authentication passes automatically.
3. **Liveness/anti-spoofing check** — layered in to confirm the voice is coming from a live speaker in real time rather than a recording or synthetic/deepfake audio, since a similarity score alone can't distinguish a genuine live caller from a sufficiently good replay or AI-generated clone.

**Where this connects to what we've already covered in this conversation:** the whole reason your Lambda passes the KVS stream metadata and ANI to Passport is so it has both the audio to run Deep Voice against and the call/device metadata to run Phoneprinting against — and the voice-mismatch detection feature mentioned earlier (alerting when a voice doesn't match an enrolled profile, especially useful for shared accounts with multiple enrolled users) is a direct output of this same matching pipeline, not a separate system.

**One thing worth double-checking with Pindrop directly for your deployment:** whether you're using active or passive enrollment changes your contact flow design meaningfully — active enrollment needs an explicit IVR prompt asking the caller to say something specific the first time, while passive enrollment builds the profile invisibly over one or more regular calls, which affects when in your flow (and how many calls in) Passport actually becomes able to return a confident match rather than an "insufficient enrollment data" result.

## How does Pindrop Protect score fraud risk in real time?

This gives a fuller picture than what I described earlier — Protect isn't a one-time score, it's a **dynamic score that updates continuously across the whole call and beyond**.

**The signal layers it fuses**
Protect is described as a multifactor solution that layers voice, device, metadata, and behavior analysis, cross-referencing calls against a **fraud consortium** — Pindrop's shared database of confirmed fraud data across its customer base. Concretely:
- **Voice** — the same Deep Voice biometric analysis used by Passport, but here applied to detect anomalies rather than confirm identity match.
- **Phoneprinting** — device type, geolocation, and carrier detection, explicitly built to work **regardless of spoofing** (i.e. it can flag when a caller's claimed number doesn't match the actual originating device/network characteristics).
- **Behavioral signals** — keypress patterns, call patterns, and interaction anomalies.
- **Fraud consortium cross-reference** — comparing this call's signals against Pindrop's aggregated pool of confirmed fraud data across all their customers, not just your organization's own history.

**When scoring happens**
Risk scoring starts at call connection, continues through the IVR, and carries through into the agent stage — a single risk score that evolves at each stage of the journey rather than being computed once and left static. That's an important detail for the contact-flow branching we discussed earlier: your `RiskScore` attribute isn't a one-time value your Lambda fetches once — it's something that can legitimately change while the caller is still on the line with an agent, which argues for re-checking it (or having Protect push updates) rather than treating the IVR-stage score as final.

**Passive by design**
It analyzes audio and extracts features in the background without disrupting the conversation or call flow — no explicit challenge to the caller, consistent with the same "no added friction" philosophy as Passport.

**Post-call: Continuous Scoring / fraud look-back**
This is worth flagging because it directly affects the S3/retention discussion from earlier — Protect's evaluation continues even after the call ends, through a capability Pindrop calls **Continuous Scoring**: it can re-score past calls using newly learned fraud intelligence, with a configurable look-back window from 24 hours up to several weeks. This means a call that scored as low-risk at the time can later get flagged as new fraud patterns emerge — which is a real argument for retaining call recordings and metadata in S3 for at least as long as your configured look-back window, since a stale/deleted recording can't be re-evaluated when Protect's models improve.

**One correction to something Pindrop's marketing surfaces that's relevant to our earlier Pulse mix-up:** the search results now show Protect and **Pulse** explicitly described as running *simultaneously* on the same call — "Pindrop Protect runs simultaneously, returning a single risk score per call... Pindrop Pulse detected deepfakes with 99.2% accuracy... when used with Pindrop Protect, and provides liveness scores to confirm a human caller." This confirms the earlier correction: Pulse is a **real-time deepfake/liveness companion to Protect**, not a batch post-call analytics product — they're two parallel real-time scoring engines feeding into the same call, not sequential stages.

## What analytics does Pindrop Pulse provide?

Now I have a clear, confirmed picture — thanks for the patience while I sorted that out earlier. Pulse is a **deepfake and liveness detection engine for calls**, not a reporting/dashboard product. Here's what it actually delivers:

**The core output: a liveness score + reason code**
Every assessment produces a **liveness score from 0–100** indicating the likelihood the voice belongs to a live human rather than a synthetic/AI-generated source, plus a **reason code** explaining what drove that score — this pairing matters because a raw number alone isn't actionable; the reason code is what makes the result explainable to an analyst or usable in a policy rule (e.g. "flagged due to unnatural pauses" vs. "flagged due to frequency distortion").

**How it technically works**
Pulse analyzes patterns that are natural for humans but hard for machines to replicate consistently — frequency distortions, voice variance, unnatural pauses, temporal anomalies — and generates what Pindrop calls a **"fakeprint"**: a compact mathematical representation of the artifacts that distinguish machine-generated speech from genuine human speech. Detection works with as little as **2 seconds of speech**, and critically, it has **no dependency on voice enrollment** — every call is analyzed atomically, unlike Passport, which needs a stored voice profile to compare against.

**Performance figures Pindrop publishes** (their own claims, worth treating as vendor-reported rather than independently verified): around 99% detection accuracy for known deepfake generation engines, 90%+ for previously unseen ones, under 1% false positive rate, and resilience testing against noise, compression, transcoding, and adversarial attacks (they cite University of Waterloo black-box test scenarios).

**Two deployment modes**
- **Pulse for Calls** — the one relevant to your Connect/contact-center architecture, working alongside Passport and Protect on the same live audio.
- **Pulse for Meetings** — a separate product for Zoom/Webex/Teams that also adds video liveness detection and geolocation checks (VPN usage, mismatched geography) — not applicable to a voice-only contact center integration, but worth knowing it exists in case your organization also handles video-based identity verification elsewhere.

**How it integrates with Passport and Protect specifically**
This is the part most relevant to the architecture we've been building: Pulse's liveness score is designed to be **consumed by Passport's authentication policies** — you can configure Passport so a suspiciously low liveness score blocks enrollment or blocks authentication outright, preventing a deepfake caller from ever getting voice-matched in the first place, regardless of how good the synthetic voice is. And it runs **simultaneously** with Protect, contributing to the same real-time risk picture rather than being a separate downstream step.

**What this means for the Lambda/contact-flow design we've discussed:** if you're licensing Pulse, your `Check contact attributes` risk logic should treat `LivenessScore` as a **gating check before** trusting a Passport auth pass, not just another independent attribute to branch on alongside `RiskScore` — a high Passport match score on a deepfake-generated voice is exactly the failure mode Pulse exists to catch, so the two need to be evaluated together rather than in isolation.