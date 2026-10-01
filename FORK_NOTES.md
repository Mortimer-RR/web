# Fork notes: upload-integrity fork of OpenCloud Web

This fork carries the web client's upload-integrity fix until upstream ships an
equivalent. The server-side fixes live in the reva fork (see its FORK_NOTES.md).

- Upstream: https://github.com/opencloud-eu/web
- Base: `ea72e3168c` (upstream `main` at the time of the audit). opencloud `main` pairs
  with web `main` in dev mode, not with the `v8.0.0` release it downloads for prod builds.
- Branch layout:
  - `fix/*`: one branch per fix, based directly on the base commit, free of fork-only
    files, so it can be sent upstream as-is.
  - `integrity`: base + this file + all fixes. Build releases from here.
- How the server picks up these assets: see "Web assets" in the opencloud fork's
  FORK_NOTES.md (build `release/web.tar.gz` here and unpack it into
  `services/web/assets/core` before building the server).
- Per AGENTS.md, commits in this repo carry no AI attribution lines.

## Divergences from upstream

### 1. `fix(upload): send a whole-file SHA1 checksum with every upload`

Branch `fix/upload-checksum`.

- `packages/web-pkg/src/services/uppy/checksum/`: `UploadChecksumPlugin`, an Uppy
  pre-processor registered by `UppyService` for both the tus and the XHR uploader. It
  hashes `file.data` in a web worker (`worker.ts`) with `hash-wasm`'s incremental SHA1,
  reading 8 MiB `Blob.slice`s, and sets `file.meta.checksum = 'sha1 <hex>'`.
  - New dependency: `hash-wasm` (`crypto.subtle.digest` cannot hash incrementally).
  - Skipped: folders, remote (companion) files, which have no local data, and files that
    already have a checksum (retries).
  - If hashing fails, the plugin emits `upload-error` for that file. Uppy then marks
    it failed and the uploader skips it, so it is never sent without a checksum. The
    other files continue.
- `uppyService.ts`: `checksum` added to `TUS_ALLOWED_META_FIELDS` (it is a hash of
  the transmitted bytes, so it reveals no path or cleartext). Forwards
  `preprocess-progress` / `preprocess-complete` as service topics.
- `useUpload.ts`: the XHR (plain PUT) fallback sends `OC-Checksum: SHA1:<hex>`, the
  desktop client's format.
- `UploadInfo.vue`: a file shows "Calculating checksum..." while it is hashed. The
  overall title already shows "Preparing upload..." until the first byte is sent.
- **Vault uploads:** `HandleUpload.applyVaultEncryption` replaces `file.data` with the
  ciphertext on `files-added`, before `uppy.upload()` runs the pre-processors. The hash
  therefore covers the ciphertext that is actually transmitted (covered by a unit test).
- **Fingerprint splicing:** tus-js-client resumes from localStorage keyed on name,
  type, size, mtime and endpoint. A different file matching all of these would resume
  another file's partial upload. The upload's `checksum` was set at creation from the
  original file, so the server now rejects the spliced result (460, reva fix 4)
  instead of storing it.
- Server format: ocdav reads `Upload-Metadata: checksum sha1 <hex>` (lower-cases the
  algorithm) and `OC-Checksum: SHA1:<hex>` for PUT. decomposedfs compares lowercase hex.

Tests (all fail before the change):

- `tests/unit/services/uppy/checksum.spec.ts`: known SHA1s, slice-by-slice hashing,
  meta field set before upload, ciphertext hashed when `file.data` is replaced, folders
  skipped, an unreadable file fails alone.
- `tests/unit/services/uppy/uppyService.spec.ts`: the tus `Upload-Metadata` fields (as
  built by `@uppy/tus` from the plugin's `allowedMetaFields`) are exactly `name`,
  `mtime`, `checksum`, with no path fields. A real HTTP round trip isn't possible in
  Vitest: tus-js-client resolves to its Node build there, which can't send Blobs.
- `tests/unit/composables/upload/useUpload.spec.ts`: XHR uploads send `OC-Checksum`.
- `web-runtime/tests/unit/components/UploadInfo.spec.ts`: "Calculating checksum..."
  shown while hashing.
- The worker tests set `VITEST_WEB_WORKER_CLONE=none`. Without it, happy-dom's `Blob`
  loses its content when structured-cloned into `@vitest/web-worker`'s emulated worker
  (browsers keep it).

Pre-existing, unrelated unit test failures on the base commit (identical with and
without this fix, Windows host): 6 tests in `CreateFolderModal.spec.ts`,
`ResourcePreview.spec.ts`, `conflictDialog.spec.ts`, `resourcesTransfer.spec.ts`.

#### Hashing cost (measured 2026-10-01, the built worker, 8 MiB slices)

| Browser                                          | 1 GB  | 10 GB  | UI thread                                  |
| ------------------------------------------------ | ----- | ------ | ------------------------------------------ |
| Chrome 154, Windows (native)                     | 3.0 s | 29.7 s | fully responsive (all timer ticks on time) |
| Firefox 155, Linux container on the same machine | 5.1 s | 50.6 s | responsive                                 |
| Chromium 153, same Linux container (calibration) | 5.2 s | 45.3 s | responsive                                 |

Native Firefox on Windows could not be launched by Playwright: Windows blocks the
unsigned Playwright Firefox build, and that security policy was left alone. The
container runs about 1.5× slower than native Chrome, and Firefox is within ~10% of
Chromium there. So native Firefox should land around **3–3.5 s per GB**. That's roughly
30–35 s of "Calculating checksum..." before a 10 GB upload starts, on top of the
upload itself. Hashing reads the file once more from local disk, but never holds more
than one 8 MiB slice in memory.

Upstream PR draft: title as the commit subject; body = the commit message + the
table above. Related issue to file: "Browser uploads carry no checksum, so damaged or
spliced uploads are stored silently".
