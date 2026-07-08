# Connectfy Account Microservice Analysis

## Overview / Purpose
`connectfy-account` is a NestJS-based backend microservice responsible for managing user accounts, profiles, user settings (privacy, general, notification), social links, and media associations (such as avatars). It functions as a hybrid microservice, listening for both TCP requests (for synchronous intra-system communication) and Kafka events (for asynchronous data projection and side-effects).

## Folder Structure
```text
connectfy-account/
├── src/
│   ├── app-settings/
│   │   ├── http-connections/
│   │   │   ├── http-connection.module.ts
│   │   │   └── http-connection.service.ts
│   │   ├── kafka-connections/
│   │   │   ├── kafka-connection.module.ts
│   │   │   ├── kafka-connection.service.ts
│   │   │   └── loggin-kafka.server.ts
│   │   ├── tcp-connections/
│   │   │   ├── tcp-connection.module.ts
│   │   │   └── tcp-connection.service.ts
│   │   └── app-settings.module.ts
│   ├── common/
│   │   └── constants/
│   │       └── environment-variables.ts
│   ├── interceptors/
│   │   └── logged-user.interceptor.ts
│   ├── modules/
│   │   ├── media/
│   │   │   ├── dto/
│   │   │   │   └── create-media.dto.ts
│   │   │   ├── entity/
│   │   │   │   └── media.entity.ts
│   │   │   ├── interface/
│   │   │   │   └── media.interface.ts
│   │   │   ├── repo/
│   │   │   │   └── media.repo.ts
│   │   │   ├── media.controller.ts
│   │   │   ├── media.module.ts
│   │   │   └── media.service.ts
│   │   ├── profile/
│   │   │   ├── dto/
│   │   │   │   ├── nested/
│   │   │   │   │   └── avatar.dto.ts
│   │   │   │   ├── add.profile.dto.ts
│   │   │   │   ├── base.profile.dto.ts
│   │   │   │   ├── edit.profile.dto.ts
│   │   │   │   ├── find.profile.dto.ts
│   │   │   │   └── remove.profile.dto.ts
│   │   │   ├── entity/
│   │   │   │   ├── nested/
│   │   │   │   │   └── avatar.entity.ts
│   │   │   │   └── profile.entity.ts
│   │   │   ├── interface/
│   │   │   │   ├── nested/
│   │   │   │   │   └── avatar.interface.ts
│   │   │   │   └── profile.interface.ts
│   │   │   ├── repo/
│   │   │   │   └── profile.repo.ts
│   │   │   ├── profile.controller.ts
│   │   │   ├── profile.module.ts
│   │   │   └── profile.service.ts
│   │   ├── settings/
│   │   │   ├── general-settings/
│   │   │   │   ├── dto/
│   │   │   │   │   ├── nested/
│   │   │   │   │   │   └── time-zone.dto.ts
│   │   │   │   │   ├── add.general-settings.dto.ts
│   │   │   │   │   ├── base.general-settings.dto.ts
│   │   │   │   │   ├── edit.general-settings.dto.ts
│   │   │   │   │   ├── find.general-settings.dto.ts
│   │   │   │   │   └── remove.general-settings.dto.ts
│   │   │   │   ├── entity/
│   │   │   │   │   ├── nested/
│   │   │   │   │   │   └── time-zone.entity.ts
│   │   │   │   │   └── general-settings.entity.ts
│   │   │   │   ├── interface/
│   │   │   │   │   ├── nested/
│   │   │   │   │   │   └── time-zone.interface.ts
│   │   │   │   │   └── general-settings.interface.ts
│   │   │   │   ├── repo/
│   │   │   │   │   └── general-settings.repo.ts
│   │   │   │   ├── general-settings.controller.ts
│   │   │   │   ├── general-settings.module.ts
│   │   │   │   └── general-settings.service.ts
│   │   │   ├── notification-settings/
│   │   │   │   ├── dto/
│   │   │   │   │   ├── add.notification-settings.dto.ts
│   │   │   │   │   ├── base.notification-settings.dto.ts
│   │   │   │   │   ├── edit.notification-settings.dto.ts
│   │   │   │   │   ├── find.notification-settings.dto.ts
│   │   │   │   │   └── remove.notification-settings.dto.ts
│   │   │   │   ├── entity/
│   │   │   │   │   └── notification-settings.entity.ts
│   │   │   │   ├── interface/
│   │   │   │   │   └── notification-settings.interface.ts
│   │   │   │   ├── repo/
│   │   │   │   │   └── notification-settings.repo.ts
│   │   │   │   ├── notification-settings.controller.ts
│   │   │   │   ├── notification-settings.module.ts
│   │   │   │   └── notification-settings.service.ts
│   │   │   ├── privacy-setting/
│   │   │   │   ├── dto/
│   │   │   │   │   ├── add.privacy-settings.dto.ts
│   │   │   │   │   ├── base.privacy-settings.dto.ts
│   │   │   │   │   ├── edit.privacy-settings.dto.ts
│   │   │   │   │   ├── find.privacy-settings.dto.ts
│   │   │   │   │   └── remove.privacy-settings.dto.ts
│   │   │   │   ├── entity/
│   │   │   │   │   └── privacy-settings.entity.ts
│   │   │   │   ├── interface/
│   │   │   │   │   └── privacy-settings.interface.ts
│   │   │   │   ├── repo/
│   │   │   │   │   └── privacy-settings.repo.ts
│   │   │   │   ├── privacy-settings.controller.ts
│   │   │   │   ├── privacy-settings.module.ts
│   │   │   │   └── privacy-settings.service.ts
│   │   │   └── settings.module.ts
│   │   ├── social-link/
│   │   │   ├── dto/
│   │   │   │   ├── add.social-link.dto.ts
│   │   │   │   ├── base.social-link.dto.ts
│   │   │   │   ├── edit.social-link.dto.ts
│   │   │   │   ├── find.social-link.dto.ts
│   │   │   │   └── remove.social-link.dto.ts
│   │   │   ├── entity/
│   │   │   │   └── social-link.entity.ts
│   │   │   ├── interface/
│   │   │   │   └── social-link.interface.ts
│   │   │   ├── repo/
│   │   │   │   └── social-link.repo.ts
│   │   │   ├── social-link.controller.ts
│   │   │   ├── social-link.module.ts
│   │   │   └── social-link.service.ts
│   │   └── modules.module.ts
│   ├── projection/
│   │   ├── user/
│   │   │   ├── dto/
│   │   │   │   ├── nested/
│   │   │   │   │   └── phoneNumber.dto.ts
│   │   │   │   ├── add.user.dto.ts
│   │   │   │   ├── base.user.dto.ts
│   │   │   │   ├── edit.user.dto.ts
│   │   │   │   ├── find.user.dto.ts
│   │   │   │   └── remove.user.dto.ts
│   │   │   ├── entity/
│   │   │   │   ├── nested/
│   │   │   │   │   └── phoneNumber.entity.ts
│   │   │   │   └── user.entity.ts
│   │   │   ├── interface/
│   │   │   │   ├── nested/
│   │   │   │   │   └── phoneNumber.interface.ts
│   │   │   │   └── user.interface.ts
│   │   │   ├── repo/
│   │   │   │   └── user.repo.ts
│   │   │   ├── user.controller.ts
│   │   │   ├── user.module.ts
│   │   │   └── user.service.ts
│   │   └── projections.module.ts
│   ├── app.module.ts
│   ├── i18n.ts
│   └── main.ts
```

