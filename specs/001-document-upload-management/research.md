# Phase 0 Research: Document Upload and Management

## Decision 1: ASP.NET/Blazor implementation shape

- **Decision:** Implement the feature in the existing Blazor Server application using Razor pages/components, scoped application services, EF Core entities, and existing cookie claims.
- **Rationale:** This is the repository's current architecture and matches offline training requirements. Authorization can be enforced consistently at the service and data boundaries.
- **Alternatives considered:** New separate API/SPA project rejected as unnecessary complexity; direct file access from pages rejected because Blazor components must not own persistence or authorization rules.

## Decision 2: Local file storage abstraction

- **Decision:** Add `IFileStorageService` with local and future cloud implementations behind dependency injection. Store files outside `wwwroot`, under a relative `AppData/uploads/{userId}/{projectId-or-personal}/{guid}.{safe-extension}` path.
- **Rationale:** Relative GUID-based paths prevent traversal and duplicate-name collisions, while DI preserves the Azure migration path.
- **Alternatives considered:** Storing files in `wwwroot` rejected because it bypasses authorization; storing original filenames rejected as unsafe; storing files as database binaries rejected as inconsistent with the stakeholder file-storage pattern.

## Decision 3: Upload transaction and partial-failure behavior

- **Decision:** Validate metadata and file format, generate a relative path, write the file, create the document record in `Pending` scan state, and enqueue an Azure Functions scan message. Only a `Clean` scan result exposes the document. On database failure after storage, the service attempts to delete the newly stored file. Queue failure marks the document `Error` and quarantines/removes its file. Replacement stores a new file and enqueues another scan; the old file remains available until replacement metadata and database state are committed and the new scan is clean.
- **Rationale:** This ordering prevents an uncreated or inaccessible record and limits orphaning. It also preserves the existing record and metadata during replacement.
- **Alternatives considered:** Save database first rejected because a storage failure would create inaccessible records; delete an old file before database update rejected because a failed replacement could destroy usable content.

## Decision 4: Malware scanning boundary

- **Decision:** Introduce `IFileMalwareScanner` and use an Azure Functions Queue Storage trigger for asynchronous file scanning after upload. The web application stores the file and document metadata in a pending-scan state, then enqueues a scan job containing the document ID and relative storage path. The function opens the file through a shared storage abstraction, calls the scanner, updates the document scan status and audit/activity records, and publishes a notification when available. Azure Functions retries queue messages for transient failures and moves repeatedly failing messages to a poison queue.
- **Rationale:** Queue-triggered processing removes scanner latency and heavy work from the upload request. Azure Functions and Queue Storage provide retry/poison-queue behavior while the storage abstraction preserves Local/Azurite and future Azure Blob implementations.
- **Alternatives considered:** Synchronous pre-storage scanning rejected because it blocks the upload request and couples web availability to scanner availability. Browser-side scanning rejected because it is not trustworthy. Direct HTTP calls from the web app to a scanner rejected because they lack durable retries and queue isolation.

## Decision 4a: Queue contract and failure states

- **Decision:** Use a JSON queue message:
  `DocumentId`, `StoragePath`, `ContentType`, `UploadDateUtc`, `Attempt`/trace identifier, and `CorrelationId`.
  The document `ScanStatus` is `Pending`, `Scanning`, `Clean`, `Threat`, or `Error`. Upload success creates metadata and returns the file as pending; it is not downloadable or previewable until `Clean`. Queue send failure marks the document `Error`, records the failure, and removes or quarantines the newly stored file. Threat results quarantine/delete the uploaded file and mark the document unavailable. Repeated transient `Error` results allow configured Azure Functions retries; terminal queue failure marks `Error` and moves work to the poison queue.
- **Rationale:** Explicit states prevent pending or scanner-failed files from being exposed. Durable queue retries handle transient scanner/outage failures, while the poison queue requires administrator review rather than silently becoming available.
- **Alternatives considered:** Returning success and making files immediately downloadable rejected because it does not establish a clean scan result; infinite retries rejected because they can repeatedly process the same invalid or unavailable file.

## Decision 5: Preview and download authorization

- **Decision:** Add authenticated Razor/HTTP endpoints for preview and download that resolve the document, check owner/project/share authorization, and stream only through the storage service.
- **Rationale:** Files outside `wwwroot` require server-mediated access. PDF and JPEG/PNG preview use an inline `Content-Disposition`; all other supported types use attachment/download.
- **Alternatives considered:** Direct local-file links rejected; signed public URLs rejected for the training deployment and shared authorization model.

## Decision 6: Authorization model

- **Decision:** Centralize document authorization in `IDocumentService`: upload is allowed for authenticated users, personal files belong to the owner, project uploads require project membership and a Project Manager is required for project-owned uploads, access requires owner, project manager/member, or explicit share, metadata/replacement/deletion require owner, and sharing requires owner. Administrator audit access is separate.
- **Rationale:** This matches the clarified requirement that only owners can manage or reshare, and satisfies the constitution that authorization exists at the service boundary.
- **Alternatives considered:** UI-only authorization rejected; stakeholder-era Project Manager deletion language is superseded by the clarification and FR-021/FR-023 requiring owner-only mutation.

## Decision 7: Data model

- **Decision:** Use integer keys for `Document`, `DocumentShare`, `DocumentTask`, and `DocumentActivity`; use text category and tag storage that supports case-preserving tags without uniqueness. Associate `Document.ProjectId` and `DocumentTask.TaskId` with existing entities.
- **Rationale:** This follows existing identity conventions and keeps category values readable and migrateable.
- **Alternatives considered:** Category enum/integer and GUID document keys rejected by requirements; a separate normalized tags table is deferred because custom tags are not separately queried in the initial release.

## Decision 8: Search and performance

- **Decision:** Build authorized EF query roots first, then filter title, description, tag text, uploader display name, project name, category, project, and date range. Add EF indexes for uploader, project, category, upload date, and share recipient. Paginate/list operations and use AsNoTracking for read models.
- **Rationale:** Filtering authorization into the query root prevents cross-tenant data exposure and supports expected list/search sizes. Indexes cover likely sorting/filtering keys.
- **Alternatives considered:** Loading all documents and filtering in memory rejected for unauthorized-result risk and poor scale; full-text indexing rejected as unnecessary for LocalDB/training scope.

## Decision 9: Notifications and activity

- **Decision:** Reuse `Notification` for recipient/project notifications and add document actions to `DocumentActivity` with integer user/document keys and UTC timestamps. Extend `NotificationType` only if needed for document share/project events; alternatively use existing notification types plus document-specific text.
- **Rationale:** Reuse minimizes changes and preserves the existing notification mechanism. A dedicated activity entity records attributable operations independently of notifications.
- **Alternatives considered:** Activity records stored in notification rows rejected because notification and audit lifecycles differ.

## Decision 10: Testing

- **Decision:** Add service/unit tests, EF integration tests against SQL LocalDB or an isolated development database, contract tests for storage/scanner interfaces, and Blazor component/manual validation. Test policy/order enforcement, mixed valid/invalid batch uploads, orphan cleanup, replacement rollback, and deny-by-default search/download.
- **Rationale:** This is the narrowest validation that covers authorization, storage, and EF behavior.
- **Alternatives considered:** UI-only tests rejected because service boundary and storage rollback are security-critical.
