---
name: vx-shop-impl
description: The implementation details related to VX or VX shop should unconditionally refer to this document. Additionally, you must read the read_gameserver_sdk together. VX is a platform money itself, and the VX shop is the shop provided within that platform. Each game only implements the interface, allowing items to be sold through the platform's vx shop and be applied to the game.
---

# VX Shop Implementation Guide
## What is VX and VX Shop?
**VX** is the primary currency of the Verse8 platform. Users purchase VX on Verse8 and use it to buy items or premium features in games created by creators.
**VX Shop** is a monetization system that enables creators to integrate revenue generation into their games:
- All payment processing handled by Verse8
- Creators only implement game logic
- 100 VX ≈ $1 USD
- Available only to CPP (Creator Partner Program) members
## Mandatory Prerequisites
**⚠️ CRITICAL: You MUST read the GameServer SDK documentation before starting VX Shop implementation.**
VX Shop requires server-side logic. All payment-related operations including purchase event handling and user state updates must be done on the server.
**Required Reading:**
- Full GameServer SDK documentation (tool: read_gameserver_sdk)
- Especially User State management: `/docs/gameserver/sdk/globalUserState`
- Remote Function calls: `/docs/gameserver/sdk/remoteFunction`
# VX Shop Implementation Steps
## Step 0: Verify Product ID
**Pre-implementation Checklist:**
Check if the user's request includes a **Product ID**. The Product ID is the unique key used to identify products in VX Shop.
**If Product ID is Missing:**
Stop work and inform the user:
"To implement VX Shop, you must first register the product on the Verse8 platform.
1. Access Verse8 game management page
2. Navigate to VX Shop tab
3. Click 'Add New Item' button
4. Enter product information:
   - Product ID: Unique identifier for code (e.g., premium-subscription, gold-pack-1000)
   - Product Name: Display name shown to users
   - Price: VX price (100 VX ≈ $1)
   - Purchase Constraints: Set purchase limits (optional)
