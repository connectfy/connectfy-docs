# connectfy-relationship Microservice Analysis

## Overview / Purpose
The `connectfy-relationship` service is a NestJS microservice responsible for handling user relationships within the Connectfy ecosystem. Specifically, it manages:
- **Friendships**: sending, accepting, declining, and canceling friendship requests; unfriending; and updating settings like close friends (favorites) and notifications (muting).
- **Blocklists**: blocking and unblocking users, which restricts friendship interactions and access.
- **Projections**: It maintains a projected (read-optimized) local copy of the `User` entities by listening to Kafka events (user creation, update, deletion).

## Full Folder Structure
```
connectfy-relationship/
├── src/
│   ├── main.ts
│   ├── i18n.ts
│   ├── app.module.ts
│   ├── common/
│   │   ├── constants/
│   │   │   └── environment-variables.ts
│   │   └── base/
│   │       └── bullmq.processor.base.ts
│   ├── interceptors/
│   │   └── logged-user.interceptor.ts
│   ├── app-settings/
│   │   ├── app-settings.module.ts
│   │   ├── kafka-connections/
│   │   │   ├── loggin-kafka.server.ts
│   │   │   ├── kafka-connection.service.ts
│   │   │   └── kafka-connection.module.ts
│   │   ├── tcp-connections/
│   │   │   ├── tcp-connection.service.ts
│   │   │   └── tcp-connection.module.ts
│   │   ├── bullMq-connection/
│   │   │   └── bullMq-connection.service.ts
│   │   └── cache/
│   │       └── cache.service.ts
│   ├── projection/
│   │   ├── projections.module.ts
│   │   └── user/
│   │       ├── user.module.ts
│   │       ├── user.controller.ts
│   │       ├── user.service.ts
│   │       ├── repo/
│   │       │   └── user.repo.ts
│   │       ├── entity/
│   │       │   ├── user.entity.ts
│   │       │   └── nested/
│   │       │       └── phoneNumber.entity.ts
│   │       ├── dto/
│   │       │   ├── add.user.dto.ts
│   │       │   ├── edit.user.dto.ts
│   │       │   ├── remove.user.dto.ts
│   │       │   ├── find.user.dto.ts
│   │       │   ├── base.user.dto.ts
│   │       │   └── nested/
│   │       │       └── phoneNumber.dto.ts
│   │       └── interface/
│   │           ├── user.interface.ts
│   │           └── nested/
│   │               └── phoneNumber.interface.ts
│   └── modules/
│       ├── modules.module.ts
│       ├── blocklists/
│       │   ├── blocklists.module.ts
│       │   └── blocklist/
│       │       ├── blocklist.module.ts
│       │       ├── blocklist.controller.ts
│       │       ├── blocklist.service.ts
│       │       ├── repo/
│       │       │   └── blocklist.repo.ts
│       │       ├── entity/
│       │       │   └── blocklist.entity.ts
│       │       ├── dto/
│       │       │   ├── add.blocklist.dto.ts
│       │       │   ├── remove.blocklist.dto.ts
│       │       │   ├── base.blocklist.dto.ts
│       │       │   ├── find.blocklist.dto.ts
│       │       │   └── actions.blocklist.dto.ts
│       │       └── interface/
│       │           └── blocklist.interface.ts
│       └── friendships/
│           ├── friendships.module.ts
│           └── friendship/
│               ├── friendship.module.ts
│               ├── friendship.controller.ts
│               ├── friendship.service.ts
│               ├── repo/
│               │   └── friendship.repo.ts
│               ├── entity/
│               │   └── friendship.entity.ts
│               ├── dto/
│               │   ├── add.friendship.dto.ts
│               │   ├── actions.friendship.dto.ts
│               │   └── find.friendship.dto.ts
│               ├── interface/
│               │   └── friendship.interface.ts
│               ├── constants/
│               │   └── friendship.bullmq.ts
│               └── publishers/
│                   └── notification/
│                       ├── friendship.notification.processor.ts
│                       └── friendship.notification.service.ts
```

## Architecture & Request/Event Flow
1. **Transport Layer**: The service runs both a TCP server (for internal synchronous RPC queries from an API Gateway or other services) and a Kafka consumer (for asynchronous event-driven state updates).
2. **Context Logging (CLS)**: The `LoggedUserInterceptor` extracts `_loggedUser` and `_lang` from incoming RPC requests, storing them using `nestjs-cls` for contextual availability throughout the request lifecycle without passing them down explicitly.
3. **Projection**: 
   - When a user is created, updated, or removed in the central `auth` service, a Kafka event (`projection.user.*`) is emitted.
   - `UserController` consumes these Kafka events and replicates changes into a local `USERS` MongoDB collection.
