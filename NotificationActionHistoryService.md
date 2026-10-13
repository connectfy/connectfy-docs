# Notification Action History Service

## Overview

`connectfy-notification-action-history` owns two things:

1. **Transactional email.** It consumes `mail.send` (from `connectfy-auth`) and sends the mail
   over SMTP, deduplicated by `mailId`.
2. **The in-app notification inbox** (the bell). Notifications are stored per recipient in
   MongoDB. Producers publish events to Kafka; this service writes the rows and publishes
   `notification.created` / `notification.removed`, which `connectfy-api-gateway` pushes to the
   recipient's sockets. The gateway reads and changes the inbox over TCP.

It also keeps a local read model of users (`projection/user`, fed by
`projection.user.created|updated|removed`), used to populate `actorId` on notifications without
calling auth.

## Folder Structure

```text
connectfy-notification-action-history/
├── src/
│   ├── app-settings/          # kafka-connections, tcp-connections, redis, cache
│   ├── common/                # constants, helpers (avatar enrichment)
│   ├── interceptors/          # logged-user.interceptor.ts (CLS)
│   ├── modules/
│   │   ├── email/             # mail.send consumer, sent-mail dedupe
│   │   └── notification/      # inbox: entity, repo, dto, controller, service
│   └── projection/user/       # read model of auth's users
├── Dockerfile.dev
└── docker-compose.*.yml
```

## Architecture & Flow

### Process model

`main.ts` boots one Nest application with two transports attached (`connectMicroservice`): a
Kafka consumer (custom `LoggingKafkaServer`, which logs registered topics missing on the broker)
and TCP on `PORT` (7000). A global `ValidationPipe({ whitelist: true, transform: true })` applies
to **both** transports.

Kafka uses `autoCommit: false`. Every handler wraps its work in `consumeKafkaEvent` (from
`connectfy-shared`), which commits even when the handler fails, so one bad message can't stall
a partition. A payload the pipe rejects is the exception (see Notes).

### Email flow

`mail.send` → `EmailService`, in three steps:

1. **Claim** the `mailId` with one atomic `insertIfAbsent` in the sent-mail collection.
2. **Send** through `@nestjs-modules/mailer` / Nodemailer.
3. **Mark** it with `sentAt`.

A redelivery of a mail already marked does nothing. One claimed but never marked (the first
attempt died before the provider accepted it) is **sent again**: a duplicate is better than a
lost verification email.

### Group join-request rows (since 2026-10-13)

Messenger sends the **whole state** of one group's pending requests after every change:
`{chatId, syncedAt, count, adminIds, bump, roomName, latestRequester…, title, body}`, keyed by
`chatId`. `syncGroupJoinRequests` applies it in one `bulkWrite`:

- an upsert for each current admin;
- a removal for anyone else's row (a demoted admin), or for every row when `count` is 0;
- an event older than the row's `metadata.syncedAt` leaves that row alone;
- `bump` (a new request) puts the row back on top, unread. Because a timestamps schema makes
  `createdAt` immutable in Mongoose, this write goes through the native driver.

Then it emits `notification.created` for each written row (with `wasUnread`, `bumped`) and
`notification.removed` for each deleted one. Plan:
`claude/docs/plans/group_join_request_followups_plan.md`.

## Entities / Models

### `notifications`

- `_id` (UUID), `eventId` (the producing event's id, `null` for legacy rows), `recipientId`,
  `actorId` (ref users).
- `type` (`NotificationType`), `status` (`unread`, `read`, `pending`, `accepted`, `declined`,
  `cancelled`, `archived`), `channel`.
- `title` / `body`: `{en, az, ru, tr}`, rendered by the producer.
- `resourceId` / `resourceType`, `metadata`.
- `readAt`, `resolvedAt`, `expiresAt` (TTL), `createdAt` / `updatedAt`.

Indexes:

- `{recipientId, status, createdAt: -1}`: the inbox read.
- `{recipientId, actorId, resourceId, type}`, unique, partial on `friendship_request_sent`: one
  friendship-request row per pair.
- `uniq_notification_event`: `{eventId}`, unique, partial on `eventId: {$type: 'string'}`. A
  Kafka redelivery writes nothing twice.
- `uniq_group_join_requests_row`: `{resourceId, recipientId}`, unique, partial on
  `group_join_requests`. One row per (group, admin).
- `{expiresAt}`: TTL.

## Core Features

- Kafka-fed email sending, deduplicated by `mailId`.
- The notification inbox: list, unread count, read/unread, remove, bulk remove.
- Friendship request notifications kept in step with the relationship (accepted, declined,
  cancelled).
- New-device sign-in alerts from auth's outbox.
- Daily friend-birthday notifications, removed once their day has passed.
- Group join-request rows for each group's admins, synced from messenger's state.
- Live delivery through the gateway (`notification.created`, `notification.removed`).

## APIs

### Kafka

Consumed:

| Topic | Producer | Handler |
|---|---|---|
| `mail.send` | auth | send the email once per `mailId` |
| `notification.friendship.created` | relationship | request sent → upsert; accepted → insert once per `eventId` |
| `notification.session.login` | auth (outbox) | new-device sign-in, once per `eventId` |
| `notification.group.join_requests` | messenger | sync a group's join-request rows (below) |
| `projection.user.created/updated/removed` | auth | user read model |

Produced:

| Topic | Payload | Consumer |
|---|---|---|
| `notification.created` | the stored row (actor populated) | gateway → `notification:new` |
| `notification.removed` | `{recipientId, ids, unreadRemoved}` | gateway → `notification:removed` |

### TCP patterns (called by the gateway)

- `notification/all`: the inbox, paginated. Cancelled/declined friendship requests are excluded
  by default. Usernames and avatars are re-resolved in batch on read.
- `notification/countUnread`: counts both `unread` and `pending`.
- `notification/markRead`, `notification/markUnread`, `notification/markAllRead`.
- `notification/remove`, `notification/removeAll`.
- `notification/friendship/metadata` and `/bulk`: relationship moves a friendship row to
  accepted / declined / cancelled.
- `notification/groupJoinRequests/markRead {chatId}`: the viewer's row for that group is read.
  Returns the id it changed, or null.
- `notification/birthday/create`, `notification/birthday/purge`: the daily sweep from
  `connectfy-cron-jobs`. Birthday rows are deleted once their day has passed.

## Technologies Used

NestJS microservices (Kafka + TCP), MongoDB + Mongoose, `@nestjs-modules/mailer` + Nodemailer,
Redis (cache), `connectfy-shared`, `connectfy-i18n`.

## Notes

- **Known gap:** a payload the `ValidationPipe` rejects fails *before* the handler
  runs, is never committed, and is retried forever, stalling its partition. A dead-letter queue is
  planned (see `claude/docs/versions/v1.63.0_group_join_requests.md`, START HERE). Until then, a
  Kafka DTO must accept every payload its producer can send. In particular, `FieldValidator`'s
  ARRAY type rejects an empty array even on an optional field.
- `autoIndex` is off in production, so new indexes (e.g. `uniq_group_join_requests_row`) must be
  built there explicitly.
- Updated 2026-10-13. The previous version described the notification module as "in
  development"; it has been live since the friendship notifications.
