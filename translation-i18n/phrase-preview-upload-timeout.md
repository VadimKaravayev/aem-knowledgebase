# Phrase preview package upload times out on large pages

- **Symptom:** a page's Phrase jobs have no preview; log shows "No answer from Phrase at .../jobPreviewPackage" and no memsource-action-id for that page.
- **Root cause:** Phrase answers `POST /api2/v1/projects/{uid}/jobPreviewPackage` only after processing the zip. A ~10 MB package (listing page, many images) outlasts a 10 s socket timeout shared by all calls; HttpClient does not retry `SocketTimeoutException` on a POST, so the best-effort upload is skipped.
- **Fix:** per-request `RequestConfig` with a longer socket timeout (60 s) for multipart uploads only; every other call keeps 10 s.
- **How to spot it:** `SocketTimeoutException: Read timed out` under `DefaultHttpResponseParser.parseHead` (request sent, no status line), and the failing page's zip is several times larger than its siblings'.
