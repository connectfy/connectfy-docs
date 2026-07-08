# Connectfy Notification Action History Microservice Analysis

## Overview / Purpose
The `connectfy-notification-action-history` service is a NestJS hybrid microservice handling event-based actions and notifications. Its main responsibilities are synchronizing user projections from the event bus (Kafka), managing and persisting in-app notifications (such as friendship requests and system alerts), and sending outgoing emails via SMTP. It communicates with other services using both TCP and Kafka protocols.

## Full Folder Structure
```text
connectfy-notification-action-history/
└── src/
    ├── main.ts
    ├── i18n.ts
    ├── app.module.ts
    ├── common/
    │   └── constants/
    │       └── environment-variables.ts
    ├── interceptors/
    │   └── logged-user.interceptor.ts
    ├── projection/
    │   ├── projections.module.ts
    │   └── user/
    │       ├── user.module.ts
    │       ├── user.controller.ts
    │       ├── user.service.ts
    │       ├── repo/
    │       │   └── user.repo.ts
    │       ├── entity/
    │       │   ├── user.entity.ts
    │       │   └── nested/
    │       │       └── phoneNumber.entity.ts
    │       ├── interface/
    │       │   ├── user.interface.ts
    │       │   └── nested/
    │       │       └── phoneNumber.interface.ts
    │       └── dto/
    │           ├── add.user.dto.ts
    │           ├── edit.user.dto.ts
    │           ├── find.user.dto.ts
    │           ├── remove.user.dto.ts
    │           ├── base.user.dto.ts
    │           └── nested/
    │               └── phoneNumber.dto.ts
    ├── modules/
    │   ├── modules.module.ts
    │   ├── email/
    │   │   ├── email.module.ts
    │   │   ├── email.controller.ts
    │   │   ├── email.service.ts
    │   │   └── dto/
    │   │       └── send-email.dto.ts
    │   └── notification/
    │       ├── notification.module.ts
    │       ├── notification.controller.ts
    │       ├── notification.service.ts
    │       ├── repo/
    │       │   └── notification.repo.ts
    │       ├── entity/
    │       │   └── notification.entity.ts
    │       ├── interface/
    │       │   └── notification.interface.ts
    │       ├── utils/
    │       │   └── migrate-notifications.ts
    │       └── dto/
    │           ├── create-notification.dto.ts
    │           ├── find.notificaitons.ts
    │           ├── mark-read.notification.dto.ts
    │           ├── remove.notification.dto.ts
    │           └── update-friendship.notification.dto.ts
    └── app-settings/
        ├── app-settings.module.ts
        ├── cache/
        │   ├── cache.module.ts
        │   └── cache.service.ts
        ├── redis/
        │   └── redis.module.ts
        ├── kafka-connections/
        │   ├── kafka-connection.module.ts
        │   ├── kafka-connection.service.ts
        │   └── loggin-kafka.server.ts
        └── tcp-connections/
            ├── tcp-connection.module.ts
            └── tcp-connection.service.ts
```

## Architecture & Request/Event Flow
1. **Startup**: The application spins up both a Kafka consumer (`NestFactory.createMicroservice<MicroserviceOptions>`) and a TCP server. It utilizes a custom `LoggingKafkaServer` that inspects Kafka brokers on startup to verify if subscribed topics exist.
2. **Context Propagation (CLS)**: The `LoggedUserInterceptor` parses incoming RPC and Kafka payloads for `_loggedUser` and `_lang` properties, injecting them into Continuation-Local Storage (`nestjs-cls`). 
3. **Event Consumption**: Controllers in `projection/user`, `modules/email`, and `modules/notification` listen to specific Kafka topics. When a message is consumed, the services process it, update the MongoDB database, and manually commit the offset via `commitKafkaOffset`.
4. **TCP Requests**: For direct actions like retrieving paginated notifications or marking them as read, the TCP controller in `notification.controller.ts` is triggered. Context is automatically resolved via the CLS interceptor.
5. **Event Production**: When outbound communication is needed (e.g., emitting a `notification.created` event), services use the centralized `KafkaConnectionService`, which automatically enriches the payload with CLS context (`_loggedUser`, `_lang`) before emitting.
6. **Data Storage**: Uses Mongoose schemas backed by the repository pattern for all database interactions. `CacheService` acts as a Redis-backed caching layer, though it currently seems predominantly structured as a utility since repositories perform direct queries.

## Entities/Models

### `UserModel` (User Projection)
- `_id`: `string` (UUID, immutable)
- `firstName`: `string` (capitalized on set)
- `lastName`: `string` (capitalized on set)
- `fullName`: `string`
- `username`: `string` (lowercase, regex validated)
- `email`: `string` (lowercase, regex validated)
- `phoneNumber`: `PhoneNumberModel | null`
- `status`: `USER_STATUS` (enum: ACTIVE, etc.)
- `avatar`: `string | null`
- `createdAt`: `Date`
- `updatedAt`: `Date`
- **Indexes**: 
  - Compound: `{email: 1, status: 1}`, `{username: 1, status: 1}`, `{role: 1, status: 1}`, `{provider: 1, status: 1}`, `{createdAt: -1}`, `{updatedAt: -1}`, `{isTwoFactorEnabled: 1}` (Note: role, provider, isTwoFactorEnabled are listed in indexes but absent in schema definition).
  - Text: `{username: 'text', email: 'text'}`
  - Sparse: `{'phoneNumber.number': 1}`

