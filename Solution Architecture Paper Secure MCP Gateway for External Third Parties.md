# Solution Architecture Paper: Secure MCP Gateway for External Third Parties

**Organisation type:** UK high street retail bank (PRA-authorised, FCA-regulated) **Document status:** Draft v0.1 for Architecture Review Board and Security Design Authority **Date:** 23 November 2026 **Classification:** Internal (proposed)

> Note: Legal and regulatory references reflect the author's understanding as at the date above. Compliance, Legal and the DPO must validate them before approval, particularly where regulation is evolving (AI, critical third parties, cyber resilience legislation).

## 1. Executive Summary

The bank intends to expose selected banking capabilities to external third parties (TPPs, fintechs, partner platforms and AI agent platforms) through a **Model Context Protocol (MCP) Gateway**. MCP lets AI agents discover and call "tools" over a standard protocol. This introduces new risks compared with conventional APIs: tool descriptions are consumed by language models, tool output can contain adversarial instructions, and calling clients may act autonomously on a customer's behalf.

This paper proposes an architecture that:

- Treats the MCP Gateway as a **Policy Enforcement Point** in front of internal MCP servers and existing API estates, never as a pass-through proxy.
- Builds on proven UK Open Banking security (FAPI-grade OAuth, mutual TLS or DPoP sender-constrained tokens, directory-based TPP identity, consent management) rather than inventing new trust mechanisms.
- Separates three identities: the **customer**, the **third-party organisation**, and the **agent or software instance**, and requires all three to be explicit in every authorisation decision.
- Is **read-only and low-risk first**, with payments and other state-changing tools gated by strong customer authentication (SCA), dynamic linking and human-in-the-loop confirmation.
- Is **fail-closed for security decisions** and **gracefully degrading for availability**, with kill switches at global, TPP, tool and consent level.
- Maps explicitly to UK obligations: FCA Principles and SYSC, Consumer Duty, PRA SS2/21 and SS1/21, the operational resilience regime, PSRs 2017, UK GDPR and DPA 2018 (as amended), the SM&CR, and the Computer Misuse Act 1990.

**Decisions requested:** approve the target architecture (Section 5), the phased roadmap (Section 15), and the key architecture decisions (Section 16).

## 2. Scope, Objectives and Assumptions

### 2.1 In scope

- External access by authorised third parties to bank-owned MCP tools (account information, balances, transactions, product information, servicing queries, and later payment initiation).
- Authentication, authorisation, identity, governance, lifecycle, fallback, defence in depth, threat modelling and regulatory compliance for that exposure.

### 2.2 Out of scope

- Internal staff use of MCP or AI assistants.
- The security of third-party LLMs and agent runtimes (the bank sets contractual and technical conditions but does not control them).
- Core banking modernisation.

### 2.3 Objectives

1. Enable secure, consented third-party and agent access without weakening existing control posture.
2. Maintain regulatory compliance and evidenceability from day one.
3. Preserve customer trust and meet Consumer Duty outcomes.
4. Keep operational resilience within impact tolerances for important business services.

### 2.4 Assumptions

- The bank already operates an Open Banking API estate, a CIAM platform with SCA, an enterprise API gateway, a SIEM and a 24x7 SOC.
- MCP is used over Streamable HTTP with the OAuth 2.1-based authorisation model defined in the MCP specification. Local (stdio) transports are not exposed externally.
- Third parties are either FCA-authorised or registered (AISP/PISP, or authorised firms acting as agents) or contracted commercial partners.
- Customer data is only released with a valid legal basis and, for payment account data, explicit customer consent.

## 3. Business Context and Use Cases

| ID | Use case | Risk tier | Phase |
| --- | --- | --- | --- |
| UC1 | AI assistant of a licensed AISP retrieves customer's consented account and transaction data | Medium | 1 |
| UC2 | Partner platform queries public product, branch and service-status information | Low | 1 |
| UC3 | Customer-authorised agent answers servicing questions (for example statement availability, standing orders list) | Medium | 2 |
| UC4 | Licensed PISP agent initiates a payment on explicit customer instruction | High | 3 |
| UC5 | Agent-initiated servicing changes (limits, payees, contact details) | Very high | Not planned; requires separate risk acceptance |

## 4. Architecture Principles

1. **Zero trust:** no implicit trust by network location, client identity or prior session.
2. **Reuse regulated rails:** build on Open Banking trust frameworks, FAPI profiles and existing consent infrastructure.
3. **Least privilege and data minimisation:** per-tool, per-field, per-purpose access.
4. **Untrusted by default:** treat TPP clients, agent behaviour, model output and tool-call parameters as untrusted input.
5. **Deterministic controls around non-deterministic actors:** policy enforcement must never rely on an LLM's judgement.
6. **Fail closed on security, degrade gracefully on availability.**
7. **Every action attributable:** customer, TPP, agent, tool, consent and purpose are logged for each call.
8. **Human accountability:** named SM&CR owner for the service; humans confirm high-risk actions.
9. **Secure by design and evidenced:** controls are testable, automated and auditable.

## 5. Target Solution Architecture

### 5.1 Logical layers (outside to inside)

