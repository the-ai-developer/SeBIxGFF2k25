# BondFlow Blockchain Layer — Architecture & Spec

Regulated, high-performance DLT for fractional corporate bonds in India. Built for trust, compliance, and atomic settlement. 🇮🇳🔐⚡️

## Executive Snapshot
- Purpose: DLT backbone for fractional corporate bonds with end-to-end compliance, auditability, and atomic delivery-versus-payment (DvP).
- Model: Permissioned blockchain + security token (ERC‑1400 family) + regulated custody of underlying bonds + UPI/RTGS cash leg.
- Highlights: KYC-gated transfers, regulator node visibility, private data, automated coupons/redemptions, HSM-backed keys, and strong ops.

## 1) Design Objectives
- Compliance-first: Native KYC/AML enforcement, SEBI visibility, auditable trails aligned with PMLA, DPDP Act, and SEBI circulars.
- Finality & performance: Low-latency, high-throughput execution for T+0/T+1 secondary trading.
- Privacy by design: Data minimization, private transactions/collections, selective disclosure.
- Operability: Clear governance, versioned upgrades, observability, safe rollbacks.
- Interoperability: EVM compatibility; ISO 20022-aligned interfaces to market infra.

## 2) Network Topology & Roles
- Participants
  - Issuer/Originator: Primary bond issuance; interfaces with depositories and RTA.
  - BondFlow Operator: Core nodes; issuance, matching, settlement, corporate actions.
  - Investors (KYC’d): Hold/trade fractional tokens via apps/APIs.
  - Custodian/Trustee: Holds underlying bonds; reconciles with NSDL/CDSL; RTA integration.
  - Regulators (SEBI, etc.): Observer/validating nodes with read/audit privileges.
  - RTA/Depositories (NSDL/CDSL): Maintain demat records for full bonds.
  - Banks/PSPs: UPI/RTGS/NEFT rails; escrow/virtual accounts for DvP.
- Node Types
  - Validator/Ordering nodes: BondFlow + designated FIs.
  - Participant nodes: Issuers, custodians, institutions.
  - Regulator observer nodes: Real-time, read-only, tamper-evident access.

## 3) Platform Choice (Permissioned)
- Why Permissioned
  - Identity-bound access (RBAC/ABAC), deterministic governance.
  - High TPS, sub-second finality, energy-efficient consensus (PoA/IBFT/Raft).
  - Private transactions/collections for sensitive trades and PII minimization.

### Candidate Platforms (Comparison)

| Feature | Hyperledger Fabric | EVM Enterprise (Quorum / Polygon Edge) |
|---|---|---|
| Privacy | Channels + Private Data Collections | Tessera private tx (Quorum), on-chain privacy patterns |
| Dev Ecosystem | Chaincode (Go/Java/Node), modular endorsement | Solidity/EVM, rich tooling and audits |
| Consensus | Raft ordering + endorsement policies | IBFT2/PoA with finality |
| Fit | Complex bilateral privacy, multi-org compartmentalization | Token-centric apps, fast dev velocity |

Recommendation: Pick Fabric if deep bilateral privacy/compartmentalization leads. Choose EVM enterprise if developer velocity and token programmability are top priority.

## 4) Tokenization Model: ERC‑1400 Family
- Why ERC‑1400 (incl. partitions like ERC‑1410)
  - Transfer restrictions: KYC/AML allowlists; rules for residency, investor type, lockups; controller transfers for legal actions.
  - Partial fungibility: Partitions per ISIN/tranche/restriction class.
  - Document anchoring: On-chain hashes of prospectus/terms; off-chain storage (IPFS/S3) with legal retention.
  - Lifecycle hooks: Issue, pause, forced transfer, redemption/cancellation.
- Token Metadata
  - ISIN, coupon, face value, maturity, day-count, payment schedule, rating snapshots, covenants.
  - Partition flags (e.g., restricted_US, QIB_only, locked_until_YYYYMMDD).

## 5) Smart Contract Suite
- BondToken (ERC‑1400)
  - issue/redeem/burn; partition balances; transferWithData; controllerTransfer; document registry.
- Compliance/KYC Oracle
  - Manages allow/deny lists; integrates CKYC, PAN, sanctions (UN/OFAC/SEBI), PEP/adverse media.
  - Consent + audit logs (DPDP alignment).
- Matching & Settlement
  - Atomic DvP for token vs. fiat (UPI/RTGS) via escrow callbacks; future-ready for tokenized cash/CBDC e₹.
  - Supports RFQ/order-book; price–time priority; cancel/replace.
- Corporate Actions
  - Coupon accrual; ex/record dates; TDS handling; retries and exception queues.
  - Maturity redemption, partial calls/buybacks; registry snapshots.
- Governance/Upgrade
  - Role-based controls, timelocked upgrades, multi-sig approvals, emergency pause.

## 6) Core Workflows
- Onboarding & KYC/AML
  - CKYC/PAN; optional Aadhaar offline eKYC (with consent); sanctions/PEP screening; AML risk scoring.
  - Status flags anchored on-chain; PII off-chain.