## Architecture & Request / Event Flow
- **Initialization:** `main.ts` bootstraps two NestJS microservices. One uses Kafka transport (listening for background events), and the other uses TCP transport (listening for synchronous RPC calls from an API Gateway or peer services).
- **Context Management:** Global validation pipes and exception filters (`AllExceptionsFilter`) are applied. `nestjs-cls` acts as Continuation-Local Storage to track context. `LoggedUserInterceptor` intercepts incoming TCP/Kafka messages, extracting `_loggedUser` and `_lang` from payloads and storing them securely in the CLS layer.
- **Event-Driven Projection:** The `/projection/user` module listens to Kafka topics emitted by an authentication service. When `projection.user.created/updated/removed` events arrive, it updates a local MongoDB read-model of `UserModel`.
- **Query / Interaction (TCP):** Data retrievals (e.g., getting a profile) trigger a sequence of outbound TCP RPC requests. For example, querying a profile first checks the Relationship Service (`blocklist/isExist`) to ensure no blocking is in effect, fetches the privacy settings, and uses `shouldShowField` helpers to strip hidden fields dynamically based on the viewer's friendship level. 
- **Outbound Communication:** Outbound requests are bridged via `TcpConnectionService`, which injects the current CLS context (`_loggedUser`, `_lang`) back into outbound payloads for uninterrupted context tracking.