### `PhoneNumberModel` (Nested in User)
- `countryCode`: `string | null`
- `number`: `string | null`
- `fullPhoneNumber`: `string | null`

### `NotificationModel`
- `_id`: `string` (UUID)
- `recipientId`: `string` (Indexed, Immutable, Refs User)
- `actorId`: `string | null` (Refs User)
- `type`: `NotificationType` enum
- `title`: `INotificationMessage | null`
- `body`: `INotificationMessage | null`
- `status`: `NotificationStatus` enum (default: Unread)
- `channel`: `NotificationChannel` enum (default: InApp)
- `resourceId`: `string | null`
- `resourceType`: `string | null`
- `metadata`: `Record<string, unknown> | null`
- `readAt`: `Date | null`
- `expiresAt`: `Date | null` (TTL Index: `expireAfterSeconds: 0`)
- `resolvedAt`: `Date | null`
- `createdAt`: `Date`
- `updatedAt`: `Date`
- **Indexes**:
  - `{recipientId: 1, status: 1, createdAt: -1}`
  - Unique Partial: `{recipientId: 1, actorId: 1, resourceId: 1, type: 1}` where type is `FRIENDSHIP_REQUEST_SENT`.

### `INotificationMessage` (Interface for title/body)
- `en`: `string`
- `az`: `string`
- `ru`: `string`
- `tr`: `string`

## Exposed APIs

### Kafka Consumed Topics (`@EventPattern`)
- `projection.user.created`
- `projection.user.updated`
- `projection.user.removed`
- `mail.send`
- `notification.friendship.created`

### TCP Message Patterns (`@MessagePattern` / `@EventPattern`)
- `notification/all` (MessagePattern: Retrieves paginated notifications)
- `notification/markRead` (MessagePattern: Marks specific notification as read)
- `notification/markAllRead` (MessagePattern: Marks all or an array of specific notifications as read)
- `notification/markUnread` (EventPattern: Marks specific notification as unread)
- `notification/countUnread` (MessagePattern: Returns integer count of unread)
- `notification/friendship/metadata` (MessagePattern: Updates status and resolution time of a friendship request notification)
- `notification/remove` (MessagePattern: Deletes specific notification)
- `notification/removeAll` (MessagePattern: Deletes multiple specific notifications)

### Kafka Produced Topics
- `notification.created` (Emitted after a notification is created or upserted)

## Core Features
1. **User Data Projection**: Maintains a read-optimized copy of user data (Projection Pattern) synced via Kafka events from the Auth/Account services.
2. **Notification Engine**: Manages the complete lifecycle of notifications (creation, pagination, read/unread toggling, resolution, and deletion). Includes specialized logic for friendship requests.
3. **Email Courier**: Dedicated consumer that intercepts `mail.send` events and dispatches them via `@nestjs-modules/mailer` through SMTP.
4. **Idempotent Friendship Notifications**: Ensures that duplicate friendship requests don't bloat the database by using MongoDB partial unique indexes and `findOneAndUpdate` with `$setOnInsert`.

## Technologies Used
- **Framework**: NestJS (Microservices module)
- **Language**: TypeScript
- **Database**: MongoDB with Mongoose
- **Message Broker**: Kafka (`kafkajs`, `@nestjs/microservices`)
- **Caching**: Redis (`ioredis`)
- **Email**: `@nestjs-modules/mailer`
- **Context Management**: `nestjs-cls` (Continuation-Local Storage)
- **Validation**: `class-validator` (via `connectfy-shared` field decorators)

## Implementation Notes & Gotchas
- **Manual Offset Commits**: Standard across controllers processing Kafka events (`commitKafkaOffset`), ensuring at-least-once delivery semantics but requiring explicit state management.
- **Context Injection Thin-Proxies**: The `TcpConnectionService` and `KafkaConnectionService` act as centralized thin-proxies. They automatically inject `_loggedUser` and `_lang` into outbound payloads so the caller doesn't have to manually propagate CLS state.
- **Cache Stampede Prevention**: `CacheService.getOrSet` utilizes an `inflightRequests` `Map` to guarantee that concurrent requests for a missing cache key only hit the database once.
- **Kafka Startup Topic Verification**: `LoggingKafkaServer` connects to the Kafka Admin API on startup to list existing topics and matches them against the `@MessagePattern`/`@EventPattern` decorators, logging helpful warnings for missing topics.
- **Schema Index Mismatch**: The `UserSchema` defines compound indexes containing fields like `role`, `provider`, and `isTwoFactorEnabled`, but these fields are not explicitly defined as `@Prop()` within the `UserModel`.
- **Friendship Upsert Logic**: `upsertFriendshipNotification` heavily relies on dynamic imports for `uuid` and `connectfy-shared` inside the method scope, likely to avoid circular dependencies or initial load time overhead.

## Known In-Progress / Partially Implemented Areas
- **Graceful Shutdown for Kafka**: `KafkaConnectionService.onModuleDestroy()` wraps `client.close()` in a try/catch with a comment noting that `close()` may or may not exist depending on the NestJS version, which might lead to unclosed connections on teardown.
- **One-off Migrations**: There is a raw mongoose script (`migrate-notifications.ts`) embedded in the `utils` directory intended to clean up duplicates and build the partial unique index. It's built to be run manually rather than via a proper migration pipeline.
- **i18n Implementation**: `i18n` is initialized and tested on bootstrap (`console.log`), but it does not appear heavily utilized dynamically across the service payload templates, relying instead on pre-built `INotificationMessage` objects with all translations baked in.
