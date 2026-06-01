# Dify File Upload Guide

This guide explains how Dify file upload works in this project, especially for file inputs such as proposal documents.

## Short Answer

Dify does not expect the browser to send the raw file and the user text in one single chat request.

The standard Dify API flow is split into two steps:

1. Upload the raw file to Dify with `POST /files/upload`.
2. Send the chat message or workflow request with the returned file id as `upload_file_id`.

So the chat request contains file references, not the binary file itself.

## Why It Is Split

Dify stores uploaded files under the current end-user first. After upload, Dify returns a file `id`. Later chat or workflow requests only reference that file id.

This design lets Dify:

- validate file type and size before chat execution
- bind uploaded files to the same `user`
- reuse the uploaded file id inside `inputs` or top-level `files`
- keep the chat endpoint JSON-based instead of multipart-based

The important rule is: the `user` used during file upload must match the `user` used during the chat or workflow request. Dify scopes uploaded files by user.

## API Flow

### 1. Upload The File

Client sends `multipart/form-data`:

```http
POST /files/upload
Authorization: Bearer <DIFY_APP_KEY>
Content-Type: multipart/form-data
```

Form fields:

```text
file=<binary file>
user=<stable user id>
```

Dify returns:

```json
{
  "id": "a1b2c3d4-5678-90ab-cdef-1234567890ab",
  "name": "proposal.docx",
  "size": 204800,
  "extension": "docx",
  "mime_type": "application/vnd.openxmlformats-officedocument.wordprocessingml.document"
}
```

The key field is `id`. This becomes `upload_file_id` in the next request.

Reference: [Dify Upload File API](https://docs.dify.ai/api-reference/files/upload-file)

### 2. Send Chat Message With File Reference

For a normal chat attachment, send JSON to `/chat-messages`:

```json
{
  "inputs": {},
  "query": "Please review this proposal",
  "response_mode": "streaming",
  "conversation_id": "",
  "user": "user_abc",
  "files": [
    {
      "type": "document",
      "transfer_method": "local_file",
      "upload_file_id": "a1b2c3d4-5678-90ab-cdef-1234567890ab"
    }
  ]
}
```

Reference: [Dify Send Chat Message API](https://docs.dify.ai/api-reference/chatflows/send-chat-message)

## File Input Variables

If the Dify app has a prompt variable or workflow input of type `file`, the file reference is usually placed inside `inputs`.

Example single file input:

```json
{
  "inputs": {
    "proposal_file": {
      "type": "document",
      "transfer_method": "local_file",
      "upload_file_id": "a1b2c3d4-5678-90ab-cdef-1234567890ab"
    }
  },
  "query": "review",
  "response_mode": "streaming",
  "conversation_id": "",
  "user": "user_abc"
}
```

Example file-list input:

```json
{
  "inputs": {
    "proposal_files": [
      {
        "type": "document",
        "transfer_method": "local_file",
        "upload_file_id": "a1b2c3d4-5678-90ab-cdef-1234567890ab"
      },
      {
        "type": "document",
        "transfer_method": "local_file",
        "upload_file_id": "b2c3d4e5-6789-0abc-def1-234567890abc"
      }
    ]
  },
  "query": "review",
  "response_mode": "streaming",
  "conversation_id": "",
  "user": "user_abc"
}
```

The input key, for example `proposal_file`, must match the variable name configured in Dify.

## Remote URL Files

Dify also supports remote files by URL. In that case there is no upload step.

```json
{
  "type": "document",
  "transfer_method": "remote_url",
  "url": "https://example.com/proposal.docx"
}
```

For local files, use:

```json
{
  "type": "document",
  "transfer_method": "local_file",
  "upload_file_id": "<id returned by /files/upload>"
}
```

## How This Project Proxies The Flow

This project does not expose the Dify API key to the browser. The browser talks to local Next.js API routes:

```text
Browser
  -> POST /api/file-upload
  -> Next.js route app/api/file-upload/route.ts
  -> Dify POST /files/upload
```

Then:

```text
Browser
  -> POST /api/chat-messages
  -> Next.js route app/api/chat-messages/route.ts
  -> Dify POST /chat-messages
```

The local backend adds the Dify `user` value on the server side. In this project it is derived from the logged-in user, selected agent app id, and session id.

## Common Mistakes

### Sending JSON Content-Type During Upload

File upload must be `multipart/form-data`. When using `FormData` in the browser, do not manually set `Content-Type`; the browser must set the boundary automatically.

Wrong:

```http
Content-Type: application/json
```

Correct:

```text
Let the browser set Content-Type for FormData.
```

### Using The Frontend Temporary File Id

The UI may create a temporary id for rendering upload progress. Dify does not know this id.

Wrong:

```json
{
  "upload_file_id": "frontend-temp-uuid"
}
```

Correct:

```json
{
  "upload_file_id": "id-returned-by-dify-files-upload"
}
```

### Hard-Coding The File Type As Image

A `.docx` proposal should be sent as `document`, not `image`.

Wrong:

```json
{
  "type": "image"
}
```

Correct:

```json
{
  "type": "document"
}
```

## End-To-End Example

User uploads `proposal.docx`, then sends `review`.

Step 1: upload file.

```text
POST /api/file-upload
FormData:
  file = proposal.docx
```

Local backend adds:

```text
user = user_admin_<app_id>:<session_id>
```

Dify returns:

```json
{
  "id": "file-id-from-dify"
}
```

Step 2: send chat message.

```json
{
  "inputs": {
    "proposal": {
      "type": "document",
      "transfer_method": "local_file",
      "upload_file_id": "file-id-from-dify"
    }
  },
  "query": "review",
  "response_mode": "streaming",
  "conversation_id": null
}
```

The project backend adds the same `user`, then forwards the request to Dify.

## Summary

Dify file upload is a two-step process:

1. Upload binary file first.
2. Send text and file reference second.

The chat request should not contain the raw file binary. It should contain `upload_file_id`, `transfer_method`, and `type`.
