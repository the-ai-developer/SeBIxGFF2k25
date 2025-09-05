# BondFlow Security Architecture — Zero‑Trust, Defense‑in‑Depth, Evidence‑Grade

Multi-layered controls across edge, app, data, blockchain, identity, and ops. Built for confidentiality, integrity, availability — and provable auditability.

## Executive Snapshot
- Trust model: Zero‑trust, least‑privilege, assume‑breach, verify‑everything
- Architecture: Segmented VPCs, service‑mesh mTLS, hardened K8s, signed supply chain, HSM/KMS keys, and SIEM‑driven detection
- Compliance anchors: SEBI cyber expectations, CERT‑In 2022 directions, DPDP Act 2023, ISO 27001, NIST CSF/800‑53 mapping
- Resilience: Multi‑AZ active‑active; RPO ≤ 5 min, RTO ≤ 30 min; immutable, tamper‑evident logging; DR drills

---

## 0) Layered Security Blueprint
```mermaid
flowchart LR
  A[Users & Admins] -->|HTTPS/TLS1.3| B[CDN + WAF + DDoS]
  B --> C[API Gateway]
  C --> D[Service Mesh (mTLS, AuthZ, Telemetry)]
  D --> E[Microservices (K8s Restricted)]
  E --> F[(Databases: Postgres/Redis/ClickHouse)]
  E --> G[(Streams: Kafka)]
  E --> H[(Object Store / Lake)]
  E --> I[Blockchain Writer/Endorser]
  I --> J[Validators/Ordering (Permissioned)]
  J --> K[SEBI Observer Node]
  subgraph Security & Ops
    L[SIEM/XDR/UEBA]
    M[KMS/HSM + Vault]
    N[DLP + CASB]
    O[OPA/Gatekeeper Policies]
    P[EDR on Hosts]
  end
  F --> L
  G --> L
  H --> L
  E --> L
  B --> L
  M --> F
  M --> E
  M --> I
```

---

## 1) Network Security Layer

### 1.1 VPC Segmentation
- Isolated per environment (Prod/Stage/Dev) across separate cloud accounts/projects
- Private subnets for DBs, blockchain nodes, Kafka; public subnets only for edge
- VPC endpoints/PrivateLink to managed services; NAT egress with explicit allow‑lists
- Egress control: deny‑all egress by default; proxy‑based outbound with URL/IP allow‑lists

Benefit: Hard isolation, minimized blast radius, tight egress governance.

### 1.2 Security Groups & NACLs
- Principle: Default‑deny inbound/outbound; least‑privilege per flow
- SGs tied to workload identity (tags), not IPs; NACLs to block broad CIDR ranges

Benefit: Enforces strict east‑west, north‑south controls and stops lateral movement.

### 1.3 Edge Protection: WAF + DDoS
- L3/L4: Provider DDoS (e.g., AWS Shield Advanced) + Anycast scrubbing
- L7: Managed WAF rules (OWASP Top 10), custom rules (rate‑limit, geo/IP reputation, bot mgmt)
- HSTS, TLS 1.2/1.3, modern ciphers; OCSP stapling; forced HTTPS

Benefit: Blocks volumetrics and app‑layer abuse before it hits origin.

### 1.4 IDS/IPS & Threat Intel
- Network IDS (Suricata/Snort) + Cloud-native detections (GuardDuty/Cloud IDS)
- eBPF telemetry for anomalous syscalls; curated threat intel feeds

Benefit: Early detection of scanning, exfil, and C2 patterns.

### 1.5 Secure Remote Access (ZTNA)
- Admin access via ZTNA/IdP (SAML/OIDC) or SSM Session Manager; MFA+FIDO2
- No public SSH/RDP; Just‑in‑Time (JIT) ephemeral credentials; audited sessions

Benefit: Strong admin control without flat VPN risks.

---

## 2) Application Security Layer