- Primary Issuance
  - Underlying bond held in NSDL/CDSL; RTA coordination.
  - Fractional tokens minted and distributed post payment confirmation.
- Secondary Trading
  - Orders placed; AI engine matches; pre-trade checks (balances, KYC status, restrictions).
  - DvP Settlement:
    - Option A: UPI/RTGS escrow + real-time status callbacks (seconds).
    - Option B: Tokenized cash/CBDC e₹ (future) for pure on-chain atomic swaps.
- Corporate Actions
  - Record-date snapshots; coupon payout net of TDS; bank reconciliation.
  - Retries for failures; investor detail updates.
- Redemption/Buyback
  - Tokens burned on redemption; principal paid; depository reconciliation.

## 7) Consensus & Finality
- Fabric: Raft ordering; multi-org endorsement policies; deterministic finality.
- EVM Enterprise: IBFT2/PoA for sub-second finality; governed validator set; policy-based offboarding (not economic slashing).

## 8) Privacy, Identity, and Data Protection
- Transaction Privacy
  - Fabric: Channels + PDCs for bilateral/club deals.
  - Quorum: Tessera private payloads; selective disclosure.
  - Optional ZK proofs (e.g., accredited status) without exposing PII.
- Identity & PKI
  - X.509 or DID-based identities bound to KYC; short-lived certs; mTLS everywhere.
- DPDP Alignment
  - Consent capture; purpose limitation; data minimization; auditable access logs.
  - PII off-chain in encrypted stores; on-chain hashes/status only.

## 9) Integration with Indian Financial Infrastructure
- NSDL/CDSL & RTA
  - Underlying demat bonds remain at depositories; BondFlow acts as regulated intermediary/custodian.
  - Daily reconciliation; exception management; RTA-triggered corporate action events.
- Payment Systems
  - UPI for retail/real-time (incl. mandates/AutoPay where applicable); RTGS/NEFT for high-value.
  - Bank-side escrow/virtual accounts; idempotent callbacks for DvP orchestration.
- Regulatory Nodes
  - SEBI/others operate observer nodes; real-time surveillance dashboards; exportable audit trails.

## 10) Security Posture
- Cryptography
  - SHA‑256/Keccak; ECDSA/EdDSA; AES‑256 at rest; TLS 1.2/1.3 in transit.
- Key Management
  - HSM-backed keys (FIPS 140‑2/3); envelope encryption; rotation; quorum ops for admin keys.
- AppSec & Audits
  - Independent smart contract audits; SAST/DAST/SCA in CI/CD; SBOM and dependency pinning.
- Access Control
  - Fine-grained RBAC/ABAC; JIT access; MFA; PAM for privileged ops.
- Threat Detection
  - SIEM + anomaly detection; on-chain analytics for wash trading/layering/suspicious flows.

## 11) Resilience & Operations
- Availability
  - Active–active clusters; multi-zone; ordering redundancy; RPO ≤ 5 min, RTO ≤ 30 min.
- Backups & Recovery
  - Encrypted snapshots; ledger checkpointing; DR/failover drills.
- Observability
  - Metrics/logs/traces; SLOs (latency, TPS, settlement success); synthetic trade probes.

## 12) Compliance, Reporting, and Audit
- Compliance Engine
  - Rule packs per investor class/jurisdiction/lockup; change-controlled, versioned policies.
- Reporting
  - Regulator extracts; ISO 20022-aligned event feeds; TDS certificates; GST/fee ledgers where applicable.
- Audit
  - Immutable event logs; evidentiary timestamps; Merkle-proof backed auditability.

## 13) Performance & Sizing (Indicative)
- Throughput: 1,000–3,000 TPS steady-state on mid-range infra; higher with horizontal scale/block-time tuning.
- Latency: Sub-second chain finality (IBFT2); end-to-end DvP gated by bank callbacks (typically seconds).
- Data: Partition by ISIN/tranche; archival policies for cold data; off-chain docs with on-chain hash anchors.

## 14) Governance
- Consortium Charter
  - Membership criteria; validator onboarding/offboarding; incident response runbooks.
- Change Management
  - Timelocked upgrades; multi-sig approvals; staging–mainnet separation; rollback playbooks.

## 15) Roadmap
- Phase 1: Sandbox/POC — single issuer, select investors, UPI escrow DvP, SEBI observer node.
- Phase 2: Pilot — multiple issuers/tranches, RTA automation, coupon automation, analytics dashboards.
- Phase 3: Scale — broader access, deeper bank integrations, CBDC e₹ cash-leg pilot, cross-market interoperability.

## 16) Key Risks & Mitigations
- Payment Leg Sync Risk
  - Mitigation: Escrow with real-time status, idempotent callbacks, reconciliation watchdogs.
- Smart Contract Bugs
  - Mitigation: Audits, formal specs for critical paths, circuit breakers, phased rollouts.
- Privacy Leakage
  - Mitigation: Private tx/collections, data minimization, redaction policies.
- Governance Disputes
  - Mitigation: Consortium legal charter, on-chain policy registry, arbitration procedures.
