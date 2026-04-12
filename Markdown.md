# Connectfy System Memory

## System Overview

Connectfy is a social and messaging platform built as a gateway-centered microservice system. The React client talks only to the API Gateway over HTTP. The gateway handles sessions, cookies, auth checks, and request context, then delegates domain logic to backend services over NestJS TCP.

Kafka is used for asynchronous workflows such as email delivery and user projection updates. MongoDB is the primary datastore for backend services. Redis is used in the gateway for session storage and auth-related caching. File uploads use MinIO through a dedicated uploader service with presigned URLs.

## Services Summary

### Client

Purpose: Web frontend for auth, profile, users, friendship, settings, and messenger flows.

Responsibilities:

- Render feature UIs and route guards
- Store access token client-side and trigger refresh flow
- Call API Gateway endpoints only

Interactions:

- Uses API Gateway for all backend communication

### API Gateway

Purpose: Single HTTP entrypoint for the system.

Responsibilities:

- Expose REST endpoints under `/api/v1`
- Manage sessions, cookies, device ID, and auth guard
- Proxy requests to backend services over TCP
- Forward file upload URL requests to the uploader service

Interactions:

- Calls Auth, Account, Relationship, Notification Action History, and File Uploader

### Auth Service

Purpose: Source of truth for identity, authentication, and session lifecycle.

Responsibilities:

- Signup, login, Google auth, 2FA, password reset
- Access/refresh token generation and device session tracking
- Account recovery, deletion, and deactivation
- Publish user projection events and email jobs

Interactions:

- Calls Account to create profile/settings
- Emits `projection.user.*` for Account and Relationship
- Emits `mail.send` for Notification Action History

### Account Service

Purpose: Owner of user-facing profile data and settings.

Responsibilities:

- Manage profiles, avatars, social links
- Manage general, privacy, and notification settings
- Enforce privacy-aware response shaping
- Maintain a local user projection

Interactions:

- Consumes `projection.user.*` from Auth
- Calls Relationship for friendship-aware profile visibility
- Serves API Gateway requests over TCP

### Relationship Service

Purpose: Owner of friendship and blocklist logic.

Responsibilities:

- Friend requests, accept/decline/cancel, unfriend
- Close-friend and mute toggles
- Blocklist checks
- Maintain a local user projection for search/listing

Interactions:

- Consumes `projection.user.*` from Auth
- Calls Account privacy settings before some actions
- Serves API Gateway requests over TCP

### Notification Action History Service

Purpose: Currently active as the asynchronous email delivery service.

Responsibilities:

- Consume `mail.send` Kafka events
- Send transactional emails through SMTP
- Store the base notification entity for future notification-history work

Interactions:

- Receives email jobs mainly from Auth via Kafka

### File Uploader

Purpose: Dedicated storage integration service.

Responsibilities:

- Generate presigned MinIO upload/download URLs
- Delete files in batches
- Consume file deletion events

Interactions:

- Called by API Gateway for upload flows
- Uses MinIO for object storage
- Consumes Kafka delete events

## Core Entities

### User

- `_id`
- `username`
- `email`
- `provider`
- `status`
- `phoneNumber`

### RefreshToken

- `userId`
- `deviceId`
- `refresh_token`
- `expiresAt`
- `isActive`

### Profile

- `userId`
- `firstName`
- `lastName`
- `username`
- `avatar`
- `defaultAvatar`
- `bio`
- `location`

### GeneralSettings

- `userId`
- `theme`
- `language`
- `startupPage`

### PrivacySettings

- `userId`
- visibility fields such as `email`, `bio`, `avatar`, `phoneNumber`
- `friendshipRequest`
- `readReceipts`

### NotificationSettings

- `userId`
- `notificationSoundMode`
- `notificationContentMode`
- category notification toggles

### SocialLink

- `userId`
- `name`
- `url`
- `platform`
- `rank`

### Friendship

- `userId`
- `friendId`
- `status`
- `isFavorite`
- `isMuted`

### Blocklist

- `blocker`
- `blocked`

### Notification

- `recipientId`
- `actorId`
- `type`
- `status`
- `channel`
- `resourceId`
- `expiresAt`

## System Architecture

- Browser -> API Gateway -> TCP microservices
- API Gateway validates bearer access tokens plus refresh-token cookies and caches verified auth state in Redis
- Auth publishes `projection.user.created|updated|removed`
- Account and Relationship consume those events to keep local read models in sync
- Auth publishes `mail.send`, consumed by Notification Action History for email delivery
- API Gateway requests presigned URLs from File Uploader
- Client uploads directly to MinIO using those presigned URLs
- File deletion events are consumed by File Uploader to remove stored objects

## Key Patterns & Design Decisions

- Gateway + microservices architecture with a single public HTTP boundary
- Thin gateway; business rules live in domain services
- TCP is the main synchronous inter-service protocol
- Kafka is used for asynchronous side effects and cross-service projections
- Account and Relationship use local projection/read-model patterns instead of querying Auth for every read
- CLS/request-context propagation carries logged user, language, and device context across service boundaries
- Privacy is enforced server-side in Account responses
- Accepted friendships are represented as two directional records

## Tech Stack Summary

- React 19
- TypeScript
- Vite
- React Router
- Zustand
- Redux Toolkit
- Axios
- NestJS
- NestJS TCP microservices
- Kafka
- MongoDB + Mongoose
- Redis
- JWT
- MinIO
- Express sessions
- Helmet
- Google OAuth
- i18next

## Developer Notes

- The API Gateway is the only client-facing HTTP boundary. New client features should normally flow through it.
- Auth is the source of truth for identity, tokens, and account lifecycle state.
- `projection.user.*` events are critical. Breaking their shape or delivery will desynchronize Account and Relationship.
- Notification Action History should currently be treated as an email-delivery service; broader notification/device-token modules are not active memory unless explicitly requested.
- File uploads are direct-to-MinIO via presigned URLs, not proxy uploads through the gateway.
- Privacy rules are enforced in Account service reads, not only in frontend logic.
- The client stores the access token locally, while refresh is cookie-based.
- Relationship logic is directional; do not assume a single record represents a mutual friendship.
