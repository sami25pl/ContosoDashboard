# Data Model

## Document

Represents metadata and storage identity for an uploaded file.

| Field | Type | Constraints |
|---|---|---|
| DocumentId | int | Primary key, integer identity |
| Title | string | Required, 1-255 characters |
| Description | string? | Optional, maximum 2000 characters |
| Category | string | Required; predefined values: Project Documents, Team Resources, Personal Files, Reports, Presentations, Other |
| ProjectId | int? | Optional FK to Project |
| Tags | string | Optional normalized, delimiter or JSON representation; 2000 characters maximum |
| UploadedByUserId | int | Required FK to User, immutable owner |
| UploadDateUtc | DateTime | Required, UTC |
| FileSizeBytes | long | Required, greater than 0 and at most 25 MB |
| MimeType | string | Required, maximum 255 characters |
| OriginalFileName | string | Required metadata only, maximum 255 characters; never used as storage path |
| FilePath | string | Required unique relative storage path, maximum 1024 characters |
| Status | string/int enum | Available only after storage and metadata succeed |
| ScanStatus | string enum | Pending, Scanning, Clean, Threat, or Error |
| ScanCompletedAtUtc | DateTime? | UTC completion time for Clean, Threat, or terminal Error |
| ScanResultDetails | string? | Optional non-sensitive scanner result summary, maximum 2000 characters |

Relationships:
- belongs to one uploader/owner (`User`)
- optionally belongs to one `Project`
- has many `DocumentShare` records
- has many `DocumentTask` records
- has many `DocumentActivity` records

## DocumentShare

Represents explicit owner sharing with a user or team.

| Field | Type | Constraints |
|---|---|---|
| DocumentShareId | int | Primary key |
| DocumentId | int | Required FK to Document |
| RecipientUserId | int? | Nullable FK to User for a direct share |
| RecipientTeamId | string/int? | Team is department-based in this model; keep nullable and resolve by department, or use a future team key |
| SharedByUserId | int | Required FK to User; must equal owner at creation |
| SharedAtUtc | DateTime | Required UTC |

Relationships:
- belongs to one `Document`
- optionally maps to one direct recipient

Validation:
- owner must exist and equal `Document.UploadedByUserId`
- at least one recipient user or team is required
- duplicate active shares are prevented by a document/user unique index
- recipients may view/download only; mutation remains owner-only

Team resolution should use the existing Department claim context or a future department/team entity. For the initial training implementation, a team share may resolve member users by department name and create recipient notifications/access entries.

## DocumentTask

Join entity for task/document association.

| Field | Type | Constraints |
|---|---|---|
| DocumentTaskId | int | Primary key |
| DocumentId | int | Required FK |
| TaskId | int | Required FK |
| AddedByUserId | int | Required FK |
| AssociatedAtUtc | DateTime | Required UTC |

Validation:
- user must be authorized to view/use the document
- user must be authorized to view the task
- task project is copied/normalized onto the document association if the document has no conflicting project
- document project must match task project where a project is set

## DocumentActivity

Attributable audit record.

| Field | Type | Constraints |
|---|---|---|
| DocumentActivityId | int | Primary key |
| DocumentId | int? | Nullable because deletion/failed operations may retain an audit identity reference |
| ActorUserId | int | Required FK to User |
| Action | string | Upload, Download, Preview, Delete, Share, Replace, MetadataUpdate |
| OccurredAtUtc | DateTime | Required UTC |
| Details | string? | Optional non-sensitive details, maximum 2000 characters |

No content, original path, or secret is stored in activity details.

## Existing Entity Adaptations

- `Project`: optional one-to-many documents. A document project association does not change Project member authorization.
- `TaskItem`: optional many-to-many documents through `DocumentTask`.
- `User`: uploader/owner, direct-share recipient, activity actor, and task/document association actor.
- `Notification`: reuse for share and project-document events. Document content is not embedded in notification records.

## State and Lifecycle

### Upload

1. Validate metadata and file.
2. Validate caller authorization.
3. Generate unique relative path.
4. Save file.
5. Create `Document` with `ScanStatus = Pending`.
6. Enqueue a `DocumentScanQueueMessage` for the Azure Functions scan worker.
7. Record upload activity.
8. Notify project members only after the document becomes `Clean`, if project-associated.

If metadata creation fails after storage, delete the stored file and return an error.
If queue enqueue fails, mark `ScanStatus = Error`, record the failure, and remove/quarantine the stored file. The record can remain for an auditable terminal result but must not expose the file.

### Virus Scan Processing

1. Queue trigger receives a JSON message with `DocumentId`, `StoragePath`, `ContentType`, and `CorrelationId`.
2. Azure Functions validates the document and storage path, then sets `ScanStatus = Scanning`.
3. The worker opens the file through the storage abstraction and invokes the malware scanner.
4. `Clean` changes the status to `Clean`, records completion, creates scan activity, and makes download/preview eligible.
5. `Threat` changes the status to `Threat`, quarantines or deletes the file, records scan details, and denies access.
6. `Error` keeps or restores the status to `Error`, records the failure, denies access, and allows Azure Functions retry for transient failures.
7. Repeatedly failing poison messages trigger terminal error handling and are retained for administrator review.

### Replacement

1. Verify owner.
2. Validate replacement metadata and file format.
3. Generate and save a new path.
4. Create or update metadata with `ScanStatus = Pending`.
5. Enqueue a replacement scan job.
6. Keep the old file and metadata usable until the replacement scan becomes `Clean`.
7. Delete the old file after successful database commit and clean replacement scan.
8. Record replacement activity.

If database update fails, delete the new file and preserve old metadata/path.

### Deletion

1. Verify owner.
2. Confirmed operation arrives at service.
3. Delete database relationships/document record.
4. Delete physical file.
5. Record deletion result.

The plan requires either successful permanent removal or a clearly reported partial failure. A database record must not remain with an unusable file without an explicit failure status.

## EF Indexing

- `Document(Document.UploadedByUserId)`
- `Document(Document.ProjectId)`
- `Document(Document.Category)`
- `Document(Document.UploadDateUtc)`
- `Document(Title)` and relevant search fields through query filters
- unique `Document(FilePath)`
- unique `(DocumentId, RecipientUserId)` on `DocumentShare`
- `(DocumentId, TaskId)` and `(TaskId, DocumentId)` on `DocumentTask`
- `(DocumentId, Action, OccurredAtUtc)` and `(ActorUserId, OccurredAtUtc)` on `DocumentActivity`