4. **Data Persistence**: Uses MongoDB (Mongoose). Repositories extend a generic `BaseRepository`.
5. **Caching**: Uses Redis cache (via `CacheService` in `app-settings/cache/cache.service.ts`) for storing privacy settings.
6. **Background Tasks/Notifications**: 
   - Uses BullMQ queues to defer processing of side effects like sending notifications (e.g. `REQUEST_SENT`, `REQUEST_ACCEPTED`).
   - `FriendshipNotificationProcessor` picks up jobs and processes notifications (likely calling other microservices via TCP).
7. **Inter-service Communication**:
   - Outbound synchronous RPC calls are made via `TcpConnectionService` (e.g. fetching account privacy settings or syncing notification action history metadata).
   - The blocklist service is injected into the friendship service to ensure blocked users cannot send friendship requests.

## Entities / Models

### 1. UserModel (Projection)
**Collection**: `USERS`
Fields:
- `_id`: `string` (UUID, default: `uuid()`, immutable)
- `firstName`: `string`
- `lastName`: `string`
- `fullName`: `string`
- `username`: `string`
- `email`: `string`
- `phoneNumber`: `PhoneNumberModel | null` (Nested object)
  - `countryCode`: `string | null`
  - `number`: `string | null`
  - `fullPhoneNumber`: `string | null`
- `status`: `USER_STATUS` enum (e.g., `ACTIVE`)
- `avatar`: `string | null`
- `createdAt`: `Date`
- `updatedAt`: `Date`

*Indexes*: `{ email: 1, status: 1 }`, `{ username: 1, status: 1 }`, `{ role: 1, status: 1 }`, `{ provider: 1, status: 1 }`, `{ createdAt: -1 }`, `{ updatedAt: -1 }`, text index on `{ username: 'text', email: 'text' }`, sparse index on `{ 'phoneNumber.number': 1 }`.

### 2. BlocklistModel
**Collection**: `BLOCKS_LIST`
Fields:
- `_id`: `string` (UUID, default: `uuid()`, immutable)
- `blocker`: `string` (UUID, required, ref to `USERS`)
- `blocked`: `string` (UUID, required, ref to `USERS`)
- `createdAt`: `Date`
- `updatedAt`: `Date`

*Indexes*: Compound unique index on `{ blocker: 1, blocked: 1 }`.

### 3. FriendshipModel
**Collection**: `FRIENDSHIPS`
Fields:
- `_id`: `string` (UUID, default: `uuid()`, immutable)
- `userId`: `string` (UUID, required, ref to `USERS`)
- `friendId`: `string` (UUID, required, ref to `USERS`)
- `status`: `FriendshipStatus` enum (e.g., `Pending`, `Accepted`)
- `isFavorite`: `boolean` (default: `false`)
- `isMuted`: `boolean` (default: `false`)
- `createdAt`: `Date`
- `updatedAt`: `Date`

*Indexes*: Unique compound index on `{ userId: 1, friendId: 1 }`. Additional indexes on `{ userId: 1, status: 1 }`, `{ userId: 1, isFavorite: 1 }`, `{ userId: 1, isMuted: 1 }`.

## Exposed APIs

### Kafka Consumers (Event Patterns)
- `@EventPattern('projection.user.created')` - Creates a local projected user.
- `@EventPattern('projection.user.updated')` - Updates a local projected user.
- `@EventPattern('projection.user.removed')` - Removes a local projected user.

### TCP Message Patterns (Message Patterns)

**Blocklist Controller:**
- `@MessagePattern('blocklist/findMany')`: Retrieves paginated lists of blocked users for a given `blockerId`.
- `@MessagePattern('blocklist/isExist')`: Checks if a block exists between two users.
- `@MessagePattern('blocklist/findBlockedUserIds')`: Retrieves an array of IDs of users blocked by or who blocked the given user.
- `@MessagePattern('blocklist/remove')`: Unblocks a user.
- `@MessagePattern('blocklist/create')`: Blocks a user.

