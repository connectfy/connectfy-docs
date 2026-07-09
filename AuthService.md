# Auth Service

## Overview
`connectfy-auth` is the identity and account lifecycle service. It owns authentication, password and provider logic, token issuance, refresh-token session tracking, user status transitions, and account recovery flows.

It communicates with:
- `connectfy-account` over TCP to create profile/settings records and read user-facing settings.
- `connectfy-notification-action-history` over Kafka for outbound email delivery.
- projection consumers in other services over Kafka using `projection.user.*` events.

## Folder Structure
```text
connectfy-auth/
├── src/
│   ├── app-settings/
│   │   ├── kafka-connections/
│   │   └── tcp-connections/
│   ├── external-modules/
│   │   ├── account/
│   │   └── notifications/
│   ├── internal-modules/
│   │   ├── bcrypt/
│   │   └── request-helper/
│   ├── interceptors/
│   └── modules/
│       ├── auth/
│       ├── tokens/
│       └── users/
├── Dockerfile.dev
└── Dockerfile.prod
```

## Architecture & Flow
The service starts as a TCP microservice. Every message is validated by a global `ValidationPipe`, then handled by module services backed by MongoDB repositories.

Main flows:
1. Signup validates uniqueness, sends a verification email, then creates the user after code verification.
2. Login supports password and Google providers, optional 2FA, and device-aware refresh-token persistence.
3. Account mutations such as username, email, password, phone number, deletion, deactivation, and restoration are handled here.
4. Auth emits Kafka projection events so `account` and `relationship` can maintain local user read models.

## Entities / Models
### User
- `_id: string` - user UUID
- `username: string` - unique handle
- `email: string` - unique email
- `role: ROLE` - authorization role
- `provider: PROVIDER` - password or Google
- `password: string` - hashed password
- `isTwoFactorEnabled: boolean` - 2FA toggle
- `phoneNumber: PhoneNumberModel | null` - optional phone data
- `status: USER_STATUS` - active/inactive/deleted-related state
- `timeZone: string | null` - inferred timezone
- `location: string | null` - inferred location

### PhoneNumber
- `countryCode: string`
- `number: string`
- `fullPhoneNumber: string`

### Token
- `_id: string`
- `userId: string`
- `token: string`
- `type: TOKEN_TYPE`
- `expiresAt: Date`
- `isUsed: boolean`

### RefreshToken
- `_id: string`
- `userId: string`
- `refresh_token: string`
- `deviceId: string | null`
- `userAgent: string | null`
- `deviceName: string | null`
- `platform: DEVICE_TYPE`
- `browser: string | null`
- `os: string | null`
- `ipAddress: string | null`
- `country: string | null`
- `countryCode: string | null`
- `city: string | null`
- `longitude: number | null`
- `latitude: number | null`
- `region: string | null`
- `timezone: string | null`
- `lastUsedAt: Date`
- `expiresAt: Date`
- `isActive: boolean`
- `fingerprint: string | null`
- `metadata: Record<string, any> | null`

### BannedUser
- `_id: string`
- `userId: string`
- `bannedToDate: Date | null`

### DeactivatedUser
- `_id: string`
- `userId: string`

### DeletedUser
- `_id: string`
- `userId: string`
- `deletedAt: Date`
- `reason: DELETE_REASON`
- `reasonCode: DELETE_REASON_CODE | null`
- `reasonDescription: string | null`

## Core Features
- Email/password signup and login
- Google login and signup
- Two-factor email verification
- Access and refresh token generation
- Device/session tracking for refresh tokens
- Forgot/reset password
- Change username, email, password, phone number
- Delete, deactivate, and restore account
- Kafka projection publishing for downstream read models

## APIs
This service exposes TCP message patterns, not REST endpoints.

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

## Technologies Used
- NestJS microservices
- MongoDB + Mongoose
- JWT
- Kafka
- `bcrypt`
- Google OAuth (`google-auth-library`)
- `nestjs-cls`
- `geoip-lite`, `ua-parser-js`

## Notes
- This service is the source of truth for identity and session state.
- `projection.user.created|updated|removed` events are critical; account and relationship depend on them.
- Email delivery is asynchronous through Kafka topic `mail.send`.
