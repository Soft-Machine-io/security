# Security Policy

Soft Machine operates a Virtual Machine–based agentic development environment.
Each user workspace runs as an isolated Firecracker microVM, brokered by a
control plane that handles authentication, workspace orchestration, billing,
and model inference. The platform executes untrusted user code as a core
function of the product; tenant isolation and platform integrity are therefore
primary security objectives, and we actively welcome coordinated disclosure
from the security research community.

This document defines what is in scope, what is not, how to report a
vulnerability, our response commitments, severity expectations, bounty
eligibility, and the legal safe harbor we extend to good-faith research.

**Policy version:** 1.0
**Effective date:** 2026-05-16
**Last reviewed:** 2026-05-16

---

## Reporting a Vulnerability

**Please report all vulnerabilities privately through GitHub Security
Advisories:**

➡️ **https://github.com/Soft-Machine-io/security/security/advisories/new**

Browse previously coordinated advisories here:
https://github.com/Soft-Machine-io/security/security/advisories

Do **not** open public GitHub issues, pull requests, or discussions for security
problems, and do not disclose the issue publicly until coordinated disclosure is
complete (see timeline below). Public issues on this repository are disabled for
exactly this reason.

A good report includes:

- The affected asset (domain, endpoint, component, or image) and version/commit
  if known.
- A clear description of the vulnerability and its security impact.
- Step-by-step reproduction instructions and a minimal proof of concept.
- Any relevant logs, request/response captures, screenshots, or payloads.
- Your assessment of severity and realistic exploitation preconditions.
- A CVSS v3.1 vector and score estimate, if you have one.
- Whether you believe the issue is being actively exploited in the wild,
  and any indicators supporting that belief.

If you believe you have found an actively exploited issue, or one that exposes
other users' data or workspaces, state this prominently at the top of the
report so we prioritize triage immediately.

---

## Scope

Soft Machine's production surface spans a Vercel-hosted frontend, a Fly.io
control-plane API, per-user Firecracker workspace microVMs, a Cloudflare Worker
port proxy, Supabase Postgres, and Cloudflare R2 object storage. The following
are explicitly in scope.

### In scope

**Frontend (Vercel)**

- `https://soft-machine.io` and `https://www.soft-machine.io`
- `https://soft-machine.vercel.app`
- Alias domains serving the same application:
  `https://wolframgraphs.com`, `https://wolframphysics.app`,
  `https://softmachine.app` (and `www.` variants)
- The client single-page application, including its security-relevant
  client-side logic (session/token handling, attestation, the
  `security-wasm` module, CSP/Trusted-Types enforcement)

**Control-plane API (Fly.io — `api.soft-machine.io`)**

- Authentication and session endpoints under `/auth/*` (WorkOS OAuth,
  session cookie issuance/refresh, anonymous sessions, email verification,
  passkey/WebAuthn, step-up / DPoP)
- CSRF / origin enforcement and session validation middleware
- Workspace and VM orchestration under `/api/cluster/*`
  (provision, status, connect, stop, destroy, heartbeat, and the
  deploy-authenticated admin image endpoint)
- Workspace CRUD, sharing, and collaboration APIs
- Billing endpoints, including the Stripe webhook signature verification
  (`/api/billing/webhook`)
- The model-inference proxy (`/api/inference`, `/api/runpod`), including
  budget/rate-limit enforcement and provider key isolation
- Security/abuse endpoints (`/api/security/csp-report`,
  `/api/security/abuse-event`) and the container-to-control-plane
  authentication path, including the credential it relies on
- File upload handling

**Workspace microVMs (Fly.io — `wss://<vmid>.fly.dev/ws`)**

- The workspace WebSocket server (terminal I/O, file-system operations,
  agent event streaming and replay)
- The agent daemon / agent runner (connection-independent execution,
  cron scheduling, JSONL event buffering)
- **VM isolation boundaries.** We are specifically interested in:
  - Cross-tenant access between workspace VMs (one workspace reading,
    writing, influencing, or observing another)
  - Escape from a workspace VM to the host, the control plane, or
    cloud-provider credentials
  - Authentication/authorization bypass on the workspace WebSocket
    (accessing a workspace you do not own or were not granted)
  - Disclosure of platform-internal material not intended to survive
    workspace initialization, including platform-internal controls and
    detection infrastructure

**Edge / proxy & supporting infrastructure**

- The Cloudflare Worker port proxy
  (`https://<port>-<vmid>.soft-machine.io` → `<vmid>.fly.dev:<port>`),
  including allowed-port enforcement and request smuggling/SSRF through
  the proxy
- Supabase Postgres Row-Level Security: any RLS bypass allowing a user to
  read or modify another user's workspaces, rules, or metadata
- Cloudflare R2 object storage: presigned-URL scoping/TTL issues, path
  traversal in workspace/backup keys, cross-workspace object access
  (`assets.soft-machine.io`)

