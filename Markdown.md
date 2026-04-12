# Connectfy - Technical Specification & Architecture Blueprint

## 1. Project Overview

**Connectfy** is a high-performance, scalable communication platform built on a microservices architecture. It enables users to manage professional profiles, establish friendships, and will eventually support real-time messaging, voice/video calls, and community-driven channels (Discord-style).

## 2. Technology Stack

- **Frontend:** React, Vite, TypeScript, RTK Query (Redux Toolkit).
- **Backend:** NestJS Microservices.
- **Communication:**
  - **External:** HTTP, WebSockets (via API Gateway).
  - **Internal:** Kafka (Event-driven), TCP (Request-Response).
- **Storage & Infrastructure:**
  - **Database:** MongoDB (Mongoose ODM).
  - **Caching:** Redis.
  - **File Storage:** Minio (S3 Compatible).
  - **Containerization:** Docker.
- **Shared Resources:** `connectfy-shared` `connectfy-i18n` (Internal NPM package for interfaces, enums, and constants).

---

## 3. Microservices Architecture

### 3.1. API Gateway

The entry point for all client requests. Handles routing, authentication guarding, and acts as a bridge for WebSocket connections.

### 3.2. Auth Service

Responsible for identity management, security, and session lifecycle.

- **Key Entities:**
  - `User`: Core identity data (Role, Provider, 2FA status).
  - `RefreshToken`: Session tracking with rich metadata (Geo-location, Device info, OS).
  - `Token`: Temporary tokens for security actions (Email/Password changes).
- **Security:** Maintains dedicated collections for `Deleted`, `Deactivated`, and `Banned` users for audit trails.

### 3.2.1 User Entity

```typescript
export interface IUser {
  _id: string;
  username: string;
  email: string;
  role: ROLE;
  provider: PROVIDER;
  password: string;
  phoneNumber: IPhoneNumber | null;
  status: USER_STATUS;
  isTwoFactorEnabled: boolean;
  timeZone: string | null;
  location: string | null;
  createdAt: Date;
  updatedAt: Date;
}
```

### 3.2.2 Deleted User Entity

```typescript
export interface IDeletedUser {
  _id: string;
  userId: string;
  deletedAt: Date;
  reason: DELETE_REASON;
  reasonCode: DELETE_REASON_CODE | null;
  reasonDescription: string | null;
  createdAt: Date;
  updatedAt: Date;
}
```

### 3.2.3 Deactivated Users Entity

```typescript
export interface IDeactivatedUser {
  _id: string;
  userId: string;
  createdAt: Date;
  updatedAt: Date;
}
```

### 3.2.4 Banned Users Entity

```typescript
export interface IBannedUser {
  _id: string;
  userId: string;
  bannedToDate: Date | null;
  createdAt: Date;
  updatedAt: Date;
}
```

### 3.2.5 Refresh Token Entity

```typescript
export interface IRefreshToken {
  _id: string;
  userId: string;
  refresh_token: string;
  deviceId: string | null;
  userAgent: string | null;
  deviceName: string | null;
  platform: DEVICE_TYPE;
  browser: string | null;
  os: string | null;
  ipAddress: string | null;
  country: string | null;
  countryCode: string | null;
  city: string | null;
  longitude: number | null;
  latitude: number | null;
  region: string | null;
  timezone: string | null;
  lastUsedAt: Date;
  expiresAt: Date;
  isActive: boolean;
  metadata: Record<string, any> | null;
  createdAt: Date;
  updatedAt: Date;
}
```

### 3.2.6 Token Entity

```typescript
export interface IToken {
  _id: string;
  userId: string;
  token: string;
  type: TOKEN_TYPE;
  expiresAt: Date;
  isUsed: boolean;
  createdAt: Date;
  updatedAt: Date;
}
```

### 3.3. Account Service

Manages user-centric data, preferences, and social presence.

- **Key Entities:**
  - `Profile`: Personal details, bio, and avatar logic (Custom vs. Default).
  - `GeneralSettings`: UI/UX preferences (Theme, Language, Startup page).
  - `PrivacySettings`: Granular visibility controls (Everyone/Friends/Nobody).
  - `NotificationSettings`: Detailed sound and content delivery configurations.
  - `SocialLinks`: User-curated links for profile display.

### 3.3.1 Profile Entity

```typescript
export interface IProfile {
  _id: string;
  userId: string;
  firstName: string;
  lastName: string;
  username: string;
  gender: GENDER;
  bio: string | null;
  location: string | null;
  avatar: IAvatar | null;
  defaultAvatar: IDefaultAvatar;
  lastSeen: Date;
  birthdayDate: Date;
  createdAt: Date;
  updatedAt: Date;
}

export interface IAvatar {
  key: string | null;
  url: string;
  isCustom: boolean;
}

export interface IDefaultAvatar {
  format: AvatarFormats;
  seed: string;
  url: string;
}
```

### 3.3.2 General Settings Entity