1. **External zone:** third-party MCP clients and agent platforms.
2. **Edge layer:** DDoS protection, CDN, WAF, bot management, TLS termination (TLS 1.2 minimum, TLS 1.3 preferred), mTLS where used.
3. **MCP Gateway (DMZ, multi-AZ and multi-region):** protocol termination, token validation, policy enforcement, rate limiting, schema validation, response filtering, audit emission.
4. **Trust and identity services:** Authorisation Server (bank CIAM/OAuth), TPP registry and certificate validation, consent service, policy decision point (PDP), secrets and key management (HSM-backed).
5. **Internal MCP server tier (tool adapters):** thin, purpose-built MCP servers, one per domain, each exposing only approved tools and calling existing internal APIs. No MCP server connects directly to core systems or databases.
6. **Existing API and core layer:** enterprise API gateway, Open Banking APIs, core banking and payments platforms.
7. **Security operations and governance plane:** SIEM, SOAR, tool registry, TPP management portal, GRC tooling, observability.

```mermaid
flowchart TB
  subgraph EXT["1. External zone"]
    TP["Third-party MCP clients and agent platforms"]
  end
  subgraph EDGE["2. Edge layer"]
    EG["DDoS protection, CDN, WAF, bot management, TLS termination, mTLS"]
  end
  subgraph DMZ["3. MCP Gateway - DMZ, multi-AZ, multi-region"]
    GW["MCP Gateway: token validation, policy enforcement, rate limiting, schema validation, response filtering, audit emission"]
  end
  subgraph TRUST["4. Trust and identity services"]
    AS["Authorisation Server - bank CIAM/OAuth"]
    REG["TPP Registry and certificate validation"]
    CON["Consent Service"]
    PDP["Policy Decision Point"]
    KMS["Secrets and key management - HSM-backed"]
  end
  subgraph MCPS["5. Internal MCP server tier"]
    MS["Thin tool-adapter MCP servers, one per domain"]
  end
  subgraph CORE["6. Existing API and core layer"]
    API["Enterprise API gateway and Open Banking APIs"]
    CB["Core banking and payments platforms"]
  end
  subgraph OPS["7. Security operations and governance plane"]
    SIEM["SIEM and SOAR"]
    TR["Tool Registry"]
    KS["Kill-switch Service"]
    GRC["TPP management portal, GRC tooling, observability"]
  end

  TP --> EG --> GW
  GW <--> AS
  GW <--> REG
  GW <--> CON
  GW <--> PDP
  AS --- KMS
  GW --> MS --> API --> CB
  GW -.->|audit events| SIEM
  MS -.->|audit events| SIEM
  TR -.->|signed tool manifests| GW
  KS -.->|disable at global, TPP, tool, consent level| GW
  GRC -.-> REG
```

Figure 5.1: Logical layers from the external zone to the core, with the security operations and governance plane acting on the Gateway from the side.

### 5.2 Request flow (summary)

1. Third-party client discovers the Gateway's protected resource metadata (RFC 9728) and the Authorisation Server metadata (RFC 8414).
2. Client, pre-registered in the bank's TPP registry, starts an authorisation request using **PAR** (RFC 9126), **PKCE** and **Rich Authorization Requests** (RFC 9396) describing the specific data or action and duration.
3. The customer authenticates and gives consent in the bank's own channel (app redirect or decoupled flow), with SCA applied per the regulatory rules.
4. The Authorisation Server issues a **short-lived, sender-constrained access token** (mTLS-bound per RFC 8705 or DPoP per RFC 9449), audience-restricted to the Gateway (resource indicators, RFC 8707).
5. On each MCP request, the Gateway validates token, sender-binding, client status, consent status and tool entitlement via the PDP, then validates tool arguments against a strict schema.
6. The Gateway exchanges the external token for a **separate internal token** (token exchange, RFC 8693) scoped to the specific internal MCP server and downstream API. **Tokens are never passed through** to internal services, which prevents confused-deputy attacks.
7. The internal MCP server calls the downstream API; the response is filtered (field-level, purpose-level), scanned and labelled as untrusted data before return.
8. Full audit event emitted to the immutable log and SIEM.

```mermaid
sequenceDiagram
  autonumber
  participant C as Third-party MCP client
  participant U as Customer
  participant AS as Authorisation Server
  participant GW as MCP Gateway
  participant PDP as Policy Decision Point
  participant MS as Internal MCP server
  participant API as Downstream API
  participant SIEM as Audit log and SIEM

  Note over C,GW: Step 1 - Discovery
  C->>GW: Request protected resource metadata (RFC 9728)
  GW-->>C: Metadata naming the Authorisation Server
  C->>AS: Request AS metadata (RFC 8414)

  Note over C,U: Steps 2 and 3 - Authorisation and consent
  C->>AS: PAR with PKCE and Rich Authorization Requests (RFC 9126, 9396)
  AS->>U: Authenticate and request consent in bank channel, with SCA
  U-->>AS: Consent granted

  Note over C,AS: Step 4 - Token issuance
  AS-->>C: Short-lived sender-constrained token (RFC 8705 or 9449), audience = Gateway (RFC 8707)

  Note over C,PDP: Step 5 - Per-request enforcement
  C->>GW: MCP tool call with access token
  GW->>PDP: Check token, sender-binding, client status, consent, tool entitlement
  PDP-->>GW: Decision and obligations
  GW->>GW: Validate tool arguments against strict schema

  Note over GW,MS: Step 6 - No token passthrough
  GW->>AS: Token exchange (RFC 8693)
  AS-->>GW: Separate internal token for one service

  Note over MS,API: Step 7 - Downstream call and response control
  GW->>MS: Tool call with internal token
  MS->>API: Downstream API call
  API-->>MS: Response
  MS-->>GW: Result
  GW->>GW: Filter fields, scan, label as untrusted data
  GW-->>C: Filtered tool result

  Note over GW,SIEM: Step 8 - Audit
  GW--)SIEM: Full audit event to immutable log
```

