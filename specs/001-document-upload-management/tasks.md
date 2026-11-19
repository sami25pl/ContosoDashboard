# Tasks: Document Upload and Management

**Input**: Design documents from `specs/001-document-upload-management/`  
**Prerequisites**: `plan.md`, `spec.md`, `data-model.md`, `contracts/document-service.md`, `research.md`, `quickstart.md`

**Tests**: No separate automated test tasks are included: the feature specification does not explicitly request test implementation or a TDD workflow. Each story below has an independent validation criterion, and implementation should follow the existing build/run guidance in `quickstart.md`.

**Organization**: Tasks are grouped by user story and ordered by dependencies. The Azure Functions Queue Storage-triggered scan is asynchronous; a document is inaccessible until its scan is clean.

## Phase 1: Setup

**Purpose**: Establish the separate scan-worker project and local configuration examples without requiring cloud credentials.

- [ ] T001 Create the Azure Functions isolated-worker project scaffold, queue-trigger dependencies, and startup files in `ContosoDashboard.Functions/ContosoDashboard.Functions.csproj`, `ContosoDashboard.Functions/Program.cs`, and `ContosoDashboard.Functions/host.json`.
- [ ] T002 [P] Add safe local configuration examples for the web app and scan worker, including the `document-scan` queue and Azurite/local storage settings, in `ContosoDashboard/appsettings.Development.json` and `ContosoDashboard.Functions/local.settings.example.json`.

## Phase 2: Foundational

**Purpose**: Create the shared persistence, storage, and authorization boundaries required by every story.

- [ ] T003 [P] Define the `Document`, `DocumentShare`, `DocumentTask`, and `DocumentActivity` entities in `ContosoDashboard/Models/Document.cs`, `ContosoDashboard/Models/DocumentShare.cs`, `ContosoDashboard/Models/DocumentTask.cs`, and `ContosoDashboard/Models/DocumentActivity.cs`; implement every field and constraint from `specs/001-document-upload-management/data-model.md`, explicitly including `Document.DocumentId`, `Document.Title`, `Document.Description`, `Document.Category`, `Document.ProjectId`, `Document.Tags`, `Document.UploadedByUserId`, `Document.UploadDateUtc`, `Document.FileSizeBytes`, `Document.MimeType`, `Document.OriginalFileName`, `Document.FilePath`, `Document.Status`, `Document.ScanStatus`, `Document.ScanCompletedAtUtc`, `Document.ScanResultDetails`, `DocumentShare.DocumentShareId`, `DocumentShare.DocumentId`, `DocumentShare.RecipientUserId`, `DocumentShare.RecipientTeamId`, `DocumentShare.SharedByUserId`, `DocumentShare.SharedAtUtc`, `DocumentTask.DocumentTaskId`, `DocumentTask.DocumentId`, `DocumentTask.TaskId`, `DocumentTask.AddedByUserId`, `DocumentTask.AssociatedAtUtc`, `DocumentActivity.DocumentActivityId`, `DocumentActivity.DocumentId`, `DocumentActivity.ActorUserId`, `DocumentActivity.Action`, `DocumentActivity.OccurredAtUtc`, and `DocumentActivity.Details`. Keep identity/user/project keys integer and category text as required by `spec.md` FR-036.
- [ ] T004 Add document DbSets, foreign-key/delete behavior, and the specified lookup/unique indexes in `ContosoDashboard/Data/ApplicationDbContext.cs`, following `specs/001-document-upload-management/data-model.md` relationships and indexes.
- [ ] T005 [P] Define the safe relative-path storage contract and local filesystem implementation in `ContosoDashboard/Services/IFileStorageService.cs` and `ContosoDashboard/Services/LocalFileStorageService.cs`; store outside `wwwroot` under `ContosoDashboard/AppData/uploads/`, reject absolute/traversal paths, and support upload, open, existence, and delete operations.
- [ ] T006 Define the shared document authorization policy/service in `ContosoDashboard/Services/IDocumentAuthorizationService.cs` and `ContosoDashboard/Services/DocumentAuthorizationService.cs`; enforce owner, project membership/manager, and explicit user/department-share access at the service boundary, with owner-only mutation and separate administrator-report authorization.
- [ ] T007 Define scan-state transition operations and fail-closed persistence behavior in `ContosoDashboard/Services/IScanStatusUpdateService.cs` and `ContosoDashboard/Services/ScanStatusUpdateService.cs`; allow trusted worker updates for Pending/Scanning/Clean/Threat/Error, record completion timestamps and activity, and never permit non-Clean files to be served.
- [ ] T008 Register the document persistence, storage, authorization, and scan-status services and configuration in `ContosoDashboard/Program.cs`.

