# Contribution 1: Integrate Azure Blob Storage as a backup destination

**Contribution Number:** 1
**Student:** Jin Gi Min
**Issue:** [Portabase/portabase#265](https://github.com/Portabase/portabase/issues/265)
**Working Branch:** [`feature/Integrate-Azure-Blob`](https://github.com/jjingi/portabase/tree/feature/Integrate-Azure-Blob)
**Status:** Phase I Complete

---

## Why I Chose This Issue

I chose this issue because it matches my interest in full-stack development and cloud storage features. The work involves TypeScript, React, backend provider logic, and UI form integration, which are areas I want to get better at through a real open-source contribution.

I also like that the issue is clearly scoped. The issue lists the exact files to create or update, and the acceptance criteria explain what "done" should look like, including screenshots of the Azure Blob Storage option and configuration form. From reading the issue, I understand that the final result should let Portabase users configure Azure Blob Storage as a backup destination.

This issue also seems active and claimable because it is recent, labeled as a good first issue, and I did not see an existing pull request or another contributor actively working on it. Before starting implementation, I plan to comment on the issue and ask if there is a preferred testing approach for the Azure provider functions.

---

## Understanding the Issue

### Problem Description

Portabase lets users register **storage channels** that backups are pushed to. Today the backend supports three working providers: `local`, `s3` (via the Minio client), and `google-drive`. "Azure Blob Storage" already shows up in the provider picker, but it is a **placeholder only** — it is flagged `preview: true` in `src/features/channel/channels-storage-helper.tsx` and has no schema, no form, and no backend handler. Selecting it does nothing useful, so users cannot actually back up to Azure.

The task is to turn that preview placeholder into a fully wired storage provider, following the same pattern the existing `s3` provider uses.

### Expected Behavior

- "Azure Blob Storage" is a selectable, non-preview option when adding a storage channel.
- Selecting it renders a configuration form (account name, account key / connection string, container name, etc.).
- The config is validated, saved as a storage channel, and a "test connection" (ping) succeeds against a real Azure container.
- Database backups dispatched to an Azure channel upload, download, delete, and copy correctly — the same five operations every other provider implements.

### Current Behavior

- "Azure Blob Storage" appears in the dropdown but is marked `preview: true` (rendered as a disabled/coming-soon option).
- There is no `'blob'` case in `renderChannelForm` in `src/features/channel/channels-helpers.tsx`, so even if selected, the default branch returns `null` and no form is shown.
- The provider is not part of the `StorageChannelFormSchema` discriminated union, not in the `StorageProviderKind` union, and not in the `handlers` map — so a dispatch would fall through to `Unsupported storage provider: blob`.

### Affected Components

| Layer | File | Change |
|---|---|---|
| Provider type | `src/features/storages/storages.types.ts` | Add `'blob'` to `StorageProviderKind` |
| Config schema | `src/features/channel/storages/blob.schema.ts` *(new)* | Zod schema for Azure config |
| Backend handler | `src/features/channel/storages/blob.ts` *(new)* | `uploadBlob/getBlob/deleteBlob/pingBlob/copyBlob` |
| Handler registry | `src/features/channel/storages/index.ts` | Register `blob` in `handlers` map |
| Form schema union | `src/features/channel/channel-form.schema.ts` | Add `blob` variant to `StorageChannelFormSchema` |
| Config form | `src/features/channel/storages/blob.form.tsx` *(new)* | React form component |
| Form router | `src/features/channel/channels-helpers.tsx` | Add `case "blob"` to `renderChannelForm` |
| Provider list | `src/features/channel/channels-storage-helper.tsx` | Remove `preview: true` from the `blob` entry |
| Dependency | `package.json` | Add `@azure/storage-blob` |

---

## Reproduction Process

### Environment Setup

Cloned my fork (`github.com/jjingi/portabase`) and added the upstream remote. The project is a Next.js app using `pnpm`, Drizzle ORM, and Docker Compose for the supporting services.

- `pnpm install` to install dependencies.
- Copied `.env.example` → `.env` and filled in local values.
- `docker compose up` to bring up the database and dependent services.
- `pnpm dev` to run the app at `http://localhost:3000`.

Main friction was making sure the local DB migrations (`src/db/migrations/`) were applied so the storage-channel tables existed before testing the storage settings UI.

### Steps to Reproduce

1. Start the app (`pnpm dev`) and log in.
2. Navigate to the storage settings / storage channels section.
3. Click "Add storage channel" and open the provider dropdown.
4. **Expected:** "Azure Blob Storage" is selectable and shows a configuration form.
5. **Actual:** "Azure Blob Storage" is rendered as a preview/coming-soon item (`preview: true`); selecting it shows no form, because `renderChannelForm` has no `"blob"` case and falls through to the `default` branch (returns `null`). There is no way to save or test an Azure channel.

### Reproduction Evidence

- **Branch showing reproduction / scoping:** [`feature/Integrate-Azure-Blob`](https://github.com/jjingi/portabase/tree/feature/Integrate-Azure-Blob)
- **Key code references:**
  - `src/features/channel/channels-storage-helper.tsx` — `{value: "blob", label: "Azure Blob Storage", icon: BlobIcon, preview: true}`
  - `src/features/channel/channels-helpers.tsx` — `renderChannelForm` switch has cases for `s3`, `google-drive`, `local`, but **no** `blob`.
  - `src/features/storages/storages.types.ts` — `StorageProviderKind` = `'local' | 's3' | 'google-drive'` (no `blob`).
- **My findings:** The UI scaffolding (label + `BlobIcon`) already exists, so this is genuinely a "wire up the implementation" task rather than a from-scratch feature. The cleanest reference is the `s3` provider, which is small (~150 lines) and implements exactly the five operations the handler interface expects.

---

## Solution Approach

### Analysis

There is no bug to fix — the feature is intentionally stubbed as a preview. The root "cause" is that the `blob` provider was never implemented end to end. Every other provider is wired through five layers (type → schema → handler → registry → UI form), and `blob` is missing all of them except the dropdown entry.

### Proposed Solution

Implement `blob` by mirroring the `s3` provider's structure, swapping the Minio client for the official `@azure/storage-blob` SDK. Each storage handler is a plain module exporting five functions matching the `ProviderHandler` interface in `src/features/channel/storages/index.ts`:

```ts
type ProviderHandler = {
  upload: (config, input) => Promise<StorageResult>;
  get:    (config, input) => Promise<StorageResult>;
  delete: (config, input) => Promise<StorageResult>;
  ping:   (config, input) => Promise<StorageResult>;
  copy:   (config, input) => Promise<StorageResult>;
};
```

### Implementation Plan

Using the UMPIRE framework (adapted):

**Understand:** "Azure Blob Storage" is a preview-only placeholder in the storage provider picker. I need to make it a real provider so users can configure an Azure container and have backups upload/download/delete/copy to it, plus a working connection test.

**Match:** The `s3` provider is the closest existing pattern:
- `src/features/channel/storages/s3.schema.ts` — Zod config schema.
- `src/features/channel/storages/s3.ts` — `uploadS3/getS3/deleteS3/pingS3/copyS3`, each returning a `StorageResult` and catching errors into `{success: false, error}`.
- Registered in the `handlers` map in `src/features/channel/storages/index.ts`.
- Added to the `StorageChannelFormSchema` discriminated union in `channel-form.schema.ts`.
- Rendered via `case "s3"` in `renderChannelForm` (`channels-helpers.tsx`) and `s3.form.tsx`.
The `google-drive` provider is a second reference for a provider that needs an SDK client.

**Plan:**
1. Add `'blob'` to `StorageProviderKind` in `src/features/storages/storages.types.ts`.
2. Add `@azure/storage-blob` to `package.json` (`pnpm add @azure/storage-blob`).
3. Create `src/features/channel/storages/blob.schema.ts` — Zod schema for the Azure config (account name, account key or connection string, container name).
4. Create `src/features/channel/storages/blob.ts` — implement `uploadBlob`, `getBlob`, `deleteBlob`, `pingBlob`, `copyBlob` using a `BlobServiceClient`, mirroring the S3 functions (including an `ensureContainer` helper analogous to `ensureBucket`).
5. Register `blob` in the `handlers` map in `src/features/channel/storages/index.ts`.
6. Add a `blob` variant to `StorageChannelFormSchema` in `src/features/channel/channel-form.schema.ts`.
7. Create `src/features/channel/storages/blob.form.tsx` and add `case "blob"` to `renderChannelForm` in `channels-helpers.tsx`.
8. Remove `preview: true` from the `blob` entry in `channels-storage-helper.tsx`.
9. Verify `pnpm lint` / typecheck pass and run the existing test suite for regressions.

**Implement:** Work tracked on [`feature/Integrate-Azure-Blob`](https://github.com/jjingi/portabase/tree/feature/Integrate-Azure-Blob). *(commit links added as work progresses)*

**Review:** Self-review against the project's `CONTRIBUTING` guidelines and commit conventions; make sure error handling matches the existing providers (never throw out of a handler — return `StorageResult` with `success: false`).

**Evaluate:** Manually reproduce the steps above and confirm the Azure option is selectable, the form renders, "test connection" (ping) succeeds against a real container, and a database backup round-trips (upload → download → delete). All existing tests should continue to pass.

---

## Testing Strategy

### Unit Tests

- [ ] `pingBlob` returns `{success: true}` against a reachable container and `{success: false}` with a clear message when the container is missing or credentials are wrong.
- [ ] `uploadBlob` handles `Buffer`, `Uint8Array`, and `Readable` inputs (matching the S3 handler's input handling).
- [ ] `getBlob` returns the stored file and, when requested, a signed/SAS URL; returns `{success: false, error: "File not found"}` for a missing blob.
- [ ] `deleteBlob` and `copyBlob` succeed and surface errors as `StorageResult` rather than throwing.

### Integration Tests

- [ ] Create a `blob` storage channel through the form and confirm it persists with the correct discriminated-union shape.
- [ ] Dispatch a real database backup to an Azure channel and confirm `backupStorage` status transitions to success.

### Manual Testing

Reproduce the four UI steps above against an Azure Storage account (or the Azurite emulator) and confirm the provider is selectable, configurable, testable, and that a backup round-trips.

---

## Implementation Notes

### Phase I Progress (Understanding & Reproduction)

- Mapped the full storage-provider architecture: `dispatchStorage` → `dispatchViaProvider` → per-provider handler module.
- Confirmed `blob` is a preview-only stub and identified the exact nine files/changes needed (table above).
- Picked `s3` as the implementation reference because it is the smallest complete provider and uses an external SDK client like Azure will.
- Decided to use the official `@azure/storage-blob` SDK rather than a generic S3-compatible shim, since the issue specifically asks for Azure Blob Storage.

### Code Changes

- **Files to be created:** `blob.schema.ts`, `blob.ts`, `blob.form.tsx`.
- **Files to be modified:** `storages.types.ts`, `index.ts`, `channel-form.schema.ts`, `channels-helpers.tsx`, `channels-storage-helper.tsx`, `package.json`.
- **Approach decision:** Mirror the S3 provider's error-handling contract exactly — handlers catch internally and return `StorageResult`, so the dispatcher's behavior stays uniform across providers.

---

## Pull Request

**PR Link:** *(to be added when submitted)*

**PR Description:** *Adapted from the sections above — problem, the nine wiring changes, and the test results.*

**Maintainer Feedback:**
- *(none yet)*

**Status:** In progress — Phase I complete, implementation not yet opened as a PR.

---

## Learnings & Reflections

### Technical Skills Gained

- How a pluggable provider/dispatch pattern is structured in a real TypeScript/Next.js codebase (typed handler map + discriminated-union form schemas).

### Challenges Overcome

- *(to be filled in during implementation)*

### What I'd Do Differently Next Time

- *(reflection after PR)*

---

## Resources Used

- [Issue #265](https://github.com/Portabase/portabase/issues/265)
- [`@azure/storage-blob` SDK docs](https://learn.microsoft.com/azure/storage/blobs/storage-quickstart-blobs-nodejs)
- Existing `s3` and `google-drive` providers in `src/features/channel/storages/` as reference implementations.
</content>
</invoke>