Figure 5.2: Request flow from discovery to audited response; the numbered notes correspond to the eight steps above.

### 5.3 Key components

| Component | Responsibility | Notes |
| --- | --- | --- |
| MCP Gateway | Terminates MCP sessions; enforces authN, authZ, quotas, schema, response controls | Stateless where possible; session state in encrypted, TTL-bound store |
| Authorisation Server | OAuth/OIDC, FAPI 2.0-aligned profile, token issuance and revocation | Open Banking FAPI 1.0 Advanced compatibility maintained during migration |
| TPP Registry | Source of truth for third-party identity, regulatory status, certificates, permitted tools | Synced with the FCA Register and the Open Banking Directory |
| Consent Service | Stores, versions and revokes customer consents; powers the customer dashboard | Single source for consent state |
| PDP (policy engine) | Evaluates ABAC/ReBAC policies as code (for example OPA or Cedar) | Policies version-controlled, tested, signed |
| Tool Registry | Catalogue of approved tools, schemas, versions, hashes, risk tier, owner | Gateway only exposes registered, signed tool manifests |
| Response Guard | Field filtering, PII minimisation, injection-pattern and anomaly detection on outbound content | Content labelled as data, not instructions |
| Kill-switch Service | Instant disablement at global, TPP, tool, consent, customer level | Propagates to Gateway in seconds |
| Audit and Telemetry | Tamper-evident logging, metrics, traces | Integrated with SOC and regulatory reporting |

## 6. Identity Architecture

### 6.1 Identity types

| Identity | Examples | Proof |
| --- | --- | --- |
| Customer (PSU) | Retail or SME account holder | Bank CIAM credentials, device binding, biometrics, SCA |
| Third-party organisation | AISP, PISP, partner firm | Open Banking Directory identity, FCA register status, UK Open Banking certificates (OBWAC, OBSEAL) or bank-issued mTLS certificates for non-OB partners |
| Client application | Registered OAuth client | Client ID, private-key JWT or mTLS authentication, software statement |
| Agent or workload instance | Specific agent deployment | Workload identity (for example SPIFFE-style SVID or signed software statement per instance) and unique instance identifier carried in token claims |
| Internal service | Gateway, MCP servers | mTLS workload identity, short-lived credentials from internal PKI |

### 6.2 Principles

- **No shared client credentials** across third parties or environments.
- **Static registration only.** Open Dynamic Client Registration is disabled. If MCP clients present Client ID Metadata Documents or DCR requests, they are routed to an admin-approved onboarding workflow.
- **Delegation is explicit.** Every token carries subject (customer), actor (TPP and agent instance, via the `act` claim), consent identifier, purpose and tool scopes.
- **Regulatory status is live.** The Gateway checks authorisation status (for example revoked or passported-out TPP) at token issuance and via a regularly refreshed cache so that status changes take effect quickly.
- **Pairwise subject identifiers** are used so that TPPs cannot correlate customers across firms.

## 7. Authentication Architecture

### 7.1 TPP and client authentication

- **Client authentication:** `private_key_jwt` or `tls_client_auth` using certificates from the Open Banking Directory (or bank PKI for non-regulated partners). Client secrets are not permitted.
- **Transport:** TLS 1.3 preferred; mTLS for all client-to-AS and (where used) client-to-Gateway traffic.
- **Sender-constrained tokens:** mTLS-bound or DPoP-bound access and refresh tokens, so a stolen token cannot be replayed.
- **Request integrity:** PAR with signed request objects (JAR) so authorisation parameters cannot be tampered with in the browser.

### 7.2 Customer authentication

- Authentication occurs only on **bank-owned surfaces** (bank app or web). TPPs and agents never see or handle customer credentials.
- **SCA** is applied in line with PSRs 2017 and the SCA regulatory technical standards: on initial account access consent, on payments, and where risk analysis demands. Account-access consent re-confirmation by the TPP follows the current FCA-permitted regime; the bank retains risk-based step-up rights.
- **Dynamic linking** for payments: authentication code is bound to amount and payee, and any change invalidates it.
- **Decoupled flows (CIBA-style)** are supported for agent-originated requests so the customer approves on their own trusted device.

### 7.3 Token profile

| Token | Lifetime | Properties |
| --- | --- | --- |
| Access token | 5 minutes or less | Sender-constrained, audience = Gateway, scopes = specific tools |
| Refresh token | Bound to consent term | Sender-constrained, rotating, reuse detection revokes the family |
| Internal exchanged token | 60 seconds or less | Audience = single internal service, no customer-wide scope |
| Authorisation code | 60 seconds | Single-use, PKCE-bound |

