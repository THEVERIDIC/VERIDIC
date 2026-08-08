# Evidence Envelope Specification v0.1

*An open format specification for independently custodied AI agent execution records, defining the schema, verification algorithm, and capture interface contract.*

VERIDIC Inc. · Open Standards · Author: Pinar Emirdag · [e.pinar@theveridic.ai](mailto:e.pinar@theveridic.ai) · April 2026 · Evidence Envelope Specification  ·  v0.1  ·  April 2026 OPEN SPECIFICATION

---


## 1. Status and Licence

**This specification** is published under the Apache 2.0 licence. It may be reproduced, implemented, and extended without restriction. The Evidence Envelope Specification governs the format, schema, and verification algorithm only. VERIDIC's custody infrastructure — the append-only chain, key management, jurisdictional SLAs, institutional access controls — remains proprietary.

This is version 0.1. The specification is stable enough for implementation and reference but may be amended in future versions. Governance of this specification will transition to a neutral Technical Steering Committee. Until that transition, VERIDIC Inc. maintains editorial control.

> **Note.** This specification governs the format, schema, and verification algorithm. Custody infrastructure — storage, access control, jurisdictional SLAs, key management — is outside scope and may be provided by any conforming implementation.

---


## 2. Scope

This specification defines three components:

| Component | Scope |
| --- | --- |
| Schema | The AgentInteractionRecord (AIR): field definitions, types, versioning rules, identity binding fields (`agent_did`, `auth_context`, `delegation_chain`), and the OutcomeRecord schema. Defined as a structured field specification (§4). A CDDL representation is provided in the companion SCITT profile for wire-format interoperability. |
| Verification | A four-step verification algorithm: SHA-256 content hash, SHA-256 chain hash, ECDSA-P256 signature verification, sequence completeness check. Requires only the record, preceding chain hash, and the Issuer’s public key. |
| Capture | Writer interface contract: what a conforming storage backend must accept and guarantee. Append-only, at-least-once delivery, deduplication by `record_id`. Defined in IDL. Capture mechanism is out of scope. |

What is *not* part of this specification: custody infrastructure, APIs, analytics, reliability computation methods, query layer, framework adaptors, storage backends, jurisdiction-specific regulatory mappings. These are proprietary to the custody provider.

> **Note 1.** The capture mechanism is deliberately out of scope. An MCP middleware hook, a LangChain callback, a framework-native wrapper, and a reverse-proxy interceptor are all valid capture implementations. The specification constrains what is *recorded*, not how it is *captured*.

---


## 3. Terminology

| Term | Definition |
| --- | --- |
| Agent | An autonomous software system that perceives inputs, makes decisions, and takes actions on behalf of a principal, using one or more AI models to drive its behaviour. |
| Material Action | An action that creates, modifies, or extinguishes an external obligation, transfers value, accesses regulated data, or produces a consequential output that cannot be trivially reversed. |
| AIR | AgentInteractionRecord. A structured, signed, hash-chained record of a single Material Action. |
| Evidence Chain | The ordered, hash-chained sequence of AIRs for a given agent deployment. |
| Evidence Custodian | The party holding the append-only Evidence Chain. Independent of the Agent Operator. |
| Evidence Receipt | A signed attestation issued by the Evidence Custodian upon admission of an AIR, confirming chain position and integrity. |
| Redaction Receipt | A record attesting that a field was redacted prior to signing, preserving the SHA-256 hash of the original value. |
| Principal | The human or institutional entity on whose behalf the Agent acts and to whose authority its actions are attributed. |

---


## 4. AgentInteractionRecord Schema

**The AIR** captures eight categories of evidence about a Material Action: what the agent was given (input), what it produced (outcome), what resources it used (tool calls), what it reasoned (reasoning hash), who it is (identity), what authorised it (auth context, delegation chain), when it acted (timestamps), and where it fits in a causal chain (hash chain, sequence).


### 4.1 Field Definitions

> **Note 2.** `record_id` uses UUID v7 (RFC 9562) rather than UUID v4. UUID v7 is time-ordered, enabling range queries on the Evidence Chain without a separate index.

