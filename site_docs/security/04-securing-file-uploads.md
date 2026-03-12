# Module 4: Securing File Uploads

## Why Uploads Are Dangerous

File uploads are the attack surface with the highest risk.

- Attacker controls the content (malware, zip bombs)
- Attacker controls the filename (path traversal: `../../../etc/passwd`)
- Original filename might contain sensitive info
- Unsafe processing can trigger RCE (remote code execution)

Rule: **Never trust files from users. Never.**

## The Steps to Secure Upload

1. **Validate file size** (limit upload size)
2. **Validate magic bytes** (file type, not extension)
3. **Scan for malware** (ClamAV)
4. **Store with UUID keys** (never use original filename)
5. **Serve via presigned URLs** (don't expose internal paths)
6. **Restrict file types** (whitelist allowed MIME types)

## Step 1: File Size Limits

### Nginx Configuration

```nginx
client_max_body_size 100m;  # Max upload size
```

Without this, Nginx rejects uploads >1MB by default.

### FastAPI Configuration

```python
from fastapi import FastAPI, File, UploadFile

@app.post("/api/documents/upload")
async def upload_document(file: UploadFile = File(...)):
    # FastAPI enforces form size limit by default
    # Max size: 1MB (configurable)
    pass
```

Increase with middleware:

```python
from starlette.requests import Request

# Set form size limit to 100MB
app.middleware('http')(
    lambda app: app  # Handled via middleware
)

# Better: set in environment
# Or use multipart form data parser
```

Actually, set in the upload endpoint:

```python
import aiofiles
from fastapi import FastAPI, File, UploadFile, HTTPException

@app.post("/api/documents/upload")
async def upload_document(file: UploadFile = File(...)):
    # Check file size before processing
    if file.size > 100 * 1024 * 1024:  # 100MB
        raise HTTPException(status_code=413, detail="File too large")
    
    return {"filename": file.filename}
```

But this comes after upload. Better: reject at Nginx level.

## Step 2: Validate Magic Bytes

File extension is useless. Attacker renames `malware.exe` to `document.pdf`.

Check magic bytes (file signature):

```python
import magic

# Magic bytes for common formats:
# PDF: %PDF
# JPEG: ff d8 ff
# PNG: 89 50 4e 47
# ZIP: 50 4b 03 04

def get_file_magic(filepath):
    mime = magic.Magic(mime=True)
    return mime.from_file(filepath)

# ALLOWED_MIMES = {'application/pdf', 'image/jpeg', 'image/png'}

@app.post("/api/documents/upload")
async def upload_document(file: UploadFile = File(...)):
    contents = await file.read()
    
    # Validate magic bytes
    mime_type = magic.from_buffer(contents, mime=True)
    
    if mime_type not in ALLOWED_MIMES:
        raise HTTPException(status_code=400, detail="Invalid file type")
    
    # Safe to proceed
    return {"filename": file.filename}
```

Install python-magic:
```bash
pip install python-magic
```

## Step 3: Scan for Malware (ClamAV)

Your platform runs ClamAV container for virus scanning.

```python
import pyclamav

@app.post("/api/documents/upload")
async def upload_document(file: UploadFile = File(...)):
    contents = await file.read()
    
    # Scan with ClamAV
    clam = pyclamav.ClamAV('localhost')
    is_infected = clam.scan_stream(contents)
    
    if is_infected:
        raise HTTPException(status_code=400, detail="File contains malware")
    
    # Safe to proceed
    return {"filename": file.filename}
```

Or use minio-py with scan directive:

```python
from minio import Minio

client = Minio(
    "minio:9000",
    access_key="minioadmin",
    secret_key="minioadmin",
    secure=False
)

# Upload prompts MinIO-integrated ClamAV
client.fput_object(
    "uploads",
    object_name="document.pdf",
    file_path="/tmp/document.pdf"
)
```

## Step 4: Store with UUID Keys

Never use the original filename as the storage key.

```python
import uuid
from pathlib import Path

@app.post("/api/documents/upload")
async def upload_document(file: UploadFile = File(...)):
    # Generate UUID key
    storage_id = str(uuid.uuid4())
    original_extension = Path(file.filename).suffix
    storage_filename = f"{storage_id}{original_extension}"
    
    # Store in MinIO (not filesystem)
    client.put_object(
        "documents",
        storage_filename,
        file.file,
        length=len(contents)
    )
    
    # Save metadata to database
    doc = Document(
        id=uuid.uuid4(),
        storage_id=storage_filename,
        original_filename=file.filename,  # For display only
        mime_type=mime_type,
        size=len(contents),
        user_id=current_user.id
    )
    db.add(doc)
    db.commit()
    
    return {
        "id": doc.id,
        "filename": file.filename,
        "size": doc.size
    }
```

Original filename is stored in the database for display. Storage uses UUID, preventing path traversal.

## Step 5: Serve via Presigned URLs

Never send the file path directly. Use presigned URLs.

```python
@app.get("/api/documents/{document_id}/download")
async def download_document(document_id: str):
    doc = db.query(Document).filter_by(id=document_id).first()
    if not doc:
        raise HTTPException(status_code=404)
    
    # Check authorization
    if doc.user_id != current_user.id:
        raise HTTPException(status_code=403)
    
    # Generate presigned URL (valid for 1 hour)
    url = client.get_presigned_download_url(
        "documents",
        doc.storage_id,
        expires=timedelta(hours=1)
    )
    
    return {"download_url": url}
```

Presigned URL contains HMAC signature. MinIO verifies it before serving.

Attacker cannot guess URLs or request files they don't have access to.

## Step 6: Whitelist File Types

```python
ALLOWED_MIMES = {
    'application/pdf',
    'application/vnd.openxmlformats-officedocument.wordprocessingml.document',  # .docx
    'application/msword',  # .doc
    'text/plain'
}

ALLOWED_EXTENSIONS = {'.pdf', '.doc', '.docx', '.txt'}

@app.post("/api/documents/upload")
async def upload_document(file: UploadFile = File(...)):
    # Validate extension
    ext = Path(file.filename).suffix.lower()
    if ext not in ALLOWED_EXTENSIONS:
        raise HTTPException(status_code=400, detail="Invalid file type")
    
    contents = await file.read()
    
    # Validate magic bytes
    mime_type = magic.from_buffer(contents, mime=True)
    if mime_type not in ALLOWED_MIMES:
        raise HTTPException(status_code=400, detail="File type mismatch")
    
    # Rest of validation...
```

## Protection Against Zip Bombs

Zip bomb: a ZIP file that expands to terabytes (e.g., 42.zip is 42KB but expands to 4.3PB).

Unzipping blindly causes DoS.

**Solution**: Don't unzipautomatically. If you must:

```python
import zipfile

@app.post("/api/bulk-import")
async def bulk_import(file: UploadFile = File(...)):
    contents = await file.read()
    
    # Check for zip bomb
    max_extraction_ratio = 100  # 1KB can extract to 100KB max
    
    try:
        with zipfile.ZipFile(contents) as zf:
            for info in zf.filelist:
                # Ratio of compressed to uncompressed
                if info.compress_size == 0:
                    continue
                ratio = info.file_size / info.compress_size
                if ratio > max_extraction_ratio:
                    raise HTTPException(status_code=400, detail="Zip bomb detected")
    except zipfile.BadZipFile:
        raise HTTPException(status_code=400, detail="Invalid ZIP file")
```

Or simpler: just don't support ZIP uploads.

## Complete Secure Upload Handler

```python
import uuid
from pathlib import Path
import magic
from fastapi import UploadFile, HTTPException

ALLOWED_MIMES = {'application/pdf'}
ALLOWED_EXTENSIONS = {'.pdf'}
MAX_FILE_SIZE = 100 * 1024 * 1024  # 100MB

async def secure_upload(file: UploadFile, current_user):
    """Secure file upload with validation"""
    
    # Read file
    contents = await file.read()
    
    # 1. Validate size
    if len(contents) > MAX_FILE_SIZE:
        raise HTTPException(status_code=413, detail="File too large")
    
    # 2. Validate extension
    ext = Path(file.filename).suffix.lower()
    if ext not in ALLOWED_EXTENSIONS:
        raise HTTPException(status_code=400, detail="Invalid file type")
    
    # 3. Validate magic bytes
    mime = magic.from_buffer(contents, mime=True)
    if mime not in ALLOWED_MIMES:
        raise HTTPException(status_code=400, detail="File type mismatch")
    
    # 4. Scan for malware
    clam = pyclamav.ClamAV('clamav:3310')
    if clam.scan_stream(contents):
        raise HTTPException(status_code=400, detail="File contains malware")
    
    # 5. Store with UUID
    storage_id = str(uuid.uuid4()) + ext
    client.put_object(
        "documents",
        storage_id,
        contents,
        len(contents)
    )
    
    # 6. Save metadata
    doc = Document(
        id=uuid.uuid4(),
        storage_id=storage_id,
        original_filename=file.filename,
        mime_type=mime,
        size=len(contents),
        user_id=current_user.id
    )
    db.add(doc)
    db.commit()
    
    return {"id": doc.id}

@app.post("/api/documents/upload")
async def upload_document(file: UploadFile, current_user = Depends(get_current_user)):
    return await secure_upload(file, current_user)
```

## Hands-On Lab

### Lab 4.1: Validate Magic Bytes

```python
import magic

# Test with fake extension
with open("malware.pdf", "wb") as f:
    f.write(b"\x4d\x5a\x90\x00")  # MZ header (EXE)

mime = magic.Magic(mime=True)
detected = mime.from_file("malware.pdf")
print(detected)  # application/x-dosexec (not PDF!)

# Attacker's trick detected
```

### Lab 4.2: Test ClamAV

```bash
# In docker-compose
clamav:
  image: clamav/clamav:latest
  ports:
    - "3310:3310"

# Test from Python
import pyclamav
clam = pyclamav.ClamAV('localhost:3310')

# Scan a file
infected = clam.scan_file("/path/to/file")
print(infected)  # {'file': ('', 'Clean')} or ('', virus_name)
```

## Cheat Sheet: Secure File Upload

```python
# Size limit (Nginx)
client_max_body_size 100m;

# Validate magic bytes
import magic
mime = magic.from_buffer(contents, mime=True)
if mime not in ALLOWED_MIMES:
    raise HTTPException(status_code=400)

# ClamAV scan
clam = pyclamav.ClamAV('clamav:3310')
if clam.scan_stream(contents):
    raise HTTPException(status_code=400)

# Store with UUID
storage_id = str(uuid.uuid4()) + extension
client.put_object("bucket", storage_id, contents, len(contents))

# Serve with presigned URL
url = client.get_presigned_download_url("bucket", storage_id)
return {"download_url": url}
```

## Key Takeaways

- **File size**: Nginx `client_max_body_size`, FastAPI endpoint check
- **Magic bytes**: Validate actual file type, not extension
- **Malware**: ClamAV scanning
- **Storage**: UUID keys, never user-controlled names
- **Serving**: Presigned URLs, not direct paths
- **Whitelist**: Only allow specific MIME types
- **Zip bombs**: Check compression ratio or disallow ZIPs
- **Authorization**: Check user ownership before download

Module 5 is payment security, the most critical data protection.
