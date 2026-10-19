# Notification Action History Service

## Overview

`connectfy-notification-action-history` owns three things:

1. **Transactional email.** It consumes `mail.send` (from `connectfy-auth`) and sends the mail
   over SMTP, deduplicated by `mailId`.
2. **The in-app notification inbox** (the bell). Notifications are stored per recipient in
   MongoDB. Producers publish events to Kafka; this service writes the rows and publishes
   `notification.created` / `notification.removed`, which `connectfy-api-gateway` pushes to the
   recipient's sockets. The gateway reads and changes the inbox over TCP.
3. **The user activity log** (since 2026-10-15). It holds each user's own actions: sign-ins,
   credential changes, profile and privacy edits, friend requests, blocks and group membership.
   Every producer publishes `activity.recorded` (the contract is in `connectfy-shared`). This
   service writes one row per event, and the owner reads their own log over TCP. See
   `claude/docs/plans/user_activity_log_plan.md`.
4. **Web Push** (since 2026-10-19). It stores one subscription per browser (`push_subscriptions`) and sends with `web-push` (VAPID keys from env). The sender is `PushSenderService.sendToUsers`. See `claude/docs/plans/push_notifications_plan.md`.

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
│   │   ├── notification/      # inbox: entity, repo, dto, controller, service
│   │   └── user-activity/     # activity log: entity, repo (+ repo module), dto, controller, service
│   └── projection/user/       # read model of auth's users
├── Dockerfile.dev
└── docker-compose.*.yml
```

## Architecture & Flow

### Process model

`main.ts` boots one Nest application with two transports attached (`connectMicroservice`): a
Kafka consumer (`connectfy-shared`'s `DeadLetterKafkaServer`, which also logs registered topics
missing on the broker) and TCP on `PORT` (7000). A global
`ValidationPipe({ whitelist: true, transform: true })` applies to **both** transports.

Kafka uses `autoCommit: false`. Every handler wraps its work in `consumeKafkaEvent` (from
`connectfy-shared`). A transient failure is retried up to 5 times. A message that can't be
applied goes to `kafka.dead-letter` first and is only then committed, so one bad message can't
stall a partition. This covers a payload the pipe rejects, a permanent handler error and an
exhausted retry budget (see Notes).

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

### `push_subscriptions`

- `_id` (UUID), `userId`, `deviceId` (the auth session's `x-device-id`), `endpoint`, `keys {p256dh, auth}`, `userAgent`, `createdAt`, `lastSuccessAt`.
- Indexes: `uniq_push_endpoint` (unique: a re-subscribe or another account on the same browser takes the row over), `push_by_user`, `push_by_session {userId, deviceId}`.
- A send deletes rows the push service answers 404/410 for, in one `bulkWrite` with the `lastSuccessAt` updates. `projection.user.removed` deletes the user's rows.

### `user_activity`

- `_id`: the producing event's `eventId`. That makes it the idempotency key, so a redelivery
  writes nothing.
- `userId`: the log's owner.
- `action` (`UserActivityAction`, 36 values), `category` (derived from `action` via
  `USER_ACTIVITY_CATEGORY`, never sent by the producer), `actorType` (`self` or `system`).
- `at`, `targetIds` (other users the row names).
- `details`: its shape depends on `action`, with before/after values where they apply.
- `device`.
- `expiresAt`: `at` + 730 days for security and account rows, + 365 for the rest.
- Append-only: every field is immutable, and there are no timestamps.

Indexes:

- `user_activity_page`: `{userId, at: -1, _id: -1}`. The unfiltered keyset page, and the purge.
- `user_activity_page_by_category`: `{userId, category, at: -1, _id: -1}`. A filtered page (a
  `SORT_MERGE` over the chosen categories).
- `user_activity_ttl`: `{expiresAt}`, TTL.

A purged account (`projection.user.removed`) loses its own log. Rows in other users' logs that
name it stay, and read as "Connectfy User".

## Core Features

- Kafka-fed email sending, deduplicated by `mailId`.
- The notification inbox: list, unread count, read/unread, remove, bulk remove.
- Friendship request notifications kept in step with the relationship (accepted, declined,
  cancelled).
- New-device sign-in alerts from auth's outbox.
- Daily friend-birthday notifications, removed once their day has passed.
- Group join-request rows for each group's admins, synced from messenger's state.
- Live delivery through the gateway (`notification.created`, `notification.removed`).
- The user activity log: one row per `activity.recorded`, the owner-only paged read, retention
  by TTL.

## APIs

### Kafka

Consumed:

| Topic | Producer | Handler |
|---|---|---|
| `mail.send` | auth | send the email once per `mailId` |
| `notification.friendship.created` | relationship | request sent → upsert; accepted → insert once per `eventId` |
| `notification.session.login` | auth (outbox) | new-device sign-in, once per `eventId` |
| `notification.group.join_requests` | messenger | sync a group's join-request rows (below) |
| `projection.user.created/updated/removed` | auth | user read model (`removed` also deletes the user's own inbox and activity log) |
| `activity.recorded` | auth, account (outbox); relationship, messenger | one activity row per `eventId` (upsert); an invalid event is dead-lettered |
| `auth.session.expired` | auth | delete that session's push subscriptions (`{userId, deviceId}`) |

Produced:

| Topic | Payload | Consumer |
|---|---|---|
| `notification.created` | the stored row (actor populated) | gateway → `notification:new` |
| `notification.removed` | `{recipientId, ids, unreadRemoved}` | gateway → `notification:removed` |

### TCP patterns (called by the gateway)

- `push/publicKey`, `push/subscribe {endpoint, keys, deviceId, userAgent?}` (the endpoint must be on a known push-service host), `push/unsubscribe {endpoint}` (the caller's own row), `push/test` (development only).

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
- `activity/find {order?, categories?, cursor?}`: one page (30) of the **caller's own** activity
  log. The owner comes from `_loggedUser`, never the payload.
  - Paging: `(at, _id)` keyset in either order, cursor `<ms>_<id>`.
  - Reply: `{items, users, nextCursor}`.
  - Rounds: the page, then the people it names (projection + account's privacy-aware avatars, in
    parallel). The second round is skipped when the page names nobody.
  - Non-ACTIVE users come back as `{isDeactivated: true}`; purged users are absent.
  - Gateway: `GET /api/v1/notification-action-history/activity?order=&categories=a,b&cursor=`.

## Technologies Used

NestJS microservices (Kafka + TCP), MongoDB + Mongoose, `@nestjs-modules/mailer` + Nodemailer,
Redis (cache), `connectfy-shared`, `connectfy-i18n`.

## Notes

- **Dead letters** (since 2026-10-14, shared 0.0.170): a payload the `ValidationPipe` rejects
  fails *before* the handler runs. `DeadLetterKafkaServer` still catches it: it writes the raw
  message with `x-dlq-*` headers to `kafka.dead-letter` (reason `rejected`) and commits. If that
  write fails, nothing is committed and the message is redelivered. List or replay the records
  with `connectfy-config/kafka/replay-dead-letter.sh`. A Kafka DTO should still accept every
  payload its producer can send: a rejected message is set aside, not applied. Note that
  `FieldValidator`'s ARRAY type rejects an empty array even on an optional field. Design:
  `claude/docs/plans/kafka_dead_letter_plan.md`.
- `autoIndex` is off in production, so new indexes (e.g. `uniq_group_join_requests_row`) must be
  built there explicitly.
- Updated 2026-10-13. The previous version described the notification module as "in
  development"; it has been live since the friendship notifications.
