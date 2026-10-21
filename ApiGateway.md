# API Gateway

## Overview
`connectfy-api-gateway` is the HTTP entry point for the system. It exposes REST endpoints for the web client, manages sessions and cookies, validates access tokens, and forwards requests to backend microservices over NestJS TCP clients.

It mainly orchestrates `connectfy-auth`, `connectfy-account`, `connectfy-relationship`, `connectfy-notification-action-history`, and the file uploader service.

## Folder Structure
```text
connectfy-api-gateway/
├── src/
│   ├── app-settings/
│   │   ├── cache/
│   │   ├── http-connection/
│   │   ├── redis/
│   │   └── tcp-connections/
│   ├── common/
│   │   ├── constants/
│   │   ├── exception-filters/
│   │   └── functions/
│   ├── guards/
│   ├── interceptors/
│   └── modules/
│       ├── auth/
│       ├── account/
│       ├── relationship/
│       └── files/
├── Dockerfile.dev
├── Dockerfile.prod
└── docker-compose.*.yml
```

Main areas:
- `app-settings/tcp-connections`: wraps downstream TCP clients.
- `guards/auth.guard.ts`: verifies JWTs, refresh cookies, and hydrates CLS context.
- `modules/*`: HTTP controllers grouped by domain.

## Architecture & Flow
1. Client sends HTTP requests to `/api/v1/*`.
2. `main.ts` applies Helmet, CORS, Redis-backed sessions, cookie parsing, logging, and device ID interception.
3. `AuthGuard` validates `Authorization: Bearer` plus `refresh_token` cookie, caches token/user lookups, and stores `_loggedUser` in CLS.
4. Controllers call service classes, which proxy requests through `TcpConnectionService`.
5. File upload requests are forwarded over HTTP to the file uploader service; business domains use TCP microservices.

## Entities / Models
This service does not own database entities. Its core runtime models are:
- Session state: Redis-backed `n_sid` session containing signup and 2FA verification data.
- CLS request context: language, device ID, and logged-in user.
- Cached auth state: access token payloads and verified user snapshots.

## Core Features
- Central REST API for auth, profile, settings, friendship, and upload flows.
- Session-based signup and login verification flows.
- Refresh-token cookie handling.
- Request context propagation to downstream services.
- Redis-backed caching for repeated auth checks.
- Runs as two instances behind nginx (`ip_hash`): Socket.IO rooms go through the Redis adapter, and 1:1 call state, group-call presence and the media-change lock live in Redis (see Notes).

## APIs
Base prefix: `/api/v1`

### Auth
- `POST /auth/signup`
- `POST /auth/signup/verify`
- `POST /auth/signup/verify/resend`
- `POST /auth/login`
- `POST /auth/login/verify`
- `POST /auth/google/login`
- `POST /auth/google/signup`
- `POST /auth/refresh`
- `POST /auth/logout`
- `POST /auth/restore-account`
- `POST /auth/forgot-password`
- `POST /auth/reset-password`
- `POST /auth/authenticate-user`
- `POST /auth/is-valid-token`

### User
- `POST /user/me`
- `PATCH /user/change-username`
- `PATCH /user/change-email`
- `PATCH /user/change-email/verify`
- `PATCH /user/change-password`
- `PATCH /user/change-phone-number`
- `POST /user/check-unique`
- `PATCH /user/two-factor`
- `POST /user/delete-account`
- `POST /user/deactivate-account`

### Account
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
- `POST /account/settings/general-settings/get`
- `PATCH /account/settings/general-settings/update`
- `PATCH /account/settings/general-settings/reset`
- `POST /account/settings/privacy-settings/get`
- `PATCH /account/settings/privacy-settings/update`
- `POST /account/settings/notification-settings/get`
- `PATCH /account/settings/notification-settings/update`

### Relationship and Files
- `POST /relationship/friendship/find-requests`
- `POST /relationship/friendship/find-friends`
- `POST /relationship/friendship/send-request`
- `POST /relationship/friendship/accept-request`
- `POST /relationship/friendship/decline-request`
- `POST /relationship/friendship/cancel-request`
- `POST /relationship/friendship/unfriend`
- `POST /relationship/friendship/update-close-friend`
- `POST /relationship/friendship/update-notification`
- `POST /files/presigned-upload`

### Web Push (since 2026-10-19, `modules/notification-action-history/push`)
All behind `AuthGuard`, forwarded to notification-action-history (`push/*` TCP patterns):
- `GET /push/public-key`: the VAPID public key (null when the service has no keys).
- `POST /push/subscription {endpoint, keys}`: upsert this browser's subscription. `deviceId` comes from `CLS_KEYS.DEVICE_ID` and the user agent from the request header, never from the body. POST, not PUT: CORS `methods` has no PUT.
- `DELETE /push/subscription {endpoint}`: the caller's own row only.
- `POST /push/test`: development only (refused in production).
- **Call pushes (Phase 5):** `ChatGateway` decides what happened to a 1:1 call and tells
  notification-action-history over TCP without waiting (`CallPushNotifier` → `push/call {kind,
  callId, chatId, callerUserId, calleeUserId}`):
  - `incoming` after `call:invite`, unless the callee is already on a call;
  - `answered` on the first `call:answer`; `declined` on `call:decline`;
  - `missed` on `call:cancel`, `call:busy`, a `call:left`/`call:hangup` while it rings, the
    caller's socket going away before an answer, and the registry's 50 s ring deadline.
  `GroupCallsService.start` sends `push/groupCall {callId, chatId, chatName, startedBy, userIds:
  ringUserIds}`.
- **Pending-invite replay:** `CallSessionRegistry` keeps the sealed invite while the call rings
  (a per-callee Redis set, cleared on answer, refusal, end or the 50 s deadline, which is a Redis sorted set drained by every instance). A socket that
  sends `call:pending` gets every invite still ringing for its user, emitted to that socket only,
  so a tab opened from the push can still answer. A `call:left` while ringing (the caller's own
  no-answer) and the deadline both send the callee `call:cancel {reason: 'no-answer'}`.

## Technologies Used
- NestJS
- Redis + `connect-redis`, `@socket.io/redis-adapter`
- JWT
- `nestjs-cls`
- TCP microservices
- Axios
- Helmet
- Express sessions

## Notes
- The gateway is intentionally thin: business rules live in downstream services.
- `AuthGuard` trusts both an access token and a persisted refresh token/device session.
- Session storage is used for short-lived verification flows, not primary user data.
- **Multi-instance (since 2026-10-20, [v1.73.0](../claude/docs/versions/v1.73.0_multi_instance_gateway.md)):** two gateways (`:3000`, `:3001`) sit behind nginx with `ip_hash` in development. Production runs **one** instance for the MVP (2026-10-21, `docker-compose.prod.yml`); adding the second back is a compose service plus an upstream server, with no code change.
  - Socket.IO uses `@socket.io/redis-adapter`, so any room emit, `fetchSockets()` or `disconnect()` reaches every instance.
  - `CallSessionRegistry` (1:1 calls, busy, pending invites, ring deadlines) and `GroupCallPresenceRegistry` are Redis-backed, atomic through Lua; their API is async.
  - Each ring deadline fires once cluster-wide (the sweeper on every instance claims with `ZREM`).
  - `GroupCallsService.serializeMedia` takes a `RedisLockService` lock per `callId:userId`; a wait over 5 s gives a 503.
  - LiveKit's webhook goes through nginx (`:8080`), so either instance can handle it.
  - New realtime state must go in Redis, never in process memory.