Opaque or signed (JWS) tokens may be used; JWT validation requires strict algorithm allow-listing (no `none`), key ID pinning via JWKS with cache and fallback (Section 11), and clock-skew limits.

### 7.4 MCP-specific authentication rules

- The Gateway acts as an OAuth **resource server** only; it does not mint tokens.
- It **must reject tokens not issued for its own audience**, and must never forward inbound tokens to downstream services.
- Session identifiers are not authentication: every request is independently authorised.
- Streaming and long-lived connections are re-validated at defined intervals and on revocation events.

## 8. Authorisation Architecture

### 8.1 Layered authorisation model

1. **Coarse:** is the TPP active, regulated for the requested role, and contractually permitted to use this tool category?
2. **Consent:** does a valid, unrevoked customer consent exist covering this tool, data cluster, account(s) and time window?
3. **Tool:** is the tool in the TPP's approved entitlement set, at the approved version?
4. **Parameter and resource:** do arguments (account ID, date range, amount) stay within consented resources? Object-level checks prevent BOLA/IDOR.
5. **Context:** risk signals (anomalous volume, new geography/ASN, velocity, device or instance change, fraud scores) may deny or force step-up.
6. **Action:** for state-changing tools, SCA with dynamic linking and explicit human confirmation.

Decisions are made by the PDP using policy-as-code, with obligations returned to the Gateway (for example "redact fields X, Y", "limit to 90 days", "require step-up").

### 8.2 Tool classification

| Class | Examples | Controls |
| --- | --- | --- |
| T0 Public | Branch locator, product info | Client auth, rate limits |
| T1 Read, non-sensitive | Statement availability | Consent, tool entitlement |
| T2 Read, sensitive financial | Balances, transactions | Consent + SCA at grant, field minimisation, anomaly detection |
| T3 Write, low impact | Preference notifications | T2 + customer confirmation |
| T4 Write, financial | Payment initiation | T2 + SCA with dynamic linking, human-in-the-loop, limits, fraud checks, APP-fraud controls |
| T5 Prohibited externally | Credentials, limit and payee changes, admin | Not exposed |

### 8.3 Fine-grained consent

Consent is expressed through Rich Authorization Requests (for example type `account_information`, specific accounts, data clusters, expiry). Consent is **purpose-bound** (UK GDPR purpose limitation) and visible in a bank-hosted **consent dashboard** where the customer can review and revoke any TPP or agent at any time. Revocation propagates to the Gateway and invalidates tokens immediately.

### 8.4 Rate limiting and abuse control

Per-TPP, per-customer, per-tool and per-agent-instance quotas; cost and token-volume budgets; adaptive throttling; concurrency caps for streaming sessions; and tarpit behaviour for suspicious clients.

## 9. Governance

### 9.1 Accountability (SM&CR)

- A named **Senior Manager** (for example the SMF holder responsible for operations and technology, or the relevant business SMF) holds a documented Statement of Responsibilities for the MCP Gateway service.
- The **Security Design Authority** owns reference architecture; **CISO** owns security risk acceptance; the **DPO** advises on data protection; **Compliance** owns regulatory mapping; **Operational Resilience** owns impact tolerances; **Third-Party Risk** owns TPP due diligence.

### 9.2 Governance forums and artefacts

| Forum or artefact | Purpose |
| --- | --- |
| MCP Service Governance Board (monthly) | Approves tools, TPP onboarding tiers, policy changes, incident lessons |
| Tool Approval Process | Threat model, DPIA, security test and business sign-off before any tool enters the Registry |
| Policy-as-code repository | Peer-reviewed, signed, versioned authorisation policies; changes through CI/CD with automated tests |
| Risk register and exceptions | Tracked with time-bound acceptances |
| Model and AI risk review | Applies where bank-operated AI components are used (for example response guard classifiers), aligned to PRA SS1/23 principles where applicable and FCA AI expectations |
| Management information | Usage, incidents, denials, consent revocations, TPP performance, complaints, fraud losses |

### 9.3 Third-party obligations

Each TPP is bound by a contract and technical onboarding terms including: security baseline (ISO 27001 or equivalent, SOC 2 Type II or independent attestation), incident notification SLA, right to audit or request evidence, data handling and retention rules, prohibition on training models on bank-supplied customer data without separate lawful basis, sub-processor transparency, and immediate suspension rights. Regulated TPPs additionally evidence FCA authorisation; the bank does not duplicate regulatory supervision but verifies status continuously.

### 9.4 Tool and prompt supply-chain governance

Tool names, descriptions, schemas and annotations are treated as **security-sensitive code**: they are signed, hashed and version-pinned. Any change requires re-approval, and clients see changes only through versioned manifests. This prevents tool poisoning and "rug pull" attacks where a tool's behaviour or description silently changes after approval.

## 10. Lifecycle Management

### 10.1 Third-party lifecycle

