# S3 Multipart Upload Workflow with Presigned URLs

> Complete step-by-step guide for a Dropbox-style file upload system where the backend
> generates presigned URLs and the client uploads chunks directly to S3.

---

## Overview

```
Client  ──►  Backend  ──►  S3 (CreateMultipartUpload)  →  uploadId
Client  ◄──  Backend  ◄──  N presigned URLs (one per chunk)
Client  ──────────────────────────────────────────────►  S3 (PUT each chunk, collect ETags)
Client  ──►  Backend  ──►  S3 (CompleteMultipartUpload)  →  final object
```

**Why presigned URLs?**  
The client uploads chunks *directly* to S3 — the binary data never passes through your backend.
This saves bandwidth, reduces latency, and offloads heavy I/O from your servers.

---

## Prerequisites

```python
# Python dependencies
pip install boto3

# Environment variables (never hardcode credentials)
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
AWS_REGION=ap-south-1
S3_BUCKET=your-bucket-name
```

---

## Step 1 — Initiate Multipart Upload (Backend)

Call `create_multipart_upload` on S3. This registers the upload session and returns an
`uploadId` that ties all subsequent parts together.

```python
import boto3
import os

s3_client = boto3.client(
    "s3",
    region_name=os.environ["AWS_REGION"],
    aws_access_key_id=os.environ["AWS_ACCESS_KEY_ID"],
    aws_secret_access_key=os.environ["AWS_SECRET_ACCESS_KEY"],
)

BUCKET = os.environ["S3_BUCKET"]

def initiate_multipart_upload(object_key: str, content_type: str = "application/octet-stream") -> str:
    """
    Registers a new multipart upload session with S3.
    Returns the uploadId — store this; you need it for every subsequent step.
    """
    response = s3_client.create_multipart_upload(
        Bucket=BUCKET,
        Key=object_key,
        ContentType=content_type,
    )

    upload_id = response["UploadId"]
    print(f"Multipart upload initiated. UploadId: {upload_id}")
    return upload_id


# --- Example usage ---
upload_id = initiate_multipart_upload(
    object_key="users/pramod/documents/large_file.zip",
    content_type="application/zip",
)
# upload_id = "VXBsb2FkIElEIGZvciBteS1rZXk..."  (opaque string from AWS)
```

**What AWS returns:**

```json
{
    "Bucket": "your-bucket-name",
    "Key": "users/pramod/documents/large_file.zip",
    "UploadId": "VXBsb2FkIElEIGZvciBteS1rZXk..."
}
```

> **Store the `uploadId` in your database** alongside the object key and user ID.
> You will need it for generating presigned URLs, listing parts, and completing the upload.

---

## Step 2 — Calculate Chunk Count (Client or Backend)

Before generating presigned URLs, determine how many parts the file will be split into.

```python
import math

CHUNK_SIZE_BYTES = 5 * 1024 * 1024  # 5 MB — S3 minimum part size (except the last part)

def calculate_parts(file_size_bytes: int) -> int:
    """
    Returns the number of parts needed for a given file size.
    S3 allows 1 to 10,000 parts. Each part (except the last) must be >= 5 MB.
    """
    return math.ceil(file_size_bytes / CHUNK_SIZE_BYTES)


# --- Example ---
file_size = 27 * 1024 * 1024  # 27 MB file
num_parts = calculate_parts(file_size)
print(f"File will be split into {num_parts} parts")
# Output: File will be split into 6 parts
# Parts 1-5 = 5 MB each, Part 6 = 2 MB (last part can be smaller)
```

**S3 part size constraints:**

| Constraint         | Value                  |
|--------------------|------------------------|
| Minimum part size  | 5 MB (except last part)|
| Maximum part size  | 5 GB                   |
| Maximum part count | 10,000                 |
| Maximum object     | 5 TB                   |

---

## Step 3 — Generate Presigned URLs in a Loop (Backend)

For each part number (1-indexed), generate a presigned URL that allows the client to `PUT`
that specific chunk directly to S3. The URL is cryptographically signed and time-limited.

