---
name: gameserver-sdk-v1
description: |
  Any logic related to the server must refer to this document. It provides methods for writing server code and communicating with the server from the client.
  This document is for legacy use. Please refer to this document when using a single server.js file in the project root.
---

# Agent8 GameServer SDK

Agent8 GameServer is a fully managed game server solution. You don't need to worry about server setup or management when developing your game. Simply create a server.js file with your game logic, and you'll have a server-enabled game ready to go.

Let's look at a basic example of calling server functions:

```js filename='server.js'
class Server {
add(a, b) {
return a + b;
}
}

// just define the class and NEVER export it like this: "module.exports = Server;" or "export default Server;"
```

```tsx filename='App.tsx'
import React from "react";
import { useGameServer } from "@agent8/gameserver";

export default function App() {
const { connected, server } = useGameServer();

    if (!connected) return "Connecting...";

    const callServer = () => {
      const args = [1, 2];
      const result = await server.remoteFunction("add", args);
      console.log(result); // expected: 3
    };

    return (
      <div>
        <button onClick={callServer}>Call Server</button>
      </div>
    );

}
```

# remoteFunction

remoteFunction can be called through the GameServer instance when connected.

```
const { connected, server } = useGameServer();
if (connected) server.remoteFunction(...);
```

Three ways to call remoteFunction:

1. Call requiring return value
   Use `await remoteFunction('add', [1, 2])` when you need to wait for the server's computation result.
   Note: Very rapid calls (several per second) may be rejected.

2. Non-response required but reliability guaranteed call (optional)
   `remoteFunction('updateMyNickname', [{nick: 'karl'}], { needResponse: false })`
   The needResponse option tells the server not to send a response, which can improve communication performance.
   Rapid calls may still be rejected.

3. Overwritable update calls (fast call)
   `remoteFunction('updateMyPosition', [{x, y}], { throttle: 50 })`
   The throttle option sends requests at specified intervals (ms), ignoring intermediate calls.
   Use this for fast real-time updates like player positions. The server only receives throttled requests, effectively reducing load.

3-1. Throttling multiple values simultaneously
`remoteFunction('updateBall', [{ballId: 1, position: {x, y}}], { throttle: 50, throttleKey: '1'})`
`remoteFunction('updateBall', [{ballId: 2, position: {x, y}}], { throttle: 50, throttleKey: '2'})`
Use throttleKey to differentiate throttling targets when updating multiple entities with the same function.
Without throttleKey, function name is used as default throttle identifier.

# Users

Users making server requests have a unique `account` ID:

- Access via `$sender.account` in server code
- Access via `server.account` in client code

```js filename='server.js'
class Server {
  getMyAccount() {
    return $sender.account;
  }
}
```

```tsx filename='App.tsx'
  const { connected, server } = useGameServer();
  const myAccount = server.account;
```

# User Authentication & Premium Features

Users can have subscription and follower status in their authentication context. These properties enable premium features and follower-exclusive content:

- Access via `$sender.isFollower`, `$sender.isSubscriber`, and `$sender.subscription` in server code
- `$sender.isFollower` - boolean indicating if user follows the creator
- `$sender.isSubscriber` - boolean indicating if user has active subscription (checks expiration)
- `$sender.subscription` is `{ tier: string, exp: number } | null` (default tier: 'basic', expandable)
- Only available from server-signed authentication tokens (security feature)

```js filename='server.js'
class Server {
  getMyStatus() {
    return {
      isFollower: $sender.isFollower,
      isSubscriber: $sender.isSubscriber,
      subscription: $sender.subscription
    };
  }
}
// Returns:
// {
//   isFollower: true | false,
//   isSubscriber: true | false,
//   subscription: { tier: 'basic', exp: 1234567890 } | null
// }
```

# Global State Management

Multiplayer games require management of user states, items, rankings, etc.
Access global state through the `$global` variable in server code.
Agent8 provides three types of persistent global state:

1. Global Shared State

- `$global.getGlobalState(): Promise<Object>` - Retrieves current global state
- `$global.updateGlobalState(state: Object): Promise<Object>` - Updates global state

```js filename='server.js'
class Server {
  async getGlobalState() {
    return $global.getGlobalState();
  }
  async updateGlobalState(state) {
    await $global.updateGlobalState(state);
  }
}
```

2. User State

- `$sender.account: string` - Request sender's account
- `$global.getUserState(account: string): Promise<Object>` - Gets specific user's state
- `$global.updateUserState(account: string, state: Object): Promise<Object>` - Updates specific user's state
- `$global.getMyState(): Promise<Object>` - Alias for <code>$global.getUserState($sender.account)</code>
- `$global.updateMyState(state: Object): Promise<Object>` - Alias for <code>$global.updateUserState($sender.account, state)</code>