| Field | Type | Description |
| --- | --- | --- |
| schema_version | `string` | MUST be `"air-1.0"`. |
| record_id | `string` | UUID v7 (time-ordered). Unique identifier for this record. |
| session_id | `string` | Workflow or session UUID grouping related actions. |
| action_type | `string` | Classification of the Material Action. See §4.3. |
| action_subtype | `string|null` | Optional refinement of the action type. |
| action_timestamp_ms | `uint` | Unix epoch in milliseconds. When the Material Action completed. |
| captured_timestamp_ms | `uint` | When the capture pipeline recorded the action. |
| written_timestamp_ms | `uint|null` | When the Evidence Custodian admitted the record. Null until admission. |
| agent_id | `string` | Stable deployment identifier for the agent. |
| agent_version | `string` | Model and framework version string. |
| agent_did | `string|null` | W3C Decentralized Identifier, if available. |
| agent_workload_id | `string|null` | SPIFFE ID for workload identity, if available. |
| operator_id | `string` | Identifier for the Agent Operator. |
| operator_pubkey_id | `string` | Key identifier for the signing key used. |
| principal_id | `string|null` | Identifier for the human or institutional principal. |
| delegation_chain | `[string]|null` | Array of Verifiable Credential references establishing the authority chain. |
| intent_attestation | `string|null` | Reference to a signed attestation of the principal's intent. |
| auth_context | `AuthContext|null` | OAuth/authorisation context at execution time. See §4.2. |
| input_hash | `bytes` | SHA-256 hash of the raw input to the agent. |
| input_summary | `string|null` | Human-readable summary of the input (post-redaction). |
| outcome_state | `string` | Terminal state: `completed`, `failed`, `partially_completed`, `reversed`, `pending_confirmation`. |
| outcome_hash | `bytes` | SHA-256 hash of the raw output. |
| outcome_summary | `string|null` | Human-readable summary of the outcome (post-redaction). |
| tool_calls | `[ToolCallRecord]` | Array of tool/API calls made during execution. See §4.2. |
| jurisdiction | `string` | ISO 3166-1 alpha-2 country code. |
| retention_class | `string` | Applicable retention period: `regulatory_7yr`, `regulatory_5yr`, `regulatory_3yr`, `operational_1yr`, `custom`. |
| policy_refs | `[string]` | References to applicable policies governing this action. |
| external_refs | `[ExternalRef]` | Cross-references to external systems (trade IDs, payment references). See §4.2. |
| parent_record_id | `string|null` | UUID of the parent AIR in a multi-step workflow. |
| workflow_id | `string|null` | Identifier for the broader workflow this action belongs to. |
| trace_id | `string|null` | OpenTelemetry trace ID for observability correlation. |
| consumer_instructions | `string|null` | Principal-defined constraints for agentic commerce (price thresholds, merchant restrictions). |
| reasoning_hash | `bytes|null` | SHA-256 hash of the reasoning trace. Preserves evidence of reasoning without requiring the raw trace in the AIR. |
| redaction_receipts | `[RedactionReceipt]` | Attestations of redacted fields. See §7. |
| integrity | `IntegrityEnvelope` | Hash chain and signature. See §5. |

---


### 4.2 Supporting Types

```
// AuthContext — authorisation state at execution time
{
  "token_type":    string,       // e.g. "Bearer"
  "scopes":        [string],     // OAuth scopes granted
  "audience":      string|null,  // intended audience
  "expires_at_ms": uint|null     // token expiry, epoch ms
}

// ToolCallRecord — a single tool or API invocation
{
  "tool_id":       string,   // tool identifier
  "tool_type":     string,   // e.g. "api_call", "mcp_tool"
  "input_hash":   bytes,    // SHA-256 of tool input
  "output_hash":  bytes,    // SHA-256 of tool output
  "is_write":     bool,     // true if tool mutates state
  "timestamp_ms": uint      // when this call completed
}

// ExternalRef — cross-reference to external system
{
  "ref_type":   string,       // e.g. "payment_ref", "trade_id"
  "ref_value":  string,       // the reference value
  "ref_system": string|null  // originating system
}

// RedactionReceipt — see §7
{
  "field_path":    string,  // dot-path to redacted field
  "original_hash": bytes,   // SHA-256 of original value
  "policy_id":     string,  // which policy triggered redaction
  "timestamp_ms":  uint    // when redaction was applied
}

// IntegrityEnvelope — see §5
{
  "content_hash":    bytes,  // SHA-256 of canonical payload
  "prev_chain_hash": bytes,  // chain_hash of preceding record
  "chain_hash":      bytes,  // SHA-256(content_hash ∥ prev_chain_hash ∥ action_timestamp_ms ∥ agent_id)
  "sequence_number": uint,   // monotonically increasing, 0-based
  "signature":       bytes   // ECDSA-P256 over chain_hash
}
```

---


### 4.3 Action Type Vocabulary