1. **Request and due diligence:** regulatory status, financial and security assessment, data protection and Consumer Duty fit.
2. **Contract and onboarding:** tiered entitlement, certificates issued or validated, sandbox access.
3. **Conformance testing:** security and functional tests, penetration evidence, agent behaviour tests (prompt-injection resilience, scope adherence).
4. **Production approval:** SM&CR-accountable sign-off, progressive quotas.
5. **Continuous monitoring:** regulatory status feed, behavioural baselines, periodic recertification (at least annual; more frequent for high tiers).
6. **Suspension and offboarding:** immediate credential, token and consent revocation; data return and deletion attestations; evidence retained for regulatory retention periods.

### 10.2 Tool lifecycle

Idea, threat model and DPIA, design review, build with SAST/DAST and dependency scanning, signed release into the Registry, staged rollout (canary, then percentage), monitoring, version deprecation with notice to consumers, retirement with revocation of entitlements. Tool versions are immutable once published.

### 10.3 Credential, key and certificate lifecycle

- Certificates: short validity, automated renewal, revocation checking (OCSP/CRL) with defined fail behaviour, directory-driven rotation for OB certificates.
- Signing keys: HSM-protected, rotated at least annually and after any compromise, with overlapping JWKS publication.
- Secrets: vault-managed, dynamic where possible, no secrets in code or configuration.

### 10.4 Consent lifecycle

Create, authenticate, activate, use, re-confirm (per applicable regime), expire, revoke. Every transition is evented, logged and visible to the customer. Consents expire automatically on term end, TPP suspension, account closure or death/bereavement flags.

### 10.5 Data lifecycle

Gateway does not persist customer payload data beyond transient processing; logs store identifiers and metadata with payloads tokenised or hashed. Retention follows the bank's retention schedule and record-keeping rules, with secure deletion and legal hold support.

## 11. Fallback, Resilience and Continuity

### 11.1 Resilience objectives

The service is mapped to relevant **important business services** (for example access to account information and, in Phase 3, payment initiation) under the FCA and PRA operational resilience rules, with defined **impact tolerances**, scenario testing against severe but plausible disruptions, and self-assessment evidence. Indicative targets (to be confirmed by Operational Resilience): availability of 99.95% for the Gateway, RPO near zero for consent state, RTO under 15 minutes for regional failover.

### 11.2 Fallback mechanisms

| Failure | Behaviour | Principle |
| --- | --- | --- |
| Authorisation Server unavailable | Existing valid tokens continue until expiry (short TTL). Gateway uses cached JWKS (bounded staleness). No new grants. Customer-facing messaging. | Fail closed for new access |
| JWKS or key retrieval failure | Use last-known-good keys within a defined grace window, then deny | Bounded degradation |
| PDP unavailable | Local embedded policy bundle (signed, last-known-good) evaluates cached decisions; high-risk tools (T3/T4) denied | Fail closed on risk |
| Consent service unavailable | Read of recently verified consent cache for T1/T2 with short TTL; deny T3/T4 | Risk-tiered |
| Revocation or kill-switch channel down | Short token TTLs bound exposure; Gateway polls a signed deny-list as secondary path | Defence in depth |
| Downstream core or API degraded | Circuit breakers, bulkheads, timeouts, retry budgets with jitter; return explicit, machine-readable error. Optional serving of clearly time-stamped stale data for low-risk reads (balances never served stale without flag) | Graceful degradation |
| Regional outage | Active-active multi-region with health-based routing; stateless Gateways; replicated consent and policy stores | Resilience |
| Gateway compromise suspected | Per-TPP or global **kill switch**, traffic shifts to read-only or full block, credentials rotated | Containment |
| Agent misbehaviour or runaway loop | Rate limits, budget exhaustion, automatic session termination, quarantine of the agent instance | Containment |

### 11.3 Degraded modes

1. **Normal:** all approved tools.
2. **Restricted:** T3/T4 disabled; read tools only.
3. **Minimal:** T0/T1 only.
4. **Blocked:** all external access disabled; customers directed to bank-channel alternatives.

Modes are switched by the Kill-switch Service, by automated policy on SOC triggers, or manually by the duty incident manager. Switching and criteria are rehearsed.

### 11.4 Fallback for customers

Bank-owned channels remain fully available; customers are not dependent on third-party agents for essential banking. Where an outage affects TPP customers, communications follow Consumer Duty expectations (clear, timely, helpful) and the bank's vulnerable customer procedures.

## 12. Security Architecture and Defence in Depth

### 12.1 Layered controls

| Layer | Controls |
| --- | --- |
| 1. Perimeter | DDoS mitigation, WAF with MCP/JSON-RPC awareness, geo and ASN risk, bot management, IP allow-listing for high tiers where feasible |
| 2. Transport | TLS 1.3, mTLS, certificate pinning policies for internal hops, HSTS, strong cipher suites |
| 3. Identity | Registered clients, sender-constrained tokens, workload identity, no shared secrets |
| 4. Gateway policy | Token and consent checks, PDP decisions, strict JSON-RPC and schema validation, size/depth limits, method allow-listing (unused MCP features such as sampling and elicitation disabled for external clients unless approved) |
| 5. Application | Thin MCP servers, input validation, output encoding, parameterised calls, no dynamic code execution, no arbitrary URL fetch (SSRF protection), separate runtime per domain |
| 6. Data | Field-level minimisation, tokenisation, encryption at rest (KMS/HSM), egress DLP, contextual redaction |
| 7. AI-specific | Response Guard marking returned text as untrusted data, neutralisation of instruction-like content in free-text fields (for example payment references), tool-description integrity checks, no tool chaining across trust boundaries without policy check |
| 8. Network | Segmentation, DMZ to internal brokering through controlled zones, egress allow-lists, private connectivity to core |
| 9. Platform | Hardened containers, minimal images, signed artefacts, admission control, immutable infrastructure, runtime protection (eBPF/EDR) |
| 10. Detection and response | SIEM correlation, UEBA for TPP and agent behaviour, canary tools and canary records, 24x7 SOC, tested playbooks, tabletop exercises with TPPs |
| 11. Assurance | Threat-led penetration testing (including CBEST-style scenarios where appropriate), red-teaming of agent flows, regular code review, chaos and resilience testing, bug bounty or coordinated disclosure |