3. Collection State (List management for specific keys, rankings)

- `$global.countCollectionItems(collectionId: string, options?: CollectionOptions): Promise<number>` - Retrieves number of items
- `$global.getCollectionItems(collectionId: string, options?: CollectionOptions): Promise<Item[]>` - Retrieves filtered/sorted items
- `$global.getCollectionItem(collectionId: string, itemId: string): Promise<Item>` - Gets single item by ID
- `$global.addCollectionItem(collectionId: string, item: any): Promise<Item>` - Adds new item
- `$global.updateCollectionItem(collectionId: string, item: any): Promise<Item>` - Updates existing item
- `$global.deleteCollectionItem(collectionId: string, itemId: string): Promise<{ \_\_id: string }>` - Deletes item
- `$global.deleteCollection(collectionId: string): Promise<string>` - Deletes Collection

CollectionOptions uses Firebase-style filtering:

```
interface QueryFilter {
field: string;
operator:
| '<'
| '<='
| '=='
| '!='
| '>='
| '>'
| 'array-contains'
| 'in'
| 'not-in'
| 'array-contains-any';
value: any;
}

interface QueryOrder {
field: string;
direction: 'asc' | 'desc';
}

interface CollectionOptions {
filters?: QueryFilter[];
orderBy?: QueryOrder[];
limit?: number;
startAfter?: any;
endBefore?: any;
}
```

**IMPORTANT**: You cannot use both `filters` and `orderBy` together in CollectionOptions because Firebase requires an index for such composite queries. You must choose either filtering OR ordering on a single field, not both simultaneously.

Important: All state updates in Agent8 use merge semantics

```js filename='server.js'
const state = await $global.getGlobalState(); // { "name": "kim" }
await $global.updateGlobalState({ age: 18 });
const newState = await $global.getGlobalState(); // { "name": "kim", "age": 18 }
```

# Rooms - Optimized for Real-Time Session Games

Rooms are optimized for creating real-time multiplayer games. While global states handle numerous users, tracking them all isn't ideal. Rooms allow real-time awareness of connected users in the same space and synchronize all user states. There's a maximum connection limit.

You can manage all rooms through $global. These functions require explicit roomId specification.

- `$global.countRooms(): Promise<number>` - Gets the total number of rooms that currently have active users.
- `$global.getAllRoomIds(): Promise<string[]>` - Returns IDs of all rooms that currently have active users.
- `$global.getAllRoomStates(): Promise<any[]>` - Returns an array of roomStates for all rooms with active users.
- `$global.getRoomUserAccounts(roomId: string): Promise<string[]>` - Gets an array of account strings for current users in a specific room.
- `$global.countRoomUsers(roomId: string): Promise<number>` - Gets the number of current users in a specific room.

- `$global.getRoomState(roomId: string): Promise<any>`
- `$global.updateRoomState(roomId: string, state: any): Promise<any>`
- `$global.getRoomUserState(roomId: string, account: string): Promise<any>`
- `$global.updateRoomUserState(roomId: string, account: string, state: any): Promise<any>`

To join/leave rooms, you must call these functions:

- `$global.joinRoom(roomId?: string): Promise<string>`: Joins the specified room. If no roomId is provided, the server will create a new random room and return its ID.
- `$global.leaveRoom(): Promise<string>`: Leaves the current room. You can call `$room.leave()` instead.

IMPORTANT: `joinRoom()` request without roomId will always create a new random room. If you want users to join the same room by default, use a default roomId as a parameter.


Eexample for joining the room
```js filename='server.js'
class Server {
  const MAX_ROOM_USER = 3;
  async joinRoom(roomId) {
    if (roomId) {
      if ($global.countRoomUsers(roomId) >= MAX_ROOM_USER) throw Error('room is full');
    }

    const joinedRoomId = await $global.joinRoom(roomId)

    if ($global.countRoomUsers(joinedRoomId) === MAX_ROOM_USER) { // or you can use `$room.countUsers() === MAX_ROOM_USER`
      await $room.updateRoomState({ status: 'START' })
    } else {
      await $room.updateRoomState({ status: 'READY' })
    }

    return joinedRoomId
  }
}
```

The $room prefix automatically acts on the room that the $sender belongs to, so you don't need to explicitly specify roomId. All actions are queried and processed for the currently joined room.



