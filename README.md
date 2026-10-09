
# Offline Sync Engine — Conflict Resolution Backend

A backend service built with **Node.js, Express, TypeScript, and PostgreSQL** that keeps data synchronized across multiple devices, even when they work offline.

When different devices make changes to the same document, the system checks for conflicts before updating the data. Changes to different fields can be merged automatically, while conflicting edits are flagged for review. It also prevents outdated updates from overwriting newer data and maintains a history of document changes.

## API Documentation and Testing

Interactive API documentation is available through Swagger when the application is running locally or through Docker.

**URL:** `http://localhost:3000/docs`

## Architecture and Synchronization Logic

Each document stored in PostgreSQL has three main properties:

- **`version`**: Tracks the current revision of the document.
- **`data`**: Stores the document's current content in JSONB format.
- **`field_versions`**: Records the last version in which each field was updated. For example, `{"title": 2, "status": 1}` means the title was last updated in version 2, while the status was last updated in version 1.

### Conflict Resolution

When a device reconnects and sends offline changes through `POST /api/v1/sync`, it includes its `base_version` and the fields it wants to update.

The server handles the request based on the current document state.

| Condition | Server Action | HTTP Status |
|---|---|---|
| The client's version matches the server's version | Applies the changes and increments the document version. | `200 OK` |
| The client's version is older, but the changes affect different fields | Merges the changes automatically. | `200 OK` |
| The client's version is older and the same field has conflicting changes | Rejects the update and returns details of the conflict. | `409 Conflict` |
| The client's version is ahead of the server's version | Rejects the invalid request. | `400 Bad Request` |

## Handling Edge Cases

### 1. Simultaneous Updates

PostgreSQL row-level locks using `SELECT ... FOR UPDATE` prevent multiple requests from modifying the same document simultaneously. Database transactions help ensure that updates are processed safely and that incomplete changes are not committed.

### 2. Out-of-Order Updates

The system checks field versions before applying changes. This prevents older offline updates from overwriting newer changes that have already been saved on the server.

### 3. Change History and Recovery

Every accepted update creates a snapshot in `document_versions` and an audit entry in `sync_logs`. This makes it possible to review previous versions and restore an earlier state when needed.

## API Reference

### 1. Create a Document

**Endpoint:** `POST /api/v1/documents`

Creates a new document using the provided initial data.

**Request Body:**

```json
{
  "initial_data": {
    "title": "Project Specification",
    "author": "Alice",
    "status": "DRAFT"
  }
}
```

The server stores the document in PostgreSQL and initializes its version tracking for future synchronization requests.

### 2. Synchronize Offline Edits

**Endpoint:** `POST /api/v1/sync`

Sends changes made on a device while offline and checks whether they can be applied safely.

**Request Body:**

```json
{
  "document_id": "c71a3554-1a9e-4a6c-82df-6fb43dcd3e12",
  "device_id": "phone-device-01",
  "base_version": 1,
  "changes": {
    "status": "PUBLISHED"
  }
}
```

**Example Response: `409 Conflict`**

This response is returned when the same field has been modified differently on the server and the client.

```json
{
  "status": "CONFLICT",
  "message": "Conflicting modifications detected.",
  "conflicts": {
    "status": {
      "server_value": "ARCHIVED",
      "client_value": "PUBLISHED"
    }
  },
  "data": {
    "title": "Project Specification",
    "author": "Alice",
    "status": "ARCHIVED"
  },
  "new_version": 2
}
```

The response provides both values so the conflict can be reviewed instead of silently overwriting existing data.

### 3. View Version History

**Endpoint:** `GET /api/v1/documents/:id/versions`

Returns the available versions of a document so previous changes can be reviewed.

### 4. Restore a Previous Version

**Endpoint:** `POST /api/v1/documents/:id/restore/:version`

Restores a document to a selected version, allowing earlier data to be recovered when required.