# CtxOS: Encryption-First Context Operating System

## 1. Vision
- Macro-context graph of world knowledge combining entities, events, and temporal relations.
- Personal context vault that remains end-to-end encrypted (E2EE).
- Context Composer orchestrating retrieval, grounding, and citation of relevant facts for LLMs.
- Zero-trust security posture: platform cannot inspect personal data at rest or in transit; enclaves gate any server-side computation.

## 2. Security Model
### 2.1 Threat Model
Adversaries include cloud operators, insiders, compromised infrastructure, supply-chain compromises, network attackers, endpoint malware, and prompt-injection attempts during inference. Goals cover confidentiality of personal data and embeddings, integrity of retrieval results, verifiability of sources, revocability of access, and minimizing trust in servers.

### 2.2 Controls
- **Transport:** Mandatory TLS 1.3 with HSTS, CT SCTs, OCSP stapling, and mutual TLS (mTLS) for internal traffic.【F:README.md†L11-L30】
- **Identity:** Passkeys/WebAuthn as default; OPAQUE aPAKE + Argon2id fallback; services authenticate with OAuth 2.1 / OIDC service identities.【F:README.md†L31-L38】
- **Encryption:** Client-side E2EE using per-object DEKs sealed with XChaCha20-Poly1305 and per-user KEKs; HPKE for cross-device and enclave key exchange; macro-context stores may remain plaintext.【F:README.md†L39-L47】
- **Computation:** Personal retrieval performed locally or inside TEEs (AWS Nitro, Azure/GCP Confidential VMs) with attestation-gated key release.【F:README.md†L48-L55】
- **PII Defense:** Optional Microsoft Presidio-based anonymization prior to macro indexing or retrieval rails.【F:README.md†L56-L58】
- **Audit:** Append-only, hash-chained, signed logs exposed to users without ever logging plaintext personal data.【F:README.md†L59-L60】

## 3. Data Architecture
### 3.1 Macro-Context Layer
Sources include Wikidata (via dumps and Query Service), Wikipedia summaries, and Wikidated change history. Storage includes a graph database (Neo4j-class), a vector index (pgvector/Qdrant/Milvus), and a temporal event index. Schema covers entities, relations, events, sources, and text chunks with embeddings.【F:README.md†L62-L87】

### 3.2 Personal Context Layer
Namespaces such as `facts/`, `preferences/`, `timeline/`, `files/`, `notes/`, `email/`, and `calendar/`. Each object uses per-object DEKs sealed with XChaCha20-Poly1305 AEAD binding ciphertext to user id, namespace, and version. Personal embeddings remain encrypted; retrieval is local or inside TEEs using attestation-gated keys.【F:README.md†L88-L101】

## 4. Retrieval and Answering Pipeline
The Context Composer parses intent, traverses the graph, performs macro vector retrieval, runs personal retrieval (local/TEE), cross-encoder reranks, and packs minimal sufficient context with citations and timestamps before LLM answering. The system abstains if coverage is weak, measured with RAGAS and RAG-HAT evaluations.【F:README.md†L103-L111】

## 5. Cryptography
- TLS 1.3 for transport with HSTS, OCSP stapling, CT, and mTLS.【F:README.md†L113-L114】
- XChaCha20-Poly1305 AEAD for personal data at rest.【F:README.md†L115-L116】
- HPKE (RFC 9180) with X25519-HKDF and ChaCha20/AES AEAD suites for key wrapping and sharing.【F:README.md†L117-L118】
- OPAQUE (RFC 9807) + Argon2id (RFC 9106) for password fallback flows.【F:README.md†L119-L120】
- AES-GCM as hardware-accelerated alternative per NIST SP 800-38D.【F:README.md†L121-L122】

## 6. Enclave Key-Release Flow
Enclaves boot signed images, produce attestation reports, and only after remote attestation validates image hashes and policies do KMS release DEKs. Enclaves decrypt personal indexes, run ANN, and return limited results without exfiltrating raw documents.【F:README.md†L124-L131】

## 7. Compliance (EU Focus)
Design adheres to GDPR Articles 5, 25, and 22 with consent-driven personal indexing, E2EE defaults, minimal retention, and ensuring answers remain assistive unless explicit consent covers automated decisions.【F:README.md†L133-L138】

## 8. Repository Layout
```
/ctxos
  /codex-cli
  /ingestor
  /graph
  /vector
  /personal-client
  /personal-enclave
  /composer
  /answering
  /pii
  /audit
  /infra
  /eval
```
Languages: CLI/client SDK in Rust or Go; services in Go/Python; enclave in Rust/Go with static linking.【F:README.md†L140-L157】