Rooms provide a tick function that enables automatic server-side logic execution without explicit user calls.
The room tick will only repeat execution while users are present in the room.
The room tick is a function that is called every 100ms~1000ms (depends on the game's logic)

```js filename='server.js'
class Server {
  $roomTick(deltaMS, roomId) {
  ...
  }
}
```

Important: Do not use `setInterval` or `setTimeout` in server.js - you must use room ticks instead.

Room state can be accessed through the `$room` public variable in server code. Agent8 provides three types of room states that are not persisted - they get cleared when all users leave the room.

1. Room Public State

- `$room.getRoomState(): Promise<Object>` - Retrieves the current room state.
- `$room.updateRoomState(state: Object): Promise<Object>` - Updates the room state with new values.

Important: roomState contains these default values:

- `roomId: string`
- `$users: string[]` - Array of all user accounts in the room, automatically updated

2. Room User State

- `$sender.roomId: string` - The ID of the room the sender is in.
- `$room.getUserState(account: string): Promise<Object>` — Retrieves a particular user's state in the room.
- `$room.updateUserState(account: string, state: Object): Promise<Object>` — Updates a particular user's state in the room.
- `$room.getMyState(): Promise<Object>` — Retrieves $sender state in the room.
- `$room.updateMyState(state: Object): Promise<Object>` — Updates $sender state in the room.
- `$room.getAllUserStates(): Promise<Object[]>` - Retrieves all user states in the room.

3. Room Collection State

- `$room.countCollectionItems(collectionId: string, options?: CollectionOptions): Promise<number>` - Retrieves number of items from a room collection based on filtering, sorting, and pagination options.
- `$room.getCollectionItems(collectionId: string, options?: CollectionOptions): Promise<Item[]>` - Retrieves multiple items from a room collection based on filtering, sorting, and pagination options.
- `$room.getCollectionItem(collectionId: string, itemId: string): Promise<Item>` - Retrieves a single item from the room collection using its unique ID.
- `$room.addCollectionItem(collectionId: string, item: any): Promise<Item>` - Adds a new item to the specified room collection and returns the added item with its unique identifier.
- `$room.updateCollectionItem(collectionId: string, item: any): Promise<Item>` - Updates an existing item in the room collection and returns the updated item.
- `$room.deleteCollectionItem(collectionId: string, itemId: string): Promise<{ \_\_id: string }>` - Deletes an item from the room collection and returns a confirmation containing the deleted item's ID.
- `$room.deleteCollection(collectionId: string): Promise<string>` - Delete room collection.

# Messaging

Simple socket messaging is also supported:

1. Global Messages

- `$global.broadcastToAll(type: string, message: any)` - Broadcasts a message to all connected users globally.
- `$global.sendMessageToUser(type: string, account: string, message: any)` - Sends a direct message to a specific user account globally.

2. Room Messages

- `$room.broadcastToRoom(type: string, message: any)` - Broadcasts a message to all users in the current room.
- `$room.sendMessageToUser(type: string, account: string, message: any)` - Sends a direct message to a specific user within the current room.

Very Important: $global, $room, and $sender are all used in `server.js` (server-side code).

# Subscribing to State Changes on Client

The `server` object from `const { server } = useGameServer()` contains these subscription functions:

1. Global State Subscriptions

- `server.subscribeGlobalState((state: any) => {}): UnsubscribeFunction`
- `server.subscribeGlobalUserState(account, (state: any) => {}): UnsubscribeFunction`
- `server.subscribeGlobalMyState((state: any) => {}): UnsubscribeFunction`
- `server.subscribeGlobalCollection(collectionId, ({ items, changes } : { items: any[], changes: { op: 'add' | 'update' | 'delete' | 'deleteAll', items: any[]}}) => {}): UnsubscribeFunction`

2. Room State Subscriptions

- `server.subscribeRoomState(roomId, (state: {...state: any, $users: string[]}) => {}): UnsubscribeFunction`
- `server.subscribeRoomUserState(roomId, account, (state: any) => {}): UnsubscribeFunction`
- `server.subscribeRoomAllUserStates(roomId, (states: { ...state: any, account: string, updated: boolean }[]) => {}): UnsubscribeFunction` - All user states, The changed state, which is the cause of the subscription, is set to `updated: true`.
- `server.subscribeRoomMyState(roomId, (state: any) => {}): UnsubscribeFunction`
- `server.subscribeRoomCollection(roomId, collectionId, ({ items, changes} : { items: any[], changes: {op: 'add' | 'update' | 'delete' | 'deleteAll', items: any[]}}) => {}): UnsubscribeFunction`
- `server.onRoomUserJoin(roomId, (account: string) => {}): UnsubscribeFunction` - Triggered when a user joins the room.
- `server.onRoomUserLeave(roomId, (account: string) => {}): UnsubscribeFunction` - Triggered when a user leaves the room.

3. Message Receiving

- `server.onGlobalMessage(type: string, (message: any) => {})`
- `server.onRoomMessage(roomId: string, type: string, (message: any) => {})`

IMPORTANT: All subscribe functions are triggered when the state is updated. When subscribing, the current value is received once.

# Real-time State with React Hooks

In environments supporting React hooks, you can get real-time server state updates on the client without explicit subscriptions.

- Getting global state:

```tsx filename='App.tsx'
import {
useGlobalState,
useGlobalMyState,
useGlobalUserState,
useGlobalCollection,
} from "@agent8/gameserver";

const globalState = useGlobalState();
const myState = useGlobalMyState();
const userState = useGlobalUserState(account); // Specific user's state
const { items } = useGlobalCollection(collectionId); // Collection
```

- Getting room state (roomId is automatically handled in hooks):

```tsx filename='App.tsx'
import {
useRoomState,
useRoomMyState,
useRoomUserState,
useRoomAllUserStates,
useRoomCollection,
} from "@agent8/gameserver";
const roomState = useRoomState(); // Room public state
const roomMyState = useRoomMyState(); // my state
const userState = useRoomUserState(account); // Specific user's state
const states = useRoomAllUserStates(); // all users states in the room ({ ...state:any, account: string }[])
const { items } = useRoomCollection(collectionId); // Collection
```

## Asset Management

Agent8 provides comprehensive asset management for in-game currencies and fungible resources. All operations are automatically validated and synchronized:

### Server-side Asset API ($asset)
- `$asset.get(assetId: string): Promise<number>` - Get asset amount
- `$asset.getAll(): Promise<Record<string, number>>` - Get all assets
- `$asset.has(assetId: string, amount: number): Promise<boolean>` - Check asset availability
- `$asset.mint(assetId: string, amount: number): Promise<Record<string, number>>` - Create assets
- `$asset.burn(assetId: string, amount: number): Promise<Record<string, number>>` - Destroy assets
- `$asset.transfer(toAccount: string, assetId: string, amount: number): Promise<Record<string, number>>` - Transfer assets

**IMPORTANT**: Never expose mint/burn directly to clients. Always wrap in server-controlled logic:

```js filename='server.js'
class Server {
  // Server determines reward amount - client only sends missionId
  async completeMission(missionId) {
    const rewards = { 'daily': 100, 'boss': 500 };
    const reward = rewards[missionId];
    if (!reward) throw new Error('Invalid mission');
    return await $asset.mint('gold', reward);
  }

  // Server defines prices - client only sends itemId
  async buyItem(itemId) {
    const prices = { 'sword': 100, 'potion': 25 };
    const price = prices[itemId];
    if (!price) throw new Error('Item not found');
    if (!await $asset.has('gold', price)) throw new Error('Insufficient gold');
    await $asset.burn('gold', price);
    return await $asset.mint(itemId, 1);
  }
}
```

### Client-side Asset Hook (useAsset)

The `useAsset` hook provides **read-only** access to the current user's assets. All modifications must be done server-side.

```tsx
import { useAsset } from "@agent8/gameserver";
const { assets, getAsset, hasAsset } = useAsset();

// Check if user can afford something
if (hasAsset('gold', 100)) {
  // Call server function to perform the purchase
  await server.remoteFunction('buyItem', ['sword']);
}
```

## System Event Handlers

Special handlers called automatically by the system. Protected from client calls (`$on` prefix).

### $onItemPurchased - VX Shop Purchase Handler

Called when a user purchases an item from VX Shop. Use to deliver purchased items.

- `account: string` - User's wallet address
- `purchaseId: number` - Unique purchase ID
- `productId: string` - Product identifier
- `quantity: number` - Number purchased
- `metadata?: object` - Optional data

```js filename='server.js'
class Server {
  async $onItemPurchased({ account, purchaseId, productId, quantity }) {
    // Grant items by productId
    if (productId === 'gold_pack') {
      await $asset.mint('gold', 100 * quantity);
    } else if (productId === 'premium_skin') {
      const user = await $global.getMyState();
      await $global.updateMyState({ skins: [...(user.skins || []), productId] });
    } else {
      await $asset.mint(productId, quantity);
    }
    return { success: true };
  }
}
```

ULTRA IMPORTANT: Does not support `setInterval` or `setTimeout` in `server.js`. NEVER use them.
ULTRA IMPORTANT: `server.js` must be placed in the root of the project. <boltAction type="file" filePath="server.js">