## Entities and Models
Exhaustive schema structures (using Mongoose Types / TypeScript Interfaces). All entities have `createdAt` and `updatedAt` strings/dates.

### 1. User (Projection)
_Collection: user projections_
- `_id`: String (UUID)
- `firstName`: String
- `lastName`: String
- `fullName`: String
- `username`: String (Unique, Indexed)
- `email`: String (Unique, Indexed)
- `phoneNumber`: Object | null
  - `countryCode`: String | null
  - `number`: String | null
  - `fullPhoneNumber`: String | null
- `status`: Enum (`USER_STATUS` e.g., ACTIVE)
- `avatar`: String | null

### 2. Profile
_Collection: profiles_
- `_id`: String (UUID)
- `userId`: String (UUID, Ref to User, Unique, Indexed)
- `firstName`: String
- `lastName`: String
- `fullName`: String
- `username`: String (Unique, Indexed)
- `gender`: Enum (`GENDER`)
- `bio`: String | null
- `location`: String | null
- `avatar`: Object | null
  - `key`: String | null
  - `url`: String
  - `isCustom`: Boolean
- `defaultAvatar`: Object
  - `format`: Enum (`AvatarFormats` e.g., Adventurer)
  - `seed`: String
  - `url`: String
- `lastSeen`: Date
- `birthdayDate`: Date

### 3. PrivacySettings
_Collection: privacy_settings_
- `_id`: String (UUID)
- `userId`: String (UUID, Unique, Indexed)
- `email`: Enum (`PRIVACY_SETTINGS_CHOICE`)
- `bio`: Enum (`PRIVACY_SETTINGS_CHOICE`)
- `gender`: Enum (`PRIVACY_SETTINGS_CHOICE`)
- `location`: Enum (`PRIVACY_SETTINGS_CHOICE`)
- `socialLinks`: Enum (`PRIVACY_SETTINGS_CHOICE`)
- `lastSeen`: Enum (`PRIVACY_SETTINGS_CHOICE`)
- `avatar`: Enum (`PRIVACY_SETTINGS_CHOICE`)
- `messageRequest`: Enum (`PRIVACY_SETTINGS_CHOICE`)
- `birthdayDate`: Enum (`PRIVACY_SETTINGS_CHOICE`)
- `phoneNumber`: Enum (`PRIVACY_SETTINGS_CHOICE`)
- `friendshipRequest`: Boolean
- `readReceipts`: Boolean

### 4. NotificationSettings
_Collection: notification_settings_
- `_id`: String (UUID)
- `userId`: String (UUID, Unique, Indexed)
- `notificationSoundMode`: Enum (`NOTIFICATION_SOUND_MODE`)
- `notificationContentMode`: Enum (`NOTIFICATION_CONTENT_MODE`)
- `sendMessageSound`: Boolean
- `receiveMessageSound`: Boolean
- `privateMessageSound`: Boolean
- `groupMessageSound`: Boolean
- `systemNotificationSound`: Boolean
- `friendshipNotificationSound`: Boolean
- `showPrivateMessageNotification`: Boolean
- `showGroupMessageNotification`: Boolean
- `showFriendshipNotification`: Boolean
- `showSystemNotification`: Boolean

### 5. GeneralSettings
_Collection: general_settings_
- `_id`: String (UUID)
- `userId`: String (UUID, Unique, Indexed)
- `theme`: Enum (`THEME` e.g., LIGHT, DARK)
- `language`: Enum (`LANGUAGE`)
- `startupPage`: Enum (`STARTUP_PAGE` e.g., MESSENGER)
- `timeZone`: Object
  - `timeFormat`: Enum (`TIME_FORMAT` e.g., H24)
  - `dateFormat`: Enum (`DATE_FORMAT` e.g., DDMMYYYY)

### 6. Media
_Collection: medias_
- `_id`: String (UUID)
- `userId`: String (Indexed)
- `key`: String
- `url`: String
- `mimeType`: String
- `size`: Number
- `moduleName`: String

### 7. SocialLink
_Collection: social_links_
- `_id`: String (UUID)
- `userId`: String (Indexed)
- `name`: String
- `url`: String
- `platform`: Enum (`SOCIAL_LINK_PLATFORM`)
- `rank`: Number

## Exposed APIs

### Kafka Consumed Event Patterns
- `projection.user.created`
- `projection.user.updated`
- `projection.user.removed`
- `account.privacy-settings.remove`
- `account.notification-settings.remove`
- `account.general-settings.remove`

