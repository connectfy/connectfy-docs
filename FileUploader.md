# File Uploader

## Overview
`connectfy-file-uploader` is a lightweight Express service dedicated to file storage operations. It generates MinIO presigned URLs for direct browser uploads, resolves presigned download URLs in batches, and deletes files both synchronously and asynchronously from Kafka events.

It is called by the API Gateway for upload URL generation and listens to Kafka for profile file deletion events.

## Folder Structure
```text
connectfy-file-uploader/
├── src/
│   ├── app.js
│   └── common/constants/environment-variables.js
├── Dockerfile.dev
├── docker-compose.dev.yml
└── docker-compose.prod.yml
```

## Architecture & Flow
1. On startup, the service initializes MinIO and ensures the configured bucket exists.
2. It applies a public read bucket policy and creates a second MinIO client for browser-accessible presigned URLs.
3. It connects a Kafka producer and consumer.
4. The consumer listens to `profile.file.delete` and removes matching objects from MinIO.
5. HTTP routes provide upload, batch read, and batch delete operations.

## Entities / Models
This service does not use a database and does not define persistence entities.

Core request models:
- Presigned upload request: `filename`, `moduleName`, `contentType`
- Batch read/delete request: `filenames: string[]`

## Core Features
- Generate presigned PUT URLs for direct uploads
- Build stable object keys using `moduleName + uuid + extension`
- Return final public file URLs
- Generate presigned GET URLs for many files at once
- Delete files in batches
- React to Kafka deletion events

## APIs
- `POST /files/presigned-upload` - generate a presigned upload URL and final file path
- `POST /files/presigned-batch` - generate presigned GET URLs for many keys
- `DELETE /files/batch` - delete multiple objects by key

## Technologies Used
- Express
- MinIO
- KafkaJS
- UUID
- CORS
- Docker

## Notes
- Uploads are direct browser-to-MinIO; the service does not proxy file bytes.
- In development, generated public URLs point to `localhost`; internally, MinIO still uses container networking.
- The bucket policy is intentionally public-read for object access.
