# async-s3

A small, async-first Python client for S3-compatible object storage — AWS S3, MinIO, Cloudflare R2, B2 Backblaze and anything else that speaks the S3 protocol.

Built on top of [`aiobotocore`](https://github.com/aio-libs/aiobotocore), `async-s3` wraps a useful subset of the S3 API into a clean, **bucket-scoped**, typed interface. No sessions to manage, no `ClientError` code unpacking, no pagination boilerplate.

```python
async with S3Client(config) as s3:
    await s3.put_bytes("notes/hello.txt", b"hello world", content_type=S3ContentType.PLAIN)
    content = await s3.get_bytes("notes/hello.txt")
```

---

## Why async-s3?

Raw `aiobotocore` is powerful but low-level — you get a thin async wrapper around AWS's own `botocore`, which means wrestling with sessions, context managers, paginated response dicts, and `ClientError` exception codes. `async-s3` is a higher-level client that removes that friction:

- **Bucket-scoped** — one client, one bucket. No per-call bucket argument, no accidental cross-bucket operations.
- **Works with any S3-compatible storage** — point it at MinIO, Cloudflare R2, Backblaze B2, or any other S3-compatible endpoint via a single config field. Not AWS-specific.
- **Fully async** — built on `aiobotocore`, plays naturally with `asyncio`, FastAPI, and similar stacks.
- **Pagination handled for you** — listing and deleting across thousands of objects just works, internally batching requests automatically.
- **Typed exceptions** — no `error.response['Error']['Code']` checks. Catch `S3ObjectNotFoundError`, `S3PutObjectError`, `S3BatchDeleteError`, and friends directly.
- **Streaming support** — read and write large objects without loading them fully into memory.
- **Minimal footprint** — deliberately covers the operations that come up 80% of the time.

---

## Requirements

- Python `>= 3.12`

---

## Installation

### From GitHub (latest)

```bash
uv add git+https://github.com/<your-org>/async-s3.git
# or
pip install "git+https://github.com/<your-org>/async-s3.git"
```

### From GitHub (specific version / tag)

```bash
uv add git+https://github.com/<your-org>/async-s3.git@v0.1.0
# or
pip install "git+https://github.com/<your-org>/async-s3.git@v0.1.0"
```

### From GitHub (specific commit)

```bash
uv add git+https://github.com/<your-org>/async-s3.git@<commit-hash>
# or
pip install "git+https://github.com/<your-org>/async-s3.git@<commit-hash>"
```

---

## Quick Start

```python
import asyncio
from async_s3 import S3Client, S3Config, S3ContentType


async def main() -> None:
    config = S3Config(
        endpoint_url="https://s3.example.com",
        region="us-east-1",
        bucket="my-bucket",
        access_key_id="YOUR_ACCESS_KEY",
        secret_access_key="YOUR_SECRET_KEY",
    )

    async with S3Client(config) as s3:
        await s3.put_bytes("notes/hello.txt", b"hello world", content_type=S3ContentType.PLAIN)

        content = await s3.get_bytes("notes/hello.txt")
        print(content.decode())  # hello world


asyncio.run(main())
```

---

## Usage

### Upload and download bytes

```python
async with S3Client(config) as s3:
    await s3.put_bytes("data/record.json", payload, content_type=S3ContentType.JSON)
    raw = await s3.get_bytes("data/record.json")
```

### Upload and download files

```python
from pathlib import Path

async with S3Client(config) as s3:
    await s3.put_file("backups/report.csv", Path("report.csv"))
    await s3.get_file("backups/report.csv", Path("downloads/report.csv"), overwrite=True)
```

`get_file` writes atomically — it streams to a temp file first and renames it into place, so a failed download never leaves a partial file at the target path.

### Stream large objects

For objects that shouldn't be fully loaded into memory:

```python
async with S3Client(config) as s3:
    # Stream upload from any BinaryIO
    with open("big-video.mp4", "rb") as f:
        await s3.put_stream("media/big-video.mp4", f, content_type=S3ContentType.MP4)

    # Stream download into any writable BinaryIO
    with open("local-copy.mp4", "wb") as f:
        bytes_written = await s3.get_stream("media/big-video.mp4", f)

    print(f"Downloaded {bytes_written} bytes")
```

### Check existence and handle missing objects

```python
from async_s3 import S3ObjectNotFoundError

async with S3Client(config) as s3:
    if await s3.exists("images/cat.png"):
        data = await s3.get_bytes("images/cat.png")
        print(f"downloaded {len(data)} bytes")

    try:
        await s3.get_bytes("images/missing.png")
    except S3ObjectNotFoundError as e:
        print(f"not found: {e.key}")
```

### List keys and subprefixes

```python
async with S3Client(config) as s3:
    # All keys under a prefix — pagination is handled automatically
    keys = await s3.list_keys("uploads/")
    print(keys)  # ["uploads/a.txt", "uploads/b.txt", ...]

    # Immediate logical "subdirectories" one level deep
    prefixes = await s3.list_subprefixes("projects/")
    print(prefixes)  # ["projects/alpha/", "projects/beta/"]

    # Both accept None or "" to mean the bucket root
    all_keys = await s3.list_keys()
```

### Delete objects

```python
async with S3Client(config) as s3:
    # Delete a single object
    await s3.delete_key("tmp/old-file.txt")

    # Delete a specific set of keys (batched automatically, up to 1000 per request)
    deleted = await s3.delete_keys(["tmp/a.txt", "tmp/b.txt", "tmp/c.txt"])
    print(f"Deleted {deleted} objects")

    # Delete everything under a prefix
    deleted = await s3.delete_prefix("tmp/")
    print(f"Deleted {deleted} objects")

    # Delete the entire bucket contents — requires explicit opt-in
    deleted = await s3.delete_prefix("", allow_root=True)
```

### Move objects within a bucket

Move is a copy + delete and is **not atomic**. Supports objects up to 5 GiB (S3 single-copy limit). If the delete step fails after a successful copy, `S3MoveObjectError` is raised with `.stage == "delete_source"`.

```python
async with S3Client(config) as s3:
    await s3.move(
        source_key="uploads/video.mp4",
        target_key="archive/video.mp4",
        overwrite=False,  # raises ValueError if target already exists
    )
```

### Build S3 keys safely

```python
# Join path segments into a clean S3 key (strips stray slashes)
key = S3Client.join("users", "42", "avatar.png")
print(key)    # users/42/avatar.png

# Split a key back into parts (ignores leading/trailing/repeated slashes)
parts = S3Client.split("/users/42/avatar.png/")
print(parts)  # ["users", "42", "avatar.png"]
```

### Manual lifecycle management

Use `open()` / `close()` when a context manager isn't convenient — e.g. inside a long-lived service object:

```python
s3 = S3Client(config)
await s3.open()
try:
    await s3.put_bytes("notes/manual.txt", b"manual lifecycle")
finally:
    await s3.close()
```

Both `open()` and `close()` are safe to call multiple times.

---

## Content Types

`S3ContentType` is a `StrEnum` of common MIME types, usable anywhere a `content_type` string is accepted:

| Constant | MIME type | Use for |
|---|---|---|
| `JSON` | `application/json` | Structured data, API payloads |
| `PARQUET` | `application/vnd.apache.parquet` | Columnar analytics files |
| `OCTET_STREAM` | `application/octet-stream` | Generic binary fallback |
| `ZIP` | `application/zip` | Compressed archives |
| `MP4` | `video/mp4` | Standard MP4 video |
| `WEBM` | `video/webm` | Web-optimized video |
| `OPUS` | `audio/ogg` | Opus audio in Ogg container |
| `MP3` | `audio/mpeg` | Compressed audio |
| `WAV` | `audio/wav` | Uncompressed audio |
| `PNG` | `image/png` | Lossless images |
| `JPEG` | `image/jpeg` | Compressed photos |
| `WEBP` | `image/webp` | Modern image format |
| `GIF` | `image/gif` | Animated or simple images |
| `HTML` | `text/html` | Web pages |
| `PLAIN` | `text/plain` | Plain text |
| `CSV` | `text/csv` | Tabular data |

You can also pass any arbitrary string for types not covered:

```python
await s3.put_bytes("model.onnx", data, content_type="application/octet-stream")
```

---

## Exception Reference

All exceptions except `S3ObjectNotFoundError` inherit from `S3OperationError`. You can catch broadly or specifically:

| Exception | Raised when | Notable attributes |
|---|---|---|
| `S3ObjectNotFoundError` | Key does not exist (get, head, move source) | `.key` |
| `S3PutObjectError` | Put operation fails | `.key`, `.bucket` |
| `S3GetObjectError` | Get fails for a reason other than missing key | `.key`, `.bucket` |
| `S3DeleteObjectError` | Single-key delete fails | `.key`, `.bucket` |
| `S3BatchDeleteError` | Batch delete fails or backend reports per-key errors | `.keys`, `.delete_errors`, `.deleted_keys`, `.bucket` |
| `S3ListObjectsError` | List operation fails | `.prefix`, `.bucket` |
| `S3HeadObjectError` | Head fails for a reason other than missing key | `.key`, `.bucket` |
| `S3MoveObjectError` | Copy or source-delete step of a move fails | `.source_key`, `.target_key`, `.stage`, `.bucket` |

```python
from async_s3 import (
    S3OperationError,
    S3ObjectNotFoundError,
    S3BatchDeleteError,
    S3MoveObjectError,
)

try:
    await s3.move("uploads/video.mp4", "archive/video.mp4")
except S3ObjectNotFoundError as e:
    print(f"source key missing: {e.key}")
except S3MoveObjectError as e:
    # e.stage is "copy" or "delete_source"
    print(f"move failed at '{e.stage}' stage: {e.source_key} -> {e.target_key}")
except S3OperationError as e:
    print(f"s3 error in bucket '{e.bucket}': {e}")
```

---

## API Reference

### `S3Config`

| Field | Type | Description |
|---|---|---|
| `endpoint_url` | `str` | Base URL of your S3-compatible storage, must include scheme (`https://`) |
| `region` | `str` | Region identifier (can be arbitrary for non-AWS providers) |
| `bucket` | `str` | The bucket this client will operate on |
| `access_key_id` | `str` | Access key (excluded from `repr`) |
| `secret_access_key` | `str` | Secret key (excluded from `repr`) |

### `S3Client`

| Method | Returns | Description |
|---|---|---|
| `put_bytes(key, data, *, content_type)` | `None` | Upload raw bytes |
| `put_file(key, path, *, content_type)` | `None` | Upload a local file |
| `put_stream(key, stream, *, content_type)` | `None` | Upload from a binary stream |
| `get_bytes(key)` | `bytes` | Download object into memory |
| `get_file(key, path, *, overwrite)` | `None` | Download object to a local file (atomic) |
| `get_stream(key, target)` | `int` | Stream object into a writable `BinaryIO`, returns bytes written |
| `exists(key)` | `bool` | Check whether an object exists |
| `list_keys(prefix)` | `list[Key]` | List all keys under a prefix, paginated |
| `list_subprefixes(prefix)` | `list[Prefix]` | List immediate logical child prefixes (one level deep) |
| `delete_key(key)` | `None` | Delete a single object |
| `delete_keys(keys)` | `int` | Delete a set of exact keys, auto-batched; returns deleted count |
| `delete_prefix(prefix, *, allow_root)` | `int` | Delete all objects under a prefix; returns deleted count |
| `move(source_key, target_key, *, overwrite)` | `None` | Copy + delete within the same bucket (max 5 GiB) |
| `open()` / `close()` | `None` | Manual lifecycle management |
| `S3Client.join(*segments)` | `str` | Build a clean S3 key from path segments |
| `S3Client.split(key)` | `list[str]` | Split a key into parts, ignoring empty segments |

---

## S3-compatible providers

| Provider | `endpoint_url` example |
|---|---|
| AWS S3 | omit, or `https://s3.amazonaws.com` |
| MinIO | `http://localhost:9000` |
| Cloudflare R2 | `https://<account-id>.r2.cloudflarestorage.com` |
| Backblaze B2 | `https://s3.<region>.backblazeb2.com` |