### Kafka Emitted Event Patterns
- `projection.user.updated` - Dispatched when avatars or names change in the profile module.
- `profile.file.delete` - Dispatched when a custom avatar file needs to be purged from object storage.

### TCP Exposed Message Patterns (RPC)
- **Profile**
  - `profile/create`
  - `profile/findOne`
  - `profile/findOneByUserId`
  - `profile/search`
  - `profile/update`
  - `profile/updateAvatar`
  - `profile/updateDefaultAvatar`
- **Privacy Settings**
  - `privacy-settings/get`
  - `privacy-settings/findOne`
  - `privacy-settings/create`
  - `privacy-settings/update`
- **Notification Settings**
  - `notification-settings/get`
  - `notification-settings/findOne`
  - `notification-settings/create`
  - `notification-settings/update`
- **General Settings**
  - `general-settings/get`
  - `general-settings/findOne`
  - `general-settings/create`
  - `general-settings/update`
  - `general-settings/reset`
- **Media**
  - `media.create`
  - `media.findOneByUserId`
- **Social Links**
  - `socialLinks/findMany`
  - `socialLinks/create`
  - `socialLinks/update`
  - `socialLinks/updateRank`
  - `socialLinks/remove`
  - `socialLinks/removeMany`

## Core Features
1. **Dynamic Avatar Management**: Fallback default avatars are dynamically seeded procedurally using Dicebear. Custom avatar paths communicate via HTTP with a file-uploader service for pre-flight validation (sizing and mimetype) before being committed to Mongo.
2. **Access Control Filtering by Privacy Settings**: When requesting a user's profile, the system fetches privacy settings and relationship blocks simultaneously, then automatically nullifies profile fields depending on if the requester is an accepted friend, a stranger, or viewing their own profile.
3. **Advanced Profile Search**: Text search dynamically fetches blocked lists from external services and filters out users accordingly. Output is enriched conditionally with user avatars dependent on friendship context.
4. **Ranking System for Social Links**: Provides granular capability for re-ordering (bulk rank updating) up to 5 social media links per user.
5. **Configurable Settings Management**: Complete tracking of settings subsets (General, Privacy, Notification). Supports batch resetting to default settings.

## Technologies Used
- **Language / Framework**: TypeScript, NestJS
- **Microservice Integration**: NestJS Microservices, `@nestjs/microservices` (TCP and KafkaJS strategies)
- **Database Architecture**: MongoDB using Mongoose, Repository Pattern implementations.
- **Context Injection**: `nestjs-cls` (Continuation-Local Storage), custom Interceptors.
- **Internal HTTP Comm**: Axios (via `@nestjs/axios`).
- **Validation**: `class-validator`, `class-transformer` combined with proprietary `connectfy-shared` wrappers.
- **I18n**: i18next (Integrated directly in validations).

## Implementation Notes & Gotchas
1. **CLS Payload Proxying**: The application strips `_loggedUser` and `_lang` from incoming RPC calls via interceptor into CLS context. When using `TcpConnectionService.sendTcpWithContext()`, these fields are auto-attached to outbound data. *Gotcha:* Never use standard NestJS ClientProxy `send` for external service calls within business logic, always use `TcpConnectionService` so authorization tracking is not lost mid-flight.
2. **File Deletion Rollback on Avatar Save**: Due to distributed storage logic, if avatar Mongo document update fails natively, a `catch` block fires `profile.file.delete` into Kafka to wipe the orphaned file cleanly off the CDN/Minio.
3. **Mongoose Sparse Indexes**: Avatar indexes in the profile schema utilize `{ sparse: true }` heavily, prioritizing memory economy for null fields. 
4. **Presence Heartbeat via Mongoose Pre-Hook**: Every `findOneAndUpdate` query to `ProfileSchema` invokes a hook overriding `{ lastSeen: new Date() }`. Meaning *any* update acts as a presence pulse.

## Known In-Progress / Partially Implemented Areas
- **Profile Text Search Index**: `ProfileSchema.index({ firstName: 'text', lastName: 'text', username: 'text' })` is currently commented out in `profile.entity.ts`. As a fallback, `ProfileService.findMany` explicitly iterates a `$or: [{ username: searchRegex }, { fullName: searchRegex }]` query strategy which may underperform at scale.
- `showNotification` and `showNotificationContent` fields appear in `INotificationSettings` interface structure but do not explicitly have `@Prop` decorators backing them up inside the actual `NotificationSettingsModel`.
