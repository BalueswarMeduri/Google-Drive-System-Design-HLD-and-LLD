# Google Drive – System Design (HLD & LLD)

A system design of a Google Drive-like file storage system that allows users to upload, download, and manage files with secure access and permissions.

## Table of Contents

1. [Functional Requirements](#1-functional-requirements)
2. [Non-Functional Requirements](#2-non-functional-requirements)
3. [High-Level Design (HLD)](#3-high-level-design-hld)
4. [Upload Design Decision](#4-upload-design-decision)
5. [File Upload Flow](#5-file-upload-flow)
6. [Failure Handling](#6-failure-handling)
7. [Idempotency](#7-idempotency)
8. [Database Design](#8-database-design)

---

## 1. Functional Requirements

These describe what our Google Drive system should do.

- **Upload File:** Users should be able to upload files to the system.
- **Download File:** Users should be able to download their uploaded files.
- **File Access and Permissions:** Users should be able to manage file permissions and control who can access their files.

## 2. Non-Functional Requirements

These describe how well our system should work.

- **Scalability:** The system should support millions of users and large file uploads as traffic increases.
- **High Availability:** The service should remain available with minimal downtime.
- **Performance:** File uploads and downloads should happen efficiently with minimal latency.

## 3. High-Level Design (HLD)

The following diagram represents the overall architecture of our Google Drive file upload system.

![Google Drive HLD](https://github.com/user-attachments/assets/0796b2b3-0519-480e-97f7-f3c7e0b12ec6)

## 4. Upload Design Decision

**Question: Who should upload the file to S3 — the Client or the Server?**

We have two approaches:

### Approach 1: Server-side Upload

In this approach:

1. The client sends the entire file to our backend server.
2. The server receives and processes the file.
3. The server uploads the file to AWS S3.

**Disadvantage:**
- The file travels through our backend server before reaching S3.
- This increases backend bandwidth and server load.
- It adds an extra step to the upload process.

### Approach 2: Client-side Upload (Our Choice)

In this approach, the client uploads the file directly to AWS S3 using pre-signed URLs.

**Advantages:**
- The file does not need to pass through our backend servers.
- Reduces backend bandwidth and server load.
- Supports efficient large file uploads using multipart upload.
- Allows clients to upload multiple chunks independently.

**Design Decision:** We choose client-side upload because it reduces backend workload and allows files to be uploaded directly to S3.

However, the backend still manages the upload session, permissions, chunk configuration, and upload completion.

## 5. File Upload Flow

Let's understand how the system works when a user uploads a large video.

### Step 1: Initiate Upload

1. The client sends an upload request to the API Gateway.
2. The request includes an Idempotency Key to prevent duplicate upload sessions.
3. The API Gateway forwards the request to the Upload Service.

### Step 2: Permission and Storage Checks

1. The Upload Service calls the Permission Service.
2. The Permission Service checks whether the user has permission to upload files.
3. It also checks whether the user has sufficient storage quota.
4. If the checks pass, the upload process continues.

### Step 3: Create Upload Session

1. The Upload Service generates a unique Session ID.
2. It sends a Multipart Upload initiation request to AWS S3.
3. S3 returns a unique Upload ID.
4. The Upload Service stores the Session ID and Upload ID mapping in the Metadata DB.

### Step 4: Decide Chunk Size

The Upload Service decides:

- Total number of chunks.
- Size of each chunk.

We use multipart upload because sending a very large file in a single request can be slow and difficult to recover if the connection fails.

Dividing the file into smaller chunks allows us to upload parts independently and retry failed parts.

### Step 5: Generate Pre-signed URLs

1. The Upload Service generates pre-signed URLs for the file chunks.
2. These URLs allow the client to upload chunks directly to S3 without exposing AWS credentials.
3. The Upload Service sends the Session ID and pre-signed URLs to the client.

### Step 6: Client Uploads Chunks

1. The client divides the file into chunks based on the configured chunk size.
2. It uploads each chunk directly to S3 using its corresponding pre-signed URL.
3. S3 returns an ETag for each successfully uploaded part.
4. The client collects the part numbers and their corresponding ETags.

### Step 7: Verify and Complete Upload

1. After uploading all chunks, the client sends the Session ID, part numbers, and ETags to the Upload Service.
2. The Upload Service uses the stored Upload ID to identify the multipart upload.
3. It verifies the uploaded parts against S3, using the ListParts API where required.
4. If everything is valid, the Upload Service calls the CompleteMultipartUpload API.
5. S3 combines all uploaded parts into the final file.
6. The Metadata DB is updated with the file's completed status.

## 6. Failure Handling

We need to handle failures during large file uploads.

**Scenario: What happens if some chunks fail to upload?**

Imagine a user is uploading a 100 MB video divided into 10 chunks.

Suppose 7 chunks are successfully uploaded, but the internet connection fails while uploading the remaining 3 chunks.

We should not ask the user to upload the entire file again.

Instead, we use the existing upload session to resume the upload.

### How does it work?

1. The client reconnects and sends the existing Session ID to the Upload Service.
2. The Upload Service retrieves the corresponding Upload ID from the Metadata DB.
3. It calls the S3 ListParts API to identify which chunks have already been uploaded.
4. It compares the uploaded parts with the expected parts.
5. The Upload Service generates new pre-signed URLs for the missing chunks.
6. These URLs are returned to the client.
7. The client uploads only the missing chunks.

This allows us to resume failed uploads without uploading the entire file again.

## 7. Idempotency

An Idempotency Key is used to prevent duplicate operations when clients retry the same request.

**Example:**

1. The client sends an upload request with an Idempotency Key.
2. The Upload Service creates an upload session.
3. Due to a network issue, the client does not receive the response.
4. The client retries the same request with the same Idempotency Key.
5. The Upload Service checks whether that key was already used for the same user and operation.
6. If an upload session already exists, it returns the existing session details instead of creating another upload session.

This prevents duplicate upload sessions when the same request is retried.

## 8. Database Design

We use a NoSQL database for storing upload metadata and session information.

The actual file content is stored in AWS S3, while the Metadata DB maintains the information required to manage and retrieve the file.

### Upload Metadata

| Field | Description |
|---|---|
| session_id | Unique application-level upload session ID |
| upload_id | S3 Multipart Upload ID |
| user_id | ID of the user uploading the file |
| file_name | Original file name |
| file_size | Total size of the file |
| chunk_size | Size of each chunk |
| total_chunks | Total number of chunks |
| status | INITIATED / UPLOADING / COMPLETED / FAILED |
| created_at | Upload creation timestamp |

### Important Notes

- **Session ID:** Generated by our application to identify an upload session.
- **Upload ID:** Generated by AWS S3 to identify a Multipart Upload.
- **ETag:** Returned by S3 for each successfully uploaded part and used during upload completion.
- **Pre-signed URL:** Allows the client to upload file chunks directly to S3 without exposing AWS credentials.
- **Metadata DB:** Stores the mapping between Session ID and S3 Upload ID, along with upload details.
- **AWS S3:** Stores the actual file content.

---
