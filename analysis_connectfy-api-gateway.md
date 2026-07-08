# Connectfy API Gateway Analysis

## Overview / Purpose
The `connectfy-api-gateway` is a NestJS-based entry point for the Connectfy application. It acts as a thin-proxy between external client traffic (HTTP REST, WebSockets) and internal domain microservices. Its main responsibilities include handling edge security (CORS, Helmet), maintaining sessions via Redis, validating authentication tokens, propagating request context (CLS), orchestrating distributed caches, and funneling HTTP requests as TCP messages to backend services. It also manages WebSocket connections for real-time notifications by consuming Kafka events.

## Full Folder Structure
```text
src/
├── app-settings/
│   ├── app-settings.module.ts
│   ├── cache/
│   │   ├── cache.module.ts
│   │   └── cache.service.ts
│   ├── http-connection/
│   │   ├── http-connection.module.ts
│   │   └── http-connection.service.ts
│   ├── kafka-connections/
│   │   └── loggin-kafka.server.ts
│   ├── redis/
│   │   └── redis.module.ts
│   └── tcp-connections/
│       ├── tcp-connection.module.ts
│       └── tcp-connection.service.ts
├── common/
│   ├── constants/
│   │   └── environment-variables.ts
│   ├── exception-filters/
│   │   └── all.filter.ts
│   └── functions/
│       └── request.ts
├── gateways/
│   └── socker.gateway.ts
├── guards/
│   ├── auth.guard.ts
│   └── safeQuery.guard.ts
├── interceptors/
│   ├── deviceId.interceptor.ts
│   └── logging.interceptor.ts
├── modules/
│   ├── modules.module.ts
│   ├── account/
│   │   ├── account.module.ts
│   │   ├── profile/
│   │   │   ├── profile.controller.ts
│   │   │   ├── profile.module.ts
│   │   │   └── profile.service.ts
│   │   ├── settings/
│   │   │   ├── settings.module.ts
│   │   │   ├── general/
│   │   │   │   ├── general-settings.controller.ts
│   │   │   │   ├── general-settings.module.ts
│   │   │   │   └── general-settings.service.ts
│   │   │   ├── notification/
│   │   │   │   ├── notification-settings.controller.ts
│   │   │   │   ├── notification-settings.module.ts
│   │   │   │   └── notification-settings.service.ts
│   │   │   └── privacy/
│   │   │       ├── privacy-settings.controller.ts
│   │   │       ├── privacy-settings.module.ts
│   │   │       └── privacy-settings.service.ts
│   │   └── social-link/
│   │       ├── social-link.controller.ts
│   │       ├── social-link.module.ts
│   │       └── social-link.service.ts
│   ├── auth/
│   │   ├── auth.module.ts
│   │   ├── auth/
│   │   │   ├── auth.controller.ts
│   │   │   ├── auth.module.ts
│   │   │   └── auth.service.ts
│   │   └── user/
│   │       ├── user.controller.ts
│   │       ├── user.module.ts
│   │       └── user.service.ts
│   ├── notification-action-history/
│   │   ├── notification-action-history.module.ts
│   │   └── notification/
│   │       ├── notification-event.controller.ts
│   │       ├── notification.controller.ts
│   │       ├── notification.gateway.ts
│   │       ├── notification.module.ts
│   │       └── notification.service.ts
│   ├── presence/
│   │   ├── presence.controller.ts
│   │   ├── presence.module.ts
│   │   └── presence.service.ts
│   └── relationship/
│       ├── relationship.module.ts
│       ├── blocklist/
│       │   ├── blocklist.controller.ts
│       │   ├── blocklist.module.ts
│       │   └── blocklist.service.ts
│       └── friendship/
│           ├── friendship.controller.ts
│           ├── friendship.module.ts
│           └── friendship.service.ts
├── app.module.ts
├── i18n.ts
└── main.ts
```

## Architecture & Request/Event Flow
1. **HTTP Requests**:
   - The Gateway is the `/api/v1` entrypoint.
   - Middlewares apply security headers (`helmet`), manage Redis-backed sessions (`express-session`), parse cookies.
   - Global Interceptors apply to the incoming request:
     - `LoggingInterceptor` calculates durations and colors the output into formatted terminal logs.
     - `DeviceIdInterceptor` validates the presence and format of an `x-device-id` header, placing it within `nestjs-cls` context.
   - Handlers marked with `@UseGuards(AuthGuard)` trigger authentication logic. The guard checks Redis for cached JWT payload validity and ensures the user exists using a direct downstream TCP call if not already cached. Token and user entities are populated into the CLS context.
   - The requested `Controller` method catches the payload and passes it directly to its paired `Service`.
   - The Service formulates a payload and invokes `TcpConnectionService`.
   - `TcpConnectionService` appends `_loggedUser` and `_lang` from `nestjs-cls` into the payload and dispatches it over TCP (`@nestjs/microservices`) to a targeted backend service.
   - Upon receiving the TCP response, the `Service` often orchestrates cache invalidation (via `CacheService`) or cache updates before returning to the frontend.

