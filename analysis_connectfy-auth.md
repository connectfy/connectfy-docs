# connectfy-auth Microservice Analysis

## Overview / Purpose
The `connectfy-auth` microservice manages authentication, authorization, user registration, device session tracking (JWT access/refresh tokens), Google OAuth2 integration, two-factor authentication (2FA), and user lifecycle events (banning, deactivation, deletion, restoration). It serves as the primary entry point for user identity validation, communicating synchronously with other microservices (Account, Messenger, Relationship) over TCP and emitting asynchronous domain events over Kafka.

## Full Folder Structure
```
connectfy-auth/
└── src/
    ├── main.ts
    ├── i18n.ts
    ├── app.module.ts
    ├── common/
    │   ├── constants/
    │   │   └── environment-variables.ts
    │   └── functions/
    │       └── function.ts
    ├── interceptors/
    │   └── logged-user.interceptor.ts
    ├── app-settings/
    │   ├── app-settings.module.ts
    │   ├── kafka-connections/
    │   │   ├── loggin-kafka.server.ts
    │   │   ├── kafka-connection.service.ts
    │   │   └── kafka-connection.module.ts
    │   └── tcp-connections/
    │       ├── tcp-connection.service.ts
    │       └── tcp-connection.module.ts
    ├── internal-modules/
    │   ├── internal-modules.module.ts
    │   ├── bcrypt/
    │   │   ├── bcrypt.service.ts
    │   │   └── bcrypt.module.ts
    │   └── request-helper/
    │       ├── request-helper.module.ts
    │       ├── request-helper.service.ts
    │       ├── dto/
    │       │   └── request-data.dto.ts
    │       └── interfaces/
    │           └── request.interface.ts
    ├── external-modules/
    │   ├── external-modules.module.ts
    │   ├── account/
    │   │   ├── account.service.ts
    │   │   └── account.module.ts
    │   └── notifications/
    │       ├── notifications.module.ts
    │       ├── notifications.service.ts
    │       └── interfaces/
    │           └── notifications.interface.ts
    └── modules/
        ├── modules.module.ts
        ├── auth/
        │   ├── auth.controller.ts
        │   ├── auth.module.ts
        │   ├── auth.service.ts
        │   ├── dto/
        │   │   ├── authenticate-user.dto.ts
        │   │   ├── forgot-password.dto.ts
        │   │   ├── login.dto.ts
        │   │   ├── logout.dto.ts
        │   │   ├── reset-password.dto.ts
        │   │   ├── restore-account.dto.ts
        │   │   ├── signup.dto.ts
        │   │   ├── validate-token.dto.ts
        │   │   ├── verify.dto.ts
        │   │   └── nested/
        │   │       └── requestData.ts
        │   └── interface/
        │       └── auth.interface.ts
        ├── tokens/
        │   ├── tokens.module.ts
        │   ├── refresh-token/
        │   │   ├── refresh-token.module.ts
        │   │   ├── refresh-token.service.ts
        │   │   ├── entity/
        │   │   │   └── refresh-token.entity.ts
        │   │   ├── interface/
        │   │   │   └── refresh-token.interface.ts
        │   │   └── repo/
        │   │       └── refresh-token.repo.ts
        │   └── token/
        │       ├── token.controller.ts
        │       ├── token.module.ts
        │       ├── token.service.ts
        │       ├── entity/
        │       │   └── token.entity.ts
        │       ├── interface/
        │       │   └── token.interface.ts
        │       ├── repo/
        │       │   └── token.repo.ts
        │       └── dto/
        │           ├── add.token.dto.ts
        │           ├── base.token.dto.ts
        │           ├── edit.token.dto.ts
        │           ├── find.token.dto.ts
        │           └── remove.token.dto.ts
        └── users/
            ├── users.module.ts
            ├── banned-user/
            │   ├── banned-user.controller.ts
            │   ├── banned-user.module.ts
            │   ├── banned-user.service.ts
            │   ├── entity/
            │   │   └── banned-user.entity.ts
            │   ├── interface/
            │   │   └── banned-user.interface.ts
            │   ├── repo/
            │   │   └── banned-user.repo.ts
            │   └── dto/
            │       ├── add.banned-user.dto.ts
            │       ├── base.banned-user.dto.ts
            │       ├── find.banned-user.dto.ts
            │       └── remove.banned-user.dto.ts
            ├── deactivated-users/
            │   ├── deactivated-users.controller.ts
            │   ├── deactivated-users.module.ts
            │   ├── deactivated-users.service.ts
            │   ├── entity/
            │   │   └── deactivated-user.entity.ts
            │   ├── interface/
            │   │   └── deactivated-user.intreface.ts
            │   ├── repo/
            │   │   └── deactivated-user.repo.ts
            │   └── dto/
            │       ├── add.deactivated-user.dto.ts
            │       ├── base.deactivated-user.dto.ts
            │       ├── find.deactivated-user.dto.ts
            │       └── remove.deactivated-user.dto.ts
            ├── deleted-user/
            │   ├── deleted-user.controller.ts
            │   ├── deleted-user.module.ts
            │   ├── deleted-user.service.ts
            │   ├── entity/
            │   │   └── deleted-user.entity.ts
            │   ├── interface/
            │   │   └── deleted-user.interface.ts
            │   ├── repo/
            │   │   └── deleted-user.repo.ts
            │   └── dto/
            │       ├── add.deleted-user.dto.ts
            │       ├── base.deleted-user.dto.ts
            │       ├── find.deleted-user.dto.ts
            │       └── remove.deleted-user.dto.ts
            └── user/
                ├── user.controller.ts
                ├── user.module.ts
                ├── user.service.ts
                ├── entity/
                │   ├── user.entity.ts
                │   └── nested/
                │       └── phoneNumber.entity.ts
                ├── interface/
                │   ├── user.interface.ts
                │   └── nested/
                │       └── phoneNumber.interface.ts
                ├── repo/
                │   └── user.repo.ts
                └── dto/
                    ├── add.user.dto.ts
                    ├── base.user.dto.ts
                    ├── change-email.dto.ts
                    ├── change-password.dto.ts
                    ├── change-phone-number.dto.ts
                    ├── change-username.dto.ts
                    ├── check-unique.dto.ts
                    ├── deactivate-account.dto.ts
                    ├── delete-account.dto.ts
                    ├── edit.user.dto.ts
                    ├── find.user.dto.ts
                    ├── remove.user.dto.ts
                    ├── two-factor.dto.ts
                    └── nested/
                        └── phoneNumber.dto.ts
```

