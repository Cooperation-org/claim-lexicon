# Hosting the `community.claim` Lexicon

This guide explains how to host and use the `community.claim` lexicon in practice.

## Overview

ATProto lexicons are **not hosted at traditional URLs**. Instead, they're published as records in AT Protocol repositories and discovered via DNS.

## How Lexicon Discovery Works

```
1. Client sees NSID "community.claim"
2. Extracts domain authority: "community"
3. Looks up DNS TXT record: _lexicon.community
4. Gets DID from TXT record: did=did:plc:xxx
5. Resolves DID to find PDS
6. Fetches: at://did:plc:xxx/com.atproto.lexicon.schema/community.claim
```

## Option 1: Use Without DNS Authority (Development/Pilot)

For development and early adoption, you can use the lexicon **without setting up DNS authority**:

```typescript
import { Lexicons } from '@atproto/lexicon'

// Load lexicon directly in your code
const communityClaimLexicon = require('./lexicons/community/claim.json')

const lexicons = new Lexicons([communityClaimLexicon])

// Validate records before publishing
const claim = {
  subject: 'https://example.org/project',
  claimType: 'impact',
  statement: 'Delivered 500 water filters',
  createdAt: new Date().toISOString()
}

lexicons.assertValidRecord('community.claim', claim)

// Publish to user's repo (or server's repo)
await agent.com.atproto.repo.putRecord({
  repo: agent.session.did,
  collection: 'community.claim',
  rkey: TID.nextStr(),
  record: claim
})
```

**Note**: PDS servers will store records even if they don't recognize the lexicon. Validation happens in your application code.

## Option 2: Full DNS Authority Setup (Production)

For production use where other applications need to discover your lexicon:

### 1. Control a Domain

You need control over a domain that matches your NSID authority. For `community.claim`:
- Authority domain: `community`
- This would require controlling `community` TLD or getting an arrangement

More realistically, use a domain you control:
- `claims.linkedtrust.us` → NSID would be `us.linkedtrust.claims.claim`
- `claim.cooperation.org` → NSID would be `org.cooperation.claim.main`

### 2. Create DNS TXT Record

```
_lexicon.claim.cooperation.org  TXT  "did=did:plc:your-did-here"
```

### 3. Publish Lexicon to Your Repository

```typescript
const agent = new AtpAgent({ service: 'https://bsky.social' })
await agent.login({ identifier: 'your.handle', password: 'app-password' })

const lexiconDoc = require('./lexicons/community/claim.json')

await agent.com.atproto.repo.putRecord({
  repo: agent.session.did,
  collection: 'com.atproto.lexicon.schema',
  rkey: 'community.claim',  // or your actual NSID
  record: lexiconDoc
})
```

## Using the Lexicon in Code

### Installation

```bash
npm install @atproto/api @atproto/lexicon
```

### Basic Usage

```typescript
import { AtpAgent } from '@atproto/api'
import { Lexicons } from '@atproto/lexicon'

// Load lexicon
const claimLexicon = require('./lexicons/community/claim.json')
const lexicons = new Lexicons([claimLexicon])

// Create agent
const agent = new AtpAgent({ service: 'https://bsky.social' })

// Login (for publishing)
await agent.login({
  identifier: 'your.handle',
  password: 'your-app-password'
})

// Create and validate a claim
const claim = {
  subject: 'did:plc:alice',
  claimType: 'skill',
  object: 'React',
  statement: 'Experienced React developer',
  source: {
    howKnown: 'FIRST_HAND'
  },
  createdAt: new Date().toISOString()
}

// Validate
lexicons.assertValidRecord('community.claim', claim)

// Publish
const result = await agent.com.atproto.repo.createRecord({
  repo: agent.session.did,
  collection: 'community.claim',
  record: claim
})

console.log('Published at:', result.uri)
// at://did:plc:xxx/community.claim/3kfxyz...
```

### With External Signatures (MetaMask)

```typescript
// User signs with MetaMask first
const signature = await signWithMetaMask(claimData)

// Add embedded proof
const claimWithProof = {
  ...claimData,
  embeddedProof: {
    type: 'EthereumEip712Signature2021',
    verificationMethod: `did:pkh:eip155:1:${userAddress}`,
    proofValue: signature,
    proofPurpose: 'assertionMethod',
    created: new Date().toISOString()
  }
}

// Validate and publish (server publishes on behalf of user)
lexicons.assertValidRecord('community.claim', claimWithProof)
await serverAgent.com.atproto.repo.createRecord({
  repo: serverAgent.session.did,  // Server's repo
  collection: 'community.claim',
  record: claimWithProof
})
```

### Reading Claims

```typescript
// Get a specific claim
const { data } = await agent.com.atproto.repo.getRecord({
  repo: 'did:plc:xxx',
  collection: 'community.claim',
  rkey: '3kfxyz'
})

// List all claims in a repo
const { data: list } = await agent.com.atproto.repo.listRecords({
  repo: 'did:plc:xxx',
  collection: 'community.claim',
  limit: 50
})
```

## Firehose Subscription (for Indexers)

To build an AppView that indexes all `community.claim` records:

```typescript
import { Firehose } from '@atproto/sync'

const firehose = new Firehose({
  service: 'wss://bsky.network'
})

firehose.on('commit', (commit) => {
  for (const op of commit.ops) {
    if (op.path.startsWith('community.claim/')) {
      if (op.action === 'create') {
        // Index new claim
        indexClaim(commit.repo, op.path, op.cid, op.record)
      } else if (op.action === 'delete') {
        // Mark as deleted
        markDeleted(commit.repo, op.path)
      }
    }
  }
})

firehose.start()
```

## Code Generation (Optional)

Generate TypeScript types from your lexicon:

```bash
npx @atproto/lex gen-api ./src/generated ./lexicons/**/*.json
```

This creates typed interfaces and validation functions.

## Current Status

The `community.claim` NSID uses `community` as the authority domain. Since `community` is not a standard TLD, there are options:

1. **Use as-is**: Many apps bundle lexicons directly and don't rely on DNS resolution
2. **Request official community namespace**: Work with ATProto team on community lexicons
3. **Use your own domain**: Rename to something like `org.cooperation.claim`

For the LinkedTrust pilot, option 1 (bundled lexicon) is recommended.

## References

- [ATProto Lexicon Spec](https://atproto.com/specs/lexicon)
- [Custom Schemas Guide](https://docs.bsky.app/docs/advanced-guides/custom-schemas)
- [@atproto/lexicon package](https://www.npmjs.com/package/@atproto/lexicon)