2. **WebSocket & Kafka Flow**:
   - Clients connect via Socket.io to `/notification` or `/`.
   - The gateway joins the client into a specific room based on the verified User ID.
   - The gateway consumes Kafka messages from the topic `notification.created`.
   - When a message arrives, `NotificationEventController` passes the payload to `NotificationGateway`.
   - `NotificationGateway` emits a `notification:new` event explicitly to the room of the designated `recipientId`.

## Entities / Models
The API Gateway does not act as the primary owner of domain schemas. Due to its proxy nature, models are handled generically via `any` or implied typing. However, based on the codebase, we can deduce fields handled directly in the gateway layer:

- **Auth Session Context**:
  - `unverifiedUser`: any (Stores signup payload during verification process)
  - `verifyCode`: string (Holds code sent for email verification)
  - `twoFaCode`: string (Holds 2FA session code)
  - `userId`: string (Holds User ID temporarily during 2FA)
- **Token Payload (Implicit)**:
  - `_id`: string (Identifies the user)
  - `language`: string (e.g., 'en', used for localization)
- **Parsed Context Context (CLS)**:
  - `DEVICE_ID`: string (UUID strictly validated)
  - `USER`: Record<string, any>
  - `LANG`: string
- **Request Data Payload**:
  - `headers['user-agent']`: string
  - `headers['x-forwarded-for']`: string
  - `headers['x-real-ip']`: string
  - `headers['cf-connecting-ip']`: string
  - `ip`: string

## Exposed APIs

### REST Endpoints (Prefixed with `/api/v1`)
**Presence:**
- `POST /user/heartbeat` - Marks a user as active in Redis.

**Auth Module:**
- `POST /auth/signup` - Init signup sequence
- `POST /auth/signup/verify` - Verify signup OTP
- `POST /auth/signup/verify/resend` - Resend signup OTP
- `POST /auth/login` - Init login sequence
- `POST /auth/login/verify` - Verify login 2FA OTP
- `POST /auth/google/login` - OAuth Google Login
- `POST /auth/google/signup` - OAuth Google Signup
- `POST /auth/refresh` - Refresh access token using cookie
- `POST /auth/logout` - Logout & clear caches
- `POST /auth/restore-account`
- `POST /auth/forgot-password`
- `POST /auth/reset-password`
- `POST /auth/authenticate-user`
- `POST /auth/is-valid-token`
- `GET /auth/internal/verify`

**User Module:**
- `POST /user/me` - Get cached current user data
- `PATCH /user/change-username`
- `PATCH /user/change-email`
- `PATCH /user/change-email/verify`
- `PATCH /user/change-password`
- `PATCH /user/change-phone-number`
- `POST /user/check-unique`
- `PATCH /user/two-factor`
- `POST /user/delete-account`
- `POST /user/deactivate-account`

**Account & Profiles:**
- `POST /account/profile/get`
- `POST /account/profile/findOneByUserId/:_id`
- `PATCH /account/profile/update`
- `PATCH /account/profile/update-avatar`
- `PATCH /account/profile/update-default-avatar`
- `POST /account/profile/search`
- `POST /account/social-link/get`
- `POST /account/social-link/create`
- `PATCH /account/social-link/update`
- `PATCH /account/social-link/update-rank`
- `DELETE /account/social-link/remove`
- `DELETE /account/social-link/removeMany`

**Settings:**
- `POST /account/settings/notification-settings/get`
- `PATCH /account/settings/notification-settings/update`
- `POST /account/settings/general-settings/get`
- `PATCH /account/settings/general-settings/update`
- `PATCH /account/settings/general-settings/reset`
- `POST /account/settings/privacy-settings/get`
- `PATCH /account/settings/privacy-settings/update`

**Relationships (Friendship/Blocklist):**
- `POST /relationship/blocklist/find-blocked`
- `POST /relationship/blocklist/unblock`
- `POST /relationship/blocklist/block`
- `POST /relationship/friendship/find-requests`
- `POST /relationship/friendship/find-friends`
- `POST /relationship/friendship/send-request`
- `POST /relationship/friendship/accept-request`
- `POST /relationship/friendship/decline-request`
- `POST /relationship/friendship/cancel-request`
- `POST /relationship/friendship/unfriend`
- `POST /relationship/friendship/update-close-friend`
- `POST /relationship/friendship/update-notification`
- `POST /relationship/friendship/requests-count`
- `POST /relationship/friendship/find-online-friends` (Uses proxy to combine friends IDs with Redis heartbeat data to return active friends)
- `POST /relationship/friendship/find-suggestions`
- `POST /relationship/friendship/find-mutual-friends`

**Notifications:**
- `POST /notification-action-history/notification/all`
- `POST /notification-action-history/notification/countUnread`
- `POST /notification-action-history/notification/markRead`
- `POST /notification-action-history/notification/markUnread`
- `POST /notification-action-history/notification/markAllRead`
- `POST /notification-action-history/notification/remove`
- `POST /notification-action-history/notification/removeAll`

### WebSockets Events
- **Namespace `/` (`MySocketGateway`)**:
  - `chat` (Subscribes to input, Emits `"salam"` back out, unused mapping for specific users exists).