### 2.1 SSDLC (Shift-Left)
- Threat modeling (STRIDE/PASTA), security stories in backlog, security champions
- SAST/Secrets scan pre‑merge; DAST/IAST per release; dependency pinning (SCA)
- Fuzzing for parsers/critical APIs; mandatory peer review with secure coding checklists

### 2.2 API Security
- OAuth 2.1 + OIDC; PKCE for public clients; short‑lived access tokens; refresh rotation
- JWT with signed (ES256/RS256) tokens; jti, aud, exp, kid; key rotation via JWKS
- Fine‑grained scopes; rate limits per user/IP/app; strict input validation and output encoding
- GraphQL (if used): disable introspection in prod; query depth/complexity limits

### 2.3 AuthN/Z & Session Mgmt
- MFA everywhere; FIDO2/WebAuthn passkeys preferred; SMS only as fallback
- RBAC/ABAC + SoD; service‑to‑service mTLS with SPIFFE/SPIRE identities
- Web sessions via httpOnly, Secure, SameSite=strict cookies; CSRF tokens for state‑changing ops
- Device binding and anomaly‑based session revocation

### 2.4 Web/Mobile Hardening
- CSP, HSTS, X‑Frame‑Options, Referrer‑Policy, X‑Content‑Type‑Options
- File uploads: content‑type validation + AV scan + sandbox conversion
- Mobile: certificate pinning, root/jailbreak detection, secure keystore (StrongBox/Keychain), anti‑hooking, MASVS compliance

---

## 3) Data Security Layer

### 3.1 Encryption In Transit
- TLS 1.2/1.3; internal mTLS via service mesh; certificate rotation automated
- Strict cipher policy; TLS downgrade protection; ALPN/HTTP2 hardened

### 3.2 Encryption At Rest
- Envelope encryption with KMS/HSM; per‑tenant/per‑dataset DEKs; automated rotation
- External key mgmt (XKS/BYOK) option; dual control for key ceremonies

### 3.3 Data Classification & Minimization
- Classes: Public / Internal / Confidential / Restricted
- PII tokenization and field‑level encryption (deterministic for joinable fields)
- Non‑prod data fully masked/synthesized; no raw Aadhaar/PAN in logs

### 3.4 DLP & Egress Controls
- DLP for email/storage/endpoints; secrets/PII detectors in repos/logs
- Egress proxy with inspection; CASB for SaaS access governance

---

## 4) Blockchain Security Layer

### 4.1 Cryptography & Standards
- EVM chains: secp256k1 + keccak256; Fabric: ECDSA/Ed25519 + SHA‑256
- Domain‑separated hashes for signatures; replay protection

### 4.2 Consensus & Node Security
- Permissioned validators (5–11) across institutions; IBFT2/Raft with finality
- Nodes in private subnets; firewall allow‑lists; mTLS between peers; static peering
- Remote attestation (where available: Nitro Enclaves/Confidential VMs) for key ops

### 4.3 Smart Contract Assurance
- Multiple independent audits; Slither/Mythril/Manticore static+sym analysis
- Property‑based tests, invariants; formal methods (Scribble/Certora/KEVM) for critical logic
- Upgrades via time‑locked proxies; multi‑sig approvals; emergency pause; minimal trusted controllers

### 4.4 Key Management (Wallets/Signers)
- HSM/CloudHSM custody; quorum approvals; MPC/TSS for hot workflows; cold storage for master keys
- Transaction policies: per‑address limits, velocity caps, anomaly checks before signing

---

## 5) Identity, Access, and Secrets

- Central IdP (Okta/Azure AD): SSO (OIDC/SAML), SCIM provisioning, conditional access, device posture checks
- RBAC/ABAC across cloud/K8s/DB; SoD enforced; “break‑glass” accounts vaulted and monitored
- PAM for privileged DB/OS access; session recording
- Secrets: Vault + KMS; dynamic DB creds; short TTL service tokens; secret zero via workload identities (IRSA/Workload Identity)

---

