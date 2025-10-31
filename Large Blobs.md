# Handling Large Blobs

📁 Large files need special handling. Use presigned URLs to let clients upload directly to blob storage and download from CDNs, avoiding your application servers as a bottleneck.

## The Problem

**Why blob storage?** Files over 10MB should go to blob storage (S3, GCS, Azure Blob), not databases. Databases handle structured queries; blob storage handles unlimited capacity with 99.999999999% durability.

**The bottleneck:** Routing files through your application servers makes them dumb proxies - adding latency and cost without value. A 2GB video upload that fails at 99% means starting over. Your servers become the bottleneck.

## The Solution

Give clients temporary credentials to interact directly with storage. Your server validates requests and generates presigned URLs, but never touches the actual bytes.

### Direct Upload with Presigned URLs

1. Client requests upload permission
2. Server validates user and generates presigned URL (15-60 min expiry)
3. Client uploads directly to blob storage using the URL
4. Storage service validates the signature

**Key features:**

- URLs are cryptographically signed with expiry times
- Add restrictions: file size limits, content-type validation
- No network call needed to generate URL (happens in memory)

### Resumable Uploads (Multipart)

For large files (>100MB), use chunked uploads:

- AWS S3: Multipart uploads (5MB+ parts)
- GCS: Resumable uploads
- Azure: Block blobs (4MB+ blocks)

If connection drops, client queries which parts succeeded and resumes from there. Don't forget cleanup - incomplete uploads cost money (set lifecycle policies for 1-2 days).

### State Synchronization

Store metadata in your database, files in blob storage. Track status: `pending` → `completed`.

**Problem:** Client might crash after upload but before notifying your API.

**Solution:** Use storage event notifications (S3 → SNS/SQS). When storage receives a file, it publishes an event. Your backend updates the database based on the storage event, not client notification.

**Backup:** Add reconciliation jobs to check files stuck in `pending` status.

### Direct Downloads

Use CDN (CloudFront, Cloud CDN) with signed URLs for fast, geographically distributed downloads. CDN caches content at edge locations worldwide (200ms → 5ms latency).

For large files, enable **range requests** - clients can download chunks and resume on failure.

## Cloud Provider Terminology

| Feature | AWS | Google Cloud | Azure |
|---------|-----|--------------|-------|
| Presigned URLs | Presigned URLs | Signed URLs | SAS tokens |
| Multipart | Multipart Upload API | Resumable Uploads | Block Blobs |
| Events | S3 Event Notifications (SNS/SQS) | Cloud Storage Pub/Sub | Event Grid |
| CDN | CloudFront | Cloud CDN | Azure CDN |

## When to Use

**Use this pattern when:**

- Files > 10MB
- Video/photo platforms (YouTube, Instagram)
- File storage systems (Dropbox)
- Chat apps with media sharing

**Don't use when:**

- Small files < 10MB (use normal API)
- Need synchronous validation during upload
- Compliance requires inspecting data before storage
- Need immediate feedback (e.g., face detection on profile photo)

## Common Interview Questions

### "What if upload fails at 99%?"

Use multipart uploads. Client tracks upload session ID, queries which parts succeeded, resumes from failed part. Store session ID in localStorage for app restarts.

### "How to prevent abuse?"

- Upload to quarantine bucket first
- Run virus scans, content validation
- Set file size limits in presigned URLs
- Only make file available after passing checks

### "How to handle metadata?"

- Store metadata in database (user_id, filename, size, status)
- Create DB record with status='pending' when generating presigned URL
- Update to 'completed' via storage event notifications
- Use storage_key pattern: `uploads/{user_id}/{timestamp}/{uuid}`

### "How to ensure fast downloads?"

- Use CDN with signed URLs for geographic caching
- Enable range requests for resumable downloads
- For very large files, consider parallel chunk downloads

## Key Takeaways

1. **Direct uploads** bypass servers - clients upload directly to blob storage using presigned URLs
2. **Multipart uploads** enable resumability for large files
3. **Event notifications** keep database and storage in sync
4. **CDNs** provide fast, cached downloads worldwide
5. Use for files > 10MB; avoid for small files or compliance-heavy scenarios