- **Namespace `/notification` (`NotificationGateway`)**:
  - `notification:new` (Emits out to user room `user:<id>`).

### Kafka Subscriptions
- **Consumed**: `notification.created`
- **Produced**: None explicitly created by the Gateway.

### TCP Microservice Integrations
Uses NestJS `Transport.TCP` mapped to remote host services.
- **AUTH_SERVICE**: `user/change-username`, `user/change-email`, `user/change-email/verify`, `user/change-password`, `user/change-phone-number`, `user/check-unique`, `user/two-factor`, `user/delete-account`, `user/deactivate-account`, `auth/signup`, `auth/verify-signup`, `auth/verify-signup/resend`, `auth/login`, `auth/verify-login`, `auth/google/login`, `auth/google/signup`, `auth/forgot-password`, `auth/reset-password`, `auth/refreshToken`, `auth/logout`, `auth/restore-account`, `auth/authenticate-user`, `auth/is-valid-token`, `auth/refresh-token/verify-token`.
- **ACCOUNT_SERVICE**: `profile/findOne`, `profile/findOneByUserId`, `profile/search`, `profile/update`, `profile/updateAvatar`, `profile/updateDefaultAvatar`, `notification-settings/get`, `notification-settings/update`, `general-settings/get`, `general-settings/update`, `general-settings/reset`, `privacy-settings/get`, `privacy-settings/update`, `socialLinks/findMany`, `socialLinks/create`, `socialLinks/update`, `socialLinks/updateRank`, `socialLinks/remove`, `socialLinks/removeMany`.
- **RELATIONSHIP_SERVICE**: `blocklist/findMany`, `blocklist/remove`, `blocklist/create`, `friendship/findRequests`, `friendship/findFriends`, `friendship/create`, `friendship/acceptFriendshipRequest`, `friendship/declineFriendshipRequest`, `friendship/cancelFriendshipRequest`, `friendship/unfriend`, `friendship/updateCloseFriend`, `friendship/updateNotification`, `friendship/requestsCount`, `friendship/findFriendIds`, `friendship/findSuggestions`, `friendship/findMutualFriends`.
- **NOTIFICATION_ACTION_HISTORY_SERVICE**: `notification/all`, `notification/countUnread`, `notification/markRead`, `notification/markUnread`, `notification/markAllRead`, `notification/remove`, `notification/removeAll`.
- **MESSENGER_SERVICE**: Connected, but explicit endpoints are not leveraged in this dump natively.

## Core Features
1. **Aggregator & Delegator**: Strips routing concerns away from the microservices.
2. **Heavy State Management & Caching Engine**: Relies deeply on Redis deduplication via `CacheService.getOrSet` implementing in-flight request deduping. User payload and profiles are persistently cached and strategically wiped during mutations.
3. **Session & Web Contextualization**: Manages HTTP context (IP, User-Agent, Devices) merging it inside CLS before TCP firing. Uses Redis-backed `express-session` purely for transient auth flows (e.g., intermediate OTP checks).
4. **Real-time Engine**: Broadcasts Kafka notifications directly to individual user WebSocket channels.
5. **Presence Tracking**: Exposes a heartbeat endpoint mapping activity directly in Redis keys (`presence:online:<id>`), which enables querying subset lists safely (e.g. `findOnlineFriends`).

## Technologies Used
- NestJS (Express adapter)
- KafkaJS (`@nestjs/microservices`)
- Socket.IO (`@nestjs/websockets`)
- Redis & ioredis (Caching, Presence, Sessions)
- connect-redis & express-session
- JWT (`@nestjs/jwt`)
- `nestjs-cls` (Continuations Local Storage implementation for Node contexts)

## Implementation Notes & Gotchas
- **Thin Proxy Pattern**: Business logic deliberately omitted. Endpoints strictly format request payloads and fire to TCP microservices. 
- **In-flight Deduplication**: `CacheService` caches unresolved Promises internally (`inflightRequests` Map) to solve Thundering Herd Cache Stampedes when resolving identical keys simultaneously.
- **Safe Query Generator**: Located in `SafeQueryGuard`, acts as an injection shield by translating arbitrary client JSON (e.g. `{"operator": "between", "value": [1,10]}`) directly into safe Mongo operators (`$gt`, `$lte`, `$regex`). It is intended to run as a guard but transforms body structure globally.
- **CLS Context Hydration**: `TcpConnectionService.sendTcpWithContext()` forces `_loggedUser` and `_lang` down into every TCP request dynamically without developers passing it individually per method.
- **Presence Composition API**: A rare piece of application logic found in the gateway is in `findOnlineFriends`. It fetches friend IDs from the relationship microservice via TCP, bulk queries Redis for heartbeats, and composes an online subset, bypassing backend persistence.

## Known In-Progress / Partially Implemented Areas
- **Socket.io Generic Root (`/`)**: Currently maps user arrays against `userSocketsMap`, but only exposes a basic `"chat"` ping-pong listener. Full messenger implementation mapping appears incomplete on the gateway side compared to real notification socket handling.
- **Message Service Integration**: `MICROSERVICE_NAMES.TCP.MESSENGER` is bound to the module and instantiated, but never actively queried or exposed to REST controllers.
