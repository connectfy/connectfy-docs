# Notification Action History Service

## Overview

`connectfy-notification-action-history` currently acts as the email delivery service for the system and contains the initial notification history model. The active implementation is Kafka-driven email sending; notification and device-token modules are present in the codebase but are still under development and are intentionally excluded here.

It receives email jobs from other services, especially `connectfy-auth`, via Kafka topic `mail.send`.

## Folder Structure

```text
connectfy-notification-action-history/
├── src/
│   ├── app-settings/
│   │   ├── kafka-connections/
│   │   └── tcp-connections/
│   ├── common/constants/
│   └── modules/
│       ├── email/
│       └── notification/
├── Dockerfile.dev
└── docker-compose.*.yml
```

Main areas:

- `modules/email`: implemented Kafka email consumer.
- `modules/notification/entities`: persisted notification schema used for future notification history flows.

## Architecture & Flow

The service starts both Kafka and TCP transports, but the implemented business flow is Kafka-based:

1. Another service emits `mail.send`.
2. `EmailController` consumes the event.
3. `EmailService` passes the payload to Nest Mailer.
4. Kafka offsets are committed after successful handling.

## Entities / Models

### Notification

- `_id: string` - notification UUID
- `recipientId: string` - target user
- `actorId: string | null` - initiating user
- `type: NotificationType` - business notification type
- `title: string` - notification title
- `body: string` - notification body
- `status: NotificationStatus` - read/unread state
- `channel: NotificationChannel` - delivery channel
- `pushDeliveryStatus: Push_DELIVERY_STATUS` - push pipeline status
- `pushRetryCount: number` - retry counter
- `resourceId: string | null` - linked domain object id
- `resourceType: string | null` - linked domain object type
- `metadata: Record<string, unknown> | null` - flexible payload
- `readAt: Date | null` - read timestamp
- `expiresAt: Date | null` - TTL-based expiration

## Core Features

- Consume email jobs from Kafka
- Send transactional emails through SMTP
- Commit Kafka offsets safely
- Define the base notification history schema for future expansion

## APIs

Implemented interface:

- Kafka event `mail.send`

No active REST or TCP business API is currently wired for documented use.

## Technologies Used

- NestJS microservices
- Kafka
- MongoDB + Mongoose
- `@nestjs-modules/mailer`
- Nodemailer

## Notes

- This doc intentionally excludes the in-progress notification and device-token modules.
- Email sending is asynchronous and decoupled from the calling service.
- `expiresAt` on `Notification` uses a MongoDB TTL index for automatic cleanup.
