# Implementation Plan: Document Upload and Management

**Branch**: `001-document-upload-management` | **Date**: 2026-11-19 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-document-upload-management/spec.md`

## Summary

Add document upload, browsing, search, preview/download, owner-only management, sharing, task association, dashboard visibility, notifications, and audit reporting to the existing ASP.NET Core/Blazor Server training application. Use EF Core integer entities, service-boundary authorization, local filesystem storage behind `IFileStorageService`, Azure Functions with Azure Queue Storage triggers for asynchronous malware scanning, and authenticated server endpoints for file delivery.

## Technical Context

**Language/Version**: C#, ASP.NET Core/Blazor Server; project currently targets net10.0  
**Primary Dependencies**: Entity Framework Core 8, SQL Server LocalDB, ASP.NET Core authentication/authorization, Bootstrap 5.3  
**Storage**: SQL Server LocalDB for metadata; local disk `AppData/uploads` for training files; Azure Queue Storage for scan work messages
**Testing**: .NET test framework project to be selected during implementation planning; Azure Functions local-run/component/manual validation
**Target Platform**: Windows training workstation using LocalDB and a browser, optionally with Azurite and Azure Functions Core Tools
**Project Type**: Single web application with one Azure Functions scan worker
**Performance Goals**: 25 MB upload within 30 seconds, list 500 documents within 2 seconds, authorized search within 2 seconds, supported preview within 3 seconds  
**Constraints**: Core operations remain locally runnable; Azure Functions/Queue Storage or Azurite must be replaceable/configurable; 25 MB per-file limit, extension/MIME validation, fail-closed malware scan, integer keys, text category values
**Scale/Scope**: Training deployment; one web application with approximately 500-document list/search target

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle/Rule | Assessment | Gate |
|---|---|---|
| Training-Only Scope | Pass. The plan documents local storage, mock authentication, and training scanner limitations. | Pass |
| Offline-First Operation | Pass for core document operations. The asynchronous Azure Queue scan worker is optional/configurable, and fail-closed behavior applies when queue or worker infrastructure is unavailable. Local/Azurite emulation supports local validation. | Pass |
| Layered Architecture | Pass. Blazor pages call document services; EF Core and storage remain in their respective layers. | Pass |
| Authorization Boundaries | Pass. Owner/project/share checks are required in services and download/preview endpoints; UI hiding alone is insufficient. | Pass |
| Simplicity and Clarity | Pass. Reuses existing services, notifications, EF patterns, and one web project. | Pass |

No constitutional exceptions are required.

## Project Structure

### Documentation (this feature)

```text
specs/001-document-upload-management/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
└── tasks.md                 # Not created by /speckit.plan
```

### Source Code (repository root)

```text
ContosoDashboard/
├── Data/
│   └── ApplicationDbContext.cs
├── Models/
│   ├── Document.cs
│   ├── DocumentShare.cs
│   ├── DocumentTask.cs
│   └── DocumentActivity.cs
├── Services/
│   ├── IDocumentService.cs
│   ├── DocumentService.cs
│   ├── IFileStorageService.cs
│   ├── LocalFileStorageService.cs
│   ├── IFileMalwareScanner.cs
│   └── LocalFileMalwareScanner.cs
├── Pages/
│   ├── Documents.razor
│   ├── DocumentDetails.razor
│   └── DocumentUpload.razor
├── AppData/
│   └── uploads/
└── Program.cs
ContosoDashboard.Functions/
├── Functions/
│   └── DocumentScanFunction.cs
├── Models/
│   └── DocumentScanQueueMessage.cs
└── Program.cs
```

**Structure Decision:** Extend the existing single Blazor Server project. The additions preserve Models/Data/Services/Pages separation and do not introduce a separate API or frontend project.

## Complexity Tracking

> **Fill only if Constitution Check has violations that must be justified.**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| None | No constitution violation is planned | N/A |

## Post-Design Constitution Re-check

*Re-evaluation after data-model and contract design.*

| Principle/Rule | Assessment | Status |
|---|---|---|
| Training-Only Scope | Data and file abstractions remain local and explicitly replaceable; no production-security claims. | Pass |
| Offline-First Operation | The design uses no external account, SDK, or network dependency. | Pass |
| Layered Architecture | Entities, EF configuration, services, abstractions, pages, and server file endpoints remain separated. | Pass |
| Authorization Boundaries | Query roots and mutations require owner/project/share checks; preview/download use the same checks. | Pass |
| Simplicity and Clarity | A focused set of four entities and two infrastructure interfaces covers the requirements. | Pass |
