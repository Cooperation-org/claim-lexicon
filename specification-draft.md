# **Verifiable Claim Lexicon for ATProtocol**

### **Draft Specification — December 2025**

---

## **1. Overview**

This document introduces `community.claim`, a minimal lexicon for representing **verifiable claims** on ATProtocol. The design is inspired by the **LinkedClaims** specification from the Decentralized Identity Foundation (DIF) and is intentionally compatible with the emerging **community.lexicon.attestation** signature trait now under discussion in the ATProto ecosystem.

The goal is to provide a small, practical building block that allows applications to publish structured, signed statements about people, events, or actions. These claims can be:

* Professional credentials
* Skill assertions
* NGO impact reports
* Community confirmations
* Endorsements or disputes
* Machine-generated attestations (e.g., timestamp proofs, sensor data)

This lexicon is deliberately minimal, allowing domain-specific extensions without imposing a universal schema.

---

## **2. Purpose and Motivation**

Many ATProto applications need a primitive for:

1. Expressing structured claims
2. Attaching cryptographic attestations
3. Linking claims to evidence
4. Supporting claim versioning
5. Enabling apps to query, verify, and display claims

Existing ATProto lexicons support record signing and identity, but **ATProto does not yet define a general-purpose claim model**. Meanwhile, the community is actively developing a shared attestation standard to allow multiple parties to sign a single record.

`community.claim` fills this gap by defining the **content structure of a claim**, while remaining compatible with whichever attestation format becomes canonical.

---

## **3. Design Principles**

### **3.1 Minimal, not maximal**

The lexicon defines only the universal aspects of a claim. Everything else can be layered on top.

### **3.2 Compatible with LinkedClaims**

The lexicon satisfies the MUST requirements:

* A claim has a **subject** (URI or DID)
* A claim is **addressable** (via ATProto record URI)
* A claim is **cryptographically signed** (using `sigs`)
* A claim can reference **deterministic content**

### **3.3 Compatible with ATProto Attestation Work**

The `sigs` field aligns with the proposed `community.lexicon.attestation` signature schema, but without locking into any evolving details.

### **3.4 Determinism**

Claims should produce stable, hashable representations to support auditability and provenance.

### **3.5 Upgradable**

Claims can reference prior versions via `prevCid`, supporting trust continuity.

---

## **4. `community.claim` — Lexicon Definition**

### **4.1 Minimal Lexicon (Recommended for Adoption)**

```jsonc
{
  "lexicon": 1,
  "id": "community.claim",
  "defs": {
    "main": {
      "type": "record",
      "key": "tid",
      "description": "A minimal LinkedClaim-style verifiable claim record.",
      "record": {
        "type": "object",
        "required": ["subject", "claimType", "content", "createdAt"],
        "properties": {
          "subject": {
            "type": "string",
            "format": "uri",
            "description": "The entity this claim is about (DID or URL)."
          },
          "claimType": {
            "type": "string",
            "description": "Type of claim (e.g., 'skill', 'credential', 'impact', 'verification')."
          },
          "content": {
            "type": "string",
            "description": "Human-readable explanation of the claim."
          },
          "evidence": {
            "type": "string",
            "format": "uri",
            "description": "Optional CID or URL for supporting evidence."
          },
          "createdAt": {
            "type": "string",
            "format": "datetime"
          },
          "effectiveDate": {
            "type": "string",
            "format": "datetime"
          },
          "prevCid": {
            "type": "string",
            "format": "cid",
            "description": "Links to a previous version of this claim."
          },
          "sigs": {
            "type": "ref",
            "ref": "community.lexicon.attestation.signature",
            "description": "Signatures attesting to this claim."
          }
        }
      }
    }
  }
}
```

---

## **5. Diagrams**

### **Diagram A: Claim + Attestation Flow**

