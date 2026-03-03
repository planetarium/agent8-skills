---
name: gameserver-sdk-v2
description: Any logic related to the server must refer to this document. It provides methods for writing server code and communicating with the server from the client. When initially creating the server code for the project, the server is configured in the server/ directory using @agent8/gameserver-node. please read this document. This is the latest server implementation document. It includes a migration guide from the standalone legacy server.js to the latest version.
---

# Agent8 GameServer SDK

Agent8 GameServer is a fully managed game server solution. You don't need to worry about server setup or management when developing your game.

## IMPORTANT: Default Development Approach

**When helping users with server development, always follow this approach:**

### CRITICAL: NEVER Manually Create Configuration Files

**DO NOT manually create these files:**
- DO NOT create `server/package.json` manually
- DO NOT create `server/tsconfig.json` manually
- DO NOT create `server/src/server.ts` manually without init
- DO NOT create any server project files manually

**ALWAYS use the init command instead:**
```bash
npx -y @agent8/gameserver-node init
```

This command automatically generates all necessary files with correct configurations.

### For New Projects or Adding Server Logic:
1. **Always use the Structured Project approach** (TypeScript-based in `server/` directory)
2. **Check if `server/` directory exists**
3. **If it doesn't exist, YOU MUST initialize first:**
   ```bash
   npx -y @agent8/gameserver-node init
   ```
   **This step is MANDATORY before writing any server code.**
4. This creates the complete project structure automatically (package.json, tsconfig.json, server.ts, etc.)
5. **ONLY AFTER init completes**, modify `server/src/server.ts` and other files as needed
6. **NEVER skip the init step** - always run it first if `server/` directory doesn't exist

### Development Goal: Write Server Code AND Tests
**IMPORTANT:** When writing server code, always write test code as well. This is a critical part of development.

**Testing Workflow:**
1. Write server functions in `server/src/server.ts`
2. Write corresponding tests in `server/test/server.test.ts`
3. Run tests with `npx -y @agent8/gameserver-node test`
4. **Check test results:**
   - If you can read the terminal output, analyze the results and fix any failing tests
   - If you cannot read the terminal output, ask the user whether tests passed or failed
5. Iterate until all tests pass

**Goal:** Ensure both server code and tests are written and passing before considering the task complete.

### For Existing Projects with server.js:
- If the project already has `server.js` in the root, you may continue using it (legacy approach)
- However, recommend migrating to Structured Project for better maintainability
- **CRITICAL:** Legacy `server.js` does NOT use `export` - just define `class Server` without any export statement

**The Structured Project approach is now the standard and should be used by default.**

### Key Difference Summary:
- **Structured (`server/src/server.ts`)**: `export class Server { ... }` - export required
- **Legacy (`server.js`)**: `class Server { ... }` - NO export

## Recommended Approach: Structured Project

**For new projects or when adding server logic, always use the Structured Project approach with TypeScript.** This provides testing capabilities, better organization, and scalability.

Initialize with:
```bash
npx -y @agent8/gameserver-node init
```

This command automatically creates the `server/` directory structure with TypeScript configuration, testing setup, and organized project files. Once initialized, you can modify the generated files to implement your game logic.

### Legacy Approach: server.js

The simple `server.js` approach is the legacy method and should only be used for existing projects that already have a `server.js` file. For all new development, use the Structured Project approach described in the next section.

**CRITICAL DIFFERENCE: Legacy server.js does NOT use export**

Here's a basic example of the legacy server.js approach:

```js filename='server.js'
class Server {
  add(a, b) {
    return a + b;
  }
}

// DO NOT export in server.js (legacy approach)
// NEVER write: module.exports = Server;
// NEVER write: export class Server;
// NEVER write: export default Server;
```

**Important:** The key difference between legacy and structured approach:
- **Legacy `server.js`**: NO export statement (just define the class)
- **Structured `server/src/server.ts`**: MUST use `export class Server`

```tsx filename='App.tsx'
import React from "react";
import { useGameServer } from "@agent8/gameserver";

export default function App() {
  const { connected, server } = useGameServer();

  if (!connected) return "Connecting...";

  const callServer = async () => {
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

# Structured Project (Recommended)

This is the **recommended approach** for all server development in Agent8. It provides TypeScript support, testing capabilities, better organization, and scalability while maintaining backward compatibility with the legacy `server.js` approach.

## Initialization

**CRITICAL: Always initialize your server project with this command first.**

**DO NOT manually create configuration files** like `package.json`, `tsconfig.json`, or `server.ts`. These files must be generated by the init command.

If the `server/` directory doesn't exist in your project, run:

```bash
npx -y @agent8/gameserver-node init
```

**This command is MANDATORY before writing any server code.** It automatically generates a complete server structure:
- `server/src/server.ts` - Your main server logic
- `server/test/server.test.ts` - Test files
- `server/package.json` - Dependencies and configuration (DO NOT create manually)
- `server/tsconfig.json` - TypeScript configuration (DO NOT create manually)

After initialization, you can modify the generated files to implement your game logic. The workflow is:
1. **First:** Run `init` command
2. **Then:** Modify generated files
3. **Never:** Skip init and create files manually

## TypeScript Server Implementation

The `server/src/server.ts` file uses TypeScript and requires an `export` statement:

```typescript filename="server/src/server.ts"
export class Server {
  async ping(): Promise<string> {
    return 'pong';
  }