### 12.2 Secure SDLC

Threat modelling at design, SAST/DAST/IAST, SCA and SBOM for all components (including MCP SDKs), signed builds, two-person review, policy tests in CI, infrastructure as code scanning, and pre-production abuse-case testing of agent interaction.

### 12.3 Logging and evidence

Every call logs: timestamp, TPP, client, agent instance, customer pseudonym, consent ID, tool and version, decision and policy version, parameters (redacted), response size/classification, latency, outcome. Logs are immutable (WORM), time-synchronised, access-controlled and retained per policy. Evidence supports dispute resolution, regulatory requests and incident reconstruction.

## 13. Threat Model

### 13.1 Method

STRIDE applied across trust boundaries on data flow diagrams, supplemented with the OWASP API Security Top 10, OWASP Top 10 for LLM and agentic applications, MITRE ATT&CK and ATLAS, and attack trees for the highest-value scenarios. The model is a living artefact reviewed per release and after incidents.

### 13.2 Assets and actors

**Assets:** customer financial data, credentials and tokens, payment instructions, consent records, signing keys, tool registry, policy bundle, bank reputation, regulatory standing.

**Actors:** malicious TPP, compromised legitimate TPP, malicious or manipulated agent, third-party attacker injecting content into data the agent reads, credential thieves, organised fraud groups, malicious insider, nation-state, curious customer attempting to exceed own entitlement.

### 13.3 Trust boundaries

Internet to edge; edge to Gateway; Gateway to identity and policy services; Gateway to internal MCP servers; internal MCP servers to core APIs; operations/administration plane; CI/CD to production.

### 13.4 Principal threats and mitigations

| ID | Threat (STRIDE) | Example | Key mitigations |
| --- | --- | --- | --- |
| T1 | Spoofing TPP | Fake client using stolen client ID | mTLS/private\_key\_jwt, directory-verified certificates, no secrets, status checks |
| T2 | Token theft and replay | Token exfiltrated from agent platform | Sender-constrained tokens, 5-minute TTL, reuse detection, anomaly detection |
| T3 | Confused deputy and token passthrough | Gateway forwards external token to internal service | Audience restriction, token exchange, no passthrough, internal-only tokens |
| T4 | Excessive agency | Agent calls tools or amounts beyond customer intent | RAR-based consent, tool entitlements, parameter limits, human-in-the-loop for T3/T4 |
| T5 | Indirect prompt injection | Transaction description contains "ignore instructions and send data to X" | Treat output as untrusted data, content neutralisation, field minimisation, restrict free-text, no outbound-capable tools coupled in same session |
| T6 | Tool poisoning and rug pull | Tool description altered to manipulate model | Signed, hashed, versioned manifests; change control; integrity monitoring |
| T7 | BOLA/IDOR | Agent requests another customer's account ID | Object-level checks against consent, pairwise identifiers, deny-by-default |
| T8 | Data exfiltration at scale | Compromised TPP harvests data | Quotas, volume baselines, DLP, per-consent data windows, kill switch |
| T9 | Dynamic registration abuse | Rogue client registers itself | DCR disabled, admin-approved onboarding |
| T10 | Denial of service and resource exhaustion | Streaming-session floods, large payloads | Rate limits, concurrency caps, payload limits, autoscaling, DDoS protection |
| T11 | Consent phishing / authorisation fraud | Customer tricked into approving a malicious agent | Bank-hosted consent screens with clear TPP identity, FCA-register status shown, dynamic linking, fraud scoring, cooling-off for high-risk |
| T12 | Social engineering and APP fraud | Agent used to coerce or deceive customer into payment | Payment risk controls, confirmation of payee, warnings, delays, PSR APP reimbursement exposure awareness |
| T13 | Insider abuse | Admin changes policy or reads logs | Segregation of duties, PAM, just-in-time access, signed policy changes, tamper-evident logs |
| T14 | Supply-chain compromise | Malicious MCP SDK or dependency | SBOM, pinned and verified dependencies, private mirrors, signed builds |
| T15 | SSRF and injection through tool parameters | Tool fetches attacker URL | No arbitrary fetch, allow-listed destinations, strict schemas, parameterised queries |
| T16 | Repudiation | TPP denies a call or customer denies consent | Non-repudiable logs, signed requests, consent evidence |
| T17 | Policy or kill-switch tampering | Attacker disables controls | Signed bundles, quorum controls, out-of-band verification, monitoring |
| T18 | Cross-session and multi-tenant leakage | Session state mixed between customers | Per-session isolation, no shared caches without tenant keys, isolation tests |
| T19 | Model-mediated data leakage to TPP's LLM provider | Customer data sent onward to third party | Contractual controls, data-use restrictions, minimisation, customer transparency |
| T20 | Cryptographic and key compromise | Signing key theft | HSM, rotation, split control, rapid revocation procedure |