Each AIR carries an `action_type` classifying the Material Action. The initial vocabulary defined by this specification:

| Value | Description |
| --- | --- |
| payment_initiation | Initiation of a payment instruction. |
| payment_execution | Confirmed execution of a payment. |
| contract_formation | Formation of a binding agreement. |
| contract_modification | Amendment of an existing agreement. |
| regulated_data_access | Access to regulated personal or financial data. |
| regulated_data_export | Export or transfer of regulated data. |
| trade_execution | Execution of a financial instrument trade. |
| credit_decision | Decision affecting credit standing or access. |
| authorisation_grant | Grant of authority to act. |
| authorisation_revocation | Revocation of previously granted authority. |
| external_commitment | A write or irreversible API call to an external system. |
| key_rotation | Rotation of an Issuer signing key. Creates a verifiable record of key lifecycle in the Evidence Chain. |

Implementations may define additional values using a reverse-DNS namespace (e.g. `com.example.custom-action`). Values without a namespace prefix are reserved for this specification.

> **Note 3.** The action type vocabulary is extensible by design. The initial set covers regulated financial services use cases. As the specification is adopted in new domains, the vocabulary will be extended through the governance process described in §9.

---


## 5. Integrity Envelope

**Every AIR carries** an integrity envelope computed by the signing component before submission to the Evidence Custodian. The cryptographic primitives are fixed: SHA-256 for hashing, ECDSA with the P-256 curve for signatures.


### 5.1 Hash Chain Construction

For the first record in a chain (`sequence_number = 0`): `prev_chain_hash` is 32 zero bytes. For all subsequent records: `prev_chain_hash` is the `chain_hash` of the immediately preceding record.

> **Note 4.** The three-timestamp model (`action_timestamp_ms`, `captured_timestamp_ms`, `written_timestamp_ms`) allows reconstruction of pipeline latency. The action timestamp is when the event happened; the captured timestamp is when the pipeline recorded it; the written timestamp is when the custodian admitted it.

```
// For every record:
content_hash   = SHA-256(canonical_payload)
chain_hash     = SHA-256(content_hash ∥ prev_chain_hash ∥ action_timestamp_ms ∥ agent_id)
signature      = ECDSA-P256.sign(chain_hash, private_key)
```

The four-input construction binds identity and temporal ordering into the chain at the cryptographic level: a receipt proves who, when, and what without requiring payload access. Both `agent_id` and `action_timestamp_ms` are mandatory fields supplied by the upstream agent platform at capture time.

The concatenation inputs are encoded as follows: `content_hash` and `prev_chain_hash` are raw 32-byte SHA-256 digest bytes. `action_timestamp_ms` is an unsigned 64-bit integer in big-endian (network) byte order (8 bytes). `agent_id` is a length-prefixed UTF-8 string: a 4-byte big-endian unsigned 32-bit integer containing the byte length of the UTF-8 encoding, followed by the UTF-8 encoded bytes (no null terminator). The total input to SHA-256 is exactly 76 + len(agent_id) bytes. This encoding is deterministic: two conforming implementations given the same field values MUST produce the same `chain_hash`.

Canonical form follows the JSON Canonicalization Scheme (RFC 8785). All fields except the `integrity` envelope are included in the canonical payload.


### 5.2 Identity Binding Levels

| Level | Requirements |
| --- | --- |
| Level 1 | `agent_id` and `operator_id` present. Signing key pre-registered. Signature verifies. |
| Level 2 | Level 1 plus `auth_context` present with valid scopes and non-expired token claims at `action_timestamp_ms`. |
| Level 3 | Level 2 plus at least one resolvable Verifiable Credential in `delegation_chain`, verified by the Issuer at capture time. |

---


## 6. Verification Algorithm

> **Architectural commitment.** The verification algorithm is architecturally separate from access control. Who can see records is permissioned. How to verify records is public by design. The algorithm has no secrets — the public key is, by definition, public.

**The following four-step algorithm** verifies any AIR. It requires only the record, the preceding record's `chain_hash` (or an Evidence Receipt), and the Issuer's public key.

> **Note.** The verification algorithm is self-contained. Any party holding the public key and the chain can verify independently, without contacting the Transparency Service or any other infrastructure provider.

| # | Operation | Pass | Fail implication |
| --- | --- | --- | --- |
| 1 | `SHA-256(canonical_payload) == content_hash` | payload intact | payload modified after signing |
| 2 | `SHA-256(content_hash ∥ prev_chain_hash ∥ action_timestamp_ms ∥ agent_id) == chain_hash` | chain intact | record reordered, inserted, or removed |
| 3 | `ECDSA-P256.verify(chain_hash, sig, pubkey)` | signature valid | record not signed by claimed Operator |
| 4 | `sequence_number == prev + 1` (monotonically increasing, no gaps) | no omissions | missing records — break point located |