## Phase 3: User Story 1 - Upload and Organize Documents (Priority: P1) 🎯 MVP

**Goal**: Accept valid single or batch uploads, securely store them, create pending records, and expose files only after asynchronous clean scans.

**Independent Test**: As an authenticated employee, upload a supported PDF or Office file up to 25 MB with title/category and confirm progress, secure storage, and My Documents visibility after a clean scan; submit unsupported and oversized files and confirm per-file rejection without stored files or records.

### Implementation for User Story 1

- [ ] T009 [P] [US1] Implement per-file upload validation for the exact extension/MIME allowlist, 1–25 MB limit, required title/category, and predefined text category values in `ContosoDashboard/Services/DocumentUploadValidator.cs`.
- [ ] T010 [P] [US1] Define the queue message and web-app enqueue abstraction, then implement an Azure Queue Storage producer configured for `document-scan` and Azurite-compatible connections in `ContosoDashboard/Models/DocumentScanQueueMessage.cs`, `ContosoDashboard/Services/IDocumentScanQueue.cs`, and `ContosoDashboard/Services/AzureDocumentScanQueue.cs`; serialize document ID, generated storage path, content type, correlation ID, and upload time.
- [ ] T011 [US1] Implement the Queue Storage-triggered Azure Function in `ContosoDashboard.Functions/Functions/DocumentScanFunction.cs` and register its storage/scanner dependencies in `ContosoDashboard.Functions/Program.cs`; validate the message and relative path, transition Pending→Scanning→Clean/Threat/Error, use the shared storage and scanner abstractions, retry transient failures, and keep poison-message outcomes inaccessible. On Clean, record scan completion/activity and send the project-member notification only after the clean result; on Threat/Error, quarantine or remove the file and record a safe result summary.
- [ ] T012 [US1] Implement authenticated single/batch upload operations in `ContosoDashboard/Services/IDocumentService.cs` and `ContosoDashboard/Services/DocumentService.cs`; authorize project uploads, store each valid file at a generated unique relative path, create metadata in Pending scan state, enqueue its scan message, return independent per-file outcomes, record upload activity, and clean up or mark Error on storage, database, or enqueue failure without exposing the file.
- [ ] T013 [US1] Build the upload page with multi-file selection, required metadata, optional description/project/tags, visible per-file progress, and success/error feedback in `ContosoDashboard/Pages/DocumentUpload.razor`.
- [ ] T014 [US1] Add the authenticated initial My Documents view showing title, category, upload date, size, and project, and hide pending/failed/threat documents from available results in `ContosoDashboard/Pages/Documents.razor`.

## Phase 4: User Story 2 - Browse, Search, and Retrieve Documents (Priority: P1)

**Goal**: Provide authorized document discovery, sorting/filtering/search, project views, previews, and downloads.

**Independent Test**: With authorized documents in multiple categories/projects, sort and filter by the specified fields, search metadata/uploader/project names, preview a PDF or image, and download an authorized file; verify that inaccessible project/shared records never appear or stream.

### Implementation for User Story 2

- [ ] T015 [US2] Implement paginated authorized browse/search queries and sorting by title/date/category/size, filtering by category/project/date range, and search across title/description/tags/uploader/project in `ContosoDashboard/Services/DocumentService.cs`; apply authorization in the query root before filtering or paging and include only Clean documents.
- [ ] T016 [P] [US2] Extend the My Documents UI with search, sort, filter, paging, empty/loading states, and matching-results feedback in `ContosoDashboard/Pages/Documents.razor`.
- [ ] T017 [P] [US2] Add the authorized project-document view for project members and managers in `ContosoDashboard/Pages/ProjectDocuments.razor`, using the document service’s project-scoped query rather than loading all documents.
- [ ] T018 [US2] Implement authenticated `/documents/{documentId}/download` and `/documents/{documentId}/preview` handlers in `ContosoDashboard/Pages/DocumentFileEndpoints.cs` and map them in `ContosoDashboard/Program.cs`; recheck service authorization and Clean scan status before opening storage, allow inline preview only for PDF/JPEG/PNG, use safe response metadata, return consistent not-found/access-denied outcomes, and record successful Preview/Download activity.

## Phase 5: User Story 3 - Manage Metadata and Document Versions (Priority: P2)

**Goal**: Allow only owners to edit metadata, stage a replacement without losing the current clean file, and permanently delete after confirmation.

**Independent Test**: As the owner, edit metadata, replace a file and verify the prior clean file remains usable until the replacement scan is clean, then confirm deletion; as a project manager or recipient who is not the owner, attempt every mutation and confirm no change.

### Implementation for User Story 3