  async getMyAccount(): Promise<string> {
    return $sender.account;
  }

  async updateScore(points: number): Promise<number> {
    const myState = await $global.getMyState();
    const newScore = (myState.score || 0) + points;
    await $global.updateMyState({ score: newScore });
    return newScore;
  }
}
```

The functionality is identical to `server.js`, but uses TypeScript with type safety.

## Testing

Write tests in `server/test/server.test.ts`:

```typescript filename="server/test/server.test.ts"
describe('Server', () => {
  test('ping returns pong', async (server) => {
    const result = await server.ping();
    expect(result).toBe('pong');
  });

  test('connect changes user', async (server) => {
    server.connect({ account: 'user-alice' });
    const account = await server.getMyAccount();
    expect(account).toBe('user-alice');
  });
});
```

Run tests from your project directory:

```bash
npx -y @agent8/gameserver-node test
```

## Handler Namespaces

For better code organization, separate functionality into handlers using the `@Handler` decorator:

```typescript filename="server/src/userHandler.ts"
@Handler('user')
class UserHandler {
  async getMyAccount(): Promise<string> {
    return $sender.account;
  }
}
```

Access namespaced functions:

**In tests:**
```typescript
const account = await server.user.getMyAccount();
```

**From client:**
```typescript
const result = await server.remoteFunction('user.getMyAccount');
```

## Building and Deployment

**Building and deployment are automatically handled by the Agent8 platform.** You don't need to manually run build or deploy commands.

When you push your code to the repository or use the platform's deployment features:
1. The platform automatically builds your TypeScript server from `server/src/` to `server/dist/server.js`
2. The platform automatically deploys the built server

**Important**: If `./server.js` exists in the project root, it takes priority over `server/dist/server.js` during deployment.

## Isolated VM Environment Limitations

Your server code runs in an **isolated-vm** environment for security and performance:

### Available
- Provided contexts: `$sender`, `$global`, `$room`, `$asset`
- Pure JavaScript/TypeScript libraries (lodash, date-fns, etc.)
- Most computation and logic processing

### Not Available
- Node.js built-in modules: `fs`, `http`, `https`, `net`, `child_process`, etc.
- Network request libraries: `axios`, `node-fetch`, etc.
- File system access
- External process execution

## Structured Project Workflow

1. **Initialize:** `npx -y @agent8/gameserver-node init` (required if `server/` directory doesn't exist)
2. **Develop:** Write server logic in `server/src/`
3. **Test:** `npx -y @agent8/gameserver-node test`
4. **Build:** `npx -y @agent8/gameserver-node build` (generates server/dist/server.js)
5. **Deploy:** Push to repository - building and deployment are handled automatically by the platform

# Migrating from Legacy server.js to Structured Project

If you have an existing `server.js` file and want to migrate to the Structured Project approach, follow these steps:

## Migration Steps

1. **Initialize the structured project:**
   ```bash
   npx -y @agent8/gameserver-node init
   ```

2. **Convert to TypeScript:**
   - Move your code from `server.js` to `server/src/server.ts`
   - Add `export` to your Server class: `export class Server { ... }`
   - Add TypeScript type annotations to function parameters and return types
   - Convert JavaScript syntax to TypeScript where needed

3. **File Organization:**
   - **If your `server.js` is large (>200 lines)**, consider splitting it into multiple files
   - You can organize code in several ways:

     **Option A: Keep everything in Server class** (no client changes needed)
     ```
     server/src/
       server.ts          (main server class with all functions)
       utils/
         calculations.ts  (helper functions - not exposed to clients)
     ```
     - Client code remains the same: `remoteFunction('hello')`

     **Option B: Use handler namespaces with `@Handler` decorator** (requires client updates)
     ```
     server/src/
       server.ts          (main server class)
       gameHandler.ts     (@Handler('game') for game logic)
       userHandler.ts     (@Handler('user') for user management)
       utils/
         calculations.ts  (helper functions)
     ```
     - **Important:** Using handlers requires updating client code
     - Example: `remoteFunction('hello')` becomes `remoteFunction('game.hello')`
     - Choose this only if you want to organize functions into namespaces

4. **Migration Example:**

   **Before (server.js):**
   ```js
   class Server {
     async updateScore(points) {
       const myState = await $global.getMyState();
       const newScore = (myState.score || 0) + points;
       await $global.updateMyState({ score: newScore });
       return newScore;
     }
   }
   ```

   **After (server/src/server.ts):**
   ```typescript
   export class Server {
     async updateScore(points: number): Promise<number> {
       const myState = await $global.getMyState();
       const newScore = (myState.score || 0) + points;
       await $global.updateMyState({ score: newScore });
       return newScore;
     }
   }
   ```

5. **Write Tests:**
   - Create tests in `server/test/server.test.ts`
   - Test all migrated functions
   - Run `npx -y @agent8/gameserver-node test` to verify

6. **Deploy:**
   - Once tests pass, push to repository
   - The platform will automatically build and deploy

**IMPORTANT:** After successful migration and deployment, you can delete the old `server.js` file from the project root.

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

```typescript filename='server/src/server.ts'
export class Server {
  getMyAccount(): string {
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

```typescript filename='server/src/server.ts'
export class Server {
  getMyStatus(): { isFollower: boolean; isSubscriber: boolean; subscription: { tier: string; exp: number } | null } {
    return {
      isFollower: $sender.isFollower,
      isSubscriber: $sender.isSubscriber,
      subscription: $sender.subscription
    };
  }
}
```

# Global State Management

Multiplayer games require management of user states, items, rankings, etc.
Access global state through the `$global` variable in server code.
Agent8 provides three types of persistent global state:

1. Global Shared State

- `$global.getGlobalState(): Promise<Object>` - Retrieves current global state
- `$global.updateGlobalState(state: Object): Promise<Object>` - Updates global state

```typescript filename='server/src/server.ts'
export class Server {
  async getGlobalState(): Promise<any> {
    return $global.getGlobalState();
  }
  async updateGlobalState(state: any): Promise<void> {
    await $global.updateGlobalState(state);
  }
}
```

2. User State

- `$sender.account: string` - Request sender's account
- `$global.getUserState(account: string): Promise<Object>` - Gets specific user's state
- `$global.updateUserState(account: string, state: Object): Promise<Object>` - Updates specific user's state
- `$global.getMyState(): Promise<Object>` - Alias for `$global.getUserState($sender.account)`
- `$global.updateMyState(state: Object): Promise<Object>` - Alias for `$global.updateUserState($sender.account, state)`

3. Collection State (List management for specific keys, rankings)

- `$global.countCollectionItems(collectionId: string, options?: CollectionOptions): Promise<number>` - Retrieves number of items
- `$global.getCollectionItems(collectionId: string, options?: CollectionOptions): Promise<Item[]>` - Retrieves filtered/sorted items
- `$global.getCollectionItem(collectionId: string, itemId: string): Promise<Item>` - Gets single item by ID
- `$global.addCollectionItem(collectionId: string, item: any): Promise<Item>` - Adds new item
- `$global.updateCollectionItem(collectionId: string, item: any): Promise<Item>` - Updates existing item
- `$global.deleteCollectionItem(collectionId: string, itemId: string): Promise<{ __id: string }>` - Deletes item
- `$global.deleteCollection(collectionId: string): Promise<string>` - Deletes Collection

CollectionOptions uses Firebase-style filtering:

```typescript
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

```typescript filename='server/src/server.ts'
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

Example for joining the room:
```typescript filename='server/src/server.ts'
export class Server {
  private MAX_ROOM_USER = 3;

  async joinRoom(roomId?: string): Promise<string> {
    if (roomId) {
      if (await $global.countRoomUsers(roomId) >= this.MAX_ROOM_USER) {
        throw new Error('room is full');
      }
    }

    const joinedRoomId = await $global.joinRoom(roomId);

    if (await $global.countRoomUsers(joinedRoomId) === this.MAX_ROOM_USER) {
      // or you can use `await $room.countUsers() === this.MAX_ROOM_USER`
      await $room.updateRoomState({ status: 'START' });
    } else {
      await $room.updateRoomState({ status: 'READY' });
    }

    return joinedRoomId;
  }
}
```

The $room prefix automatically acts on the room that the $sender belongs to, so you don't need to explicitly specify roomId. All actions are queried and processed for the currently joined room.

Rooms provide a tick function that enables automatic server-side logic execution without explicit user calls.
The room tick will only repeat execution while users are present in the room.
The room tick is a function that is called every 100ms~1000ms (depends on the game's logic)

```typescript filename='server/src/server.ts'
export class Server {
  $roomTick(deltaMS: number, roomId: string): void {
    // ... your game logic here
  }
}
```

Important: Do not use `setInterval` or `setTimeout` in server code - you must use room ticks instead.

Room state can be accessed through the `$room` public variable in server code. Agent8 provides three types of room states that are not persisted - they get cleared when all users leave the room.

1. Room Public State

- `$room.getRoomState(): Promise<Object>` - Retrieves the current room state.
- `$room.updateRoomState(state: Object): Promise<Object>` - Updates the room state with new values.

Important: roomState contains these default values:

- `roomId: string`
- `$users: string[]` - Array of all user accounts in the room, automatically updated

2. Room User State

- `$sender.roomId: string` - The ID of the room the sender is in.
- `$room.getUserState(account: string): Promise<Object>` - Retrieves a particular user's state in the room.
- `$room.updateUserState(account: string, state: Object): Promise<Object>` - Updates a particular user's state in the room.
- `$room.getMyState(): Promise<Object>` - Retrieves $sender state in the room.
- `$room.updateMyState(state: Object): Promise<Object>` - Updates $sender state in the room.
- `$room.getAllUserStates(): Promise<Object[]>` - Retrieves all user states in the room.

3. Room Collection State

- `$room.countCollectionItems(collectionId: string, options?: CollectionOptions): Promise<number>` - Retrieves number of items from a room collection.
- `$room.getCollectionItems(collectionId: string, options?: CollectionOptions): Promise<Item[]>` - Retrieves multiple items from a room collection.
- `$room.getCollectionItem(collectionId: string, itemId: string): Promise<Item>` - Retrieves a single item from the room collection.
- `$room.addCollectionItem(collectionId: string, item: any): Promise<Item>` - Adds a new item to the specified room collection.
- `$room.updateCollectionItem(collectionId: string, item: any): Promise<Item>` - Updates an existing item in the room collection.
- `$room.deleteCollectionItem(collectionId: string, itemId: string): Promise<{ __id: string }>` - Deletes an item from the room collection.
- `$room.deleteCollection(collectionId: string): Promise<string>` - Delete room collection.

# Messaging

Simple socket messaging is also supported:

1. Global Messages

- `$global.broadcastToAll(type: string, message: any)` - Broadcasts a message to all connected users globally.
- `$global.sendMessageToUser(type: string, account: string, message: any)` - Sends a direct message to a specific user account globally.

2. Room Messages

- `$room.broadcastToRoom(type: string, message: any)` - Broadcasts a message to all users in the current room.
- `$room.sendMessageToUser(type: string, account: string, message: any)` - Sends a direct message to a specific user within the current room.

Very Important: $global, $room, and $sender are all used in server code (`server/src/server.ts` for Structured Projects).

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

```typescript filename='server/src/server.ts'
export class Server {
  // Server determines reward amount - client only sends missionId
  async completeMission(missionId: string): Promise<Record<string, number>> {
    const rewards: Record<string, number> = { 'daily': 100, 'boss': 500 };
    const reward = rewards[missionId];
    if (!reward) throw new Error('Invalid mission');
    return await $asset.mint('gold', reward);
  }

  // Server defines prices - client only sends itemId
  async buyItem(itemId: string): Promise<Record<string, number>> {
    const prices: Record<string, number> = { 'sword': 100, 'potion': 25 };
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

```typescript filename='server/src/server.ts'
export class Server {
  async $onItemPurchased({
    account,
    purchaseId,
    productId,
    quantity
  }: {
    account: string;
    purchaseId: number;
    productId: string;
    quantity: number;
  }): Promise<{ success: boolean }> {
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

ULTRA IMPORTANT: Does not support `setInterval` or `setTimeout` in server code. NEVER use them.

ULTRA IMPORTANT: Project Initialization:
- **ALWAYS run `npx -y @agent8/gameserver-node init` FIRST** if `server/` directory doesn't exist
- **NEVER manually create** `server/package.json`, `server/tsconfig.json`, or `server/src/server.ts` without running init
- **DO NOT skip the init command** - it generates all necessary configuration files correctly
- Only modify files AFTER init completes successfully

ULTRA IMPORTANT: Export syntax differences:
- **Structured Project (`server/src/server.ts`)**: MUST use `export class Server` - the export statement is REQUIRED
- **Legacy (`server.js`)**: MUST NOT use any export statement - just define `class Server` without export

ULTRA IMPORTANT: File locations:
- **Structured Projects**: server code must be in `server/src/server.ts` and built to `server/dist/server.js`
- **Legacy projects**: `server.js` must be placed in the project root
