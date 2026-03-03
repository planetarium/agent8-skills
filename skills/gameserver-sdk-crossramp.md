---
name: gameserver-sdk-crossramp
description: This document focuses on work related to crossramp (cross-chain) integration.
Any logic involving the server should reference this document, which provides methods for writing server code and communicating with the server from the client.
---

# CROSS Mini Hub Integration System Prompt

This document provides guidance for CROSS Mini Hub integration in Agent8 games.

---

## PREREQUISITE - MUST READ FIRST

**DEPENDENCY**: Before using this document, you MUST first read the Agent8 GameServer SDK documentation via `read_gameserver_sdk_v2` tool. CROSS Mini Hub integration depends on:
- `useGameServer` hook and `server` object
- `$asset` API for asset management
- Understanding of Agent8's authentication system

**DO NOT** provide CROSS Mini Hub integration guidance without first ensuring the base GameServer SDK context is loaded.

---

## TRIGGER CONDITIONS

**ACTIVATION SCOPE** - Reference this document when CROSS Mini Hub-related keywords appear in:
- The current user request, OR
- Any previous messages in the conversation context

**Trigger Keywords** (case-insensitive, any language):

| Type | Keywords | Condition |
|------|----------|-----------|
| Standalone | `crossramp`, `CrossRamp`, `크로스램프`, `cross ramp`, `크로스 램프` | Immediate activation |
| Standalone | `CROSS Mini Hub`, `mini hub`, `미니허브`, `미니 허브` | Immediate activation |
| Standalone | `Cross Game Token`, `크로스 게임 토큰` | Immediate activation |
| Standalone | `getCrossRampShopUrl` | Immediate activation |
| Combined | `cross` / `크로스` | When mentioned with `교환`, `exchange`, `DEX`, or `token` |

**DO NOT** reference this document when:
- User only mentions "blockchain", "블록체인", "token", "토큰" without crossramp/cross keywords
- User asks about general game development or asset management
- User asks about NFT, smart contracts, or Web3 without crossramp/cross keywords
- User asks about regular game shop or inventory systems

**DO NOT proactively mention or recommend CROSS Mini Hub** - Only respond when the user explicitly asks about it first or when trigger keywords have appeared in the conversation.

---

## CROSS Mini Hub Integration

CROSS Mini Hub enables players to easily exchange game items, coins, and CROSS. Integration is simple: generate a URL and open it in a popup.

### Available Features

| Feature | Method | Purpose |
|---------|--------|---------|
| **CROSS Mini Hub** | `getCrossRampShopUrl()` | Open CROSS Mini Hub for item and token exchange |

**Important**: This method is SDK built-in. Call it directly on the `server` object. Do NOT implement server-side code or use `remoteFunction()` for CROSS Mini Hub APIs.

### Quick Start

```tsx
// Open CROSS Mini Hub in a popup
const url = await server.getCrossRampShopUrl("en");
window.open(url, "CrossRampShop", "width=1024,height=768");
```

### Supported Languages

- `"en"` - English
- `"ko"` - Korean
- `"zh"` - Chinese (Simplified)
- `"zh-Hant"` - Chinese (Traditional)
- `"ja"` - Japanese

### Full Example

```tsx filename='App.tsx'
import React, { useState } from "react";
import { useGameServer } from "@agent8/gameserver";

export default function CrossRampShop() {
  const { connected, server } = useGameServer();
  const [loading, setLoading] = useState(false);

  const openShop = async () => {
    if (!connected || !server) return;
    setLoading(true);
    try {
      const url = await server.getCrossRampShopUrl("en");
      window.open(url, "CrossRampShop", "width=1024,height=768");
    } catch (error) {
      console.error("Failed to open CROSS Mini Hub:", error);
    } finally {
      setLoading(false);
    }
  };

  if (!connected) return <div>Connecting...</div>;

  return (
    <div>
      <button onClick={openShop} disabled={loading}>
        Shop
      </button>
    </div>
  );
}
```

**Note**: CROSS Mini Hub handles all blockchain complexity. Asset synchronization happens automatically.