If all four steps pass: the record is **VERIFIED** — intact, untampered, correctly positioned in the chain, and signed by the claimed Issuer.


### 6.1 Pseudocode

```
function verify_chain(records, public_key):
    prev_chain_hash = 0x00 * 32  // 32 zero bytes for first record
    expected_seq = 0

    for record in records:
        // Step 1: payload integrity
        payload = canonicalise(record, exclude=["integrity"])
        content_hash = sha256(payload)
        assert content_hash == record.integrity.content_hash

        // Step 2: chain integrity
        chain_hash = sha256(
            content_hash + prev_chain_hash +
            record.action_timestamp_ms + record.agent_id
        )
        assert chain_hash == record.integrity.chain_hash

        // Step 3: signature validity
        assert ecdsa_p256_verify(
            chain_hash,
            record.integrity.signature,
            public_key
        )

        // Step 4: sequence completeness
        assert record.integrity.sequence_number == expected_seq

        prev_chain_hash = chain_hash
        expected_seq += 1

    return VERIFIED
```

---


## 7. Redaction Semantics

**Evidence records** in regulated environments frequently contain personally identifiable information (PII) that must not persist in the custody layer. The redaction mechanism preserves evidence that the data existed at execution time while removing the data itself.

When a field is redacted, the signing component: computes the SHA-256 hash of the original field value; replaces the field value with a sentinel (e.g. `"[REDACTED]"`); and appends a `RedactionReceipt` to the `redaction_receipts` array. The canonical payload — and therefore the `content_hash` — is computed *after* redaction. This means the integrity envelope covers the redacted form. The `original_hash` in the Redaction Receipt allows a party with legitimate access to the original data to prove that the redacted field contained a specific value, without the custody layer ever holding the raw data.

> **Note 5.** For the following action types, at least one Redaction Receipt is required: `regulated_data_access`, `regulated_data_export`, `payment_initiation`, `payment_execution`, `credit_decision`. This ensures PII-containing fields are processed before signing.

The redaction order in the write pipeline is: Capture → Normalise → **Redact** → Sign → Write. Redaction occurs before signing. The signed record never contains raw PII.

---


## 8. Capture Interface Contract

**A conforming storage backend** — whether operated by VERIDIC or by any other party implementing this specification — must accept and guarantee the following:


### 8.1 Writer Interface (IDL)

```
interface EvidenceWriter {

  // Submit a signed AIR for admission to the Evidence Chain.
  // Returns an Evidence Receipt on success.
  // Rejects if schema validation, hash chain, or signature fails.
  async submit(record: SignedAIR): EvidenceReceipt

  // Retrieve a record by its record_id.
  // Returns null if the record does not exist or
  // the caller lacks access.
  async get(record_id: string): SignedAIR | null

  // Retrieve the Evidence Receipt for a given record.
  async get_receipt(record_id: string): EvidenceReceipt | null

  // Retrieve a range of records by sequence number.
  async get_range(
    agent_id: string,
    from_seq: uint,
    to_seq: uint
  ): [SignedAIR]
}
```


### 8.2 Guarantees

| Property | Requirement |
| --- | --- |
| Append-only | Records once admitted cannot be deleted, modified, or reordered. Enforced by both hash chain structure and operational policy. |
| At-least-once | The writer must accept resubmission of a record with the same `record_id` without error, deduplicating by `record_id`. |
| Deduplication | If a record with a given `record_id` has already been admitted, resubmission returns the existing Evidence Receipt. |
| Hash chain validation | Before admitting a record, the writer must verify that `prev_chain_hash` matches the `chain_hash` of the last admitted record for this `agent_id`. |
| Signature validation | Before admitting a record, the writer must verify the ECDSA-P256 signature using the pre-registered public key for the `operator_pubkey_id`. |
| Receipt issuance | The writer must issue a signed Evidence Receipt for every admitted record. |

> **Note 6.** The capture mechanism — how agent actions are intercepted and normalised into AIRs — is deliberately outside this specification. MCP middleware, framework callbacks (LangChain, CrewAI), reverse proxies, and direct instrumentation are all valid. The specification constrains the *output* of capture, not its mechanism.

---


## 9. Conformance