```typescript
export interface IGeneralSettings {
  _id: string;
  userId: string;
  theme: THEME;
  language: LANGUAGE;
  startupPage: STARTUP_PAGE;
  timeZone: ITimeZone;
  createdAt: Date;
  updatedAt: Date;
}
```

### 3.3.3 Privacy Settings Entity

```typescript
export interface IPrivacySettings {
  _id: string;
  userId: string;
  email: PRIVACY_SETTINGS_CHOICE;
  bio: PRIVACY_SETTINGS_CHOICE;
  gender: PRIVACY_SETTINGS_CHOICE;
  location: PRIVACY_SETTINGS_CHOICE;
  socialLinks: PRIVACY_SETTINGS_CHOICE;
  lastSeen: PRIVACY_SETTINGS_CHOICE;
  avatar: PRIVACY_SETTINGS_CHOICE;
  messageRequest: PRIVACY_SETTINGS_CHOICE;
  birthdayDate: PRIVACY_SETTINGS_CHOICE;
  phoneNumber: PRIVACY_SETTINGS_CHOICE;
  friendshipRequest: boolean;
  readReceipts: boolean;
  createdAt: Date;
  updatedAt: Date;
}
```

### 3.3.4 Notification Settings Entity

```typescript
export interface INotificationSettings {
  _id: string;
  userId: string;
  notificationSoundMode: NOTIFICATION_SOUND_MODE;
  notificationContentMode: NOTIFICATION_CONTENT_MODE;
  sendMessageSound: boolean;
  receiveMessageSound: boolean;
  notificationSound: boolean;
  privateMessageSound: boolean;
  groupMessageSound: boolean;
  systemNotificationSound: boolean;
  friendshipNotificationSound: boolean;
  showPrivateMessageNotification: boolean;
  showGroupMessageNotification: boolean;
  showFriendshipNotification: boolean;
  showSystemNotification: boolean;
  createdAt: Date;
  updatedAt: Date;
}
```

### 3.3.5 Social Links Entity

```typescript
export interface ISocialLink {
  _id: string;
  userId: string;
  name: string;
  url: string;
  rank: number;
  platform: SOCIAL_LINK_PLATFORM;
  createdAt: Date;
  updatedAt: Date;
}
```

### 3.4. Relationship Service

Handles the social graph and user interactions.

- **Key Entities:**
  - `Friendship`: Status-based relations (Pending, Accepted, Blocked) with Favorite/Mute features.
  - `Blocklist`: Advanced user restriction logic (Upcoming).

### 3.4.1 Friendship Entity

```typescript
export interface IFriendship {
  _id: string;
  userId: string;
  friendId: string;
  status: FriendshipStatus;
  isFavorite: boolean;
  isMuted: boolean;
  createdAt: Date;
  updatedAt: Date;
}
```

### 3.4.2 Blocklist Entity

```typescript
export interface IBlocklist {
  _id: string;
  blocker: string;
  blocked: string;
  createdAt: Date;
  updatedAt: Date;
}
```

### 3.5 Notifications

Manages user-centric notification presence. (Development in Progress)

- **Key Entities:**
  - `Email`: Sends email notifications.
  - `Notification`: Notification entity (Upcoming).

### 4. Project Directory Structure

Standardized folder structure across the ecosystem to ensure consistency and scalability.

### 4.1 Backend (NestJS Microservices)

Each microservice follows a modular architecture where every domain logic is encapsulated within its own module.

```plaintext
src/
├── common/             # Global filters, decorators, and shared logic
├── interceptors/       # Request/Response interceptors
├── app-settings/       # Configuration management
├── projection/         # Optimized DB query projections
└── modules/
    └── [module-name]/  # e.g., profile, friendship, auth
        ├── dto/        # Data Transfer Objects (Validation)
        ├── entity/     # Database schemas/models
        ├── interface/  # Local TypeScript interfaces
        ├── repo/       # Repository pattern for DB access
        ├── [name].controller.ts
        ├── [name].module.ts
        └── [name].service.ts
```

### 4.2 Frontend (React Client)

The client-side uses a feature-based structure within the modules directory for high modularity.

```plaintext
src/
├── assets/             # Static files (images, fonts)
├── common/             # Shared utility functions
├── components/         # Reusable UI components (Atomic design)
├── context/            # Global React Context providers
├── layouts/            # Page layout wrappers
├── store/              # Redux/Zustand global state management
└── modules/
    └── [module-name]/  # e.g., auth, messenger, profile
        ├── api/        # RTK Query services or Axios calls
        ├── router/     # Feature-specific route definitions
        ├── types/      # TypeScript types/interfaces
        ├── ui/         # Feature-specific components and views
        ├── constants/  # (Optional) Feature constants
        └── hooks/      # (Optional) Custom hooks for the feature
```

## 5. Data Models & Interface Contracts

### 5.1. Core Identity (`IUser`)