- [ ] T019 [US3] Implement owner-only metadata updates with required title/category, authorized project reassignment, validation, concurrency handling, and MetadataUpdate activity in `ContosoDashboard/Services/DocumentService.cs`.
- [ ] T020 [US3] Add durable staged-replacement state to the document persistence model and EF configuration in `ContosoDashboard/Models/Document.cs` and `ContosoDashboard/Data/ApplicationDbContext.cs`, so the active clean file path/status remains distinguishable from a pending replacement path/status until its queue scan completes.
- [ ] T021 [US3] Implement owner-only replacement in `ContosoDashboard/Services/DocumentService.cs`; validate and store a new generated path, enqueue the same asynchronous scan contract, preserve the old file and metadata on failure, atomically activate the new path only after a Clean result, then delete the old file and record Replace activity.
- [ ] T022 [US3] Implement confirmed owner-only permanent deletion, relationship/file cleanup, and clearly reported partial failures in `ContosoDashboard/Services/DocumentService.cs`; record Delete activity without retaining file content or sensitive paths.
- [ ] T023 [US3] Add owner metadata editing, replacement status/progress, and a confirmation step for deletion in `ContosoDashboard/Pages/DocumentDetails.razor`; do not expose mutation controls as a substitute for service authorization.

## Phase 6: User Story 4 - Share Documents and Integrate with Tasks (Priority: P2)

**Goal**: Let owners share with named users or departments and notify recipients; let authorized task users attach existing or newly uploaded documents.

**Independent Test**: Share a clean document with a user and department and verify in-app notifications and Shared with Me visibility while recipient mutations are denied; attach an existing authorized document and upload a new one from an authorized task, then verify the task/project association.

### Implementation for User Story 4

- [ ] T024 [US4] Implement owner-only user/department sharing, deterministic department-member resolution, duplicate-share prevention, and Share activity in `ContosoDashboard/Services/DocumentService.cs`; create in-app notifications through the existing notification service without embedding document content.
- [ ] T025 [US4] Add owner sharing controls and an authorized Shared with Me view that permits recipients to view/download but not reshare, edit, replace, or delete in `ContosoDashboard/Pages/DocumentDetails.razor` and `ContosoDashboard/Pages/SharedDocuments.razor`.
- [ ] T026 [US4] Implement task-document association and task-context upload in `ContosoDashboard/Services/DocumentService.cs` and `ContosoDashboard/Pages/Tasks.razor`; require authorization for both task and document, enforce matching or unset project association, normalize the task project onto the document, and persist the `DocumentTask` relationship.

## Phase 7: User Story 5 - Dashboard Visibility and Audit Reporting (Priority: P3)

**Goal**: Show the employee’s five latest documents and total count, and provide administrator reports from attributable document activity.

**Independent Test**: Upload/download/share/delete documents, confirm the dashboard shows the five latest owned documents and count, then request administrator reports for document types, active uploaders, and access patterns; verify every completed action is attributed to actor and UTC time.

### Implementation for User Story 5

- [ ] T027 [P] [US5] Extend the dashboard service read model and dashboard page with the current user’s five newest Clean documents and document count in `ContosoDashboard/Services/DashboardService.cs` and `ContosoDashboard/Pages/Index.razor`.
- [ ] T028 [US5] Implement administrator-authorized aggregation for document types, active uploaders, and access patterns from `DocumentActivity` in `ContosoDashboard/Services/DocumentReportService.cs`; scope each report to attributable records and avoid disclosing document contents or storage paths.
- [ ] T029 [US5] Add the protected administrator document-activity report page and navigation entry in `ContosoDashboard/Pages/DocumentReports.razor` and `ContosoDashboard/Shared/NavMenu.razor`.

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: Keep setup and known limitations clear for local training and verify the documented asynchronous workflow.

- [ ] T030 [P] Update `specs/001-document-upload-management/quickstart.md` with the implemented local/Azurite queue and Functions configuration, pending/clean/failure behavior, owner-only management, and worker startup/validation steps without requiring cloud credentials.
- [ ] T031 Update `README.md` with the document feature entry points, local storage location, optional Azure Functions/Azurite worker setup, and explicit training-only scanner/authentication limitations.

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: T001 and T002 have no prerequisites and may run in parallel.
- **Foundational (Phase 2)**: Depends on Setup. T003/T005 may run in parallel; T004 and T006/T007 depend on the entities; T008 follows the foundational service definitions and blocks all stories.
- **User Stories**: Depend on Phase 2. US1 is the first deliverable. US2 builds on the upload/list service; US3 and US4 build on US1 and can proceed independently of each other and most US2 work. US5 consumes activity from the preceding stories.
- **Polish (Phase 8)**: Depends on the stories selected for delivery, including the queue worker and reporting behavior described in the quickstart.