## 9. Terminal UX ("codex" CLI)
Provides install, workspace initialization, trust bootstrap (TLS 1.3, CT, mTLS), passkey enrollment, personal vault management, macro ingestion, personal data connection, TEE deployment, and query workflows with citations.【F:README.md†L159-L193】

## 10. APIs
- Retrieval API (`POST /v1/compose`) returns structured context packs with policy metadata.【F:README.md†L195-L204】
- Vault SDK supports put/search/share/rotate operations for encrypted personal objects.【F:README.md†L205-L209】

## 11. Pipelines
- Macro ETL: ingest Wikidata/Wikidated, normalize Wikipedia citations, build graph/vector/temporal indexes.【F:README.md†L211-L214】
- Personal ingestion: local chunking, optional Presidio PII scan, local embedding, encryption, local DB storage, optional encrypted sync.【F:README.md†L215-L219】
- Compose: entity/time extraction, graph/vector retrieval, personal retrieval, rerank, pack, enforce abstain threshold, emit answer with citations.【F:README.md†L220-L223】

## 12. Key Management
User KEKs derive from passkeys or OPAQUE transcripts hardened with Argon2id; 256-bit DEKs wrapped via HPKE for devices and enclave policies; AEAD AAD binds to user/namespace/version; rotations rewrap keys upon device changes.【F:README.md†L225-L233】

## 13. Enclave Design
Static-linked minimal binaries with no shell and constrained egress; attest before decrypting; perform ANN and return only IDs/summaries.【F:README.md†L235-L238】

## 14. Evaluation & QA
Use RAGAS/RAG-HAT for groundedness (≥95%), privacy regression tests, prompt-injection red teaming, attestation negative tests, and crypto known-answer tests guarding nonce reuse and corruption detection.【F:README.md†L240-L244】

## 15. Operations & Supply Chain
Service mesh with SPIFFE/SPIRE identities and deny-by-default egress, SLSA provenance with in-toto and sigstore cosign enforcement, cloud KMS for macro keys while personal keys remain unexposed, enclave-only DEK unsealing under attestation.【F:README.md†L246-L252】

## 16. Sample Configurations
`enclave_pcr_policy.json` defines approved image hashes, signers, and networking policies. `codex.yaml` enforces TLS 1.3, CT, strict mTLS, OIDC, and cryptographic defaults for personal/macro/enclave subsystems.【F:README.md†L254-L276】

## 17. Outcomes
Trust (user-held keys, server blindness), verifiability (citations and timestamps), scalability (centralized macro + decentralized personal), and strong GDPR posture.【F:README.md†L278-L282】

## 18. 90-Day Execution Plan
1. Weeks 1–3: deliver CLI, identity, TLS stack; ingest Wikidata + Wikidated and build baseline retrieval.【F:README.md†L284-L286】
2. Weeks 3–6: ship personal vault SDK with XChaCha20-Poly1305; integrate Presidio PII service.【F:README.md†L287-L288】
3. Weeks 6–9: build enclave image, attestation, KMS policy; enable TEE retrieval.【F:README.md†L289-L290】
4. Weeks 9–12: implement Context Composer with temporal reasoning; add RAGAS/RAG-HAT dashboards.【F:README.md†L291-L292】

## 19. Risk Handling
Encrypted vector search handled locally or in TEEs; OPAQUE/Argon2id mitigate password risks; temporal drift countered through event graph with timestamps and Wikidated deltas.【F:README.md†L294-L298】

## 20. References
[1] TechCrunch – Wikidata Embedding Project.  
[2] RFC 8446 – TLS 1.3.  
[3] W3C WebAuthn Level 3.  
[4] RFC 9807 – OPAQUE aPAKE.  
[5] OpenID Connect Core.  
[6] libsodium – XChaCha20-Poly1305.  
[7] RFC 9180 – HPKE.  
[8] AWS Nitro Enclaves + KMS attestation.  
[9] Microsoft Presidio.  
[10] Wikidata Query Service.  
[11] AWS Nitro Enclave workflow.  
[12] Qdrant groundedness documentation.  
[13] NIST SP 800-38D.  
[14] GDPR Article 5.  
[15] GDPR Article 25.  
[16] GDPR Article 22.  
[17] Wikidated 1.0 paper.  
[18] FIDO Alliance – Passkeys.  
[19] Cloudflare – mTLS overview.  
[20] SLSA/in-toto/cosign reference.  
[21] AWS KMS cryptographic attestation.
[22] Google Confidential Computing attestation guidance.

