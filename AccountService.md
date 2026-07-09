# Account Service

## Overview
`connectfy-account` manages user-facing profile data and settings. It stores profile metadata, privacy settings, notification settings, general preferences, and social links. It also keeps a local projection of core auth user data so it can query profile information without synchronous reads back to auth for every request.

It interacts with:
- `connectfy-auth` via Kafka projections (`projection.user.*`)
- `connectfy-relationship` via TCP for friendship-aware profile/privacy decisions
- `connectfy-api-gateway` as its main HTTP-facing caller

## Folder Structure
```text
connectfy-account/
├── src/
│   ├── app-settings/
│   │   ├── kafka-connections/
│   │   └── tcp-connections/
│   ├── interceptors/
│   ├── modules/
│   │   ├── profile/
│   │   ├── settings/
│   │   │   ├── general-settings/
│   │   │   ├── privacy-setting/
│   │   │   └── notification-settings/
│   │   └── social-link/
│   └── projection/
│       └── user/
```

## Architecture & Flow
The service runs both Kafka and TCP transports.

Flow:
1. Auth publishes user projection events.
2. `projection/user` consumes them and updates the local `User` read model.
3. API Gateway sends TCP messages for profile/settings/social-link actions.
4. Profile reads may call `relationship` to resolve friendship status and count, then filter fields using privacy settings.
5. Profile updates publish `projection.user.updated` when username or avatar changes so other services stay synchronized.

## Entities / Models
### Profile
- `_id: string`
- `userId: string`
- `firstName: string`
- `lastName: string`
- `username: string`
- `gender: GENDER`
- `bio: string | null`
- `location: string | null`
- `avatar: AvatarModel | null`
- `defaultAvatar: DefaultAvatarModel`
- `lastSeen: Date`
- `birthdayDate: Date`

### Avatar
- `key: string | null`
- `url: string`
- `isCustom: boolean`

### DefaultAvatar
- `format: AvatarFormats`
- `seed: string`
- `url: string`

### SocialLink
- `_id: string`
- `userId: string`
- `name: string`
- `url: string`
- `platform: SOCIAL_LINK_PLATFORM`
- `rank: number`

### GeneralSettings
- `_id: string`
- `userId: string`
- `theme: THEME`
- `language: LANGUAGE`
- `startupPage: STARTUP_PAGE`
- `timeZone: TimeZone`

### TimeZone
- `timeFormat: TIME_FORMAT`
- `dateFormat: DATE_FORMAT`

### PrivacySettings
- `_id: string`
- `userId: string`
- `email: PRIVACY_SETTINGS_CHOICE`
- `bio: PRIVACY_SETTINGS_CHOICE`
- `gender: PRIVACY_SETTINGS_CHOICE`
- `location: PRIVACY_SETTINGS_CHOICE`
- `socialLinks: PRIVACY_SETTINGS_CHOICE`
- `lastSeen: PRIVACY_SETTINGS_CHOICE`
- `avatar: PRIVACY_SETTINGS_CHOICE`
- `messageRequest: PRIVACY_SETTINGS_CHOICE`
- `birthdayDate: PRIVACY_SETTINGS_CHOICE`
- `phoneNumber: PRIVACY_SETTINGS_CHOICE`
- `friendshipRequest: boolean`
- `readReceipts: boolean`

### NotificationSettings
- `_id: string`
- `userId: string`
- `notificationSoundMode: NOTIFICATION_SOUND_MODE`
- `notificationContentMode: NOTIFICATION_CONTENT_MODE`
- `sendMessageSound: boolean`
- `receiveMessageSound: boolean`
- `privateMessageSound: boolean`
- `groupMessageSound: boolean`
- `systemNotificationSound: boolean`
- `friendshipNotificationSound: boolean`
- `showPrivateMessageNotification: boolean`
- `showGroupMessageNotification: boolean`
- `showFriendshipNotification: boolean`
- `showSystemNotification: boolean`

### User Projection
- `_id: string`
- `firstName: string`
- `lastName: string`
- `username: string`
- `email: string`
- `phoneNumber: PhoneNumberModel | null`
- `status: USER_STATUS`
- `avatar: string | null`

### PhoneNumber Projection
- `countryCode: string`
- `number: string`
- `fullPhoneNumber: string`

## Core Features
- Create and update user profiles
- Generate and switch default/custom avatars
- Search profiles by username or name
- Enforce privacy-aware field exposure
- Manage general, privacy, and notification settings
- Manage up to five ordered social links
- Maintain a local projection of auth user data

## APIs
TCP message patterns:
- `profile/create`
- `profile/findOne`
- `profile/findOneByUserId`
- `profile/search`
- `profile/update`
- `profile/updateAvatar`
- `profile/updateDefaultAvatar`
- `general-settings/get`
- `general-settings/findOne`
- `general-settings/create`
- `general-settings/update`
- `general-settings/reset`
- `privacy-settings/get`
- `privacy-settings/findOne`
- `privacy-settings/create`
- `privacy-settings/update`
- `notification-settings/get`
- `notification-settings/findOne`
- `notification-settings/create`
- `notification-settings/update`
- `socialLinks/findMany`
- `socialLinks/create`
- `socialLinks/update`
- `socialLinks/updateRank`
- `socialLinks/remove`
- `socialLinks/removeMany`

Kafka consumers:
- `projection.user.created`
- `projection.user.updated`
- `projection.user.removed`

## Technologies Used
- NestJS microservices
- MongoDB + Mongoose
- Kafka
- TCP transport
- `nestjs-cls`
- DiceBear avatar generation

## Notes
- Profile reads are privacy-aware and relationship-aware; this is one of the most important design points.
- The local `projection/user` model is a read model, not the source of truth for identity.
- Username and avatar changes are re-published so other services can update their projections.
