# connectfy-file-uploader Source Dump

## File: connectfy-file-uploader/src/app.js
```javascript
import express from "express";
import { Client as MinioClient } from "minio";
import { Kafka } from "kafkajs";
import { v4 as uuidv4 } from "uuid";
import path from "path";
import cors from "cors";
import { ENVIRONMENT_VARIABLES } from "./common/constants/environment-variables.js";

// ─── App ─────────────────────────────────────────────────────────────────────

const app = express();
app.use(express.json());
app.use(cors({ origin: true, credentials: true }));

// ─── MinIO ───────────────────────────────────────────────────────────────────

// Internal client: container-to-container (minio hostname)
const minioClient = new MinioClient({
  endPoint: ENVIRONMENT_VARIABLES.MINIO_ENDPOINT,
  port: ENVIRONMENT_VARIABLES.MINIO_PORT,
  useSSL: false,
  accessKey: ENVIRONMENT_VARIABLES.MINIO_ACCESS_KEY,
  secretKey: ENVIRONMENT_VARIABLES.MINIO_SECRET_KEY,
});

// Public client: generates presigned URLs pointing to Nginx/Public endpoint
const publicUrl = new URL(ENVIRONMENT_VARIABLES.MINIO_PUBLIC_URL);
const publicMinioClient = new MinioClient({
  endPoint: publicUrl.hostname,
  port: publicUrl.port ? parseInt(publicUrl.port, 10) : (publicUrl.protocol === 'https:' ? 443 : 80),
  useSSL: publicUrl.protocol === 'https:',
  accessKey: ENVIRONMENT_VARIABLES.MINIO_ACCESS_KEY,
  secretKey: ENVIRONMENT_VARIABLES.MINIO_SECRET_KEY,
  region: "us-east-1", // Prevents network region discovery; URL generation stays offline
});

const BUCKET = ENVIRONMENT_VARIABLES.MINIO_BUCKET;

const initMinio = async (retries = 5, delayMs = 3000) => {
  for (let attempt = 1; attempt <= retries; attempt++) {
    try {
      const exists = await minioClient.bucketExists(BUCKET);

      if (!exists) {
        await minioClient.makeBucket(BUCKET, "us-east-1");
        console.log(`✅ MinIO bucket "${BUCKET}" created`);
      }

      await minioClient.setBucketPolicy(
        BUCKET,
        JSON.stringify({
          Version: "2012-10-17",
          Statement: [
            {
              Effect: "Allow",
              Principal: { AWS: ["*"] },
              Action: ["s3:GetBucketLocation", "s3:ListBucket", "s3:GetObject"],
              Resource: [`arn:aws:s3:::${BUCKET}`, `arn:aws:s3:::${BUCKET}/*`],
            },
          ],
        }),
      );

      console.log("✅ MinIO ready");
      return;
    } catch (err) {
      console.warn(
        `⚠️ MinIO init attempt ${attempt}/${retries} failed:`,
        err.message,
      );
      if (attempt < retries) await new Promise((r) => setTimeout(r, delayMs));
    }
  }

  console.error("❌ MinIO initialization failed after all retries");
};

// ─── Kafka ───────────────────────────────────────────────────────────────────

const kafka = new Kafka({
  clientId: ENVIRONMENT_VARIABLES.SERVICE_NAME,
  brokers: [ENVIRONMENT_VARIABLES.BROKER1, ENVIRONMENT_VARIABLES.BROKER2],
});

const producer = kafka.producer();
const consumer = kafka.consumer({ groupId: "file-uploader-group" });

const initKafka = async () => {
  await producer.connect();
  console.log("✅ Kafka producer connected");

  await consumer.connect();
  await consumer.subscribe({
    topic: "profile.file.delete",
    fromBeginning: false,
  });

  await consumer.run({
    eachMessage: async ({ message }) => {
      if (!message.value) return;

      try {
        const { fileKey } = JSON.parse(message.value.toString());

        if (!fileKey) {
          console.error("❌ Missing 'fileKey' in message");
          return;
        }

        await minioClient.removeObject(BUCKET, fileKey);
        console.log(`🗑️ Deleted: ${fileKey}`);
      } catch (err) {
        console.error("❌ Failed to process delete message:", err.message);
      }
    },
  });

  console.log("✅ Kafka consumer running");
};

// ─── Routes ──────────────────────────────────────────────────────────────────

// Generate a presigned PUT URL for direct browser-to-MinIO upload
app.post("/files/presigned-upload", async (req, res) => {
  const { filename, moduleName, contentType } = req.body;

  if (!filename || !moduleName) {
    return res
      .status(400)
      .json({ error: "filename and moduleName are required" });
  }

  try {
    const objectKey = `${moduleName}/${uuidv4()}${path.extname(filename).toLowerCase()}`;

    const uploadUrl = await publicMinioClient.presignedUrl(
      "PUT",
      BUCKET,
      objectKey,
      15 * 60, // 15 minutes
      { "Content-Type": contentType || "application/octet-stream" },
    );

    return res.json({
      uploadUrl,
      fileKey: objectKey,
      fileUrl: `${ENVIRONMENT_VARIABLES.MINIO_PUBLIC_URL}/${BUCKET}/${objectKey}`,
    });
  } catch (err) {
    console.error("❌ Presigned URL error:", err.message);
    return res.status(500).json({ error: "Could not generate upload URL" });
  }
});

// Generate presigned GET URLs for a batch of file keys
app.post("/files/presigned-batch", async (req, res) => {
  const { filenames } = req.body;

  if (!Array.isArray(filenames) || filenames.length === 0) {
    return res.status(400).json({ error: "filenames array is required" });
  }

  const EXPIRY = 24 * 60 * 60; // 24 hours

  const entries = await Promise.allSettled(
    filenames.filter(Boolean).map(async (key) => {
      const url = await minioClient.presignedGetObject(BUCKET, key, EXPIRY);
      return [key, url];
    }),
  );

  const result = Object.fromEntries(
    entries.filter((e) => e.status === "fulfilled").map((e) => e.value),
  );

  return res.json(result);
});

// Delete a batch of files by key
app.delete("/files/batch", async (req, res) => {
  const { filenames } = req.body;

  if (!Array.isArray(filenames) || filenames.length === 0) {
    return res.status(400).json({ error: "filenames array is required" });
  }

  try {
    await minioClient.removeObjects(BUCKET, filenames);
    return res.json({ success: true, count: filenames.length });
  } catch (err) {
    console.error("❌ Batch delete error:", err.message);
    return res.status(500).json({ error: "Failed to delete files" });
  }
});

// Get object metadata (size, contentType)
app.post("/files/stat", async (req, res) => {
  const { key } = req.body;

  if (!key) {
    return res.status(400).json({ error: "key is required" });
  }

  try {
    const stat = await minioClient.statObject(BUCKET, key);
    return res.json({
      size: stat.size,
      contentType: stat.metaData['content-type'] || 'application/octet-stream',
    });
  } catch (err) {
    console.error("❌ Stat object error:", err.message);
    return res.status(404).json({ error: "Object not found or accessible" });
  }
});

// ─── Bootstrap ───────────────────────────────────────────────────────────────

const bootstrap = async () => {
  await initMinio();
  await initKafka();

  app.listen(ENVIRONMENT_VARIABLES.PORT, "0.0.0.0", () => {
    console.log(
      `🚀 File service running on port ${ENVIRONMENT_VARIABLES.PORT}`,
    );
  });
};

bootstrap();

```

## File: connectfy-file-uploader/src/common/constants/environment-variables.js
```javascript
import * as dotenv from "dotenv";
import * as path from "path";

const appEnv = process.env.NODE_ENV || "remote";

const envPath = path.resolve(process.cwd(), `.env.${appEnv}`);

dotenv.config({ path: envPath });

export const ENVIRONMENT_VARIABLES = {
  // Core
  PORT: process.env.PORT,
  NODE_ENV: process.env.NODE_ENV,

  MINIO_BUCKET: process.env.MINIO_BUCKET,
  MINIO_ENDPOINT: process.env.MINIO_ENDPOINT,
  MINIO_ACCESS_KEY: process.env.MINIO_ACCESS_KEY,
  MINIO_SECRET_KEY: process.env.MINIO_SECRET_KEY,
  MINIO_PORT: Number(process.env.MINIO_PORT),
  MINIO_PUBLIC_URL: process.env.MINIO_PUBLIC_URL,

  // Kafka
  SERVICE_NAME: process.env.SERVICE_NAME,
  BROKER1: process.env.BROKER1 || "",
  BROKER2: process.env.BROKER2 || "",
};

```

