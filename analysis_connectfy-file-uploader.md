# Connectfy File Uploader - Service Analysis

## Overview / Purpose
The `connectfy-file-uploader` is a Node.js microservice responsible for managing file operations within the Connectfy ecosystem. It acts as a lightweight orchestrator for file storage, utilizing an S3-compatible MinIO object storage. Its primary purpose is to generate presigned URLs to facilitate secure, direct browser-to-MinIO file uploads and downloads, offloading network traffic and storage processing from the service itself. It also handles batch file deletions and responds to Kafka events to delete files asynchronously.

## Full Folder Structure
```text
connectfy-file-uploader/
└── src/
    ├── app.js
    └── common/
        └── constants/
            └── environment-variables.js
```

## Architecture & Request/Event Flow
1. **Startup (Bootstrap):** 
   - Connects to MinIO. If the target bucket does not exist, it creates it and sets a public read policy (`s3:GetObject`).
   - Connects to Kafka (initializes both a producer and a consumer).
   - Starts the Express HTTP server.
2. **Direct Upload Flow:**
   - Client sends a `POST` request to `/files/presigned-upload` with file metadata (filename, module name).
   - Service generates a unique object key (`moduleName/uuid.ext`).
   - Service uses a specialized "public" MinIO client to generate a presigned PUT URL valid for 15 minutes.
   - Service returns the presigned URL and the final public file URL.
   - Client uploads the file directly to MinIO using the presigned URL.
3. **Batch Retrieval Flow:**
   - Client sends a `POST` request to `/files/presigned-batch` with an array of object keys.
   - Service iterates through the keys and generates presigned GET URLs valid for 24 hours.
   - Service returns an object mapping object keys to their respective presigned URLs.
4. **Asynchronous Deletion Flow:**
   - The service consumes messages from the Kafka topic `profile.file.delete`.
   - Upon receiving a payload containing a `fileKey`, it calls MinIO to delete the corresponding object from the bucket.

## Entities / Models

### 1. Environment Variables Model
Defined and loaded in `src/common/constants/environment-variables.js`:
- `PORT`: String / Number
- `NODE_ENV`: String
- `MINIO_BUCKET`: String
- `MINIO_ENDPOINT`: String
- `MINIO_ACCESS_KEY`: String
- `MINIO_SECRET_KEY`: String
- `MINIO_PORT`: Number
- `MINIO_PUBLIC_URL`: String
- `SERVICE_NAME`: String
- `BROKER1`: String
- `BROKER2`: String

### 2. Kafka Event Payload: `profile.file.delete`
- `fileKey`: String (The exact object key in MinIO to delete)

## Exposed APIs

### REST Endpoints
- **`POST /files/presigned-upload`**
  - **Description:** Generates a presigned URL for direct file upload.
  - **Request Body:**
    - `filename`: String (Required)
    - `moduleName`: String (Required)
    - `contentType`: String (Optional, defaults to `"application/octet-stream"`)
  - **Response (200 OK):**
    - `uploadUrl`: String (The presigned MinIO URL for PUT request)
    - `fileKey`: String (The generated unique key for the object)
    - `fileUrl`: String (The public accessible URL for the file)
  - **Error Responses:** `400 Bad Request` (missing parameters), `500 Internal Server Error`

- **`POST /files/presigned-batch`**
  - **Description:** Generates presigned GET URLs for a batch of existing files.
  - **Request Body:**
    - `filenames`: Array of Strings (Required)
  - **Response (200 OK):**
    - JSON Object acting as a map: `{ [filename]: "presigned-url" }`
  - **Error Responses:** `400 Bad Request` (missing/invalid array)

- **`DELETE /files/batch`**
  - **Description:** Synchronously deletes a batch of files by their keys.
  - **Request Body:**
    - `filenames`: Array of Strings (Required)
  - **Response (200 OK):**
    - `success`: Boolean (`true`)
    - `count`: Number (length of deleted array)
  - **Error Responses:** `400 Bad Request` (missing/invalid array), `500 Internal Server Error`

- **`POST /files/stat`**
  - **Description:** Retrieves object metadata (size and content type) from MinIO.
  - **Request Body:**
    - `key`: String (Required)
  - **Response (200 OK):**
    - `size`: Number (Bytes)
    - `contentType`: String
  - **Error Responses:** `400 Bad Request` (missing key), `404 Not Found` (object inaccessible or not found)

### Kafka Topics Consumed
- **Topic:** `profile.file.delete`
  - **Consumer Group:** `file-uploader-group`
  - **Behavior:** Parses JSON value and executes `minioClient.removeObject` for the provided `fileKey`.

### Kafka Topics Produced
- **None active.** (The producer is connected during initialization, but no `producer.send()` calls exist in the codebase).

## Core Features
- Direct-to-storage file uploads via presigned URLs.
- Batch reading access delegation via presigned GET URLs.
- Metadata querying for existing MinIO objects.
- Automatic bootstrapping of MinIO buckets and public bucket policies.
- Asynchronous and synchronous object deletion.

## Technologies Used
- Node.js
- Express.js (HTTP routing, JSON parsing, CORS)
- MinIO JavaScript Client (`minio`)
- KafkaJS (`kafkajs`)
- UUID (`uuid` for object key generation)
- Dotenv (`dotenv` for environment configuration)

## Implementation Notes / Gotchas
- **Two MinIO Clients Pattern:** The application explicitly initializes two distinct MinIO clients:
  1. `minioClient`: Configured with internal container endpoints (`MINIO_ENDPOINT`) for server-to-server operations like bucket initialization, batch deletes, and file stats.
  2. `publicMinioClient`: Configured explicitly from the `MINIO_PUBLIC_URL`. This client is strictly used to generate presigned upload URLs. It forces `region: "us-east-1"` to prevent the client from attempting network region discovery, ensuring that URL generation stays completely offline and resolves to the correct public Nginx/Proxy endpoint for browser clients.
- **Auto-Provisioning of Bucket Policies:** The service automatically sets a permissive `s3:GetObject` public policy allowing anonymous reads to `arn:aws:s3:::<BUCKET>/*` on startup.
- **Direct Upload Proxy Pattern:** The service never touches the file buffers. It only acts as an authorization and URL generation proxy, which heavily reduces the RAM and CPU footprint of this service.
- **CORS:** CORS is globally enabled for all origins with credentials allowed (`app.use(cors({ origin: true, credentials: true }))`).

## Known In-progress / Partially Implemented Areas
- **Unused Kafka Producer:** The Kafka producer is fully instantiated and connected (`await producer.connect()`), but it is not utilized anywhere in the codebase. This suggests an intention to broadcast events (e.g., a `file.uploaded` event) that hasn't been implemented yet.