## Architecture & Request/Event Flow
1. **Transport Layer**: The application uses `NestFactory.createMicroservice` to instantiate dual transport servers (TCP and Kafka). The `AuthController` and `UserController` subscribe strictly to TCP `@MessagePattern` events.
2. **Context Passing (CLS)**: As requests enter via TCP, the `LoggedUserInterceptor` captures `_loggedUser` and `_lang` fields from the TCP payload, mounting them globally using `nestjs-cls` (Continuation Local Storage). This eliminates the need to pass `user` or `language` objects manually down the service chains.
3. **Validation**: Requests pass through a global `ValidationPipe` leveraging NestJS `class-validator` (via custom `connectfy-shared` field decorators).
4. **Services & Repositories**: The controllers hand logic off to `AuthService` or `UserService`, which use thin Repository wrappers (`BaseRepository`) around Mongoose Models for DB operations. Internal services like `BcryptService` and `RequestHelperService` are used for security and parsing logic (User-Agent, IP, GeoLocation).
5. **Cross-Service Side-effects**:
    - During signup and profile generation, `AccountService` acts over TCP (`tcpConnectionService`) to orchestrate dependent resources synchronously in the `connectfy-account` microservice.
    - Updates and life-cycle events are emitted asynchronously via the `KafkaConnectionService` onto `projection.user.*` topics to eventually consistent systems. Notifications are pushed to `mail.send`.

## Exhaustive Entity/Model Definitions

### `UserModel` (Collection: `users`)
- `_id`: String (uuid, immutable)
- `username`: String (lowercase, min: 3, max: 30)
- `email`: String (lowercase, max: 254)
- `role`: ROLE enum (default: USER)
- `provider`: PROVIDER enum (default: PASSWORD)
- `password`: String (max: 100, select: false)
- `isTwoFactorEnabled`: Boolean (default: false)
- `phoneNumber`: Object | null
  - `countryCode`: String | null
  - `number`: String | null
  - `fullPhoneNumber`: String | null