**Supply chain & deployment**

- The published workspace container image
  (`ghcr.io/lukalotl/workspace`) and its build/signing pipeline
- The deploy authentication mechanism used by CI to update the image SHA
- The Ed25519 audit-log and service-token signing scheme, and the
  honeypot / honey-token / anomaly-scoring controls (bypass or forgery,
  not mere enumeration)

**Vulnerability classes of particular interest**

- Tenant isolation breaks (VM↔VM, VM↔control-plane, RLS, R2)
- Authentication and session flaws (cookie/token theft or forgery,
  CSRF bypass, step-up/DPoP bypass, anonymous→authenticated escalation)
- Server-side request forgery, especially via the inference proxy or
  port proxy
- Remote code execution in the control plane (as distinct from intended
  code execution inside a user's own workspace VM)
- Injection (SQLi against non-RLS paths, command injection in the
  workspace server/agent), insecure deserialization
- Privilege escalation within the control plane or to platform identity
- Secret/credential exposure (provider keys, signing keys, shared
  secrets, R2 credentials)
- Billing manipulation / inference-budget bypass

### Out of scope

The following are **not** eligible. Reports consisting solely of these will be
closed as informational.

- **Intended behavior of the workspace VM.** Workspaces deliberately run
  user-supplied code as root inside the container. Obtaining a root shell,
  running arbitrary binaries, installing tools, or general "RCE in my own
  workspace" is the product working as designed. Only *crossing the VM
  isolation boundary* (to another tenant, the host, or the control plane)
  is in scope.
- **Absence of a specific hardening layer.** Reports that a specific
  hardening layer is absent, without a working cross-tenant or
  host-escape exploit demonstrating concrete impact, are out of scope.
- Missing rate limiting, or rate-limiting bypass, without a concrete
  resulting impact.
- SPF, DKIM, DMARC, BIMI, MTA-STS, or other email-configuration findings;
  email spoofing absent a demonstrated, impactful attack.
- Self-XSS, or attacks requiring the victim to paste attacker-supplied
  content into their own console/terminal.
- Social engineering, phishing, or physical attacks against Soft Machine,
  its staff, users, or infrastructure providers.
- Denial of service, resource exhaustion, volumetric, or load/stress
  testing of any kind. **Do not run DoS or automated high-volume scanning
  against production.**
- Theoretical vulnerabilities, or findings without a working
  proof of concept demonstrating realistic security impact.
- Missing security headers, cookie flags, or TLS/cipher configuration
  without a demonstrated exploit.
- Reports generated solely by automated scanners with no manual
  validation, and version-banner / "outdated software" findings without a
  demonstrated exploit.
- Clickjacking on pages with no sensitive state-changing action.
- Content spoofing or text injection without a demonstrated attack vector.
- Enumeration of honeypot routes, honey tokens, or anomaly-detection
  thresholds. Forging or bypassing the audit-log signing or anomaly
  controls *is* in scope.
- Vulnerabilities in third-party services we depend on but do not control
  (WorkOS, Stripe, Supabase platform, Fly.io platform, Cloudflare,
  Vercel, RunPod, model providers). Report those to the respective vendor.
  Misconfiguration *on our side* of these services is in scope.
- Outdated browsers, or issues only reproducible on end-of-life software.
- Third-party-application deep-link handling of the `soft-machine://`
  protocol absent a concrete exploit.

---

## Severity Guidelines

We assess severity by realistic impact on tenant isolation, the control
plane, and user data. Use this table to self-assess before submitting; it
also reflects how we prioritize triage and bounty determination. Final
severity is set by Soft Machine at triage.

| Severity | Representative impact for Soft Machine |
|---|---|
| **Critical** | Cross-tenant workspace VM escape; workspace VM escape to the host or control plane; remote code execution in the control plane; full Supabase RLS bypass exposing arbitrary tenants' data; compromise of platform signing keys, provider credentials, or the image build/deploy pipeline. |
| **High** | Authentication or session bypass; account takeover; workspace WebSocket authorization bypass granting access to a workspace you do not own; SSRF reaching internal services or cloud metadata; billing or inference-budget bypass with material financial impact. |
| **Medium** | Significant exposure or modification of data within your own tenant boundary that should not be reachable; stored XSS with a realistic victim path; CSRF on a sensitive state-changing action; meaningful integrity loss without cross-tenant reach. |
| **Low** | Limited-impact issues requiring unusual preconditions, significant user interaction, or yielding minimal security value. |

---

## Rules of Engagement

When conducting research against in-scope assets:

- Only ever test against **your own** account, workspaces, and data. Never
  access, modify, or destroy data that does not belong to you. If you
  inadvertently encounter another user's data, stop, do not save it, and
  report it immediately.
- Use your own account and create your own workspace VMs for testing.
  Do not target other tenants' VMs.
- Do not run denial-of-service, volumetric, or high-rate automated
  scanning against production. Throttle aggressively.
- Do not exfiltrate data, pivot to other systems, establish persistence,
  or maintain access beyond what is necessary to demonstrate the issue.
  Use a benign, non-destructive proof of concept.
- You must not attempt to access, exfiltrate, or retain any other user's
  data, workspace contents, or credentials. If you inadvertently encounter
  such data, stop immediately, do not copy or retain it, and notify Soft
  Machine without delay through the advisory channel.
- Do not use honey tokens, decoy credentials, or any credentials you
  discover to access systems beyond confirming the finding.
- Do not disclose before remediation or the agreed coordinated-disclosure
  date, per the timeline below.

---

## Safe Harbor

Soft Machine considers security research and vulnerability disclosure
conducted in accordance with this policy to be authorized, beneficial, and
lawful. We adopt the [disclose.io](https://disclose.io) gold-standard safe
harbor terms:

- We consider security research and vulnerability disclosure activities
  conducted consistent with this policy as **authorized** conduct under the
  Computer Fraud and Abuse Act, the DMCA, and analogous state and
  international laws, and we will not initiate or support legal action
  against you for accidental, good-faith violations of this policy.
- We will not bring a claim against you for circumventing the technological
  measures we use to protect the in-scope applications, to the extent your
  research is conducted consistent with this policy.
- You are expected, as always, to comply with all applicable laws. If legal
  action is initiated by a third party against you for activities that were
  conducted in accordance with this policy, we will make this authorization
  known.
- Soft Machine will not take adverse action against researchers who report
  in good faith under this policy, including account suspension or
  termination, for the research activities themselves.
- If at any time you have concerns or are uncertain whether your research is
  consistent with this policy, **contact us through the advisory channel
  above before going further**, and we will clarify.

This safe harbor applies only to legal claims under the control of Soft
Machine, and only to the extent your activities remain consistent with this
policy. It does not apply to good-faith violations of others' rights, or to
testing performed against assets that are out of scope or owned by third
parties.

---

## Response Targets

We commit to the following service levels for reports submitted through the
GitHub Security Advisory channel:

| Stage | Target |
|---|---|
| **Acknowledgement** of your report | Within **72 hours** |
| **Triage** — validity confirmed, severity assigned, scope determined | Within **7 days** |
| **Bounty determination** | At time of triage, or shortly after |
| **Status updates** during remediation | At least every 14 days |
| **Resolution** target | Risk-based; critical/high prioritized |

We keep you informed at every stage and answer questions about your report or
its status at any time via the advisory thread.

---

## Coordinated Disclosure

We follow a coordinated disclosure model:

1. You report the issue privately via GitHub Security Advisories and refrain
   from public disclosure.
2. We acknowledge, triage, and validate the report within the targets above
   and work with you on remediation, keeping you updated.
3. Once a fix is deployed (or after the disclosure window elapses), we
   coordinate public disclosure with you, including timing and content.
4. We credit you for the finding. Credit is given in the published GitHub
   Security Advisory, in release notes where applicable, and — with your
   consent — on the Hall of Thanks in this repository. If you prefer to
   remain anonymous, tell us and we will honor that; anonymity requests are
   always respected.

**Default disclosure timeline: 90 days** from the date we acknowledge the
report. We work to remediate well before then and publish as soon as a fix is
shipped. If a fix requires more time, we will tell you and agree a revised
timeline with you. We may request expedited (mutually agreed) disclosure for
actively exploited issues, and we ask that you do not publicly disclose before
the agreed date.

### Bounty

Valid reports that meet this policy are eligible for a monetary bounty. Bounty
amounts are determined case by case at triage, based on the confirmed severity
(see Severity Guidelines), the quality of the report, and the realistic impact
of the issue. We have not yet published a fixed reward table; award tiers are
to be determined and decided on a per-report basis until then. Eligibility and
amount are at Soft Machine's reasonable discretion, and we will communicate the
determination to you through the advisory thread.

---

## Questions

For anything that is not itself a vulnerability report — policy
clarifications, scope questions, or safe-harbor concerns — open a private
draft advisory at the link above and label it a question, or reach out
through the channel listed on https://soft-machine.io.

Scope-clarification questions submitted as draft advisories are answered
within the same **72-hour acknowledgement window** that applies to reports.
You do not need to begin testing under scope uncertainty: ask first, and we
will confirm whether your intended target and approach are in scope before
you proceed.

---

## Hall of Thanks

We publicly recognize researchers whose coordinated disclosures have helped
secure Soft Machine and its users. This section will be populated as
advisories are resolved and published. Researchers who request anonymity are
honored and will not be listed.

_No entries yet — be the first._

---

Thank you for helping keep Soft Machine and its users safe.
