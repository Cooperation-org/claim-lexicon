# **AppView Indexing Guide for `community.claim`**

### **For ATProtocol Implementers — Draft, December 2025**

---

# 1. Overview

This guide explains how an AppView should:

1. Index `community.claim` records from the ATProto firehose
2. Determine the actual signer (repo owner vs embedded proof)
3. Verify external signatures (when `embeddedProof` is present)
4. Index claims-about-claims (attestations, disputes, endorsements)
5. Produce efficient query results for trust graphs

AppViews allow ATProto applications to build *derived views* ("server-side indexes") over user repositories. For verifiable claims, AppViews are essential for:

* Querying claims by subject, signer, or type
* Validating embedded signatures
* Building trust graphs (who endorsed whom)
* Computing aggregate trust signals

---

# 2. Architecture Diagram

```
               ┌───────────────────────────┐
               │     User Repositories     │
               │  (Personal Data Servers)  │
               └──────────────┬────────────┘
                              |
                   ATProto Sync & Firehose
                              |
                              ▼
                ┌───────────────────────────┐
                │         AppView           │
                │  community.claim Indexer  │
                └───────────────────────────┘
                              |
          ┌───────────────────┼────────────────────┐
          ▼                   ▼                    ▼
   Claim Index         Signer Index         Graph Index
 (subject, type)    (who signed what)    (claims-about-claims)

          └───────────────────┬────────────────────┘
                              ▼
                Application Query Layer / APIs
```

---

# 3. Data Types Being Indexed

## 3.1 Claim Record Structure

```ts
type Claim = {
  // Required
  subject: string;        // Any URI (DID, URL, AT-URI, IPFS, etc.)
  claimType: string;      // "skill", "impact", "endorsement", "dispute", etc.
  createdAt: string;      // ISO datetime

  // Optional
  object?: string;        // Object of the claim (skill name, rating value, etc.)
  statement?: string;     // Human-readable explanation
  source?: ClaimSource;   // Evidence/provenance
  effectiveDate?: string; // When claim became true
  confidence?: number;    // 0-1
  stars?: number;         // 1-5 rating

  // For externally-signed claims
  embeddedProof?: EmbeddedProof;
};
```

## 3.2 ClaimSource Structure

```ts
type ClaimSource = {
  uri?: string;            // Evidence URI
  digestMultibase?: string; // Hash for integrity
  howKnown?: string;       // FIRST_HAND, SECOND_HAND, WEB_DOCUMENT, etc.
  dateObserved?: string;   // When evidence was observed
  author?: string;         // Original author (if different from signer)
  curator?: string;        // Who surfaced this evidence
};
```

## 3.3 EmbeddedProof Structure

```ts
type EmbeddedProof = {
  type: string;              // e.g., "EthereumEip712Signature2021"
  verificationMethod: string; // DID or key ID of actual signer
  proofValue: string;        // The signature
  proofPurpose: string;      // Usually "assertionMethod"
  created: string;           // When signature was created
};
```

---

# 4. Determining the Signer

**Critical for trust graphs:** Who actually signed this claim?

```ts
function getClaimSigner(record: Claim, repoOwnerDid: string): string {
  // If there's an embedded proof, the actual signer is in verificationMethod
  if (record.embeddedProof?.verificationMethod) {
    return record.embeddedProof.verificationMethod;
  }

  // Otherwise, the repo owner is the signer
  return repoOwnerDid;
}
```

### Signer Scenarios

| Scenario | `embeddedProof` | Signer |
|----------|-----------------|--------|
| User publishes to own repo | absent | Repo owner DID |
| Server publishes user's MetaMask-signed claim | present | `embeddedProof.verificationMethod` |
| Server publishes its own claim | absent | Server DID (repo owner) |

---

# 5. Indexing Workflow