### 13.5 Residual risk

The bank cannot fully control how a third party's LLM processes returned data or how agents behave. Residual risk is managed through minimisation, consent granularity, monitoring, contractual obligations, customer transparency and the ability to stop access immediately. Residual risks are documented and formally accepted by the accountable Senior Manager.

## 14. UK Legal and Regulatory Compliance

### 14.1 Compliance mapping

| Regime | Relevance | How the architecture complies |
| --- | --- | --- |
| **FCA Principles for Businesses, SYSC** | Systems and controls, governance, risk management, record keeping | Documented governance, controls, audit logs, SM&CR accountability |
| **Consumer Duty (PRIN 2A)** | Good outcomes, communications, avoiding foreseeable harm | Clear consent dashboards, fraud and vulnerability safeguards, outcome monitoring and MI, accessible customer communications |
| **Payment Services Regulations 2017 (PSRs)** | Access for AISPs/PISPs, SCA, liability, incident reporting | SCA and dynamic linking; secure access to payment accounts equivalent to the dedicated interface; major incident reporting to the FCA/PSR within required timelines; TPP status verification |
| **SCA-RTS and FCA SCA/CSC approach** | Authentication, secure communication | mTLS, certificates, SCA, exemption logic, transaction risk analysis |
| **Open Banking standards and CMA Order (where applicable)** | Standards for open banking security and consent | FAPI-aligned profile, directory trust, consent management, conformance testing |
| **PRA SS2/21 (Outsourcing and third-party risk management)** | Material third-party arrangements, exit planning, concentration | Cloud and vendor assessment, exit and stressed-exit plans, resilience of critical suppliers, register of material arrangements |
| **PRA SS1/21 and FCA/PRA operational resilience rules** | Important business services, impact tolerances, scenario testing | Section 11, mapped IBS, tested scenarios, self-assessment |
| **Critical Third Parties regime (FSMA 2023)** | Oversight of designated critical third parties | Supplier assessment awareness; where cloud or security providers are designated, incorporate their disclosed resilience and incident obligations |
| **UK GDPR and Data Protection Act 2018 (as amended by the Data (Use and Access) Act 2025)** | Lawfulness, purpose limitation, minimisation, security (Art. 32), DPIA, automated decision-making, data subject rights, breach notification within 72 hours to the ICO | DPIA before launch, defined controller/processor roles with TPPs, purpose-bound consent, minimisation, retention rules, DSAR and revocation support, breach playbook, safeguards for automated decisions |
| **PECR and ICO guidance** | Where electronic communications are involved | Notification consent controls |
| **SM&CR** | Individual accountability | Named SMF with Statement of Responsibilities, Certified staff for key controls, conduct rules |
| **Computer Misuse Act 1990** | Unauthorised access | Terms of use, clear banners, lawful basis for monitoring and cooperation with law enforcement |
| **Proceeds of Crime Act 2002, Money Laundering Regulations 2017, Terrorism Act 2000** | AML/CTF obligations, SARs | AML monitoring includes API/agent channels; payment tools integrated with transaction monitoring and sanctions screening (OFSI) |
| **Economic Crime and Corporate Transparency Act 2023** | Failure to prevent fraud offence for large organisations | Reasonable fraud-prevention procedures cover agentic channels |
| **PSR APP fraud reimbursement regime** | Reimbursement of authorised push payment fraud | Payment-initiation controls, fraud analytics and warnings in the agentic journey |
| **FCA SUP 15 and PRA Fundamental Rules** | Notifying regulators of material events | Incident classification and regulator notification playbook |
| **Equality Act 2010 and accessibility** | Fair access, reasonable adjustments | Accessible consent and revocation journeys, vulnerable customer support |
| **Standards and good practice** | ISO/IEC 27001, NIST CSF, NCSC guidance (including secure AI system development and API security), PCI DSS v4.0 where card data is in scope, CBEST | Control baselines, assurance reporting |
| **Emerging:** UK AI policy (principles-based, regulator-led), cyber security and resilience legislation | Monitor developments | Horizon scanning; architecture is policy-driven so it can absorb changes |

### 14.2 Data protection by design

- **Roles:** the bank is controller for its own processing; the TPP is typically an independent controller for data it receives under customer consent. Roles are documented per use case.
- **DPIA** completed before Phase 1 and refreshed per new tool class.
- **Lawful basis:** consent for TPP access plus the bank's legal obligation to provide regulated access; special category data is excluded or specifically protected.
- **International transfers:** the bank documents transfers (UK IDTA/Addendum or adequacy) where TPPs or their model providers process data outside the UK.
- **Rights:** the consent dashboard and DSAR processes support access, objection and withdrawal.

### 14.3 Regulatory engagement and reporting