```

┌──────────────────┐
│   User A (Claim)  │
└───────┬──────────┘
        │ creates
        ▼
┌─────────────────────────────┐
│ community.claim record      │
│ - subject                   │
│ - claimType                 │
│ - content                   │
│ - evidence?                 │
│ - createdAt                 │
│ - sigs[]  (optional)        │
└─────────┬───────────────────┘
          │ published as ATProto record (URI + CID)
          ▼
     Viewable by apps
          │
          │ User B attests
          ▼
┌──────────────────────────────────┐
│ community.lexicon.attestation    │
│ appended to claim.sigs[]         │
│ - issuer: DID_B                  │
│ - signature: base64              │
│ - issuedAt: datetime             │
└──────────────────────────────────┘

AppViews validate:

* signature correctness
* subject integrity
* record determinism

```

---

### **Diagram B: Claim Versioning & Integrity**

```

      Original Claim (v1)

┌────────────────────────────────────────────┐
│ id: at://did:alice/.../v1                 │
│ content: "React developer"                │
│ prevCid: null                             │
│ sigs: [coworker, bootcamp]                │
└──────────────────────────┬─────────────────┘
                           │ updated
                           ▼
                  New Version Claim (v2)
┌────────────────────────────────────────────┐
│ id: at://did:alice/.../v2                 │
│ content: "React & Next.js developer"      │
│ prevCid: <CID-v1>                         │
│ sigs: [] (new attestations needed)        │
└────────────────────────────────────────────┘

AppViews determine:

* whether attestations still apply
* what changed between versions
* which version is referenced by downstream claims

```

---

## **6. Usage Examples**

### **6.1 Skill Claim**

```json
{
  "$type": "community.claim",
  "subject": "did:example:alice",
  "claimType": "skill",
  "content": "React developer with 3 years of production experience.",
  "evidence": "ipfs://bafy…",
  "createdAt": "2025-02-14T18:22:00Z"
}
```

### **6.2 NGO Impact Claim**

```json
{
  "$type": "community.claim",
  "subject": "https://ngo.example.org/projects/water-kibera",
  "claimType": "impact",
  "content": "Distributed 500 water filters in Kibera, Nairobi.",
  "evidence": "at://did:plc:communityhost/evidence/xyz",
  "createdAt": "2025-02-10T09:45:12Z"
}
```

### **6.3 Claim Attested by a Third Party**

```json
{
  "$type": "community.claim",
  "subject": "did:example:alice",
  "claimType": "skill",
  "content": "React developer with 3 years of production experience.",
  "createdAt": "2025-02-14T18:22:00Z",
  "sigs": [
    {
      "issuer": "did:example:bob",
      "signature": "<base64 signature>",
      "issuedAt": "2025-02-14T18:30:00Z"
    }
  ]
}
```

---

## **7. Technical Considerations**

### **7.1 Deterministic Serialization**

To ensure verification and reproducibility, implementations should:

* Normalize fields before signing
* Exclude the `sigs` field during signature computation
* Use DAG-CBOR or equivalent canonical encoding

### **7.2 Versioning**

The `prevCid` field supports:

* Audit trails
* Attestation continuity
* Change detection
* Forensic reconstruction

### **7.3 Privacy**

The lexicon supports:

* public claims
* selectively disclosed claims
* private evidence with public attestations

### **7.4 Future Extensions**

Possible future lexicons:

* `community.claim.attestation` (standalone attestations)
* `community.claim.review` (structured evaluations)
* `community.claim.graph` (trust graph indices)
* Domain profiles: employment, credentials, aid projects, datasets

---

## **8. Relationship to Existing Standards**

### **8.1 LinkedClaims (DIF)**

`community.claim` is effectively a **LinkedClaim profile for ATProto**, satisfying core requirements and remaining extensible.

### **8.2 ATProto Attestation Proposal**

This lexicon aligns with the proposed `community.lexicon.attestation.signature` without assuming finalization, ensuring forward compatibility.

### **8.3 Verifiable Credentials (W3C)**

While simpler than VCs, this lexicon can interoperate with VC systems when needed.

---

## **9. Status and Next Steps**

This lexicon is a **draft** intended to:

* Support real-world pilots (credentials + philanthropy)
* Guide future community standards
* Possibly spark discussion within the lexicon-community group
* Enable developers to experiment with verifiable claims today

We could later open a GitHub issue with this proposal, share implementation notes from pilots, and collaborate with maintainers working on the attestation lexicon.