## 6) Cloud & Kubernetes Hardening

- Baselines: CIS Benchmarks (Cloud/K8s/OS), STIG‑aligned hardening
- K8s security:
  - Pod Security Admission: restricted; drop ALL capabilities; seccomp=runtime/default; AppArmor; read‑only root FS; non‑root UIDs; no hostPath/privileged
  - NetworkPolicies (Calico/Cilium) default‑deny; egress policies enforced
  - Admission control: OPA/Gatekeeper policies (no :latest tags, image signing required)
  - Image supply chain: distroless/minimal; Cosign/Sigstore signing; provenance (SLSA lvl ≥2)
- Cloud controls:
  - IMDSv2 only; no public S3/GCS; bucket policies with TLS/SSL enforced
  - Tight IAM: least‑privilege roles, access analyzer, permission boundaries
  - Config/Drift detection: Config/Policy service with auto‑remediation

---

## 7) Supply Chain Security (CI/CD)

- Signed commits (GPG/Sigstore keyless); mandatory code reviews; dependency pinning
- SBOM (CycloneDX/SPDX) generated per build; artifact signing; provenance attestations
- Container/image scanning (OS + libs) per build and in‑registry; block on critical CVEs without exceptions
- Secrets scanning (pre‑commit and CI); rotate if leaked; repo‑level DLP

---

## 8) Security Operations, Monitoring, and Detection

- SIEM/XDR: Centralized telemetry (CloudTrail, VPC Flow, K8s audit, app logs, EDR, WAF, blockchain events)
- Detections mapped to MITRE ATT&CK; UEBA for user/service anomalies
- On‑chain analytics: mempool anomalies, validator health, unusual transfer patterns
- Alerting: Multi‑burn‑rate SLO alerts; runbooks attached; automated enrichment (asset owner, priority, IOC context)
- Threat hunting & purple teaming quarterly; tabletop exercises biannually

---

## 9) Vulnerability & Patch Management

- Continuous vuln scanning (internal/external); authenticated scans of hosts/containers
- Patch SLAs: Critical ≤ 7 days; High ≤ 15 days; Medium ≤ 30 days (exceptions time‑bound, risk‑accepted)
- Third‑party library risk: SCA with license compliance; auto PRs for safe upgrades
- Annual external penetration testing; targeted red teaming for critical paths

---

## 10) Data Logging, Forensics, and Evidence

- Structured logs with correlation IDs, no PII; sensitive fields redacted at source
- Time sync via NTP; monotonic clocks; log immutability (WORM/S3 Object Lock)
- Evidence anchoring: periodic Merkle roots of logs anchored on permissioned chain
- Forensics: snapshot playbooks, chain‑of‑custody, memory capture for critical nodes

---

## 11) Incident Response (IR), BCP, and DR

- IR lifecycle: Detect → Triage → Contain → Eradicate → Recover → Lessons learned
- Sev matrix with comms plan (internal, regulator, customers); no‑blame postmortems
- DR: Cross‑AZ HA; warm secondary region; encrypted backups, restore drills; RPO ≤ 5 min, RTO ≤ 30 min
- CERT‑In & SEBI reporting timelines integrated into IR runbook

---

## 12) Third‑Party and Vendor Risk

- Due diligence: ISO 27001/SOC 2 preferred; DPDP‑aligned DPAs; data flow maps
- Access boundary: least‑privilege, tenant isolation, audit logging for partner access
- Continuous monitoring: Attack surface (EASM), expiring certs/DNS hygiene, leaked creds scans
- Bug bounty + VDP: safe harbor, coordinated disclosure

---

## 13) Controls Mapping (Quick View)

| NIST CSF | BondFlow Controls |
|---|---|
| Identify | Asset inventory, data classification, SBOM, vendor registry |
| Protect | WAF/DDoS, mTLS, IAM/RBAC/ABAC, HSM/KMS, Vault, PSP‑restricted K8s, CSP/HSTS |
| Detect | SIEM/XDR, IDS/IPS, UEBA, on‑chain analytics, drift detection |
| Respond | IR runbooks, auto‑isolation (quarantine), comms plan, CERT‑In/SEBI reporting |
| Recover | DR failover, tested backups, config-as-code restore, postmortems |

