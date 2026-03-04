# Discord Alternatives with Nostr Integration Potential

**Date:** 2026-03-03  
**Category:** Nostr/Technology  
**Tags:** Discord, Nostr, WoT, NIP-29, group chat, decentralized

---

## Executive Summary

This research maps the landscape of Discord alternatives that either already integrate with Nostr or are strong candidates for adaptation. The analysis covers three categories:

1. **Nostr-Native Solutions** — Already built on Nostr, supporting NIP-29 groups
2. **High-Potential Adaptation Candidates** — Open-source Discord alternatives with clean architectures for Nostr login/WoT integration
3. **Bridge/Hybrid Approaches** — Platforms that could connect to Nostr via bridges or plugins

**Key Finding:** The most viable paths are either (a) using existing NIP-29 group chat clients like "Groups" which already have the Nostr foundation, or (b) adapting Stoat (formerly Revolt) or Spacebar which have modular architectures ideal for Nostr auth integration.

---

## Table of Contents

1. [Nostr-Native Group Chat Solutions](#1-nostr-native-group-chat-solutions)
2. [Open-Source Discord Alternatives](#2-open-source-discord-alternatives)
3. [WoT Integration Considerations](#3-wot-integration-considerations)
4. [Technical Integration Paths](#4-technical-integration-paths)
5. [Recommendation Matrix](#5-recommendation-matrix)
6. [Sources](#6-sources)

---

## 1. Nostr-Native Group Chat Solutions

### NIP-29 "Groups" Client
**Status:** Production-ready Nostr-native  
**URL:** https://groups.nip29.com  
**Repo:** https://github.com/max21dev/groups

The most mature NIP-29 group chat implementation. Relay-based groups with moderation capabilities.

**Features:**
- ✅ Send/delete messages
- ✅ Reactions, replies, zaps
- ✅ Polls (single/multiple choice)
- ✅ Image uploads
- ✅ Group creation with custom settings
- ✅ Full moderation: add/remove users, admins, delete messages
- ✅ Real-time updates
- ✅ Dark mode
- 🚧 Threads, notifications, file attachments (in progress)

**WoT Integration:** Would need custom implementation. NDK library has WoT module that could be integrated.

**Assessment:** Best starting point if you want pure Nostr. Add WoT filtering via @nostr-dev-kit/wot package.

---

### Coracle
**Status:** Production client with groups support  
**URL:** https://coracle.social  
**Repo:** https://github.com/coracle-social/coracle

Full-featured Nostr client with NIP-72 communities and NIP-87 closed groups.

**Features:**
- ✅ NIP-87 closed groups
- ✅ NIP-72 communities
- ✅ Web of Trust scores for spam filtering
- ✅ NIP-17 DMs
- ✅ Customizable feeds
- ✅ White-label support
- ✅ Full moderation tools
- ✅ Zaps integration

**WoT Integration:** **Already built-in.** Uses WoT for content recommendations and spam filtering.

**Assessment:** Strong candidate. Already has WoT. Missing Discord-style UX (servers/channels hierarchy).

---

### noStrudel
**Status:** Experimental client  
**URL:** https://nostrudel.ninja  
**Repo:** https://github.com/hzrd149/nostrudel

Sandbox for exploring Nostr protocol. Shows underlying events — good for developers.

**Assessment:** More of a protocol explorer than a polished chat UX. Useful for testing.

---

### relay29 (Server-side)
**Repo:** https://github.com/fiatjaf/relay29

Go library for creating NIP-29 relays. Works with khatru, strfry, and relayer.

**Use Case:** Deploy your own NIP-29 relay with custom moderation rules.

```go
state.AllowAction = func(ctx context.Context, group nip29.Group, role *nip29.Role, action relay29.Action) bool {
    // Custom permission logic
    if role == adminRole {
        return true
    }
    // etc.
}
```

**Assessment:** Essential if you're building custom NIP-29 infrastructure.

---

## 2. Open-Source Discord Alternatives

### Tier 1: High Adaptation Potential

#### Stoat (formerly Revolt)
**URL:** https://stoat.chat  
**Repo:** https://github.com/stoatchat

**Tech Stack:**
- Backend: Rust (stoatchat/stoatchat)
- Web Frontend: Solid.js PWA (stoatchat/for-web)
- Mobile: Native Android (Kotlin), iOS (Swift)
- SDK: TypeScript (stoatchat/javascript-client-sdk)

**Stars:** 2.8k (backend), 960 (legacy web)

**Key Strengths:**
- Clean modular architecture
- Self-hosted Docker deployment
- Fine-grained permissions
- Built in Europe (GDPR-focused)
- No tracking/ads
- Active development

**Nostr Integration Path:**
1. Replace auth system with NIP-07/NIP-46 signer
2. Add WoT scoring via NDK's @nostr-dev-kit/wot
3. Optionally bridge messages to Nostr relays via NIP-29

**Complexity:** Medium. Rust backend would need auth module replacement.

**Assessment:** ⭐⭐⭐⭐⭐ Top candidate. Modern stack, active community, clean architecture.

---

#### Spacebar (Discord-compatible)
**URL:** https://spacebar.chat  
**Repo:** https://github.com/spacebarchat/server

**What it is:** Re-implementation of Discord's backend API. Existing Discord bots/clients work with it.

**Tech Stack:**
- Backend: TypeScript (API, Gateway, CDN, WebRTC)
- Compatible with any Discord client

**Key Strengths:**
- 100% Discord API compatible
- Existing bots work out of the box
- Self-hosted
- Active development

**Nostr Integration Path:**
1. Add Nostr login alongside existing auth
2. Bridge channels to NIP-29 groups
3. Add WoT-based permissions layer

**Complexity:** Medium-High. Benefit: preserves Discord UX and bot ecosystem.

**Assessment:** ⭐⭐⭐⭐ Good choice if you want Discord compatibility while adding Nostr identity.

---

### Tier 2: Viable but Higher Effort

#### Rocket.Chat
**URL:** https://rocket.chat  
**Repo:** https://github.com/RocketChat/Rocket.Chat

**Tech Stack:** TypeScript, Go, React, Single Linux binary, PostgreSQL

**Stars:** 40k+

**Features:**
- Team collaboration
- Omnichannel (WhatsApp, SMS integration)
- Self-hosted + cloud options
- 700+ marketplace integrations
- Federation support
- Enterprise-grade security

**Nostr Integration Path:**
1. Build OAuth provider for NIP-07 login
2. Create Nostr relay bridge integration
3. Add WoT as a moderation layer

**Complexity:** High. Enterprise-focused, more overhead.

**Assessment:** ⭐⭐⭐ Overkill for most use cases. Good if you need enterprise features.

---

#### Element/Matrix
**URL:** https://element.io  
**Repo:** https://github.com/element-hq/element-web

**Tech Stack:** TypeScript, Matrix protocol

**Features:**
- Federated by design
- E2E encryption
- Bridge ecosystem (IRC, Slack, Discord bridges exist)
- Enterprise support

**Nostr Integration Path:**
1. Build Matrix-Nostr bridge (none exists yet)
2. Nostr identities map to Matrix accounts
3. WoT scores influence room permissions

**Complexity:** High. Two federated protocols = complex bridging.

**Assessment:** ⭐⭐⭐ Federation alignment is interesting, but bridging complexity is significant.

---

#### Mattermost
**URL:** https://mattermost.com  
**Repo:** https://github.com/mattermost/mattermost

**Tech Stack:** Go + React

**Features:**
- DevSecOps focused
- Self-hosted
- Plugin/integration system
- MIT licensed

**Nostr Integration Path:**
1. Build Nostr auth plugin
2. Channel-to-relay bridge plugin
3. WoT scoring plugin

**Complexity:** Medium. Plugin architecture helps.

**Assessment:** ⭐⭐⭐ Viable, but more enterprise/dev-team focused than community chat.

---

#### Zulip
**URL:** https://zulip.com  
**Repo:** https://github.com/zulip/zulip

**Tech Stack:** Python (Django) + TypeScript

**Features:**
- Unique topic-based threading
- 1500+ contributors
- Free for open source projects
- Self-hosted + cloud

**Nostr Integration Path:**
1. Python auth backend for NIP-07
2. WoT integration via API
3. Thread model could map well to Nostr reply threads

**Complexity:** Medium. Python ecosystem different from typical Nostr JS/TS.

**Assessment:** ⭐⭐⭐ Interesting threading model. Good for async communities.

---

### Tier 3: Different Approach

#### Ditto
**URL:** https://docs.soapbox.pub/ditto/  
**Repo:** https://github.com/soapbox-pub/ditto

**What it is:** Nostr server that speaks Mastodon API. Use any Mastodon client to interact with Nostr.

**Tech Stack:** Deno (TypeScript)

**Key Insight:** This is the reverse approach — instead of adding Nostr to Discord, it makes Nostr accessible via familiar Mastodon interfaces.

**Features:**
- Built-in Nostr relay
- Works with Mastodon apps
- Self-hosted
- Zaps support

**Assessment:** ⭐⭐⭐⭐ Different paradigm. If users are comfortable with Mastodon UX, this is simpler.

---

## 3. WoT Integration Considerations

### NDK WoT Package
**Repo:** https://github.com/nostr-dev-kit/ndk

The @nostr-dev-kit/wot package provides:
- Web of Trust filtering and ranking
- Configurable depth (1-hop, 2-hop, etc.)
- Score thresholds
- Auto-filtering on subscriptions

```typescript
// Load WoT data
await ndk.wot.load({ maxDepth: 2 });

// Enable automatic filtering
ndk.wot.enableAutoFilter({
  maxDepth: 2,
  minScore: 0.5,
  includeUnknown: false
});

// All subscriptions now filter by WoT
const notes = ndk.subscribe({ kinds: [1] });
```

### Integration Patterns

**Pattern 1: Auth-Only WoT**
- Users login with Nostr keys
- WoT scores influence permissions (who can post, moderate, join)
- Messages stay on traditional backend

**Pattern 2: Hybrid Bridge**
- Traditional server for real-time chat
- Messages bridged to Nostr relays
- WoT filters spam/low-trust content
- Best of both worlds

**Pattern 3: Full Nostr Native**
- All messages are Nostr events
- Use NIP-29 for groups
- WoT for content ranking
- Relay selection for persistence

---

## 4. Technical Integration Paths

### Path A: Adapt Stoat/Revolt
**Effort:** 2-4 weeks  
**Result:** Discord-like UX with Nostr login + WoT

Steps:
1. Fork stoatchat/stoatchat
2. Add NIP-07/NIP-46 auth endpoint
3. Integrate NDK WoT for trust scoring
4. Map WoT scores to permission levels

### Path B: Build on Groups Client
**Effort:** 1-2 weeks  
**Result:** Pure Nostr, needs UX polish

Steps:
1. Fork max21dev/groups
2. Add @nostr-dev-kit/wot integration
3. Filter messages by WoT score
4. Add channel-like UI improvements

### Path C: Spacebar + Nostr Bridge
**Effort:** 4-6 weeks  
**Result:** Discord-compatible with Nostr identity layer

Steps:
1. Add Nostr OAuth provider to Spacebar
2. Build message bridge to NIP-29 relay
3. Implement WoT-based moderation
4. Maintain Discord bot compatibility

---

## 5. Recommendation Matrix

| Platform | Nostr-Ready | WoT-Ready | Discord UX | Self-Host | Effort | Score |
|----------|-------------|-----------|------------|-----------|--------|-------|
| **Groups (NIP-29)** | ✅ Native | ❌ Needs add | ⚠️ Basic | ✅ | Low | ⭐⭐⭐⭐ |
| **Coracle** | ✅ Native | ✅ Built-in | ⚠️ Social-first | ✅ | Low | ⭐⭐⭐⭐ |
| **Stoat** | ❌ Needs add | ❌ Needs add | ✅ Full | ✅ | Medium | ⭐⭐⭐⭐⭐ |
| **Spacebar** | ❌ Needs add | ❌ Needs add | ✅ 100% | ✅ | High | ⭐⭐⭐⭐ |
| **Rocket.Chat** | ❌ Needs add | ❌ Needs add | ⚠️ Different | ✅ | High | ⭐⭐⭐ |
| **Element/Matrix** | ❌ Complex | ❌ Needs add | ⚠️ Different | ✅ | Very High | ⭐⭐ |
| **Ditto** | ✅ Native | ⚠️ Partial | ❌ Mastodon | ✅ | Low | ⭐⭐⭐ |

---

## 6. Final Recommendations

### For Nostr-First (fastest path):
**Use: Groups + NDK WoT**  
- Fork max21dev/groups
- Add @nostr-dev-kit/wot
- Deploy with relay29 for your own NIP-29 relay
- Timeline: 1-2 weeks

### For Discord UX (best user experience):
**Use: Stoat (Revolt)**  
- Fork stoatchat backend
- Add Nostr auth module
- Integrate WoT for permissions
- Timeline: 3-4 weeks

### For Maximum Compatibility (bots, existing tools):
**Use: Spacebar**  
- Fork spacebarchat/server
- Add Nostr identity layer
- Bridge to NIP-29
- Timeline: 4-6 weeks

---

## 7. Sources

- NIP-29 Specification: https://github.com/nostr-protocol/nips/blob/master/29.md
- Groups Client: https://github.com/max21dev/groups
- Coracle: https://github.com/coracle-social/coracle
- Stoat: https://github.com/stoatchat
- Spacebar: https://github.com/spacebarchat/server
- NDK: https://github.com/nostr-dev-kit/ndk
- relay29: https://github.com/fiatjaf/relay29
- Rocket.Chat: https://github.com/RocketChat/Rocket.Chat
- Element: https://github.com/element-hq/element-web
- Mattermost: https://github.com/mattermost/mattermost
- Zulip: https://github.com/zulip/zulip
- Ditto: https://github.com/soapbox-pub/ditto

---

## Uncertainty Flags

- **NIP-29 maturity:** NIP-29 is implemented but not as battle-tested as NIP-01/NIP-17. Group relay ecosystem is smaller.
- **WoT scaling:** NDK WoT works well for 2-hop, but deeper graphs have performance implications.
- **Bridge reliability:** No production Nostr↔Matrix or Nostr↔Discord bridges exist yet. Would be novel work.
- **Stoat auth refactor:** Rust backend changes require familiarity with their codebase. Active community may help.