Once you've registered the product, provide me with the Product ID to begin implementation.
Detailed guide: /docs/vxshop/registering-products"
**If Product ID is Present:**
- Proceed to next step
- Product ID examples: \`'premium-subscription'\`, \`'gold-pack-1000'\`, \`'gacha-10pull'\`
## Step 1: UI Implementation
Design shop UI freely according to user's request. VX Shop does not impose UI restrictions.
**Common UI Patterns:**
- Dedicated shop screen
- In-game popup
- Inline purchase button
- Purchase option in inventory/settings screen
**Information to Include in UI:**
- Product name and description
- Price (VX)
- Purchase button
- Purchase status (already purchased, available to purchase)
## Step 2: VX Shop Integration
Use the \`useVXShop\` hook from \`@verse8/platform\` package to integrate with VX Shop.
### Basic Usage
\`\`\`tsx
import { useVXShop } from '@verse8/platform';
import { useGlobalMyState } from '@agent8/gameserver';
import { useEffect } from 'react';
function Shop() {
  const { getItem, buyItem, refresh, onClose } = useVXShop();
  const myState = useGlobalMyState();
  // Refresh product list when VX Shop dialog closes
  useEffect(() => {
    const cancel = onClose(() => refresh());
    return cancel;
  }, [onClose, refresh]);
  const item = getItem('premium-subscription'); // Query product by Product ID
  return (
    <button onClick={() => buyItem('premium-subscription')}>
      Buy Premium - {item?.price} VX
    </button>
  );
}
\`\`\`
### API Reference
**useVXShop() Returns:**
- \`getItem(productId)\` - Query specific product
- \`buyItem(productId)\` - Open purchase dialog (start payment)
- \`refresh()\` - Refresh product list
- \`onClose(callback)\` - Register callback when dialog closes
**VXShopItem Key Properties:**
- \`productId\` - Product ID
- \`price\` - VX price
- \`purchasable\` - Whether purchase is available
- \`purchaseLimit\` - Purchase limit
- \`remainingPurchaseQuantity\` - Remaining purchase count
- \`purchaseLimitReached\` - Whether limit is reached
### UI Handling by Purchase State
\`\`\`tsx
const item = getItem('product-id');
// Unlimited purchases available
if (item?.purchasable && !item.purchaseLimit) {
  return <button onClick={() => buyItem(item.productId)}>{item.price} VX</button>;
}
// Limited purchases (show remaining count)
if (item?.purchasable && item.purchaseLimit && !item.purchaseLimitReached) {
  return (
    <button onClick={() => buyItem(item.productId)}>
      {item.price} VX ({item.remainingPurchaseQuantity}/{item.purchaseLimit})
    </button>
  );
}
// Limit reached
if (item?.purchaseLimitReached) {
  return <button disabled>Purchased</button>;
}
// Not purchasable
if (!item?.purchasable) {
  return <button disabled>{item.purchaseBlockReason || 'Cannot Buy'}</button>;
}
\`\`\`
## Step 3: Server Code Implementation
**IMPORTANT:** User state changes upon purchase completion MUST be handled on the server.
### Implement $onItemPurchased in server.js
\`\`\`javascript
// server.js
async $onItemPurchased({ account, purchaseId, productId, quantity, metadata }) {
  switch (productId) {
    case 'premium-subscription':
      // One-time purchase: Set permanent flag
      await $global.updateUserState(account, { 
        isPremium: true,
        premiumDate: Date.now()
      });
      break;
      
    case 'gold-pack-1000':
      // Consumable: Add to existing value
      const userState = await $global.getUserState(account);
      await $global.updateUserState(account, { 
        gold: (userState.gold || 0) + 1000 
      });
      break;
  }
  
  return { success: true };
}
\`\`\`
### Parameters
- \`account\`: Purchaser's account ID
- \`purchaseId\`: Unique transaction ID
- \`productId\`: Purchased product's Product ID
- \`quantity\`: Purchase quantity
- \`metadata\`: Custom JSON data
## Step 4: Real-time State Updates on Client
When server changes state, client automatically receives updates.
### Using useGlobalMyState (Recommended)
\`\`\`tsx
import { useGlobalMyState } from '@agent8/gameserver';
function GameScreen() {
  const myState = useGlobalMyState(); // Auto-sync
  
  const isPremium = myState?.isPremium || false;
  const gold = myState?.gold || 0;
  
  return (
    <div>
      {isPremium ? <div>Premium Member ✨</div> : <AdComponent />}
      <div>Gold: {gold}</div>
    </div>
  );
}
\`\`\`
# Example Code
## Example 1: Premium Subscription
\`\`\`tsx
// Client
import { useVXShop } from '@verse8/platform';
import { useGlobalMyState } from '@agent8/gameserver';
function PremiumShop() {
  const { getItem, buyItem, refresh, onClose } = useVXShop();
  const myState = useGlobalMyState();
  useEffect(() => {
    const cancel = onClose(() => refresh());
    return cancel;
  }, [onClose, refresh]);
  const premiumItem = getItem('premium-subscription');
  const isPremium = myState?.isPremium || false;
  if (isPremium) {
    return <div>✨ Premium Member</div>;
  }
  return (
    <button onClick={() => buyItem('premium-subscription')}>
      Get Premium - {premiumItem?.price} VX
    </button>
  );
}
\`\`\`
\`\`\`javascript
// Server
async $onItemPurchased({ account, productId }) {
  if (productId === 'premium-subscription') {
    await $global.updateUserState(account, { isPremium: true });
  }
  return { success: true };
}
\`\`\`
## Example 2: Gacha System
\`\`\`tsx
// Client
function GachaShop() {
  const { getItem, buyItem, refresh, onClose } = useVXShop();
  const myState = useGlobalMyState();
  useEffect(() => {
    const cancel = onClose(async (payload) => {
      refresh();
      if (payload.purchased && payload.productId === 'gacha-10pull') {
        const result = await remoteFunction('openGachaBox', {});
        setPullResult(result);
      }
    });
    return cancel;
  }, [onClose, refresh]);
  return (
    <button onClick={() => buyItem('gacha-10pull')}>
      Buy 10-Pull
    </button>
  );
}
\`\`\`
\`\`\`javascript
// Server
async $onItemPurchased({ account, productId }) {
  if (productId === 'gacha-10pull') {
    const state = await $global.getUserState(account);
    await $global.updateUserState(account, {
      gachaTickets: (state.gachaTickets || 0) + 10
    });
  }
  return { success: true };
}
async openGachaBox() {
  const state = await $global.getMyState();
  if ((state.gachaTickets || 0) < 10) throw new Error('Not enough tickets');
  
  const results = [];
  for (let i = 0; i < 10; i++) {
    results.push(rollWithProbability());
  }
  
  await $global.updateMyState({
    gachaTickets: state.gachaTickets - 10
  });
  
  return { items: results };
}
\`\`\`
# Critical Warnings
## ❌ Never Do This
1. Change state directly on client
2. Call buyItem without checking purchasable
3. Forget to call refresh()
## ✅ Correct Approach
1. Change state only on server
2. Always check purchasable
3. Always call refresh()
