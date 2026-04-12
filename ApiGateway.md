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

## Technologies Used
- NestJS
- Redis + `connect-redis`
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