```typescript
export interface IUser {
  _id: string;
  username: string;
  email: string;
  role: ROLE;
  provider: PROVIDER;
  password: string;
  phoneNumber: IPhoneNumber | null;
  status: USER_STATUS;
  isTwoFactorEnabled: boolean;
  timeZone: string | null;
  location: string | null;
  createdAt: Date;
  updatedAt: Date;
}
```

### 5.2. Session Management (IRefreshToken)

```typescript
export interface IRefreshToken {
  _id: string;
  userId: string;
  refresh_token: string;
  deviceId: string | null;
  platform: DEVICE_TYPE;
  ipAddress: string | null;
  location: { country: string; city: string; timezone: string } | null;
  lastUsedAt: Date;
  expiresAt: Date;
  isActive: boolean;
}
```

### 5.3 Entity Example

```typescript
import { v4 as uuid, validate } from "uuid";
import { Prop, Schema, SchemaFactory } from "@nestjs/mongoose";
import { IFriendship } from "../interface/friendship.interface";
import { COLLECTIONS, FriendshipStatus, LANGUAGE } from "connectfy-shared";
import { HydratedDocument } from "mongoose";
import { t } from "i18next";

@Schema({
  timestamps: true,
  collection: COLLECTIONS.RELATIONSHIP.FRIENDSHIPS,
  toJSON: { virtuals: true, versionKey: false, getters: true },
  toObject: { virtuals: true, versionKey: false },
  autoIndex: process.env.NODE_ENV !== "production",
  minimize: false,
  strict: true,
  strictQuery: true,
})
export class FriendshipModel implements IFriendship {
  @Prop({ type: String, default: () => uuid(), immutable: true })
  _id: string;

  @Prop({
    type: String,
    required: [
      true,
      t("validation_messages.required", {
        lng: LANGUAGE.EN,
        field: "userId",
      }),
    ],
    index: true,
    immutable: true,
    trim: true,
    validate: {
      validator: function (value: string | null): boolean {
        return !!(value && validate(value));
      },
      message: t("validation_messages.uuid", {
        lng: LANGUAGE.EN,
        field: "userId",
      }),
    },
    ref: COLLECTIONS.AUTH.USER.USERS,
  })
  userId: string;

  @Prop({
    type: String,
    required: [
      true,
      t("validation_messages.required", {
        lng: LANGUAGE.EN,
        field: "friendId",
      }),
    ],
    index: true,
    immutable: true,
    trim: true,
    validate: {
      validator: function (value: string | null): boolean {
        return !!(value && validate(value));
      },
      message: t("validation_messages.uuid", {
        lng: LANGUAGE.EN,
        field: "friendId",
      }),
    },
    ref: COLLECTIONS.AUTH.USER.USERS,
  })
  friendId: string;

  @Prop({
    type: String,
    enum: {
      values: Object.values(FriendshipStatus),
      message: t("validation_messages.enum", {
        lng: LANGUAGE.EN,
        field: "status",
        values: Object.values(FriendshipStatus),
      }),
    },
    default: FriendshipStatus.Pending,
    index: true,
  })
  status: FriendshipStatus;

  @Prop({
    type: Boolean,
    default: false,
    index: true,
  })
  isFavorite: boolean;

  @Prop({
    type: Boolean,
    default: false,
    index: true,
  })
  isMuted: boolean;

  createdAt: Date;
  updatedAt: Date;
}

export const FriendshipSchema = SchemaFactory.createForClass(FriendshipModel);

FriendshipSchema.index({ userId: 1, friendId: 1 }, { unique: true });
FriendshipSchema.index({ userId: 1, status: 1 });
FriendshipSchema.index({ userId: 1, isFavorite: 1 });
FriendshipSchema.index({ userId: 1, isMuted: 1 });

export type FriendshipDocument = HydratedDocument<FriendshipModel>;
```

### 6. Architectural Patterns & Logic

#### 6.1. Identification Strategy

- **UUID v4:** All documents use uuid instead of MongoDB's default ObjectId to ensure compatibility across distributed systems and simplify migrations (e.g., to PostgreSQL).

#### 6.2. Database Optimization

- **Projection Logic:** Optimized data retrieval using projections to avoid heavy populate operations.
- **Indexing:** Compound indexes on Friendship (e.g., { userId: 1, friendId: 1 }).

Specific indexes for filtering: status, isFavorite, isMuted.

### 6.3. Internationalization (i18n)

Backend validation messages are localized using i18next supporting multiple languages (AZ, EN, RU, TR).

### 7. Development Roadmap

- **[ ] Phase 1 (Current):** Core Auth, Profile Management, and Friendship system.
- **[ ] Phase 2:** Real-time Messaging (1v1) using WebSockets and Kafka.
- **[ ] Phase 3:** Multimedia support (Minio integration for file sharing).
- **[ ] Phase 4:** Voice & Video communication (WebRTC).
- **[ ] Phase 5:** Community/Channels (Discord-style interaction).
