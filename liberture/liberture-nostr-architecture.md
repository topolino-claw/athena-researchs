# Liberture Nostr-Native Architecture Specification

**Version:** 1.0  
**Date:** 2026-03-03  
**Author:** Athena (Research Agent)  
**For:** Adonis (Implementation) + Fabri (Review)

---

## Executive Summary

This document specifies the complete architecture for making Liberture's content layer **Nostr-native**. Nostr becomes the canonical source of truth for all content (protocols, knowledge articles, people, books, organizations), while Postgres serves as a cache/SEO layer with WoT-based curation.

**Key architectural decisions:**
- **Kind 30818** for all wiki-style content (protocols, knowledge articles) — multiple authors can publish competing versions
- **Kind 38381/38382** for reviews and replies (mirroring mappingbitcoin pattern)
- **Kind 0** for people profiles (standard Nostr metadata)
- **Custom application tags** within events for liberture-specific taxonomy (pillars, categories)
- **WoT oracle** at `wot-oracle.mappingbitcoin.com` determines content surfacing and auto-publish thresholds
- **Prisma/Postgres** as cache layer for fast queries, full-text search, and SEO

---

## Table of Contents

1. [Nostr Event Schema](#1-nostr-event-schema)
2. [Relay Strategy](#2-relay-strategy)
3. [Indexer / Cache Pipeline](#3-indexer--cache-pipeline)
4. [WoT Integration](#4-wot-integration)
5. [Reviews Architecture](#5-reviews-architecture)
6. [Database Schema Changes](#6-database-schema-changes)
7. [Migration Strategy](#7-migration-strategy)
8. [SEO Implications](#8-seo-implications)
9. [Implementation Checklist](#9-implementation-checklist)

---

## 1. Nostr Event Schema

### 1.1 Protocols (Kind 30818)

Protocols are the core content type in Liberture. Each protocol is a **parameterized replaceable event** (NIP-54 wiki format).

**Why 30818:** Addressable events where `pubkey + kind + d-tag` = unique address. Multiple authors can publish competing versions of the same protocol (e.g., different "cold-exposure-protocol" versions). WoT determines which surfaces first.

**Content format:** AsciiDoc (per NIP-54 spec) with wikilinks support.

```json
{
  "kind": 30818,
  "pubkey": "<author-pubkey-hex>",
  "created_at": 1709487600,
  "tags": [
    ["d", "cold-exposure-protocol"],
    ["title", "Cold Exposure Protocol"],
    ["summary", "A protocol for deliberate cold exposure to improve metabolic health and resilience"],
    ["pillar", "exercise"],
    ["category", "recovery"],
    ["difficulty", "intermediate"],
    ["duration", "10-15 minutes"],
    ["equipment", "cold shower, ice bath, or cold plunge"],
    ["t", "cold-exposure"],
    ["t", "hormesis"],
    ["t", "recovery"],
    ["L", "liberture.pillar"],
    ["l", "exercise", "liberture.pillar"],
    ["published", "true"]
  ],
  "content": "= Cold Exposure Protocol\n\n== Overview\n\nDeliberate cold exposure triggers hormetic stress responses...\n\n== Steps\n\n1. Start with cold water at the end of your shower (30 seconds)\n2. Gradually increase duration over 2 weeks\n3. Progress to full cold showers (2-3 minutes)\n4. Optional: Ice baths at 10-15°C for 3-10 minutes\n\n== Benefits\n\n* Increased dopamine and norepinephrine\n* Improved insulin sensitivity\n* Enhanced brown fat activation\n* Mental resilience\n\n== Risks\n\n* Hypothermia risk with extended exposure\n* Contraindicated for cardiovascular conditions\n\n== References\n\n* [[huberman-cold-protocol|Huberman Lab Episode on Cold Exposure]]\n* nostr:npub1... (link to Susanna Søberg's profile)\n",
  "sig": "<signature>"
}
```

**Required tags:**
| Tag | Description | Example |
|-----|-------------|---------|
| `d` | Normalized identifier (slug) | `cold-exposure-protocol` |
| `title` | Display title | `Cold Exposure Protocol` |
| `pillar` | Liberture pillar | `sleep`, `exercise`, `nutrition`, `mind`, `work`, `finance` |

**Optional tags:**
| Tag | Description |
|-----|-------------|
| `summary` | Short description for lists |
| `category` | Sub-category within pillar |
| `difficulty` | `beginner`, `intermediate`, `advanced` |
| `duration` | Time estimate |
| `equipment` | Required equipment |
| `t` | Hashtags for discoverability |
| `L` | Label namespace (NIP-32) |
| `l` | Label value |
| `a` | Fork reference (if forked from another protocol) |
| `e` | Specific version fork reference |
| `published` | Author's intent (`true`/`false`) — NOT the same as cache `published` |

**d-tag normalization rules (per NIP-54):**
- Lowercase all letters
- Replace whitespace with `-`
- Remove punctuation/symbols
- Collapse multiple `-` to single
- Trim leading/trailing `-`
- Preserve non-ASCII UTF-8

### 1.2 Knowledge Articles (Kind 30818)

Knowledge articles also use **Kind 30818**. They're wiki entries that aren't actionable protocols but provide background knowledge.

**Differentiation via tags:** Use `["type", "knowledge"]` to distinguish from protocols.

```json
{
  "kind": 30818,
  "pubkey": "<author-pubkey-hex>",
  "created_at": 1709487600,
  "tags": [
    ["d", "circadian-rhythm"],
    ["title", "Circadian Rhythm"],
    ["summary", "The body's internal 24-hour clock that regulates sleep-wake cycles"],
    ["type", "knowledge"],
    ["pillar", "sleep"],
    ["t", "sleep"],
    ["t", "chronobiology"],
    ["L", "liberture.type"],
    ["l", "knowledge", "liberture.type"]
  ],
  "content": "= Circadian Rhythm\n\n== Definition\n\nThe circadian rhythm is a natural, internal process that regulates the sleep-wake cycle...\n\n== Key Concepts\n\n=== Zeitgebers\n\nExternal cues that synchronize the circadian clock:\n\n* Light (most powerful)\n* Temperature\n* Meal timing\n* Social interaction\n\n== Related Protocols\n\n* [[morning-sunlight-protocol]]\n* [[sleep-hygiene-protocol]]\n",
  "sig": "<signature>"
}
```

### 1.3 People Profiles (Kind 0)

People (researchers, authors, experts) use **standard Nostr Kind 0** metadata events. Liberture doesn't define custom kinds for people — it indexes existing profiles.

```json
{
  "kind": 0,
  "pubkey": "<person-pubkey-hex>",
  "created_at": 1709487600,
  "tags": [],
  "content": "{\"name\":\"Andrew Huberman\",\"about\":\"Neuroscientist at Stanford. Host of Huberman Lab podcast.\",\"picture\":\"https://example.com/huberman.jpg\",\"nip05\":\"huberman@hubermanlab.com\",\"website\":\"https://hubermanlab.com\",\"lud16\":\"huberman@getalby.com\"}",
  "sig": "<signature>"
}
```

**Liberture-specific indexing:** The indexer watches for Kind 0 events from pubkeys that:
1. Have authored protocols on Liberture
2. Are referenced in protocols via `p` tags
3. Are in a curated "people to index" list (Kind 30000 or similar)

**Extended profile data:** If Liberture needs extra fields (credentials, specialties), create **Kind 30000** (curated list) to tag people with metadata:

```json
{
  "kind": 30000,
  "pubkey": "<liberture-bot-pubkey>",
  "tags": [
    ["d", "liberture-experts"],
    ["p", "<huberman-pubkey>", "wss://relay.example.com", "Andrew Huberman"],
    ["specialty", "<huberman-pubkey>", "neuroscience"],
    ["specialty", "<huberman-pubkey>", "sleep"],
    ["credentials", "<huberman-pubkey>", "PhD Stanford"]
  ],
  "content": "",
  "sig": "<signature>"
}
```

### 1.4 Books (Kind 30818)

Books are wiki entries describing books relevant to human optimization.

```json
{
  "kind": 30818,
  "pubkey": "<author-pubkey-hex>",
  "created_at": 1709487600,
  "tags": [
    ["d", "why-we-sleep-matthew-walker"],
    ["title", "Why We Sleep"],
    ["summary", "Matthew Walker's comprehensive guide to the science of sleep"],
    ["type", "book"],
    ["pillar", "sleep"],
    ["author", "Matthew Walker"],
    ["isbn", "978-1501144318"],
    ["published_year", "2017"],
    ["t", "sleep"],
    ["t", "book"],
    ["L", "liberture.type"],
    ["l", "book", "liberture.type"]
  ],
  "content": "= Why We Sleep\n\n== Overview\n\nMatthew Walker's seminal work on sleep science...\n\n== Key Takeaways\n\n* Sleep is the foundation of health\n* 8 hours is non-negotiable for most adults\n* Sleep debt cannot be repaid\n\n== Related Protocols\n\n* [[sleep-hygiene-protocol]]\n* [[caffeine-cutoff-protocol]]\n",
  "sig": "<signature>"
}
```

### 1.5 Organizations (Kind 30818)

Organizations (research institutions, companies, communities) are also wiki entries.

```json
{
  "kind": 30818,
  "pubkey": "<author-pubkey-hex>",
  "created_at": 1709487600,
  "tags": [
    ["d", "huberman-lab"],
    ["title", "Huberman Lab"],
    ["summary", "Science-based podcast and media company focused on neuroscience"],
    ["type", "organization"],
    ["category", "media"],
    ["website", "https://hubermanlab.com"],
    ["p", "<huberman-pubkey>", "", "founder"],
    ["t", "podcast"],
    ["t", "neuroscience"],
    ["L", "liberture.type"],
    ["l", "organization", "liberture.type"]
  ],
  "content": "= Huberman Lab\n\n== About\n\nHuberman Lab is a science-based media company...\n",
  "sig": "<signature>"
}
```

### 1.6 Tag Namespace Summary

Liberture uses the following custom tag patterns within Kind 30818:

| Tag | Values | Purpose |
|-----|--------|---------|
| `pillar` | `work`, `sleep`, `nutrition`, `mind`, `exercise`, `finance` | Primary categorization |
| `type` | `protocol`, `knowledge`, `book`, `organization` | Content type (default: `protocol`) |
| `category` | Free-form sub-category | Secondary categorization |
| `difficulty` | `beginner`, `intermediate`, `advanced` | Protocol difficulty |
| `duration` | Free-form time string | Protocol time estimate |
| `equipment` | Free-form | Required equipment |
| `L` + `l` | NIP-32 labels | Machine-readable taxonomy |

**NIP-32 Label Pattern:**
```
["L", "liberture.pillar"]
["l", "sleep", "liberture.pillar"]
["L", "liberture.type"]
["l", "protocol", "liberture.type"]
```

---

## 2. Relay Strategy

### 2.1 Own Relay: Yes

Liberture SHOULD run its own relay for:
- **Guaranteed availability** of liberture-published content
- **Custom filtering** (only accept liberture-relevant kinds)
- **Write permissions** controlled (prevent spam)
- **Faster indexing** (subscribe locally)

**Recommended setup:**
- **strfry** or **nostream** relay implementation
- Hosted on same infra as liberture backend
- Write-restricted: only accept from whitelisted pubkeys OR require NIP-42 auth
- Read-open: anyone can fetch liberture content

**Relay URL:** `wss://relay.liberture.com`

### 2.2 Public Relay Subscription

The indexer subscribes to multiple relays to discover content from the broader community:

**Primary relays (high reliability):**
```
wss://relay.damus.io
wss://relay.nostr.band
wss://nos.lol
wss://relay.snort.social
wss://nostr.wine (paid, high quality)
```

**Wiki-specific relays:**
```
wss://wikifreedia.xyz (if running, from wikistr)
```

### 2.3 Outbox Model (NIP-65)

For reading content **from** a specific author:
1. Fetch author's Kind 10002 (relay list)
2. Extract their `write` relays
3. Query those relays for their content

**Implementation:**
```typescript
async function getAuthorRelays(pubkey: string): Promise<string[]> {
  const relayListEvent = await pool.get(
    defaultRelays,
    { kinds: [10002], authors: [pubkey] }
  );
  if (!relayListEvent) return defaultRelays;
  
  return relayListEvent.tags
    .filter(t => t[0] === 'r' && (!t[2] || t[2] === 'write'))
    .map(t => t[1]);
}
```

### 2.4 Subscription Filters

**Primary subscription (protocols + knowledge):**
```json
{
  "kinds": [30818],
  "#L": ["liberture.pillar"],
  "since": <last_sync_timestamp>
}
```

**Alternative broad subscription:**
```json
{
  "kinds": [30818],
  "#t": ["sleep", "nutrition", "exercise", "biohacking", "productivity", "finance", "mental-health"],
  "since": <last_sync_timestamp>
}
```

**Reviews subscription:**
```json
{
  "kinds": [38381, 38382],
  "#a": ["30818:*"],
  "since": <last_sync_timestamp>
}
```

**Profile subscription (for indexed authors):**
```json
{
  "kinds": [0],
  "authors": [<list_of_known_author_pubkeys>],
  "since": <last_sync_timestamp>
}
```

---

## 3. Indexer / Cache Pipeline

### 3.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         Nostr Relays                            │
│   relay.liberture.com | relay.damus.io | nos.lol | ...          │
└─────────────────────────────┬───────────────────────────────────┘
                              │ WebSocket subscriptions
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Liberture Indexer Service                    │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────────┐  │
│  │ Relay Pool  │──│ Event Router │──│ WoT Score Enrichment   │  │
│  │ (nostr-tools│  │              │  │ (HTTP to wot-oracle)   │  │
│  │  SimplePool)│  │              │  │                        │  │
│  └─────────────┘  └──────────────┘  └────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    Prisma Upsert Layer                    │  │
│  │  - Deduplication (eventId unique)                         │  │
│  │  - Version replacement (pubkey + kind + d → latest wins)  │  │
│  │  - Published flag logic (WoT threshold)                   │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      PostgreSQL (Prisma)                        │
│   Protocol | KnowledgeArticle | Person | Book | Org | Review    │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Indexer Service Implementation

**Technology:** Node.js/TypeScript with `nostr-tools`

```typescript
// indexer/src/main.ts
import { SimplePool, Event } from 'nostr-tools';
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();
const pool = new SimplePool();

const RELAYS = [
  'wss://relay.liberture.com',
  'wss://relay.damus.io',
  'wss://nos.lol',
  'wss://relay.nostr.band',
];

const WOT_ORACLE_URL = 'https://wot-oracle.mappingbitcoin.com';
const LIBERTURE_ROOT_PUBKEY = '<fabri-pubkey-or-liberture-bot-pubkey>';
const WOT_PUBLISH_THRESHOLD = 3; // Minimum hops to auto-publish

async function main() {
  // Subscribe to protocols and knowledge articles
  const sub = pool.subscribeMany(
    RELAYS,
    [
      { kinds: [30818], since: await getLastSyncTimestamp() },
      { kinds: [38381, 38382], since: await getLastSyncTimestamp() },
      { kinds: [0], authors: await getKnownAuthorPubkeys() },
    ],
    {
      onevent: handleEvent,
      oneose: () => console.log('Initial sync complete'),
    }
  );

  // Keep alive
  process.on('SIGINT', () => {
    sub.close();
    prisma.$disconnect();
    process.exit(0);
  });
}

async function handleEvent(event: Event) {
  try {
    switch (event.kind) {
      case 30818:
        await handleWikiEvent(event);
        break;
      case 38381:
        await handleReviewEvent(event);
        break;
      case 38382:
        await handleReviewReplyEvent(event);
        break;
      case 0:
        await handleProfileEvent(event);
        break;
    }
  } catch (error) {
    console.error(`Error handling event ${event.id}:`, error);
  }
}

async function handleWikiEvent(event: Event) {
  const dTag = event.tags.find(t => t[0] === 'd')?.[1];
  if (!dTag) return;

  const type = event.tags.find(t => t[0] === 'type')?.[1] || 'protocol';
  const pillar = event.tags.find(t => t[0] === 'pillar')?.[1];
  
  // Skip if not liberture-relevant
  if (!pillar && !event.tags.some(t => t[0] === 'L' && t[1]?.startsWith('liberture'))) {
    return;
  }

  // Get WoT score
  const wotScore = await getWotScore(event.pubkey);
  const shouldPublish = wotScore !== null && wotScore <= WOT_PUBLISH_THRESHOLD;

  // Upsert based on type
  const commonData = {
    nostrEventId: event.id,
    authorPubkey: event.pubkey,
    nostrDTag: dTag,
    wotScore,
    published: shouldPublish,
    title: event.tags.find(t => t[0] === 'title')?.[1] || dTag,
    summary: event.tags.find(t => t[0] === 'summary')?.[1],
    pillar,
    content: event.content,
    rawTags: JSON.stringify(event.tags),
    nostrCreatedAt: new Date(event.created_at * 1000),
    updatedAt: new Date(),
  };

  if (type === 'protocol') {
    await upsertProtocol(event, commonData);
  } else if (type === 'knowledge') {
    await upsertKnowledgeArticle(event, commonData);
  } else if (type === 'book') {
    await upsertBook(event, commonData);
  } else if (type === 'organization') {
    await upsertOrganization(event, commonData);
  }
}

async function upsertProtocol(event: Event, data: any) {
  // Unique constraint: authorPubkey + nostrDTag
  await prisma.protocol.upsert({
    where: {
      authorPubkey_nostrDTag: {
        authorPubkey: data.authorPubkey,
        nostrDTag: data.nostrDTag,
      },
    },
    update: {
      ...data,
      // Only update if this event is newer
    },
    create: {
      id: event.id,
      slug: `${data.nostrDTag}-${data.authorPubkey.slice(0, 8)}`,
      ...data,
      difficulty: event.tags.find(t => t[0] === 'difficulty')?.[1] || 'intermediate',
      duration: event.tags.find(t => t[0] === 'duration')?.[1],
      equipment: event.tags.find(t => t[0] === 'equipment')?.[1],
    },
  });
}

async function getWotScore(pubkey: string): Promise<number | null> {
  try {
    const response = await fetch(
      `${WOT_ORACLE_URL}/api/v1/distance?source=${LIBERTURE_ROOT_PUBKEY}&target=${pubkey}`
    );
    if (!response.ok) return null;
    const data = await response.json();
    return data.distance ?? null;
  } catch {
    return null;
  }
}

async function getLastSyncTimestamp(): Promise<number> {
  const latest = await prisma.syncState.findFirst({
    orderBy: { lastSync: 'desc' },
  });
  return latest ? Math.floor(latest.lastSync.getTime() / 1000) : 0;
}

main().catch(console.error);
```

### 3.3 Deduplication Strategy

**Event-level deduplication:**
- `nostrEventId` is unique — same event from multiple relays is ignored
- Check `event.id` before processing

**Version replacement:**
- For addressable events (30818), the combination `authorPubkey + nostrDTag` is unique
- Only the **latest** event (by `created_at`) is kept
- On upsert, compare `created_at` and only update if newer

```typescript
async function shouldUpdate(
  existingCreatedAt: Date,
  newCreatedAt: number
): boolean {
  return newCreatedAt > existingCreatedAt.getTime() / 1000;
}
```

### 3.4 Published Flag Logic

```typescript
const WOT_THRESHOLDS = {
  AUTO_PUBLISH: 3,      // Distance ≤ 3 hops → auto-publish
  MODERATION_QUEUE: 6,  // Distance 4-6 → moderation queue
  REJECT: 7,            // Distance > 6 → not indexed (or indexed but never published)
};

function determinePublishStatus(wotScore: number | null): {
  published: boolean;
  moderationRequired: boolean;
} {
  if (wotScore === null) {
    return { published: false, moderationRequired: true };
  }
  if (wotScore <= WOT_THRESHOLDS.AUTO_PUBLISH) {
    return { published: true, moderationRequired: false };
  }
  if (wotScore <= WOT_THRESHOLDS.MODERATION_QUEUE) {
    return { published: false, moderationRequired: true };
  }
  return { published: false, moderationRequired: false };
}
```

---

## 4. WoT Integration

### 4.1 WoT Oracle API

The existing `wot-oracle.mappingbitcoin.com` provides:

**Endpoint:** `GET /api/v1/distance?source=<pubkey>&target=<pubkey>`

**Response:**
```json
{
  "source": "<source-pubkey>",
  "target": "<target-pubkey>",
  "distance": 2,
  "path": ["<source>", "<intermediate>", "<target>"]
}
```

**Distance meanings:**
- `0` — Same pubkey (self)
- `1` — Direct follow
- `2` — Friend-of-friend
- `3` — 3 hops
- `null` — Not connected / unreachable

### 4.2 Trust Seed (Root Pubkey)

**Option A: Fabri's personal npub**
- Pro: Aligns with Fabri's existing WoT
- Con: Mixes personal social graph with liberture curation

**Option B: Dedicated liberture bot pubkey**
- Pro: Clean separation, can curate specifically for liberture relevance
- Con: Needs to build its own follow graph

**Recommendation:** Start with **Option A** (Fabri's npub), with a plan to transition to **Option B** once liberture has established credibility. The bot pubkey can follow the same experts Fabri would trust for health/bio content.

**Liberture root pubkey setup:**
1. Generate dedicated keypair for liberture
2. Follow ~20-50 seed pubkeys (Huberman, Attia, known biohackers, etc.)
3. Use this as `LIBERTURE_ROOT_PUBKEY` in the indexer

### 4.3 Score Thresholds

| WoT Distance | Action | UI Treatment |
|--------------|--------|--------------|
| 0-2 | Auto-publish, featured eligible | ✅ Green trust badge |
| 3 | Auto-publish | ✅ Trust badge (subtle) |
| 4-5 | Moderation queue | ⏳ "Pending review" |
| 6+ | Index but don't publish | Hidden from public |
| null | Index but don't publish | Hidden from public |

### 4.4 UI Trust Display

```typescript
function getTrustBadge(wotScore: number | null): {
  label: string;
  color: string;
  show: boolean;
} {
  if (wotScore === null) {
    return { label: 'Unknown', color: 'gray', show: false };
  }
  if (wotScore <= 2) {
    return { label: 'Highly Trusted', color: 'green', show: true };
  }
  if (wotScore <= 3) {
    return { label: 'Trusted', color: 'blue', show: true };
  }
  if (wotScore <= 5) {
    return { label: 'Community', color: 'yellow', show: true };
  }
  return { label: 'Unverified', color: 'gray', show: false };
}
```

**UI toggle:** Allow users to "Show all versions" which reveals lower-WoT versions of a protocol. Default view shows highest-WoT version only.

---

## 5. Reviews Architecture

### 5.1 Protocol Review (Kind 38381)

Mirroring the mappingbitcoin pattern:

```json
{
  "kind": 38381,
  "pubkey": "<reviewer-pubkey>",
  "created_at": 1709487600,
  "tags": [
    ["d", "<unique-review-id>"],
    ["a", "30818:<protocol-author-pubkey>:<protocol-d-tag>", "wss://relay.liberture.com"],
    ["rating", "5"],
    ["L", "reviews"],
    ["l", "protocol-review", "reviews"],
    ["pillar", "sleep"]
  ],
  "content": "This protocol changed my sleep quality dramatically. After 2 weeks of consistent application, I went from 6.5 to 8+ hours of quality sleep. The steps are clear and easy to follow. Highly recommended for anyone struggling with sleep onset.",
  "sig": "<signature>"
}
```

**Required tags:**
| Tag | Description |
|-----|-------------|
| `d` | Unique identifier for this review (can be UUID or `<protocol-d-tag>-<timestamp>`) |
| `a` | Reference to the protocol being reviewed |
| `rating` | Star rating 1-5 |

**Optional tags:**
| Tag | Description |
|-----|-------------|
| `L` + `l` | Label namespace |
| `pillar` | Pillar context |
| `t` | Hashtags |
| `imeta` | Image metadata (if including photos) |

### 5.2 Review Reply (Kind 38382)

Protocol authors can reply to reviews:

```json
{
  "kind": 38382,
  "pubkey": "<protocol-author-pubkey>",
  "created_at": 1709487700,
  "tags": [
    ["e", "<review-event-id>", "wss://relay.liberture.com"],
    ["p", "<reviewer-pubkey>"],
    ["a", "30818:<protocol-author-pubkey>:<protocol-d-tag>"]
  ],
  "content": "Thank you for the detailed feedback! Glad to hear it's working well. For even better results, try combining with the morning-sunlight-protocol.",
  "sig": "<signature>"
}
```

**Verification:** Only the author of the reviewed protocol can have their reply displayed as "Author Reply". Check `event.pubkey === protocol.authorPubkey`.

### 5.3 WoT-Weighted Aggregate Ratings

```typescript
interface WeightedRating {
  averageRating: number;
  totalReviews: number;
  weightedAverage: number;
  confidence: number;
}

async function calculateWeightedRating(
  protocolId: string
): Promise<WeightedRating> {
  const reviews = await prisma.review.findMany({
    where: { protocolId },
    select: { rating: true, reviewerPubkey: true, wotScore: true },
  });

  if (reviews.length === 0) {
    return { averageRating: 0, totalReviews: 0, weightedAverage: 0, confidence: 0 };
  }

  // Weight by inverse WoT distance (closer = more weight)
  let totalWeight = 0;
  let weightedSum = 0;

  for (const review of reviews) {
    const weight = getReviewWeight(review.wotScore);
    weightedSum += review.rating * weight;
    totalWeight += weight;
  }

  const weightedAverage = totalWeight > 0 ? weightedSum / totalWeight : 0;
  const confidence = Math.min(reviews.length / 10, 1); // 0-1 based on review count

  return {
    averageRating: reviews.reduce((sum, r) => sum + r.rating, 0) / reviews.length,
    totalReviews: reviews.length,
    weightedAverage,
    confidence,
  };
}

function getReviewWeight(wotScore: number | null): number {
  if (wotScore === null) return 0.1;
  if (wotScore <= 1) return 1.0;
  if (wotScore <= 2) return 0.8;
  if (wotScore <= 3) return 0.6;
  if (wotScore <= 4) return 0.4;
  return 0.2;
}
```

---

## 6. Database Schema Changes

### 6.1 Complete Updated Prisma Schema

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ============================================
// PROTOCOLS
// ============================================

model Protocol {
  id              String    @id @default(uuid())
  
  // Nostr-native fields
  nostrEventId    String    @unique
  authorPubkey    String
  nostrDTag       String
  nostrCreatedAt  DateTime
  rawTags         String    // JSON string of all tags
  
  // Derived/display fields
  slug            String    @unique
  title           String
  summary         String?
  content         String    // AsciiDoc content
  
  // Taxonomy
  pillar          String    // sleep, exercise, nutrition, mind, work, finance
  category        String?
  
  // Protocol-specific
  difficulty      String    @default("intermediate") // beginner, intermediate, advanced
  duration        String?
  equipment       String?
  
  // WoT + Curation
  wotScore        Int?      // Distance from liberture root (null = unreachable)
  published       Boolean   @default(false)
  featured        Boolean   @default(false)
  moderationStatus String   @default("pending") // pending, approved, rejected
  
  // Aggregates (cached from reviews)
  averageRating   Float     @default(0)
  reviewCount     Int       @default(0)
  weightedRating  Float     @default(0)
  
  // Timestamps
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
  
  // Relations
  reviews         Review[]
  author          Person?   @relation(fields: [authorPubkey], references: [pubkey])
  
  // Indexes
  @@unique([authorPubkey, nostrDTag])
  @@index([pillar])
  @@index([published])
  @@index([wotScore])
  @@index([nostrCreatedAt])
}

// ============================================
// KNOWLEDGE ARTICLES
// ============================================

model KnowledgeArticle {
  id              String    @id @default(uuid())
  
  // Nostr-native fields
  nostrEventId    String    @unique
  authorPubkey    String
  nostrDTag       String
  nostrCreatedAt  DateTime
  rawTags         String
  
  // Derived/display fields
  slug            String    @unique
  title           String
  summary         String?
  content         String
  
  // Taxonomy
  pillar          String
  category        String?
  
  // WoT + Curation
  wotScore        Int?
  published       Boolean   @default(false)
  
  // Timestamps
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
  
  // Relations
  author          Person?   @relation(fields: [authorPubkey], references: [pubkey])
  
  @@unique([authorPubkey, nostrDTag])
  @@index([pillar])
  @@index([published])
}

// ============================================
// PEOPLE (Authors/Experts)
// ============================================

model Person {
  pubkey          String    @id
  
  // From Kind 0
  name            String?
  about           String?
  picture         String?
  nip05           String?
  website         String?
  lud16           String?
  
  // Extended (from liberture curation)
  specialties     String[]  // e.g., ["sleep", "neuroscience"]
  credentials     String?
  
  // WoT
  wotScore        Int?
  
  // Nostr metadata
  nostrEventId    String?
  nostrCreatedAt  DateTime?
  
  // Timestamps
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
  
  // Relations
  protocols       Protocol[]
  articles        KnowledgeArticle[]
  books           Book[]
  reviews         Review[]
  
  @@index([wotScore])
}

// ============================================
// BOOKS
// ============================================

model Book {
  id              String    @id @default(uuid())
  
  // Nostr-native fields
  nostrEventId    String    @unique
  authorPubkey    String
  nostrDTag       String
  nostrCreatedAt  DateTime
  rawTags         String
  
  // Derived/display fields
  slug            String    @unique
  title           String
  summary         String?
  content         String
  
  // Book-specific
  bookAuthor      String?   // Author of the book (not Nostr event author)
  isbn            String?
  publishedYear   String?
  
  // Taxonomy
  pillar          String?
  
  // WoT + Curation
  wotScore        Int?
  published       Boolean   @default(false)
  
  // Timestamps
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
  
  // Relations
  contributor     Person?   @relation(fields: [authorPubkey], references: [pubkey])
  
  @@unique([authorPubkey, nostrDTag])
}

// ============================================
// ORGANIZATIONS
// ============================================

model Organization {
  id              String    @id @default(uuid())
  
  // Nostr-native fields
  nostrEventId    String    @unique
  authorPubkey    String
  nostrDTag       String
  nostrCreatedAt  DateTime
  rawTags         String
  
  // Derived/display fields
  slug            String    @unique
  title           String
  summary         String?
  content         String
  website         String?
  
  // Taxonomy
  category        String?
  
  // WoT + Curation
  wotScore        Int?
  published       Boolean   @default(false)
  
  // Timestamps
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
  
  @@unique([authorPubkey, nostrDTag])
}

// ============================================
// REVIEWS
// ============================================

model Review {
  id              String    @id @default(uuid())
  
  // Nostr-native fields
  nostrEventId    String    @unique
  reviewerPubkey  String
  nostrDTag       String
  nostrCreatedAt  DateTime
  rawTags         String
  
  // Review data
  rating          Int       // 1-5
  content         String
  
  // Reference to what's being reviewed
  protocolId      String
  protocolEventAddress String // "30818:<pubkey>:<d-tag>"
  
  // WoT
  wotScore        Int?
  
  // Timestamps
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
  
  // Relations
  protocol        Protocol  @relation(fields: [protocolId], references: [id])
  reviewer        Person?   @relation(fields: [reviewerPubkey], references: [pubkey])
  replies         ReviewReply[]
  
  @@index([protocolId])
  @@index([reviewerPubkey])
  @@index([wotScore])
}

model ReviewReply {
  id              String    @id @default(uuid())
  
  // Nostr-native fields
  nostrEventId    String    @unique
  authorPubkey    String
  nostrCreatedAt  DateTime
  
  // Reply data
  content         String
  isAuthorReply   Boolean   // True if authorPubkey === protocol author
  
  // Reference
  reviewId        String
  
  // Timestamps
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
  
  // Relations
  review          Review    @relation(fields: [reviewId], references: [id])
  
  @@index([reviewId])
}

// ============================================
// SYNC STATE
// ============================================

model SyncState {
  id              String    @id @default(uuid())
  relayUrl        String
  lastSync        DateTime
  lastEventId     String?
  
  @@unique([relayUrl])
}

// ============================================
// MODERATION
// ============================================

model ModerationAction {
  id              String    @id @default(uuid())
  
  targetType      String    // protocol, article, review
  targetId        String
  targetEventId   String
  
  action          String    // approve, reject, feature, unfeature
  reason          String?
  moderatorPubkey String?
  
  createdAt       DateTime  @default(now())
  
  @@index([targetId])
  @@index([createdAt])
}
```

### 6.2 Migration Notes

**Breaking changes from current schema:**
- `Protocol.id` changes from content-based to UUID (Nostr event ID stored separately)
- `Protocol.slug` now includes author prefix for uniqueness across authors
- New unique constraint: `authorPubkey + nostrDTag`
- `creator` field renamed to `authorPubkey` (hex pubkey, not display name)
- Many fields become nullable or have different sources

**Slug generation:**
```typescript
function generateSlug(dTag: string, authorPubkey: string): string {
  // Format: <d-tag>-<first-8-chars-of-pubkey>
  // e.g., "cold-exposure-protocol-a1b2c3d4"
  return `${dTag}-${authorPubkey.slice(0, 8)}`;
}
```

---

## 7. Migration Strategy

### 7.1 Phase 1: Schema Migration (Week 1)

1. **Create new Prisma schema** with Nostr-native fields
2. **Run migration** adding new columns (nullable initially)
3. **No data loss** — existing content remains accessible

```sql
-- Example migration
ALTER TABLE "Protocol" ADD COLUMN "nostrEventId" TEXT;
ALTER TABLE "Protocol" ADD COLUMN "authorPubkey" TEXT;
ALTER TABLE "Protocol" ADD COLUMN "nostrDTag" TEXT;
ALTER TABLE "Protocol" ADD COLUMN "wotScore" INTEGER;
ALTER TABLE "Protocol" ADD COLUMN "nostrCreatedAt" TIMESTAMP;
ALTER TABLE "Protocol" ADD COLUMN "rawTags" TEXT;
```

### 7.2 Phase 2: Publish Existing Content to Nostr (Week 2)

**Script to migrate existing protocols to Nostr events:**

```typescript
// scripts/migrate-to-nostr.ts
import { generateSecretKey, getPublicKey, finalizeEvent, SimplePool } from 'nostr-tools';
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();
const pool = new SimplePool();

// Liberture's signing key (keep secure!)
const LIBERTURE_PRIVATE_KEY = process.env.LIBERTURE_NOSTR_PRIVATE_KEY!;
const LIBERTURE_PUBKEY = getPublicKey(hexToBytes(LIBERTURE_PRIVATE_KEY));

async function migrateProtocols() {
  const protocols = await prisma.protocol.findMany({
    where: { nostrEventId: null },
  });

  for (const protocol of protocols) {
    // Build Nostr event
    const event = {
      kind: 30818,
      created_at: Math.floor(protocol.createdAt.getTime() / 1000),
      tags: [
        ['d', protocol.slug],
        ['title', protocol.name],
        ['summary', protocol.description],
        ['pillar', protocol.pillar],
        ['difficulty', protocol.difficulty],
        ['type', 'protocol'],
        ['L', 'liberture.pillar'],
        ['l', protocol.pillar, 'liberture.pillar'],
        ['migrated', 'true'],
      ],
      content: buildAsciiDocContent(protocol),
    };

    if (protocol.duration) event.tags.push(['duration', protocol.duration]);
    if (protocol.equipment) event.tags.push(['equipment', protocol.equipment]);

    // Sign and publish
    const signedEvent = finalizeEvent(event, hexToBytes(LIBERTURE_PRIVATE_KEY));
    
    await pool.publish(['wss://relay.liberture.com'], signedEvent);

    // Update database
    await prisma.protocol.update({
      where: { id: protocol.id },
      data: {
        nostrEventId: signedEvent.id,
        authorPubkey: LIBERTURE_PUBKEY,
        nostrDTag: protocol.slug,
        nostrCreatedAt: protocol.createdAt,
        rawTags: JSON.stringify(signedEvent.tags),
      },
    });

    console.log(`Migrated: ${protocol.slug}`);
  }
}

function buildAsciiDocContent(protocol: Protocol): string {
  return `= ${protocol.name}

== Overview

${protocol.description}

== Steps

${protocol.steps}

== Benefits

${protocol.benefits}

${protocol.risks ? `== Risks\n\n${protocol.risks}` : ''}

${protocol.references ? `== References\n\n${protocol.references}` : ''}
`;
}

migrateProtocols();
```

### 7.3 Phase 3: Enable Indexer (Week 3)

1. Deploy indexer service
2. Subscribe to relays
3. Start ingesting new community content
4. Enable WoT scoring

### 7.4 Phase 4: UI Updates (Week 4)

1. Show author attribution (npub/profile)
2. Display WoT trust badges
3. Enable "View all versions" toggle
4. Launch review submission UI

### 7.5 Backward Compatibility

**During transition:**
- Old URLs (`/protocol/cold-exposure`) continue to work
- Redirect to new URL structure if needed (`/protocol/cold-exposure-a1b2c3d4`)
- Or: Keep old slugs for liberture-authored content, new slugs for community

**URL strategy:**
```
/protocol/cold-exposure           → liberture's version (default)
/protocol/cold-exposure-a1b2c3d4  → specific author's version
/protocol/cold-exposure/versions  → list all versions with WoT ranking
```

---

## 8. SEO Implications

### 8.1 Sitemap Generation

Only include `published: true` content:

```typescript
// app/sitemap.ts (Next.js 15)
import { prisma } from '@/lib/prisma';

export default async function sitemap() {
  const protocols = await prisma.protocol.findMany({
    where: { published: true },
    select: { slug: true, updatedAt: true },
  });

  const articles = await prisma.knowledgeArticle.findMany({
    where: { published: true },
    select: { slug: true, updatedAt: true },
  });

  return [
    {
      url: 'https://liberture.com',
      lastModified: new Date(),
      changeFrequency: 'daily',
      priority: 1,
    },
    ...protocols.map((p) => ({
      url: `https://liberture.com/protocol/${p.slug}`,
      lastModified: p.updatedAt,
      changeFrequency: 'weekly',
      priority: 0.8,
    })),
    ...articles.map((a) => ({
      url: `https://liberture.com/knowledge/${a.slug}`,
      lastModified: a.updatedAt,
      changeFrequency: 'weekly',
      priority: 0.7,
    })),
  ];
}
```

### 8.2 JSON-LD for Protocols

```typescript
// components/ProtocolJsonLd.tsx
import { Person, Protocol } from '@prisma/client';
import { nip19 } from 'nostr-tools';

interface Props {
  protocol: Protocol & { author: Person | null };
}

export function ProtocolJsonLd({ protocol }: Props) {
  const npub = protocol.authorPubkey
    ? nip19.npubEncode(protocol.authorPubkey)
    : null;

  const jsonLd = {
    '@context': 'https://schema.org',
    '@type': 'HowTo',
    name: protocol.title,
    description: protocol.summary,
    author: {
      '@type': 'Person',
      name: protocol.author?.name || 'Anonymous',
      url: npub ? `https://njump.me/${npub}` : undefined,
      identifier: npub,
    },
    datePublished: protocol.nostrCreatedAt?.toISOString(),
    dateModified: protocol.updatedAt.toISOString(),
    mainEntityOfPage: {
      '@type': 'WebPage',
      '@id': `https://liberture.com/protocol/${protocol.slug}`,
    },
    aggregateRating: protocol.reviewCount > 0 ? {
      '@type': 'AggregateRating',
      ratingValue: protocol.weightedRating.toFixed(1),
      reviewCount: protocol.reviewCount,
      bestRating: 5,
      worstRating: 1,
    } : undefined,
  };

  return (
    <script
      type="application/ld+json"
      dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd) }}
    />
  );
}
```

### 8.3 Canonical URLs

**Rule:** The highest-WoT-score version of a protocol is the canonical URL.

```typescript
// In page metadata
export async function generateMetadata({ params }) {
  const protocol = await getProtocolBySlug(params.slug);
  
  // Find canonical (highest WoT score) version
  const canonical = await prisma.protocol.findFirst({
    where: { nostrDTag: protocol.nostrDTag, published: true },
    orderBy: { wotScore: 'asc' }, // Lower score = closer = better
  });

  return {
    alternates: {
      canonical: `https://liberture.com/protocol/${canonical.slug}`,
    },
  };
}
```

### 8.4 robots.txt

```
User-agent: *
Allow: /

# Don't index unpublished or low-trust content
Disallow: /protocol/*?trust=all
Disallow: /api/

Sitemap: https://liberture.com/sitemap.xml
```

---

## 9. Implementation Checklist

### Phase 1: Foundation (Week 1-2)

- [ ] Generate liberture Nostr keypair (store securely)
- [ ] Set up `relay.liberture.com` (strfry or nostream)
- [ ] Create Prisma migration with new schema
- [ ] Run migration on staging database
- [ ] Build migration script for existing protocols

### Phase 2: Indexer (Week 2-3)

- [ ] Implement indexer service (TypeScript + nostr-tools)
- [ ] Connect to WoT oracle API
- [ ] Implement deduplication logic
- [ ] Implement `published` flag logic
- [ ] Deploy indexer as background service
- [ ] Monitor and tune subscription filters

### Phase 3: Publishing (Week 3)

- [ ] Run migration script to publish existing protocols to Nostr
- [ ] Verify all events are on relay.liberture.com
- [ ] Test indexer picks them back up correctly

### Phase 4: Reviews (Week 4)

- [ ] Implement Review model in database
- [ ] Add review subscription to indexer
- [ ] Build review submission UI
- [ ] Implement WoT-weighted rating calculation
- [ ] Cache aggregate ratings in Protocol model

### Phase 5: UI Updates (Week 4-5)

- [ ] Show author profile (name, picture from Kind 0)
- [ ] Display WoT trust badges
- [ ] Add "View all versions" toggle
- [ ] Show reviews with WoT weighting
- [ ] Update sitemap generation
- [ ] Add JSON-LD structured data

### Phase 6: Polish (Week 5-6)

- [ ] Moderation dashboard for queued content
- [ ] Admin tools to manually approve/reject
- [ ] Performance testing
- [ ] Documentation for community authors

---

## Appendix A: Tag Reference

### Kind 30818 Tags (Protocols/Articles)

| Tag | Required | Values | Description |
|-----|----------|--------|-------------|
| `d` | ✅ | Normalized slug | Unique identifier within pubkey |
| `title` | ✅ | String | Display title |
| `pillar` | ✅ | `sleep`, `exercise`, `nutrition`, `mind`, `work`, `finance` | Primary category |
| `type` | ❌ | `protocol`, `knowledge`, `book`, `organization` | Content type (default: protocol) |
| `summary` | ❌ | String | Short description |
| `category` | ❌ | String | Sub-category |
| `difficulty` | ❌ | `beginner`, `intermediate`, `advanced` | Difficulty level |
| `duration` | ❌ | String | Time estimate |
| `equipment` | ❌ | String | Required equipment |
| `t` | ❌ | String | Hashtag |
| `L` | ❌ | Namespace | NIP-32 label namespace |
| `l` | ❌ | Value | NIP-32 label value |
| `a` | ❌ | Event address | Fork source |

### Kind 38381 Tags (Reviews)

| Tag | Required | Values | Description |
|-----|----------|--------|-------------|
| `d` | ✅ | Unique ID | Review identifier |
| `a` | ✅ | `30818:<pubkey>:<d-tag>` | Reviewed protocol |
| `rating` | ✅ | `1`-`5` | Star rating |

### Kind 38382 Tags (Replies)

| Tag | Required | Values | Description |
|-----|----------|--------|-------------|
| `e` | ✅ | Event ID | Review being replied to |
| `p` | ✅ | Pubkey | Reviewer's pubkey |
| `a` | ✅ | Event address | Protocol context |

---

## Appendix B: API Endpoints (Suggested)

```typescript
// REST API for frontend

GET /api/protocols
  ?pillar=sleep
  ?published=true
  ?sort=wotScore|createdAt|rating
  
GET /api/protocols/:slug
GET /api/protocols/:slug/versions

GET /api/reviews/:protocolSlug
POST /api/reviews (requires Nostr auth)

GET /api/authors/:pubkey
GET /api/authors/:pubkey/protocols

GET /api/wot/score/:pubkey
```

---

## Appendix C: Environment Variables

```env
# Database
DATABASE_URL=postgresql://...

# Nostr
LIBERTURE_NOSTR_PRIVATE_KEY=<hex-private-key>
LIBERTURE_RELAY_URL=wss://relay.liberture.com

# WoT Oracle
WOT_ORACLE_URL=https://wot-oracle.mappingbitcoin.com
LIBERTURE_ROOT_PUBKEY=<hex-pubkey>

# Thresholds
WOT_AUTO_PUBLISH_THRESHOLD=3
WOT_MODERATION_THRESHOLD=6
```

---

*Document prepared by Athena for the Liberture development team. For questions, ping Fabri.*