```
ATProto Firehose Stream
        │
        ▼
╔════════════════════════════════════╗
║  1. Filter for community.claim     ║
╚════════════════════════════════════╝
        │
        ▼
╔════════════════════════════════════╗
║  2. Parse Record                   ║
║  - Extract all fields              ║
║  - Determine signer                ║
╚════════════════════════════════════╝
        │
        ▼
╔════════════════════════════════════╗
║  3. Validate (if embeddedProof)    ║
║  - Verify signature                ║
║  - Check proof type support        ║
╚════════════════════════════════════╝
        │
        ├──────────────► on failure: flag but still index
        ▼
╔════════════════════════════════════╗
║  4. Insert into DB                 ║
║  - claims table                    ║
║  - claim_graph table (if subject   ║
║    is another claim AT-URI)        ║
╚════════════════════════════════════╝
```

---

# 6. Database Schema

## **Table: claims**

| column           | type      | notes                              |
| ---------------- | --------- | ---------------------------------- |
| `uri`            | string PK | at://did/community.claim/tid       |
| `cid`            | string    | content-address (immutable ID)     |
| `repo_did`       | string    | DID of repo owner                  |
| `signer`         | string    | Actual signer (computed)           |
| `subject`        | string    | What the claim is about            |
| `claim_type`     | string    | skill, impact, endorsement, etc.   |
| `object`         | string    | Optional claim object              |
| `statement`      | text      | Human-readable content             |
| `created_at`     | timestamp | When record was created            |
| `effective_date` | timestamp | When claim became true             |
| `confidence`     | decimal   | 0-1                                |
| `stars`          | int       | 1-5                                |
| `has_embedded_proof` | boolean | Whether external signature exists |
| `proof_valid`    | boolean   | NULL if no proof, true/false otherwise |
| `indexed_at`     | timestamp | When we indexed this               |

## **Table: claim_sources**

| column            | type      | notes                     |
| ----------------- | --------- | ------------------------- |
| `claim_uri`       | string FK | Reference to claim        |
| `uri`             | string    | Evidence URI              |
| `digest_multibase`| string    | Hash for integrity        |
| `how_known`       | string    | FIRST_HAND, etc.          |
| `date_observed`   | timestamp | When evidence was seen    |
| `author`          | string    | Original author           |
| `curator`         | string    | Who surfaced it           |

## **Table: claim_graph**

For efficiently querying claims-about-claims:

| column         | type      | notes                           |
| -------------- | --------- | ------------------------------- |
| `claim_uri`    | string FK | The claim making the assertion  |
| `target_uri`   | string    | The claim being referenced      |
| `claim_type`   | string    | endorsement, dispute, etc.      |
| `signer`       | string    | Who made this assertion         |

Populated when `subject` matches pattern `at://*/community.claim/*`

---

# 7. Signature Validation

When a claim has `embeddedProof`, validate the signature:

```ts
async function validateEmbeddedProof(
  record: Claim,
  proof: EmbeddedProof
): Promise<boolean> {
  // 1. Create canonical form (exclude embeddedProof itself)
  const { embeddedProof, ...claimContent } = record;
  const canonicalBytes = canonicalize(claimContent);

  // 2. Resolve the verification method
  const publicKey = await resolveVerificationMethod(proof.verificationMethod);

  // 3. Verify based on proof type
  switch (proof.type) {
    case 'EthereumEip712Signature2021':
      return verifyEthereumSignature(canonicalBytes, proof.proofValue, publicKey);
    case 'Ed25519Signature2020':
      return verifyEd25519Signature(canonicalBytes, proof.proofValue, publicKey);
    case 'JsonWebSignature2020':
      return verifyJwsSignature(canonicalBytes, proof.proofValue, publicKey);
    default:
      // Unknown proof type - log and mark as unverified
      return false;
  }
}
```

### Handling Validation Failures

**Index the claim anyway**, but mark `proof_valid = false`. This allows:
- Clients to decide their own policy
- Debugging of signature issues
- Graceful handling of unsupported proof types

---

# 8. Query API Surface

## 8.1 **Get all claims about a subject**

```
GET /xrpc/community.claim.getBySubject?subject=did:plc:alice
```

