# Tubely

A video hosting API built as a platform for learning file storage, AWS S3, and CDN delivery in Go. The core server skeleton was provided; the file handling, S3 integration, image processing, video processing, and CDN wiring were written as the learning exercise.

## Stack

| Concern | Tool |
|---|---|
| Database | SQLite (auto-migrated) |
| Auth | JWT (access) + opaque refresh tokens |
| Local asset storage | Filesystem (`/assets`) |
| Video/thumbnail storage | AWS S3 |
| CDN | AWS CloudFront |
| Video processing | `ffmpeg` / `ffprobe` |

## Running

**Prerequisites:** Go 1.21+, `ffmpeg`, AWS credentials configured (`~/.aws/credentials` or env vars)

Copy and fill in the env file:

```bash
cp .env.example .env
```

Required variables:

```
DB_PATH=./tubely.db
JWT_SECRET=your-secret
PLATFORM=dev
FILEPATH_ROOT=./app
ASSETS_ROOT=./assets
S3_BUCKET=your-bucket-name
S3_REGION=us-east-1
S3_CF_DISTRO=your-cloudfront-domain.cloudfront.net
PORT=8080
```

Run:

```bash
go run .
```

Download sample assets for testing:

```bash
bash samplesdownload.sh
```

Server serves the frontend at `http://localhost:8080/app/`.

## API

```
POST /api/users                          create account
POST /api/login                          get JWT + refresh token
POST /api/refresh                        exchange refresh token for new JWT
POST /api/revoke                         revoke refresh token

POST /api/videos                         create video record         [JWT]
GET  /api/videos                         list user's videos          [JWT]
GET  /api/videos/{videoID}               get video metadata          [JWT]
DELETE /api/videos/{videoID}             delete video                [JWT]
POST /api/thumbnail_upload/{videoID}     upload JPEG/PNG thumbnail   [JWT]
POST /api/video_upload/{videoID}         upload MP4                  [JWT]

POST /admin/reset                        wipe DB (dev only)
```

## What I Built

The routing, database schema, and auth helpers were provided. Everything related to file handling, S3, and CDN delivery was written from scratch:

- **Multipart thumbnail upload** — parse `multipart/form-data`, validate MIME type (JPEG/PNG only), write to local `/assets` filesystem, update video record with URL.
- **Multipart video upload** — 1 GB body limit via `http.MaxBytesReader`; stream upload to a temp file to avoid holding the entire payload in memory.
- **Video processing** — shell out to `ffprobe` to read stream dimensions and classify aspect ratio (`16:9` → `landscape/`, `9:16` → `portrait/`, else `other/`); shell out to `ffmpeg` with `-movflags faststart` to move the MP4 `moov` atom to the front of the file before upload, enabling progressive streaming.
- **S3 upload** — `PutObject` with correct `ContentType`; S3 key prefixed with the aspect-ratio directory so the bucket self-organizes.
- **CloudFront delivery** — video URLs are constructed as `https://<cf-distribution>/<key>` rather than direct S3 URLs, routing all reads through the CDN edge layer.
- **No-cache middleware** — local `/assets` responses get `Cache-Control: no-cache` so thumbnail updates are reflected immediately during development.

## What I Learned

From the [boot.dev](https://boot.dev) file storage course:

1. **Stream to disk, not memory** — buffering a 1 GB upload in RAM kills a server fast; writing to a temp file and seeking back to position 0 is the standard pattern.
2. **MP4 `moov` atom placement matters** — without `faststart`, the browser must download the entire file before playback can begin; `ffmpeg -movflags faststart` fixes this with a single remux pass.
3. **S3 "directories" are a naming convention** — there are no real folders; prefixing keys (`landscape/abc.mp4`) is just a UX layer that S3 and most clients understand.
4. **Direct S3 URLs vs CloudFront** — serving from S3 directly incurs per-request egress cost and latency from a single region; CloudFront caches at edge locations and can front a private bucket with signed URL enforcement.
5. **Private buckets + CDN is the right default** — public S3 buckets are a common source of data leaks; keeping the bucket private and routing all reads through CloudFront gives both security and performance.
6. **S3 durability ≠ availability** — 11 nines of durability (data won't be lost) is a separate guarantee from availability (data can be read right now); replication and versioning address different failure modes.
