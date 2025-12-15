# Claim Lexicon

A minimal ATProto lexicon for verifiable claims, implementing the [LinkedClaims](https://github.com/decentralized-identity/labs-linkedclaims) specification.

## Documents

1. [proposal.md](proposal.md) - Initial proposal and rationale
2. [specification-draft.md](specification-draft.md) - Technical specification
3. [appview-indexing-guide.md](appview-indexing-guide.md) - Implementation guide for AppViews
4. [HOSTING.md](HOSTING.md) - How to host and use this lexicon

## Lexicon Files

The actual lexicon JSON is in:
- [`lexicons/community/claim.json`](lexicons/community/claim.json)

## Quick Start

```typescript
import { Lexicons } from '@atproto/lexicon'
import { AtpAgent } from '@atproto/api'

// Load lexicon
const claimLexicon = require('./lexicons/community/claim.json')
const lexicons = new Lexicons([claimLexicon])

// Create a claim
const claim = {
  subject: 'https://example.org/project',
  claimType: 'impact',
  statement: 'Delivered 500 water filters',
  source: { howKnown: 'FIRST_HAND' },
  createdAt: new Date().toISOString()
}

// Validate
lexicons.assertValidRecord('community.claim', claim)

// Publish to ATProto
const agent = new AtpAgent({ service: 'https://bsky.social' })
await agent.login({ identifier: 'handle', password: 'app-password' })
await agent.com.atproto.repo.createRecord({
  repo: agent.session.did,
  collection: 'community.claim',
  record: claim
})
```

## Key Design Decisions

### Claims are Immutable
No versioning (`prevCid` removed). To update or correct a claim, create a new claim that references the original.

### Attestations are Just Claims
No special `sigs` field. An endorsement or dispute is a claim where the `subject` is another claim's AT-URI.

### External Signatures Supported
The `embeddedProof` field allows claims signed by MetaMask, aca-py, or other tools to be published on ATProto while preserving the original signer identity.

## Example Claims

**Skill claim:**
```json
{
  "subject": "did:plc:alice",
  "claimType": "skill",
  "object": "React",
  "statement": "3 years production experience",
  "createdAt": "2025-02-14T18:22:00Z"
}
```

**Endorsement of another claim:**
```json
{
  "subject": "at://did:plc:alice/community.claim/3kfxyz",
  "claimType": "endorsement",
  "statement": "I can confirm this is accurate",
  "source": { "howKnown": "FIRST_HAND" },
  "createdAt": "2025-02-15T10:00:00Z"
}
```

**Claim with external signature:**
```json
{
  "subject": "did:plc:bob",
  "claimType": "endorsement",
  "statement": "Trustworthy collaborator",
  "createdAt": "2025-02-14T18:30:00Z",
  "embeddedProof": {
    "type": "EthereumEip712Signature2021",
    "verificationMethod": "did:pkh:eip155:1:0x1234...",
    "proofValue": "0xabc123...",
    "proofPurpose": "assertionMethod",
    "created": "2025-02-14T18:30:00Z"
  }
}
```

## Related Projects

- [LinkedTrust](https://github.com/Cooperation-org/LinkedTrust) - Reference implementation
- [LinkedClaims Spec](https://github.com/decentralized-identity/labs-linkedclaims) - DIF specification
- [cooperation.org/credentials/v1](https://cooperation.org/credentials/v1/) - JSON-LD vocabulary

## License

MIT