- `status`: USER_STATUS enum (default: ACTIVE)
- `timeZone`: String | null (max: 100)
- `location`: String | null (max: 100)
- `createdAt`: Date
- `updatedAt`: Date

### `RefreshTokenModel` (Collection: `refresh_tokens`)
- `_id`: String (uuid, immutable)
- `userId`: String (uuid reference to User)
- `refresh_token`: String (unique, select: false)
- `deviceId`: String | null
- `userAgent`: String | null (max: 500)
- `deviceName`: String | null (max: 100)
- `platform`: DEVICE_TYPE enum (default: UNKNOWN)
- `browser`: String | null (max: 50)
- `os`: String | null (max: 50)
- `ipAddress`: String | null
- `country`: String | null (max: 100)
- `countryCode`: String | null (max: 3)
- `city`: String | null (max: 100)
- `longitude`: Number | null (min: -180, max: 180)
- `latitude`: Number | null (min: -90, max: 90)
- `region`: String | null (max: 100)
- `timezone`: String | null (max: 50)
- `lastUsedAt`: Date
- `expiresAt`: Date
- `isActive`: Boolean (default: true)
- `fingerprint`: String | null (select: false)
- `metadata`: Record<string, any> | null
- `createdAt`: Date
- `updatedAt`: Date

### `TokenModel` (Collection: `tokens`)
*Used for short-lived OTPs, email verifications, and account lifecycle actions.*
- `_id`: String (uuid, immutable)
- `userId`: String (uuid reference to User)
- `token`: String (unique, hashed via SHA256 before storage)
- `type`: TOKEN_TYPE enum (default: PASSWORD_RESET)
- `expiresAt`: Date
- `isUsed`: Boolean (default: false)
- `createdAt`: Date
- `updatedAt`: Date

### `BannedUserModel` (Collection: `banned_users`)
- `_id`: String (uuid, immutable)
- `userId`: String (uuid reference to User, unique)
- `bannedToDate`: Date | null
- `createdAt`: Date
- `updatedAt`: Date

### `DeletedUserModel` (Collection: `deleted_users`)
- `_id`: String (uuid, immutable)
- `userId`: String (uuid reference to User, unique)
- `deletedAt`: Date (default: Date.now)
- `reason`: DELETE_REASON enum
- `reasonCode`: DELETE_REASON_CODE enum | null
- `reasonDescription`: String | null (max: 200)
- `createdAt`: Date
- `updatedAt`: Date

### `DeactivatedUserModel` (Collection: `deactivated_users`)
- `_id`: String (uuid, immutable)
- `userId`: String (uuid reference to User, unique)
- `createdAt`: Date
- `updatedAt`: Date

## Exposed APIs

### TCP Endpoints (Consumed by API Gateway)
- `auth/signup`
- `auth/verify-signup`
- `auth/verify-signup/resend`
- `auth/login`
- `auth/verify-login`
- `auth/google/login`
- `auth/google/signup`
- `auth/forgot-password`
- `auth/reset-password`
- `auth/is-valid-token`
- `auth/logout`
- `auth/refresh-token/verify-token`
- `auth/refreshToken`
- `auth/authenticate-user`
- `auth/restore-account`
- `user/change-username`
- `user/change-email`
- `user/change-email/verify`
- `user/change-password`
- `user/change-phone-number`
- `user/check-unique`
- `user/two-factor`
- `user/delete-account`
- `user/deactivate-account`

### Kafka Topics Consumed
- None explicitly registered using `@MessagePattern(..., Transport.KAFKA)` or `@EventPattern` in the provided code dump. 

### Kafka Topics Produced
- `mail.send` (Triggered via `NotificationsService`)
- `projection.user.created` (Triggered during signup verification and Google signup)
- `projection.user.updated` (Triggered upon changes in username, email, phone number, deactivation, deletion)