### User Story Completion Order

```text
Setup → Foundational → US1
                         ├──→ US2 ──→ US4
                         ├──→ US3
                         └──→ US5 (complete relevant activity-producing stories first)
```

US1 and US2 are both P1, but US2 depends on the upload/document service foundation delivered by US1. US3 does not require the advanced browse UI; US4 uses the authorized document query behavior from US2 for Shared with Me. US5 may be implemented after US1 and may be completed incrementally as US3/US4 activity becomes available.

### Within Each User Story

- Complete the story’s service/data behavior before its UI integration.
- Reuse service-level authorization for all queries, mutations, previews, and downloads; UI visibility is not an authorization boundary.
- For upload and replacement, persist the file/metadata in Pending state, enqueue the durable `document-scan` message, and permit access only after a clean worker result.
- Do not add separate automated test tasks unless the specification or user explicitly requests them; use each story’s independent test criterion and `quickstart.md` during implementation validation.

## Parallel Opportunities

- **Setup**: T001 and T002 touch separate project/configuration files.
- **Foundational**: T003 and T005 are independent; after T003, T004 and the authorization/status service work can be split across their separate files.
- **US1**: T009 validation and T010 queue-producer work can proceed in parallel; T013 upload UI can proceed separately from T011 worker implementation once their contracts are agreed.
- **US2**: T016 list UI and T017 project view can proceed in parallel after the authorized query contract is established; T018 endpoint work is separate from those components.
- **US3**: Metadata UI work can proceed separately from persistence/service work once the owner-only operation contract is fixed; replacement staging must precede replacement activation.
- **US4**: Sharing UI and task association can proceed in parallel after service operation contracts are settled.
- **US5**: Dashboard widgets and report aggregation/page work can proceed in parallel after activity fields and authorization are established.

## Parallel Examples

### User Story 1

```text
T009: Implement file and metadata validation in ContosoDashboard/Services/DocumentUploadValidator.cs
T010: Implement the Queue Storage producer and message contract in ContosoDashboard/Services/ and ContosoDashboard/Models/
```

### User Story 2

```text
T016: Extend the My Documents browse UI in ContosoDashboard/Pages/Documents.razor
T017: Add the project-scoped document view in ContosoDashboard/Pages/ProjectDocuments.razor
```

### User Story 3

```text
T019: Implement owner-only metadata updates in ContosoDashboard/Services/DocumentService.cs
T023: Add metadata/replacement/deletion controls in ContosoDashboard/Pages/DocumentDetails.razor after the service contract is agreed
```

### User Story 4

```text
T024: Implement sharing and notifications in ContosoDashboard/Services/DocumentService.cs
T026: Implement task association and task-context upload in ContosoDashboard/Services/DocumentService.cs and ContosoDashboard/Pages/Tasks.razor
```

### User Story 5

```text
T027: Add dashboard document count/recent items in ContosoDashboard/Services/DashboardService.cs and ContosoDashboard/Pages/Index.razor
T028: Add administrator report aggregation in ContosoDashboard/Services/DocumentReportService.cs
```

## Implementation Strategy

### MVP First (User Story 1)

1. Complete Setup and Foundational phases.
2. Complete US1 upload, per-file validation, pending queue message, Azure Functions scan worker, and My Documents visibility after Clean.
3. Validate US1 independently using its criterion and the queue-triggered scenario in `quickstart.md`.
4. Stop at this checkpoint for an MVP: no pending, failed, or threatening document is accessible.

### Incremental Delivery

1. Complete Setup + Foundational → shared persistence, storage, and authorization ready.
2. Add US1 → secure upload and asynchronous clean-scan gating.
3. Add US2 → authorized discovery and retrieval.
4. Add US3 → owner-only metadata, replacement, and deletion.
5. Add US4 → sharing, notifications, and task integration.
6. Add US5 → dashboard visibility and administrator reporting.
7. Finish with local/Azurite quickstart and training limitation documentation.

## Notes

- Every task uses the required checklist form `- [ ] T### [P?] [US#?] Description with file path`; setup/foundational/polish tasks omit story labels, and user-story tasks include them.
- `[P]` marks work that can proceed independently in different files after its stated prerequisites.
- `data-model.md` is canonical for entity field constraints; T003 explicitly maps each constrained field to its `Entity.field` definition.
- The scanner is asynchronous: the web app enqueues to Azure Queue Storage; `DocumentScanFunction` processes the message, retries transient failures, and poison/failure states remain inaccessible. Local/Azurite configuration is optional for offline training runs.
