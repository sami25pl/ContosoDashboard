# Quickstart Validation Guide

## Prerequisites

- .NET 8 SDK or later, matching the repository environment
- SQL Server LocalDB
- PowerShell
- Repository checkout on Windows
- A browser
- Azure Functions Core Tools when validating the Azure scan worker
- Azurite or Azure Storage Emulator when validating Queue Storage locally

## Setup

From the repository root:

```powershell
cd ContosoDashboard
dotnet restore
dotnet run
```

Open the HTTPS URL reported by `dotnet run`, then log in with one of the seeded mock users.

To recreate a broken development database:

```powershell
dotnet ef database drop --force
dotnet run
```

Only run the database recreation command in a disposable training database.

## Azure Functions Scan Worker Setup

For local queue/worker validation:

```powershell
# Terminal 1: start storage emulation
azurite

# Terminal 2: start the Azure Functions scan worker
cd ContosoDashboard.Functions
copy local.settings.example.json local.settings.json
func start
```

Configure the worker connection to Azurite/Azure Queue Storage and configure the web application with the same queue name and storage implementation. The queue trigger should use `document-scan`.

Do not require Azure Queue Storage or Azure Functions for database/UI-only training scenarios. If queue processing is unavailable, uploads must remain `Error`/pending and inaccessible rather than being exposed as scanned.

## Validation Scenarios

### 1. Standard upload

1. Log in as `Ni Kang` (`Employee`).
2. Open Documents and start upload.
3. Select a PDF smaller than 25 MB.
4. Enter a title and category.
5. Submit.

**Expected:** progress and success feedback, one document record, secure file placement outside `wwwroot`, and appearance in My Documents.

### 2. Batch mixed validation

1. Select one valid PDF, one unsupported `.exe`, and one file over 25 MB.
2. Submit as a batch.

**Expected:** valid file succeeds, unsupported/oversized files receive separate user-readable errors, and no invalid record or file is created.

### 3. Scan failure

1. Configure or provide a file that the local scanner marks `Threat` or `Error`.
2. Submit it.

**Expected:** the record is immediately marked `Error` or `Threat`, no download/preview access is granted, and a threat file is quarantined or removed with a clear failure result.

### 3a. Queue Storage asynchronous scan

1. Start Azurite and the Azure Functions worker.
2. Upload a valid PDF.
3. Inspect the document status before and after the queue trigger completes.

**Expected:** the upload record is immediately `Pending`, a `document-scan` queue message is created, the worker changes status to `Scanning`, and a clean result changes it to `Clean`. Download/preview are denied while pending or failing and become available after `Clean`.

### 4. Browse/filter/search

1. Seed or upload documents in at least two categories and projects.
2. Sort by title/upload date/category/file size.
3. Filter by category/project/date.
4. Search title, description, tag, uploader name, or project name.

**Expected:** authorized matching records load within 2 seconds for an approximately 500-document dataset and cannot reveal unauthorized project/shared records.

### 5. Preview/download

1. Open an authorized PDF or JPEG.
2. Use preview.
3. Download an authorized Office/text document.

**Expected:** preview loads for supported PDF/image types within 3 seconds; download works for all supported types; another user's unauthorized document cannot be previewed or downloaded.

### 6. Owner-only management

1. As the owner, edit title/category/tags/project and replace the file.
2. Log in as a project manager or recipient who does not own the document.
3. Attempt edit, replace, reshare, or delete.
4. As the owner, delete with confirmation.

**Expected:** owner changes succeed; all unauthorized mutations fail and make no change; confirmed deletion removes record, relationships, and file.

### 7. Sharing and notification

1. As an owner, share a document with another seeded user and a team/department.
2. Log in as a recipient.

**Expected:** recipient receives an in-app notification and sees the document in Shared with Me, but cannot mutate or reshare it.

### 8. Task attachment

1. Open an authorized task.
2. Attach an existing authorized document.
3. Upload a new document from the task page.

**Expected:** task association is created and document project matches the task project.

### 9. Dashboard and activity

1. Upload and download/share/delete documents as a seeded user.
2. Open the dashboard.
3. Log in as Administrator and request reports.

**Expected:** dashboard shows the five latest owned documents and count; reports include document types, active uploaders, and access patterns; each completed action creates an attributable activity record.

## Reporting

After implementation:

```powershell
dotnet build
dotnet test
dotnet run
```

Report build/test warnings and any remaining training limitations. Do not run or require cloud services for this feature.