### Outbound TCP Calls (Client Proxy)
The application makes outbound requests to external microservices via `@nestjs/microservices` `ClientProxy`:
- **Account Service:** 
  - `profile/create`
  - `profile/findOne`
  - `privacy-settings/create`
  - `privacy-settings/findOne`
  - `general-settings/create`
  - `general-settings/findOne`
  - `notification-settings/create`
  - `notification-settings/findOne`
- **Messenger Service:** *(Instantiated, but no endpoints explicitly called in this dump)*
- **Relationship Service:** *(Instantiated, but no endpoints explicitly called in this dump)*
- **Notification Action History Service:** *(Instantiated, but no endpoints explicitly called in this dump)*

## Core Features
1. **Multi-Factor / OAuth Integration**: Handles standard password authentication + Google OAuth + Two-Factor Authentication (OTP-based emails).
2. **Device Identity & Session Handling**: Utilizes extensive request parsing (via `ua-parser-js`, `geoip-lite`) to map user agents and IP addresses to precise physical/browser contexts in `refresh_tokens`. Provides strong JWT Token and Refresh Token rotation. 
3. **Advanced Security on Verification Links**: Verification tokens (e.g., password reset, delete account, restore account) are generated as secure strings, but their corresponding `TokenModel` entry stores them *only* as SHA256 hashes (`this.tokenService.hashToken(token)`).
4. **Graceful User Deletion and Restoration**: Account deletion moves users into an inactive/soft-deleted phase, issuing a specialized time-bound `TOKEN_TYPE.RESTORE_ACCOUNT`. Deactivation operates similarly but is resolved upon normal login.
5. **Localization**: Translates backend errors and dispatches emails with native localized templates utilizing `i18n` strings based on `CLS_KEYS.LANG`.

## Technologies Used
- **Framework**: NestJS, NestJS Microservices
- **Communication Layer**: TCP (internal synchronous), Kafka (internal asynchronous publish)
- **Database**: MongoDB via Mongoose, `mongoose-lean-virtuals`
- **Security**: JWT (`@nestjs/jwt`), Bcrypt, Google Auth Library, UUID 
- **Parsing/Utilities**: `ua-parser-js` (User Agent parsing), `geoip-lite` (Geo-location), `nestjs-cls` (Continuation Local Storage context sharing), `i18next`, `i18n-iso-countries`
- **Shared DTO/Models**: Custom local `connectfy-shared` package.

## Implementation Notes / Gotchas
- **Continuation Local Storage Pattern (CLS)**: The `LoggedUserInterceptor` parses `_loggedUser` and `_lang` from incoming RPC requests, bypassing standard NestJS decorators (like `@Req()`). This makes dependency injection much cleaner but obfuscates incoming dependencies at the controller level; developers must ensure clients format requests specifically to accommodate `_loggedUser` fields alongside real payloads.
- **`geoip-lite` Caveats**: The internal `RequestHelperService` mentions explicitly that `geoip-lite` does not provide city information accurately, so `city` is hardcoded to `null` on geolocation lookup.
- **Hashed Security Tokens**: When finding tokens for password resets or email verifications, the raw token from the request is immediately hashed (`crypto.createHash('sha256')`) to query the database.
- **Lack of Distributed Transaction Logic**: Creating users involves persisting data in the Auth MongoDB cluster (`this.userService.create`) and simultaneously making `tcpConnectionService.account` calls (`profile/create`, `general-settings/create`, etc.). There is no explicit distributed transaction roll-back logic handling instances where TCP fails *after* local Mongo persistence, though Kafka projections theoretically help achieve eventual consistency. 
- **Fat Services & Skinny Controllers**: The entirety of business logic and complex DTO composition resides entirely within `AuthService` and `UserService`.

## Known In-Progress / Partially Implemented Areas
- **Unused Controllers**: `TokenController`, `DeletedUserController`, `BannedUserController`, and `DeactivatedUsersController` are registered but have empty class bodies with zero endpoints.
- **Metadata and Fingerprints**: `RefreshTokenModel` contains `metadata` and `fingerprint` fields that do not appear to be utilized or populated within the `RefreshTokenService` generation cycle. 
- **Unused Outbound Connections**: The `TcpConnectionService` holds client proxies for Messenger, Relationship, and Notification Action History microservices, but does not use them inside the bounds of the provided code dump.
