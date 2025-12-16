# **Verifiable Claim Lexicon for ATProtocol**

### **Draft Specification — December 2025**

---

## **1. Overview**

This document introduces `community.claim`, a minimal lexicon for representing **verifiable claims** on ATProtocol. The design is inspired by the **LinkedClaims** specification from the Decentralized Identity Foundation (DIF) and follows the core principle that **claims are immutable, signed assertions**.

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
2. Attaching cryptographic signatures (including non-ATProto signatures)
3. Linking claims to evidence with provenance
4. Enabling apps to query, verify, and display claims
5. Supporting claims-about-claims (attestations, disputes, endorsements)

Existing ATProto lexicons support record signing and identity, but **ATProto does not yet define a general-purpose claim model**. `community.claim` fills this gap by defining the **content structure of a claim**.

---

## **3. Design Principles**

### **3.1 Minimal, not maximal**

The lexicon defines only the universal aspects of a claim. Everything else can be layered on top.

### **3.2 Compatible with LinkedClaims**

The lexicon satisfies the MUST requirements:

* A claim has a **subject** (any URI)
* A claim is **addressable** (via ATProto record URI: `at://did/community.claim/tid`)
* A claim is **cryptographically signed** (repo signing, plus optional embedded proof)
* A claim can reference **deterministic content**

### **3.3 Claims are Immutable**

Claims, once published, are immutable. Their identity is their content (CID). This ensures:
* Signatures remain valid forever
* References to claims are stable
* Audit trails are trustworthy

**There is no versioning of claims.** If you want to update, correct, or supersede a claim, you create a new claim that references the original (e.g., `subject: "at://did/community.claim/original-tid"`). This new claim might have `claimType: "supersedes"` or `claimType: "disputes"`.

### **3.4 Attestations are Just Claims**

An attestation, endorsement, or dispute is simply a claim where the `subject` is another claim's AT-URI. No special mechanism is needed.

### **3.5 Determinism**

Claims should produce stable, hashable representations to support auditability and provenance.

### **3.6 Support for External Signatures**

Claims may be signed by non-ATProto methods (Ethereum wallets, DIDs, aca-py, Digital Bazaar libraries, etc.) and then published to ATProto. The `embeddedProof` field carries these external signatures.

---

## **4. `community.claim` — Lexicon Definition**

### **4.1 Lexicon (Recommended for Adoption)**

```jsonc
{
  "lexicon": 1,
  "id": "community.claim",
  "defs": {
    "main": {
      "type": "record",
      "key": "tid",
      "description": "A LinkedClaim-style verifiable claim record.",
      "record": {
        "type": "object",
        "required": ["subject", "claimType", "createdAt"],
        "properties": {
          "subject": {
            "type": "string",
            "format": "uri",
            "description": "The entity this claim is about. Can be any URI: https URL, DID, AT-URI (for claims-about-claims), IPFS CID, etc."
          },
          "claimType": {
            "type": "string",
            "description": "Category of claim: skill, credential, impact, endorsement, dispute, rating, etc."
          },
          "object": {
            "type": "string",
            "description": "Optional object of the claim (e.g., skill name, rating value, credential type)."
          },
          "statement": {
            "type": "string",
            "maxLength": 10000,
            "description": "Human-readable explanation of the claim."
          },
          "source": {
            "type": "ref",
            "ref": "#claimSource",
            "description": "Structured evidence/provenance for this claim."
          },
          "createdAt": {
            "type": "string",
            "format": "datetime",
            "description": "When this record was created."
          },
          "effectiveDate": {
            "type": "string",
            "format": "datetime",
            "description": "When the claim became/becomes true (may differ from createdAt)."
          },
          "embeddedProof": {
            "type": "ref",
            "ref": "#embeddedProof",
            "description": "For claims signed by external methods (MetaMask, aca-py, etc.) before being published to ATProto."
          },
          "confidence": {
            "type": "number",
            "minimum": 0,
            "maximum": 1,
            "description": "Signer's confidence in this claim (0-1)."
          },
          "stars": {
            "type": "integer",
            "minimum": 1,
            "maximum": 5,
            "description": "Star rating (for rating-type claims)."
          }
        }
      }
    },

    "claimSource": {
      "type": "object",
      "description": "Structured provenance/evidence for a claim. Based on cooperation.org/credentials/v1 ClaimSource.",
      "properties": {
        "uri": {
          "type": "string",
          "format": "uri",
          "description": "URI of the evidence (URL, IPFS CID, AT-URI, etc.)."
        },
        "digestMultibase": {
          "type": "string",
          "description": "Multibase-encoded hash of the evidence content for integrity verification."
        },
        "howKnown": {
          "type": "string",
          "knownValues": [
            "FIRST_HAND",
            "SECOND_HAND",
            "WEB_DOCUMENT",
            "VERIFIED_LOGIN",
            "SIGNED_DOCUMENT",
            "BLOCKCHAIN",
            "RESEARCH",
            "OPINION",
            "OTHER"
          ],
          "description": "How the signer knows this claim to be true."
        },
        "dateObserved": {
          "type": "string",
          "format": "datetime",
          "description": "When the evidence was observed/collected."
        },
        "author": {
          "type": "string",
          "format": "uri",
          "description": "Original author of the evidence (if different from claim signer)."
        },
        "curator": {
          "type": "string",
          "format": "uri",
          "description": "Entity that curated/surfaced this evidence."
        }
      }
    },

    "embeddedProof": {
      "type": "object",
      "description": "Cryptographic proof from external signing methods. Allows claims signed via MetaMask, aca-py, Digital Bazaar, etc. to be published on ATProto.",
      "required": ["type", "verificationMethod", "proofValue", "created"],
      "properties": {
        "type": {
          "type": "string",
          "description": "Signature type (e.g., EthereumEip712Signature2021, Ed25519Signature2020, JsonWebSignature2020)."
        },
        "verificationMethod": {
          "type": "string",
          "description": "DID or key identifier of the actual claim signer (e.g., did:pkh:eip155:1:0xABC..., did:key:z6Mk...)."
        },
        "proofValue": {
          "type": "string",
          "description": "The signature value."
        },
        "proofPurpose": {
          "type": "string",
          "default": "assertionMethod",
          "description": "Purpose of the proof (typically assertionMethod)."
        },
        "created": {
          "type": "string",
          "format": "datetime",
          "description": "When the signature was created."
        }
      }
    }
  }
}
```

