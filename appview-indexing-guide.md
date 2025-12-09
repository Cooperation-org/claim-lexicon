# 📘 **AppView Indexing Guide for `community.claim` + Attestations**

### **For ATProtocol Implementers — Draft, February 2025**

---

# 1. Overview

This guide explains how an AppView should:

1. Index `community.claim` records
2. Resolve and verify attestations (`sigs`)
3. Handle claim versioning (`prevCid`)
4. Produce efficient query results
5. Expose application-level APIs for trust graphs
6. Maintain deterministic views for external consumers

AppViews allow ATProto applications to build *derived views* (“server-side indexes”) over user repositories. For verifiable claims, AppViews are essential for:

* querying claims efficiently
* validating signatures
* finding attestations
* tracking version history
* computing trust signals

---

# 2. Architecture Diagram (Conceptual)

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
   Claim Index          Attestation Index    Version Graph
 (by subject, type)     (by issuer, claim)       (prevCid)

          └───────────────────┬────────────────────┘
                              ▼
                Application Query Layer / APIs
```

AppView responsibilities include:

* indexing all `community.claim` records
* extracting + validating `sigs`
* linking attestations to claim CIDs
* detecting updates and invalidations
* surfacing structured data to clients

---

# 3. Data Types Being Indexed

## 3.1 Claim Record Structure

```ts
type Claim = {
  subject: string;      // DID or URI
  claimType: string;    // "skill", "impact", etc.
  content: string;
  evidence?: string;    // CID or URL
  createdAt: string;
  effectiveDate?: string;
  prevCid?: string;
  sigs?: Signature[];
};
```

## 3.2 Signature Block Structure (simplified)

```ts
type Signature = {
  issuer: string;       // DID of attestor
  signature: string;    // base64
  issuedAt?: string;
  notBefore?: string;
  notAfter?: string;
  revocation?: string;
};
```

---

# 4. Indexing Workflow (Step-by-Step)

## **Diagram B: Indexing Pipeline**

```
ATProto Firehose Stream
        │
        ▼
╔════════════════════════════════════╗
║     1. Filter for community.claim  ║
╚════════════════════════════════════╝
        │
        ▼
╔════════════════════════════════════╗
║     2. Parse & Normalize Record    ║
║   - strip sigs for hash checks     ║
║   - canonicalize JSON              ║
╚════════════════════════════════════╝
        │
        ▼
╔════════════════════════════════════╗
║     3. Validate Signatures         ║
║   - resolution of issuer DID       ║
║   - timestamp windows              ║
║   - recomputed signature           ║
╚════════════════════════════════════╝
        │
        ├──────────────► on failure: flag record
        ▼
╔════════════════════════════════════╗
║  4. Insert Claim into DB Index     ║
╚════════════════════════════════════╝
        │
        ▼
╔════════════════════════════════════╗
║  5. Version Graph Update           ║
║   - prevCid link                   ║
║   - recompute chain integrity      ║
╚════════════════════════════════════╝
```

---

# 5. Database Schema (Recommended)

Your AppView DB should have these tables:

---

## **Table: claims**

| column           | type      | notes                        |
| ---------------- | --------- | ---------------------------- |
| `uri`            | string PK | at://...                     |
| `cid`            | string    | content-address              |
| `subject`        | string    | DID or URI                   |
| `claim_type`     | string    | skill, impact, etc           |
| `content`        | text      | human-readable msg           |
| `evidence`       | text      | optional CID/URL             |
| `created_at`     | timestamp | claim created                |
| `effective_date` | timestamp | optional                     |
| `prev_cid`       | string    | link to previous             |
| `is_valid`       | boolean   | (after signature validation) |

---

## **Table: claim_attestations**

| column      | type      | notes                               |
| ----------- | --------- | ----------------------------------- |
| `claim_cid` | string    | FK to claims                        |
| `issuer`    | string    | DID                                 |
| `signature` | text      | base64                              |
| `issued_at` | timestamp | optional                            |
| `status`    | enum      | valid / expired / revoked / invalid |

---

## **Table: claim_versions**

| column     | type      | notes                   |
| ---------- | --------- | ----------------------- |
| `cid`      | string PK | claim version           |
| `prev_cid` | string    | pointer                 |
| `depth`    | int       | computed chain depth    |
| `root_cid` | string    | CID of original version |

---

# 6. Signature Validation Logic

### **Signature validation requires:**

```ts
function validateSignature(record, signatureBlock) {
  // 1. Remove sigs field
  const withoutSigs = removeSigs(record);

  // 2. Insert $sig object for canonical encoding
  const canonicalForm = {
    ...withoutSigs,
    $sig: { 
      issuer: signatureBlock.issuer,
      did: extractDid(record),
      collection: extractCollection(record),
      ...otherMetadata
    }
  };

  // 3. Encode via DAG-CBOR
  const bytes = dagCborEncode(canonicalForm);

  // 4. Verify using issuer’s public key
  const publicKey = resolveDidKey(signatureBlock.issuer);
  
  return verifySignature(publicKey, bytes, signatureBlock.signature);
}
```

### **Signature Failure Modes**

* issuer DID cannot be resolved
* signature mismatch
* timestamp outside `notBefore/notAfter`
* attestation revoked
* claim mutated (CID mismatch)

AppView should store failure reason.

---

# 7. Version Graph Resolution

When indexing:

```
if claim.prevCid exists:
    confirm that prevCid exists in database
    chain = buildVersionChain(cid)
    store rootCid and depth
```

### **Version Integrity Checks**

* Detect if updates invalidate attestations
* Signal to consumers when they are viewing a stale version
* Keep historical audit trail

---

# 8. Query API Surface

Your AppView should expose:

---

## 8.1 **Get all claims about a subject**

```
GET /x/community/claim/by-subject?did=did:alice
```

Response includes:

* all versions
* valid/invalid status
* attestations summary

---

## 8.2 **Get attestations for a specific claim**

```
GET /x/community/claim/attestations?cid=bafy...
```

---

## 8.3 **Get full version chain for a claim**

```
GET /x/community/claim/version-graph?cid=bafy...
```

---

## 8.4 **Trust-weighted summaries** (optional enhancement)

Return:

* count of attestations
* trust graph weight (based on issuer relationships)
* reputation score

---

# 9. Derived Views

AppViews can compute:

* “top validated skills” for a user
* “verified NGO impact” lists
* “disputed claims” flags
* “highly-attested claims”
* “claims with revoked attestations”

These become UI elements in consumer apps.

---

# 10. Best Practices

### ✔ Always store “raw record” and “normalized record”

For auditability.

### ✔ Treat signature verification as asynchronous

Attestations may arrive later.

### ✔ Index by CID AND URI

CID = immutable
URI = current pointer

### ✔ Cache issuer DID documents

Reduces validation latency.

---

# 11. Summary Diagram: End-to-End Trust Flow

```
User → Creates Claim → Signed → Stored in Repo
         │
         ▼
Other Users → Attest → Signature added
         │
         ▼
ATProto Firehose → AppView → Verify → Index
         │
         ▼
Clients Query Verified Claims & Trust Graphs
```