```python
from typing import List, Dict

def generate_presigned_urls(
    object_key: str,
    upload_id: str,
    num_parts: int,
    expires_in_seconds: int = 1800,  # 30 minutes per part
) -> List[Dict]:
    """
    Generates one presigned PUT URL per part.
    Each URL is scoped to a specific partNumber and uploadId — it cannot
    be reused for a different part or a different upload session.

    Returns a list of dicts: [{"part_number": 1, "presigned_url": "https://..."}, ...]
    """
    presigned_urls = []

    for part_number in range(1, num_parts + 1):
        url = s3_client.generate_presigned_url(
            ClientMethod="upload_part",
            Params={
                "Bucket": BUCKET,
                "Key": object_key,
                "UploadId": upload_id,
                "PartNumber": part_number,
            },
            ExpiresIn=expires_in_seconds,
            HttpMethod="PUT",
        )

        presigned_urls.append({
            "part_number": part_number,
            "presigned_url": url,
        })

        print(f"  Generated URL for part {part_number}/{num_parts}")

    return presigned_urls


# --- Example ---
presigned_urls = generate_presigned_urls(
    object_key="users/pramod/documents/large_file.zip",
    upload_id=upload_id,
    num_parts=num_parts,
    expires_in_seconds=1800,
)

# Return this list to the client in your API response
print(f"Generated {len(presigned_urls)} presigned URLs")
```

**What a presigned URL looks like:**

```
https://your-bucket.s3.ap-south-1.amazonaws.com/users/pramod/documents/large_file.zip
  ?X-Amz-Algorithm=AWS4-HMAC-SHA256
  &X-Amz-Credential=AKIAIOSFODNN7EXAMPLE%2F20240315%2Fap-south-1%2Fs3%2Faws4_request
  &X-Amz-Date=20240315T103000Z
  &X-Amz-Expires=1800
  &X-Amz-SignedHeaders=host
  &partNumber=1
  &uploadId=VXBsb2FkIElEIGZvciBteS1rZXk...
  &X-Amz-Signature=abcdef1234567890...
```

**Security properties encoded in this URL:**

| Parameter           | Purpose                                                   |
|---------------------|-----------------------------------------------------------|
| `X-Amz-Credential`  | Scoped to your AWS account, region, and service (s3)      |
| `X-Amz-Date`        | Signing timestamp — S3 computes expiry from this          |
| `X-Amz-Expires`     | TTL in seconds — request rejected after Date + Expires    |
| `partNumber`        | Part is locked to this number; URL invalid for other parts|
| `uploadId`          | Tied to this upload session only                          |
| `X-Amz-Signature`   | HMAC-SHA256 of all the above — tamper-proof               |

> **Your AWS Secret Key never leaves the backend.**
> S3 re-derives the same HMAC using its internal copy of your key and compares signatures.

---

## Step 4 — Client Uploads Chunks Directly to S3

The client receives the presigned URL list, splits the file into chunks, and PUTs each chunk
directly to S3 — no backend involved at this stage.

```javascript
// Client-side JavaScript (browser or Node.js)

const CHUNK_SIZE = 5 * 1024 * 1024; // 5 MB

async function uploadChunksToS3(file, presignedUrls) {
    const etags = [];

    for (const { part_number, presigned_url } of presignedUrls) {
        const start = (part_number - 1) * CHUNK_SIZE;
        const end   = Math.min(start + CHUNK_SIZE, file.size);
        const chunk = file.slice(start, end);

        console.log(`Uploading part ${part_number}, bytes ${start}–${end}`);

        const response = await fetch(presigned_url, {
            method: "PUT",
            body: chunk,
            headers: {
                "Content-Type": "application/octet-stream",
            },
        });

        if (!response.ok) {
            throw new Error(`Part ${part_number} failed: HTTP ${response.status}`);
        }

        // S3 returns the ETag in the response header — MUST collect this
        const etag = response.headers.get("ETag");
        etags.push({ part_number, etag });

        console.log(`  Part ${part_number} uploaded. ETag: ${etag}`);
    }

    return etags; // Send these back to your backend
}


// --- Example usage ---
const file = document.getElementById("file-input").files[0];

// presignedUrls comes from your backend API response (Step 3)
const etags = await uploadChunksToS3(file, presignedUrls);

// etags = [
//   { part_number: 1, etag: '"abc123..."' },
//   { part_number: 2, etag: '"def456..."' },
//   ...
// ]

// Now send etags + uploadId to your backend to complete the upload
await fetch("/api/complete-upload", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ upload_id: uploadId, etags }),
});
```