---

## **5. Diagrams**

### **Diagram A: Claim Creation Paths**

```
┌─────────────────────────────────────────────────────────────────┐
│                    CLAIM CREATION PATHS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Path A: User signed via ATProto (Bluesky OAuth)               │
│  ───────────────────────────────────────────────               │
│  User OAuth's with Bluesky → App writes to USER's repo          │
│  • Record: at://did:plc:USER/community.claim/xxx               │
│  • Signer = User's DID (implicit via repo ownership)           │
│  • No embeddedProof needed                                      │
│                                                                 │
│  Path B: User signed via external method (MetaMask, etc.)      │
│  ────────────────────────────────────────────────────          │
│  User signs claim payload → Server publishes to SERVER's repo   │
│  • Record: at://did:plc:SERVER/community.claim/xxx             │
│  • ATProto signer = Server's DID                               │
│  • Actual claim signer = embeddedProof.verificationMethod      │
│                                                                 │
│  Path C: Server-signed claim                                    │
│  ──────────────────────────                                     │
│  Server creates claim → publishes to SERVER's repo              │
│  • Record: at://did:plc:SERVER/community.claim/xxx             │
│  • Signer = Server's DID (implicit)                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### **Diagram B: Claims About Claims (Attestation Pattern)**

```
        Alice's Claim (original)
┌────────────────────────────────────────────┐
│ uri: at://did:plc:alice/community.claim/a1 │
│ subject: "https://example.org/project"     │
│ claimType: "impact"                        │
│ statement: "Delivered 500 water filters"   │
└────────────────────────────────────────────┘
                     ▲
                     │ subject references
                     │
        Bob's Claim (endorsement)
┌────────────────────────────────────────────┐
│ uri: at://did:plc:bob/community.claim/b1   │
│ subject: "at://did:plc:alice/...claim/a1"  │  ← Points to Alice's claim
│ claimType: "endorsement"                   │
│ statement: "I witnessed this distribution" │
│ source: { howKnown: "FIRST_HAND" }         │
└────────────────────────────────────────────┘

        Carol's Claim (dispute)
┌────────────────────────────────────────────┐
│ uri: at://did:plc:carol/community.claim/c1 │
│ subject: "at://did:plc:alice/...claim/a1"  │  ← Points to Alice's claim
│ claimType: "dispute"                       │
│ statement: "The count was only 200"        │
│ source: { uri: "https://...", howKnown: "WEB_DOCUMENT" }
└────────────────────────────────────────────┘