## 21. Component Workstreams
- **codex-cli:** ship a composable command graph (init, trust, ingest, ask) with offline-first state, deterministic config parsing, and pluggable transport adapters for local vs. enclave retrieval. Instrument with structured tracing for end-to-end latency correlation.
- **ingestor:** implement resumable dump ingestion, deduplicate entities via stable QIDs, validate checksums, and emit change events onto a Kafka-compatible bus feeding graph/vector services.
- **graph:** expose gRPC + REST endpoints, enforce Cypher parameterization, support temporal predicates, and integrate policy checks for macro data export.
- **vector:** provide ANN APIs with deterministic pagination and cite-aware payloads; include reindex pipelines triggered by ingestor change events.
- **personal-client:** maintain secure keychain storage, offline-first sync queues, and background rewrap jobs; expose WASI bindings for cross-platform CLI integration.
- **personal-enclave:** minimize system calls, integrate attestation verifier, and implement streaming ANN responses with configurable leakage guards (top-k, distance thresholds).
- **composer:** deliver deterministic planning traces, guard against prompt injection by sanitizing retrieved text, and expose instrumentation hooks for evaluation harnesses.
- **answering:** abstract over LLM providers, enforce grounding policy (min coverage, abstain), and support templated response formatters for CLI and API outputs.
- **pii:** add policy-driven pipelines (redaction, pseudonymization) with audit logs and privacy budget tracking for optional macro ingestion sanitization.
- **audit:** maintain Merkle-tree backed append-only logs, provide inclusion proofs via CLI, and expose webhook integrations for security tooling.
- **infra:** codify Terraform/ Pulumi stacks for dedicated VPCs, service mesh policy, KMS bindings, attestation allow-lists, and CI/CD supply-chain enforcement.
- **eval:** automate nightly RAGAS/RAG-HAT runs, provide diff-based regressions, and publish dashboards tracking groundedness, latency, and abstain rates.

## 22. Testing & Verification Strategy
- **Cryptographic validation:** continuous known-answer test suites for AEAD/HPKE primitives, randomized misuse resistance fuzzing, and attestation transcript replay tests ensuring key release halts on mismatched PCRs.
- **Retrieval correctness:** golden query corpora spanning macro-only, personal-only, and hybrid scenarios; regression harness verifying citation coverage ≥ 0.8 and latency budgets per mode.
- **Security hardening:** automated prompt-injection adversarial tests, enclave memory integrity scans, and dependency SCA scans with supply-chain attestations.
- **Client quality:** cross-platform CLI integration tests, offline/online sync simulations, and deterministic config snapshot comparisons for reproducibility.
- **Operational readiness:** chaos drills for service mesh certificate rotation, KMS unavailability failover tests, and audit log tamper-evidence verification via Merkle inclusion proofs.

## 23. Observability & Operations
- **Metrics:** adopt OpenTelemetry for uniform tracing/metrics; key signals include retrieval latency, groundedness scores, enclave attestation success rates, and key rotation durations.
- **Logging:** structured, redaction-aware logs with field-level encryption for macro services; personal data processing confined to client/enclave with zero logging of plaintext payloads.
- **Alerting:** SLO-backed alerts for retrieval success rates, enclave attestation failures, CT/mTLS handshake anomalies, and audit log append gaps.
- **Runbooks:** document incident response for attestation failures, key rotation anomalies, and ingestion pipeline stalls; include CLI-first remediation commands.
- **Cost & capacity:** automated reports on vector store growth, enclave utilization, and client sync queue backlogs to inform scaling decisions.

## 24. Post-MVP Next Steps
1. **Collaborative sharing:** build cryptographic group sharing via HPKE multi-recipient envelopes with per-device revocation lists and transparency receipts.
2. **Policy language:** design a declarative policy DSL for personal data usage, enabling users to constrain retrieval by namespace, time, or sensitivity tags.
3. **Model fine-tuning hooks:** support secure hand-off of grounded context into fine-tuning pipelines with provenance tracking and consent gating.
4. **Marketplace integrations:** expose signed connectors for enterprise knowledge bases, ensuring ingestion policies inherit zero-trust constraints.
5. **Resilience enhancements:** replicate macro services across regions with deterministic reconciliation and gossip-based audit log replication.