**S3 responds to each PUT with:**

```
HTTP 200 OK
ETag: "abc123def456..."    ← MD5 hash of the uploaded chunk bytes
```

> **The ETag is your chunk receipt.** It is the MD5 hash of that part's bytes.
> Collect and store every ETag — you cannot complete the upload without them.

---

## Step 5 — Verify Parts Were Received (Backend, Optional but Recommended)

Before completing the upload, your backend can call `list_parts` to get the authoritative
list of parts S3 actually received. Compare this against the ETags the client reported
to catch missing parts or client-side bugs.

```python
def verify_uploaded_parts(
    object_key: str,
    upload_id: str,
    expected_num_parts: int,
) -> List[Dict]:
    """
    Asks S3 which parts it has actually received for this uploadId.
    Returns the verified list of parts. Raises if any part is missing.
    """
    response = s3_client.list_parts(
        Bucket=BUCKET,
        Key=object_key,
        UploadId=upload_id,
    )

    received_parts = response.get("Parts", [])
    received_count = len(received_parts)

    print(f"S3 has received {received_count}/{expected_num_parts} parts")

    if received_count != expected_num_parts:
        missing = set(range(1, expected_num_parts + 1)) - {p["PartNumber"] for p in received_parts}
        raise ValueError(f"Missing parts: {sorted(missing)}")

    # Log part details for debugging
    for part in received_parts:
        print(f"  Part {part['PartNumber']}: size={part['Size']} bytes, ETag={part['ETag']}")

    return received_parts


# --- Example ---
verified_parts = verify_uploaded_parts(
    object_key="users/pramod/documents/large_file.zip",
    upload_id=upload_id,
    expected_num_parts=num_parts,
)
```

**S3 list_parts response:**

```json
{
    "Parts": [
        { "PartNumber": 1, "Size": 5242880, "ETag": "\"abc123...\"", "LastModified": "..." },
        { "PartNumber": 2, "Size": 5242880, "ETag": "\"def456...\"", "LastModified": "..." },
        { "PartNumber": 3, "Size": 2097152, "ETag": "\"ghi789...\"", "LastModified": "..." }
    ],
    "StorageClass": "STANDARD"
}
```

---

## Step 6 — Complete the Multipart Upload (Backend)

Send the complete ordered list of `(partNumber, ETag)` pairs to S3. S3 assembles all chunks
into a single object at your specified key. If any ETag doesn't match what S3 stored, this
call fails — providing automatic integrity enforcement.

```python
def complete_multipart_upload(
    object_key: str,
    upload_id: str,
    etags: List[Dict],  # [{"part_number": 1, "etag": '"abc123..."'}, ...]
) -> str:
    """
    Instructs S3 to assemble all uploaded parts into the final object.
    etags must be in ascending partNumber order.
    Returns the final S3 object URL.
    """
    # Build the parts list in the format S3 expects
    parts = [
        {
            "PartNumber": item["part_number"],
            "ETag": item["etag"],
        }
        for item in sorted(etags, key=lambda x: x["part_number"])
    ]

    response = s3_client.complete_multipart_upload(
        Bucket=BUCKET,
        Key=object_key,
        UploadId=upload_id,
        MultipartUpload={"Parts": parts},
    )

    final_url = response["Location"]
    print(f"Upload complete! Object available at: {final_url}")
    return final_url


# --- Example ---
# etags received from client (Step 4) or from list_parts (Step 5)
client_etags = [
    {"part_number": 1, "etag": '"abc123..."'},
    {"part_number": 2, "etag": '"def456..."'},
    {"part_number": 3, "etag": '"ghi789..."'},
]

final_url = complete_multipart_upload(
    object_key="users/pramod/documents/large_file.zip",
    upload_id=upload_id,
    etags=client_etags,
)
```

**S3 complete response:**

```json
{
    "Location": "https://your-bucket.s3.amazonaws.com/users/pramod/documents/large_file.zip",
    "Bucket": "your-bucket-name",
    "Key": "users/pramod/documents/large_file.zip",
    "ETag": "\"combined-etag-d41d8cd98f00b204e9800998ecf8427e-6\""
}
```