Returns all claims where `subject` matches, regardless of signer.

## 8.2 **Get claims by signer**

```
GET /xrpc/community.claim.getBySigner?signer=did:plc:bob
```

Returns all claims where computed `signer` matches.

## 8.3 **Get attestations for a claim**

```
GET /xrpc/community.claim.getAttestations?uri=at://did:plc:alice/community.claim/xyz
```

Returns claims where `subject` equals the given AT-URI (endorsements, disputes, etc.)

## 8.4 **Get trust graph**

```
GET /xrpc/community.claim.getTrustGraph?subject=did:plc:alice&depth=2
```

Returns:
- Direct claims about the subject
- Claims about those claims (attestations)
- Optionally: transitive trust paths

---

# 9. Claims-About-Claims Pattern

This is the core mechanism for attestations, disputes, and endorsements.

### Example: Endorsement Chain

```
Alice claims:
  at://did:plc:alice/community.claim/a1
  subject: "https://ngo.org/project"
  claimType: "impact"
  statement: "Delivered 500 water filters"

Bob endorses:
  at://did:plc:bob/community.claim/b1
  subject: "at://did:plc:alice/community.claim/a1"  ← Points to Alice's claim
  claimType: "endorsement"
  source: { howKnown: "FIRST_HAND" }

Carol disputes:
  at://did:plc:carol/community.claim/c1
  subject: "at://did:plc:alice/community.claim/a1"  ← Points to Alice's claim
  claimType: "dispute"
  statement: "I counted only 200 filters"
```

### Indexing Strategy

When indexing, check if `subject` looks like an AT-URI for a claim:

```ts
function isClaimUri(uri: string): boolean {
  return uri.match(/^at:\/\/[^\/]+\/community\.claim\//) !== null;
}

// During indexing:
if (isClaimUri(record.subject)) {
  await db.claim_graph.insert({
    claim_uri: recordUri,
    target_uri: record.subject,
    claim_type: record.claimType,
    signer: computedSigner
  });
}
```

---

# 10. Trust Score Computation (Optional)

AppViews can compute derived trust metrics:

```ts
type TrustScore = {
  subject: string;
  endorsements: number;
  disputes: number;
  uniqueEndorsers: string[];
  weightedScore?: number;  // Based on endorser reputation
};

async function computeTrustScore(subjectUri: string): Promise<TrustScore> {
  const attestations = await db.claim_graph.findByTarget(subjectUri);

  return {
    subject: subjectUri,
    endorsements: attestations.filter(a => a.claim_type === 'endorsement').length,
    disputes: attestations.filter(a => a.claim_type === 'dispute').length,
    uniqueEndorsers: [...new Set(attestations.map(a => a.signer))],
    // Optional: weight by endorser reputation
  };
}
```

---

# 11. Best Practices

### Always store raw AND normalized data
Keep the original record for auditability, plus normalized fields for querying.

### Index by CID AND URI
- URI = current mutable pointer
- CID = immutable content reference

### Handle missing/invalid proofs gracefully
Don't reject claims with bad signatures - index them and let clients decide policy.

### Cache DID resolution
Resolving `did:pkh:eip155:1:0x...` or `did:key:z6Mk...` is expensive. Cache aggressively.

### Detect claim deletions
When a repo deletes a claim record, mark it as deleted in your index (don't remove - audit trail).

---

# 12. Summary: End-to-End Flow

```
User creates claim (frontend)
        │
        ├─► Signed by ATProto (own repo)
        │   OR
        └─► Signed externally (MetaMask, etc.) + published to server repo
                │
                ▼
        ATProto Firehose
                │
                ▼
        AppView Indexer
        - Parse record
        - Determine signer
        - Validate embeddedProof (if present)
        - Index to DB
        - Build claim graph (for claims-about-claims)
                │
                ▼
        Query APIs
        - By subject
        - By signer
        - Attestations for claim
        - Trust graphs
                │
                ▼
        Client applications display trust data
```