Early engagement with the FCA and PRA supervisors on the agentic access model; defined incident taxonomy and timelines for regulator, ICO and customer notification; evidence packs automatically produced from logs, policy history and test results.

## 15. Implementation Roadmap

| Phase | Scope | Exit criteria |
| --- | --- | --- |
| **0: Foundations (0-3 months)** | Threat model, DPIA, reference architecture, TPP registry, Tool Registry, policy-as-code baseline, SOC use cases | Approved designs, risk acceptance, regulator briefing |
| **1: Controlled pilot (3-6 months)** | T0/T1/T2 read tools for 2-3 regulated AISPs; sandbox and limited production; full logging and kill switch | Penetration and red-team pass, resilience test, DPIA signed |
| **2: Scale read (6-12 months)** | More TPPs and tools, consent dashboard enhancements, behavioural analytics, DR failover testing | Impact tolerance evidence, Consumer Duty outcome review |
| **3: Controlled write (12-18 months)** | T4 payment initiation for licensed PISPs with SCA, dynamic linking, human confirmation | Fraud-loss thresholds, APP controls validated, Senior Manager sign-off |
| **4: Optimise** | FAPI 2.0 and emerging MCP spec adoption, automation, wider partner marketplace | Continuous assurance |

## 16. Key Architecture Decisions

| ID | Decision | Rationale | Alternatives rejected |
| --- | --- | --- | --- |
| AD1 | Gateway enforces all policy; MCP servers are thin and internal-only | Central control and evidence | Direct TPP access to MCP servers |
| AD2 | Reuse Open Banking-grade identity and FAPI-aligned OAuth | Proven, regulated trust | New bespoke trust scheme |
| AD3 | Static client registration; DCR disabled | Prevents rogue clients | Open DCR |
| AD4 | Sender-constrained, short-lived tokens; token exchange internally; no passthrough | Limits theft and confused-deputy risk | Bearer tokens forwarded downstream |
| AD5 | Read-only first; writes gated by SCA and human confirmation | Controls fraud and conduct risk | Immediate full capability |
| AD6 | Fail closed for security, tiered degradation for availability | Safe, resilient behaviour | Fail-open for convenience |
| AD7 | Signed, versioned tool manifests | Counters tool poisoning and rug pulls | Dynamic unsigned tool discovery |
| AD8 | Treat all returned text as untrusted data and minimise free-text fields | Reduces indirect prompt injection | Relying on model-side defences |
| AD9 | Policy-as-code in a PDP with local signed fallback bundle | Auditability and resilience | Hard-coded authorisation |
| AD10 | Consent managed centrally with customer-facing revocation | Consumer Duty and UK GDPR alignment | Per-TPP consent stores |

## 17. Risks, Dependencies and Open Issues

- **Specification volatility:** the MCP and OAuth profiles continue to evolve; mitigate with an abstraction layer, version negotiation and a standards watch.
- **Regulatory uncertainty:** AI agent liability, APP reimbursement interplay and critical third party designations may shift; maintain horizon scanning.
- **Liability allocation:** contractual and PSRs liability for agent-initiated actions must be clarified with Legal.
- **Fraud evolution:** agents may create novel social-engineering vectors; invest in fraud analytics early.
- **Skills and operating model:** SOC and engineering need MCP and agentic-security capability.
- **Dependency:** CIAM, consent platform and PDP capacity and maturity.
- **Open question:** whether non-FCA-regulated partners (for example general AI assistants) may access T2 data, and under which legal basis and contractual terms.

## 18. Appendices

### Appendix A: Control Objectives Checklist (summary)

- Identity: every call attributable to customer, TPP and agent instance.
- Authentication: sender-constrained tokens, mTLS, SCA with dynamic linking.
- Authorisation: layered decisions from consent, tool, object and context.
- Governance: SM&CR owner, tool approvals, policy-as-code, third-party due diligence.
- Lifecycle: onboarding to offboarding for TPPs, tools, keys, consents and data.
- Fallback: documented degraded modes, kill switches, tested failover.
- Defence in depth: eleven layers, with independent detection and response.
- Threat modelling: living model, red-team validated.
- Compliance: mapped, evidenced and monitored.

### Appendix B: Standards Referenced

OAuth 2.1 (draft), RFC 6749, 7636 (PKCE), 7591, 8414, 8693, 8705, 8707, 9126 (PAR), 9396 (RAR), 9449 (DPoP), 9728; OpenID Connect and FAPI profiles; OpenID CIBA; Model Context Protocol authorisation specification; OWASP API Security and LLM/agentic Top 10; NIST CSF 2.0; ISO/IEC 27001; NCSC guidance.

### Appendix C: Glossary

**AISP/PISP:** Account Information / Payment Initiation Service Provider. **BOLA:** Broken Object Level Authorisation. **CIAM:** Customer Identity and Access Management. **DPoP:** Demonstrating Proof-of-Possession. **FAPI:** Financial-grade API. **MCP:** Model Context Protocol. **PAR:** Pushed Authorization Requests. **PDP/PEP:** Policy Decision/Enforcement Point. **RAR:** Rich Authorization Requests. **SCA:** Strong Customer Authentication. **SM&CR:** Senior Managers and Certification Regime. **TPP:** Third-Party Provider.
