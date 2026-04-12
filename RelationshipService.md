# Relationship Service

## Overview
`connectfy-relationship` manages friendship state and relationship-based rules. It stores friend requests, accepted relationships, close-friend flags, muted-notification flags, and block relationships. It also keeps a local projection of auth user data for search and friendship listing.

It interacts with:
- `connectfy-account` to read privacy settings during friendship decisions
- `connectfy-auth` indirectly through Kafka user projection events
- `connectfy-api-gateway` as the main caller

## Folder Structure
```text
connectfy-relationship/
├── src/
│   ├── app-settings/
│   │   ├── kafka-connections/
│   │   └── tcp-connections/
│   ├── interceptors/
│   ├── modules/
│   │   ├── blocklists/
│   │   └── friendships/
│   └── projection/
│       └── user/
```

## Architecture & Flow
The service runs both Kafka and TCP transports.

Flow:
1. Auth publishes `projection.user.*` events.
2. `projection/user` updates the local user read model used for search and listing.
3. Gateway calls friendship actions through TCP.
4. Friendship service checks duplicate relationships, privacy settings, and blocklist rules before creating or updating records.
5. Friend queries join relationship records with projected users to support search and pagination.

## Entities / Models
### Friendship
- `_id: string`
- `userId: string`
- `friendId: string`
- `status: FriendshipStatus`
- `isFavorite: boolean`
- `isMuted: boolean`

### Blocklist
- `_id: string`
- `blocker: string`
- `blocked: string`

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
- Send, accept, decline, and cancel friend requests
- Unfriend existing relationships
- Toggle close-friend and notification-muted states
- Count and fetch friendships for profile views
- Enforce blocklist checks
- Search friends by projected user data

## APIs
TCP message patterns:
- `friendship/create`
- `friendship/acceptFriendshipRequest`
- `friendship/declineFriendshipRequest`
- `friendship/cancelFriendshipRequest`
- `friendship/unfriend`
- `friendship/updateCloseFriend`
- `friendship/updateNotification`
- `friendship/findManyInternal`
- `friendship/findOneInternalWithCount`
- `friendship/findFriends`
- `friendship/findRequests`

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

## Notes
- Friendship records are directional, so accepted friendships are stored as two records.
- Privacy checks for sending requests rely on `account` service settings.
- The projected `User` model exists to avoid cross-service joins on every friendship query.