ISO 27001 Annex A coverage includes A.5–A.9 (policy/roles), A.8 (asset), A.9–A.12 (access/crypto), A.14 (ops), A.15 (supplier), A.16 (incident), A.17 (BCP), A.18 (compliance).

---

## 14) Sample Configs & Policies

### 14.1 Kubernetes NetworkPolicy (default‑deny + allow mesh)
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: trading
spec:
  podSelector: {}
  policyTypes: ["Ingress","Egress"]
  ingress: []
  egress: []
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-mesh
  namespace: trading
spec:
  podSelector:
    matchLabels:
      app: trading-svc
  ingress:
    - from:
        - namespaceSelector:
            matchLabels: { istio-injection: enabled }
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - namespaceSelector:
            matchLabels: { kafka: core }
      ports:
        - protocol: TCP
          port: 9092
```

### 14.2 OPA/Gatekeeper: deny privileged pods
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPPrivilegedContainer
metadata:
  name: disallow-privileged
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```

### 14.3 Content Security Policy (CSP)
```http
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-<rand>'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; connect-src 'self' https://api.bondflow.in; frame-ancestors 'none'; base-uri 'self';
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Referrer-Policy: no-referrer
```

### 14.4 KMS Key Policy (grant to signer role)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {"Sid":"AllowRoot","Effect":"Allow","Principal":{"AWS":"arn:aws:iam::ACCOUNT:root"},"Action":"kms:*","Resource":"*"},
    {"Sid":"Signers","Effect":"Allow","Principal":{"AWS":"arn:aws:iam::ACCOUNT:role/blockchain-signer"},"Action":["kms:Encrypt","kms:Decrypt","kms:GenerateDataKey*","kms:Sign","kms:Verify"],"Resource":"*","Condition":{"Bool":{"kms:GrantIsForAWSResource":"true"}}}
  ]
}
```

### 14.5 Vault Dynamic DB Creds (Postgres)
```hcl
path "database/roles/trading-app" {
  capabilities = ["read"]
}
# Role definition (rotation, TTL)
db_name   = "pg-prod"
creation_statements = "CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT,INSERT,UPDATE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";"
default_ttl = "1h"
max_ttl     = "4h"
```

---

## 15) KPIs & Guardrails

- Security SLOs: Critical vulns > 7 days = 0; phishing failure rate < 3%; privileged session approvals within 15 min
- Detection: MTTD < 15 min high‑fidelity alerts; MTTR < 4 hours Sev‑1
- Hardening: 100% workloads with mTLS, image signing, restricted PSP; > 95% infra as code coverage
- Privacy: 0 PII in logs; ≥ 99% successful key rotations on schedule

---

## 16) Implementation Roadmap

- Phase 1 (Baseline): WAF/DDoS, VPC seg, mTLS mesh, IAM hardening, Vault+KMS, SIEM ingest, CSP/HSTS, CIS baselines
- Phase 2 (Assurance): OPA/Gatekeeper, image signing + SBOM, fuzzing, external pen test, MDR/XDR, IR tabletops
- Phase 3 (Advanced): MPC/TSS signers, confidential compute for key ops, formal verification for critical contracts, ZK attestations for privacy
- Phase 4 (Operational Excellence): Automated drift remediation, purple team exercises, evidence anchoring pipelines, continuous control monitoring

---

## 17) Quick “Do Not” List (Safety Rails)

- No public admin endpoints; no long‑lived static creds; no shared accounts
- No PII in logs/screenshots/tickets; no real data in non‑prod
- No container privileges; no hostPath; no :latest images; no unsigned images
- No off‑exchange P2P transfers; no uncontrolled egress