No special "attestation" mechanism needed.
Attestations, endorsements, disputes are all just claims.
```

---

## **6. Usage Examples**

### **6.1 Skill Claim (ATProto-native signing)**

User publishes directly to their own repo via Bluesky OAuth:

```json
{
  "$type": "community.claim",
  "subject": "did:plc:alice",
  "claimType": "skill",
  "object": "React",
  "statement": "3 years of production experience with React.",
  "source": {
    "howKnown": "FIRST_HAND"
  },
  "createdAt": "2025-02-14T18:22:00Z"
}
```

### **6.2 NGO Impact Claim with Evidence**

```json
{
  "$type": "community.claim",
  "subject": "https://ngo.example.org/projects/water-kibera",
  "claimType": "impact",
  "statement": "Distributed 500 water filters in Kibera, Nairobi.",
  "source": {
    "uri": "ipfs://bafy...",
    "digestMultibase": "zQm...",
    "howKnown": "FIRST_HAND",
    "dateObserved": "2025-02-10T09:45:12Z"
  },
  "createdAt": "2025-02-10T10:00:00Z",
  "effectiveDate": "2025-02-10T09:45:12Z"
}
```

### **6.3 Claim with External Signature (MetaMask)**

User signed with MetaMask, server publishes to ATProto:

```json
{
  "$type": "community.claim",
  "subject": "did:plc:bob",
  "claimType": "endorsement",
  "object": "trustworthy",
  "statement": "I have worked with Bob for 2 years and vouch for their integrity.",
  "source": {
    "howKnown": "FIRST_HAND"
  },
  "createdAt": "2025-02-14T18:30:00Z",
  "embeddedProof": {
    "type": "EthereumEip712Signature2021",
    "verificationMethod": "did:pkh:eip155:1:0x1234567890abcdef...",
    "proofValue": "0xabc123...",
    "proofPurpose": "assertionMethod",
    "created": "2025-02-14T18:30:00Z"
  }
}
```

### **6.4 Endorsement of Another Claim**

```json
{
  "$type": "community.claim",
  "subject": "at://did:plc:alice/community.claim/3kf5xyz",
  "claimType": "endorsement",
  "statement": "I can confirm this impact claim is accurate.",
  "source": {
    "howKnown": "FIRST_HAND"
  },
  "createdAt": "2025-02-15T10:00:00Z"
}
```

### **6.5 Rating Claim**

```json
{
  "$type": "community.claim",
  "subject": "https://restaurant.example.com",
  "claimType": "rating",
  "object": "food-quality",
  "stars": 4,
  "statement": "Excellent pasta, slightly slow service.",
  "createdAt": "2025-02-14T20:00:00Z"
}
```

---

## **7. Technical Considerations**

### **7.1 Deterministic Serialization**

To ensure verification and reproducibility, implementations should:

* Normalize fields before signing
* Exclude the `embeddedProof` field during signature computation (for external signatures)
* Use DAG-CBOR or equivalent canonical encoding when computing CIDs

### **7.2 Claim Identity**

A claim's identity is:
* **AT-URI**: `at://did:plc:xxx/community.claim/tid` (human-friendly, mutable pointer)
* **CID**: Content hash (immutable, canonical reference)

When referencing claims, prefer CID for cryptographic guarantees, AT-URI for human readability.

### **7.3 Who Signed This Claim?**

| Scenario | Signer Identity |
|----------|-----------------|
| Published to user's own repo (no embeddedProof) | Repo owner DID |
| Has embeddedProof field | `embeddedProof.verificationMethod` |
| Published to server's repo (no embeddedProof) | Server DID |

AppViews should normalize this into a canonical `signer` field when indexing.

### **7.4 Privacy**

The lexicon supports:

* Public claims (default on ATProto)
* Private evidence with public claims (evidence URI may be access-controlled)
* Claims can reference pseudonymous DIDs

### **7.5 Revocation**

To revoke a claim:
1. Delete the record from the repo (ATProto native)
2. Or publish a new claim with `subject: <original-claim-uri>` and `claimType: "revocation"`

Option 2 leaves an audit trail and is preferred for transparency.

---

## **8. Relationship to Existing Standards**

### **8.1 LinkedClaims (DIF)**

`community.claim` is effectively a **LinkedClaim profile for ATProto**, satisfying core requirements:
* Subject is any URI ✓
* Claim has URI identity ✓
* Cryptographically signed ✓
* Can reference evidence ✓
* Claims can link to claims ✓

### **8.2 W3C Verifiable Credentials**

While simpler than VCs, this lexicon can interoperate:
* `embeddedProof` can carry VC-style proofs
* Claims can reference VC documents as evidence
* The `source` structure aligns with VC evidence patterns

### **8.3 cooperation.org/credentials/v1**

The `claimSource` structure is based on the LinkedClaims JSON-LD vocabulary at `cooperation.org/credentials/v1`, ensuring semantic compatibility.

---

## **9. What Was Removed (and Why)**

### **9.1 `prevCid` (Versioning)**

**Removed.** Versioning breaks the immutability model:
* If Alice signs Claim v1, then "updates" to v2, Alice's original signature doesn't cover v2
* This creates confusion about what was actually signed

**Alternative:** Create a new claim with `subject: <original-claim-uri>` and `claimType: "supersedes"`. This is explicit, auditable, and preserves signature validity.

### **9.2 `sigs` (Accumulated Signatures)**

**Removed.** Third-party attestations don't need a special field:
* Bob attesting to Alice's claim = Bob creating his own claim with `subject: <alice's-claim-uri>`
* This is the LinkedClaims pattern: claims about claims
* No modification of the original record needed
* Each attestation is independently signed and stored

---

## **10. Status and Next Steps**

This lexicon is a **draft** intended to:

* Support real-world pilots (credentials, philanthropy, professional attestations)
* Guide future community standards
* Enable developers to experiment with verifiable claims today
* Align with the LinkedClaims specification work at DIF

### **Open Questions**

1. Should `claimType` be an open string or a closed enum?
2. Should we define a `community.claim.batch` for bulk operations?
3. How should AppViews handle claims with invalid `embeddedProof` signatures?

### **Next Steps**

1. Publish lexicon JSON to a well-known location
2. Implement reference AppView indexer
3. Build SDK for claim creation/verification
4. Pilot with LinkedTrust and other community applications