**Friendship Controller:**
- `@MessagePattern('friendship/create')`: Sends a friendship request.
- `@MessagePattern('friendship/acceptFriendshipRequest')`: Accepts a pending friendship request.
- `@MessagePattern('friendship/declineFriendshipRequest')`: Declines a pending friendship request.
- `@MessagePattern('friendship/cancelFriendshipRequest')`: Cancels a sent friendship request.
- `@MessagePattern('friendship/unfriend')`: Removes an accepted friendship connection.
- `@MessagePattern('friendship/updateCloseFriend')`: Toggles `isFavorite` (Close Friend) status.
- `@MessagePattern('friendship/updateNotification')`: Toggles `isMuted` (Notifications) status.
- `@MessagePattern('friendship/findManyInternal')`: Retrieves relationships involving the `currentUserId` and an array of `targetUserIds` for internal API aggregation.
- `@MessagePattern('friendship/findOneInternalWithCount')`: Retrieves relationship between two users along with an optional count.
- `@MessagePattern('friendship/findFriends')`: Retrieves paginated list of friends (with filters for `isFavorite`/`isMuted` if requester is owner).
- `@MessagePattern('friendship/findRequests')`: Retrieves paginated friend requests (sent or received).
- `@MessagePattern('friendship/requestsCount')`: Returns count of pending friend requests received.
- `@MessagePattern('friendship/findFriendIds')`: Returns an array of friend IDs.
- `@MessagePattern('friendship/findSuggestions')`: Finds user suggestions based on mutual connections.
- `@MessagePattern('friendship/findMutualFriends')`: Returns paginated mutual friends between the current user and a target user.

## Core Features
1. **Directional Friendship Requests**: `userId` initiates the request to `friendId` with `status: Pending`. Upon acceptance, a reciprocal record may be created or the original is updated to `status: Accepted` (based on internal architecture flow logic; code updates the existing request and creates a reciprocal one).
2. **Blocking System**: Allows blocking a user. The friendship service enforces block checks (e.g. blocking prevents sending/accepting requests).
3. **Mutual Friends and Suggestions**: Advanced aggregation pipelines to calculate mutual connections and provide friend suggestions. It cascades down to random users if no mutual connection suggestions exist.
4. **Context-Aware RPCs**: Context propagation is strictly maintained (using `nestjs-cls` to pass down the logged-in user and localized language keys).
5. **Notifications via Queue**: Notification creation tasks (`REQUEST_SENT`, `REQUEST_ACCEPTED`) are deferred to a BullMQ worker (`friendship.notification.processor.ts`).
6. **Cross-Service Validations**: Makes TCP RPC calls to other services like `account` to fetch privacy settings (`privacy-settings/findOne`) to check if the target allows friend requests.

## Technologies Used
- **Framework**: NestJS (Microservices module)
- **Database**: MongoDB (with Mongoose)
- **Message Broker / Transport**: KafkaJs (Asynchronous), TCP (Synchronous RPC)
- **Queues**: BullMQ & Redis (Background Tasks, Caching)
- **Context Management**: nestjs-cls (Continuation Local Storage)
- **Internationalization**: i18next

## Implementation Notes / Gotchas
- **Projection Pattern**: Instead of making synchronous calls to an auth/user service for user data, this microservice subscribes to `projection.user.*` Kafka topics and replicates user data into its own DB, providing faster `populate()` and aggregation.
- **Directional Friendship Storage**: When a request is accepted, a reciprocal record is created (`friendId -> userId`), so lookups become fast regardless of who queried the friendship.
- **Aggregations over Population**: Complex pipelines in `findManyInternal` and `findSuggestions` use `$lookup` and `$facet` rather than relying solely on Mongoose `populate()`, highlighting performance-centric database logic.
- **Interceptors for CLS**: The `LoggedUserInterceptor` parses the `_loggedUser` from the RPC `data` payload and saves it to CLS, removing the need for manual context parsing in services.
- **BullMQ for Notification Workload**: Decoupling real-time notification sending out of the core request-response loop into a BullMQ processor `FriendshipNotificationProcessor`.
- **Manual Commit**: The `commitKafkaOffset(context)` is explicitly called when Kafka events are successfully processed by the projection consumers.

## Known In-Progress / Partially Implemented Areas
- Friend suggestions fallback logic defaults to fetching random users when no mutual friends are found, which may not be the most optimal "suggestion" algorithm but acts as a functional stopgap.
- Some queries (e.g. `isExist` in BlocklistService) retrieve a larger subset of documents to evaluate the relationship rather than relying solely on optimal database-level boolean aggregations.