**A record is EES-conformant** if and only if all three of the following conditions hold:

| Condition | Requirement |
| --- | --- |
| Schema | The record satisfies the AgentInteractionRecord schema defined in §4, including all required fields with correct types and a valid `schema_version` value. |
| Cryptography | The integrity envelope (§5) uses SHA-256 for content and chain hashing, and ECDSA with the P-256 curve for signature. No other hash or signature algorithm is conformant. |
| Verification | The record passes all four steps of the verification algorithm defined in §6 when evaluated with the Issuer’s public key and the preceding record’s `chain_hash` (or 32 zero bytes for the first record in a chain). |

A conformant implementation is any system that produces records satisfying all three conditions above. Conformance is assessed at the record level, not the system level — a system is conformant if every record it emits is conformant.

> **Note.** Conformance is deliberately minimal. The specification does not prescribe how records are captured, transported, or stored. It constrains the output — the record itself — not the implementation that produces it.

---


## 10. Versioning and Governance

**The `schema_version` field** in every AIR is the mechanism for forward compatibility. Version `"air-1.0"` is defined by this specification. Future versions will be designated `"air-1.1"`, `"air-2.0"`, etc., following semantic versioning principles: minor versions add optional fields; major versions may change required fields or the integrity construction.

Governance of this specification will transition to a neutral Technical Steering Committee. Until that transition is complete, VERIDIC Inc. maintains editorial control.

---


## 11. Relationship to the SCITT Profile

**This specification** and the [AI Agent Execution Profile of SCITT](https://datatracker.ietf.org/doc/draft-emirdag-scitt-ai-agent-execution/) (draft-emirdag-scitt-ai-agent-execution-00) are complementary. The EES defines the canonical AIR schema, the four-step verification algorithm, and the writer interface contract. The SCITT profile defines the binding: how EES-conforming records are encoded as COSE_Sign1 structures, submitted to a SCITT Transparency Service via SCRAPI, validated against a Registration Policy, and receipted using SCITT Evidence Receipts. The SCITT profile also defines the hash chain as a verifiable data structure and specifies how Evidence Receipts may be embedded in the AIR as Transparent Statements (COSE label 394).

The AgentInteractionRecord schema reproduced in the SCITT profile (§5.2 of that document) is a conformant subset of the EES schema defined here. The canonical definition, including the full field set and extensibility model, is maintained in this specification. The SCITT profile stands alone for implementation purposes; the EES provides additional guidance for implementers building capture interfaces and verification tooling.

The underlying cryptographic parameters are identical: SHA-256 for hashing, ECDSA with P-256 for signatures. The difference is wire format: the EES defines a JSON-native representation; the SCITT profile encodes the same payload as CBOR within a COSE_Sign1 structure. Key material is interchangeable.

> **Note 7.** The EES is governed by VERIDIC (transitioning to an independent TSC). The SCITT profile is governed by the IETF.

---


## 12. References

| Reference | Document |
| --- | --- |
| RFC 8785 | Rundgren et al. — JSON Canonicalization Scheme (JCS). June 2020. |
| RFC 9052 | Schaad, J. — CBOR Object Signing and Encryption (COSE). August 2022. |
| RFC 9562 | Davis et al. — Universally Unique Identifiers (UUIDs). May 2024. |
| SCITT | Birkholz et al. — draft-ietf-scitt-architecture-22. October 2025. |
| SCITT Profile | Emirdag, P. — AI Agent Execution Profile of SCITT Architecture. draft-emirdag-scitt-ai-agent-execution-00. April 2026. |
| W3C DID | Sporny et al. — Decentralized Identifiers (DIDs) v1.0. July 2022. |
| W3C VC | Sporny et al. — Verifiable Credentials Data Model v2.0. 2024. |
| SPIFFE | Feldman et al. — Secure Production Identity Framework for Everyone. 2024. |
| OAuth 2.1 | Hardt, Parecki — The OAuth 2.1 Authorization Framework. 2026. |

---

**Evidence Envelope Specification v0.1** · Published April 2026 · VERIDIC Inc.

**Canonical URL:** [theveridic.github.io/VERIDIC/standards/ees/v0.1](https://theveridic.github.io/VERIDIC/standards/ees/v0.1) · **Contact:** [e.pinar@theveridic.ai](mailto:e.pinar@theveridic.ai)

This specification is published under the Apache 2.0 licence. It may be reproduced, implemented, and extended without restriction. The Evidence Envelope Specification governs the format, schema, and verification algorithm only. Custody infrastructure remains proprietary to the custody provider.