> The final ETag for a multipart object follows the format `"<md5>-<part_count>"` — it is
> NOT the MD5 of the full file. This is by design and expected behavior.

---

## Step 7 — Abort on Failure (Backend, Error Handling)

If the upload is abandoned or fails permanently, abort it to free S3 storage. Incomplete
multipart uploads accrue storage costs even before completion.

```python
def abort_multipart_upload(object_key: str, upload_id: str) -> None:
    """
    Cancels the upload session and deletes all uploaded parts.
    Call this on unrecoverable errors or user cancellations.
    Any presigned URLs for this uploadId will return 404 NoSuchUpload after this.
    """
    s3_client.abort_multipart_upload(
        Bucket=BUCKET,
        Key=object_key,
        UploadId=upload_id,
    )
    print(f"Upload {upload_id} aborted. All parts deleted.")


# --- Example ---
try:
    complete_multipart_upload(object_key, upload_id, etags)
except Exception as e:
    print(f"Upload failed: {e}")
    abort_multipart_upload(object_key, upload_id)
```

> **Tip:** Add an S3 lifecycle rule to auto-abort incomplete multipart uploads older than
> 7 days as a safety net for orphaned sessions:
> ```json
> { "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 7 } }
> ```

---

## Full Backend API Endpoint (Putting It Together)

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()


class InitiateRequest(BaseModel):
    filename: str
    file_size: int
    content_type: str = "application/octet-stream"


class CompleteRequest(BaseModel):
    upload_id: str
    object_key: str
    etags: List[Dict]  # [{"part_number": 1, "etag": "..."}, ...]


@app.post("/api/upload/initiate")
def initiate_upload(req: InitiateRequest):
    object_key = f"uploads/{req.filename}"
    num_parts   = calculate_parts(req.file_size)

    upload_id      = initiate_multipart_upload(object_key, req.content_type)
    presigned_urls = generate_presigned_urls(object_key, upload_id, num_parts)

    # Persist to DB: (upload_id, object_key, num_parts, status="in_progress")

    return {
        "upload_id":      upload_id,
        "object_key":     object_key,
        "num_parts":      num_parts,
        "presigned_urls": presigned_urls,
    }


@app.post("/api/upload/complete")
def complete_upload(req: CompleteRequest):
    try:
        # Optional: verify with list_parts before completing
        verify_uploaded_parts(req.object_key, req.upload_id, len(req.etags))

        final_url = complete_multipart_upload(req.object_key, req.upload_id, req.etags)

        # Update DB: status="complete", s3_url=final_url

        return {"status": "complete", "url": final_url}

    except Exception as e:
        abort_multipart_upload(req.object_key, req.upload_id)
        raise HTTPException(status_code=500, detail=str(e))
```

---

## Sequence Summary

| Step | Actor   | Action                                    | Key Output           |
|------|---------|-------------------------------------------|----------------------|
| 1    | Backend | `CreateMultipartUpload`                   | `uploadId`           |
| 2    | Backend | Calculate chunk count from file size      | `numParts`           |
| 3    | Backend | Loop → `generate_presigned_url` per part  | N presigned PUT URLs |
| 4    | Client  | Split file, PUT each chunk to S3 directly | N ETags              |
| 5    | Backend | `ListParts` → verify all parts received   | Verified parts list  |
| 6    | Backend | `CompleteMultipartUpload` with ETags      | Final S3 object URL  |
| 7    | Backend | `AbortMultipartUpload` on failure         | Parts cleaned up     |

---

## Security Checklist

- [ ] Presigned URLs generated server-side only — client never touches AWS credentials
- [ ] TTL set to 15–30 minutes per part (not the S3 max of 7 days)
- [ ] `uploadId` stored in your DB, validated before generating new URLs
- [ ] `object_key` includes user ID prefix — prevents path traversal across users
- [ ] S3 bucket is **not** public — objects accessed via presigned GET URLs only
- [ ] Lifecycle rule configured to abort incomplete uploads after 7 days
- [ ] `AbortMultipartUpload` called in all error/cancel paths
