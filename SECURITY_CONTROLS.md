# ScamMap security controls

## Anonymous report quota

- The API issues a random 256-bit anonymous session token in an `HttpOnly`, `SameSite=Strict` cookie.
- Only the SHA-256 hash of that token is stored in SQLite.
- The server derives its pseudonymous reporter ID from the session-token hash; clients cannot submit or choose reporter IDs.
- The backend enforces a maximum of **5 accepted reports per session per Indian Standard Time calendar day**.
- Quota reservations use a SQLite `BEGIN IMMEDIATE` transaction so simultaneous requests cannot exceed the limit.
- Failed duplicate inserts release the reserved slot.
- The frontend displays used and remaining submissions, but the backend is authoritative.
- The existing short-window reporter/IP limiter remains as an additional abuse control.

This is a per-browser-session control, not proof of one human per session. Clearing cookies or using another browser/device can create another anonymous session. Production deployments should combine this control with abuse monitoring. Forwarded IP headers are ignored by default; set `TRUST_PROXY_HEADERS=1` only when a trusted reverse proxy overwrites `X-Forwarded-For`. Do not use invasive identity collection as a shortcut.

## Evidence-based credibility

The trust engine scores evidence internally using:
- reporter history and prior corroboration;
- quality and specificity of incident notes;
- exact and near-duplicate note detection;
- shared network-prefix signals;
- unusual reporting bursts (15-minute, 1-hour and 24-hour windows);
- large cohorts of first-time reporters.

A sudden group of first-time reports is down-weighted even when participants vary the wording. Shared IP/network signals are only correlation signals, not proof that people are the same person. The score is not a public accusation.

## Public lookup states

- `NO_REPORTS`: no community report has been received.
- `NOT_VERIFIED`: one or more reports exist, but the identifier has not been independently verified.
- `VERIFIED`: the identifier exists in the trusted verification registry.

Community report volume and credibility scores never automatically produce `VERIFIED` or a public `SUSPICIOUS` / `KNOWN_SCAM` label.

## Reporter privacy

- The anonymous quota cookie is `HttpOnly` and is not readable by page JavaScript.
- The raw quota token is not stored in SQLite.
- Public lookup never returns reporter IDs, full IP addresses, notes, or exact reporter locations.
- The report receipt summary is returned only to the submitting browser and redacts obvious phone numbers, email/UPI IDs, OTP/PIN/CVV values, URLs and long numeric identifiers.
- Existing stored IP-prefix signals remain server-side and are not returned in public lookup results.

## Deployment reminder

The original project archive includes local demo data and a license private-key file. Keep those files out of public repositories and production deployments. Use a separately managed production signing key and secrets store.

## Security hardening added in this revision

- `/admin/reset`, `/admin/challenges`, `/demo/seed` and `/demo/license/issue` require `X-Admin-Token`; comparison uses `secrets.compare_digest`. If `SCAMMAP_ADMIN_TOKEN` is not configured, these endpoints fail closed.
- CORS uses the explicit `CORS_ORIGINS` allow-list. Wildcard origins are not enabled.
- `DEMO_MODE` and `ALLOW_DEMO_LICENSE_ISSUE` default to disabled. The bundled demo licence is installed only when demo mode is explicitly enabled.
- The signing private key is not included in the project archive. Configure `SCAMMAP_PRIVATE_KEY` only on a protected issuer machine if demo licence issuance is deliberately enabled.
- A submitted dispute remains a moderation flag and no longer suppresses corroboration or changes trust scoring.
- SMS reports accept `message_text` / `raw_sms`; the server extracts supported identifiers and redacts common personal/security values before storing message text. `GET /lookup?message=...` performs similarity screening. Similarity is not a fraud verdict.
- Security response headers are added to normal application responses.

## Backend hardening and capacity notes (November 2026)

- JSON endpoints parse at most 64 KB; SMS screenshot uploads are streamed and capped at 5 MB, with an early `Content-Length` rejection when available.
- OCR work is moved off the async event loop, limited to two concurrent jobs per process, and uses per-image Tesseract timeouts. A timeout or OCR failure is `INCONCLUSIVE`, never a match.
- `/lookup`, `/sms/evidence/{report_id}`, and `/challenges` have endpoint-specific per-IP sliding-window limits. The limiter prunes stale keys when its map grows. It is process-local and is not a distributed rate limiter.
- VirusTotal checks for the same URL share one in-flight request; outbound checks are capped at four concurrent operations per process.
- Screenshot hash detection and evidence insertion run inside one SQLite `BEGIN IMMEDIATE` transaction. A duplicate image is retained for audit but is forced to `INCONCLUSIVE`.
- Trust burst calculations use prefix sums for constant-time sliding-window aggregates. Exact duplicate notes are checked across the full history; near-duplicate comparisons are bounded to the latest 200 notes per identifier to avoid unbounded quadratic work.
- Run one Uvicorn worker until trust-cache invalidation, rate limits, and external API cache state are moved to shared storage/versioning. Multiple workers can otherwise disagree temporarily about fresh reports/evidence.
- These changes improve resource bounds but do not constitute a production pentest or a distributed load test. Use a reverse proxy with its own body-size, connection, and timeout limits before public exposure.


## Coordinated-report abuse screening

- Public SMS analysis includes only reports whose screenshot evidence passed the current consistency checks (`MATCHED`). Missing, mismatched, inconclusive, duplicate and near-duplicate evidence is not counted as a verified public record.
- SHA-256 detects exact screenshot reuse; a 64-bit difference hash screens for near-identical screenshots after resizing or recompression.
- Five or more distinct anonymous reporter sessions submitting the same normalized SMS against the same identifier within 15 minutes move the cluster to `INCONCLUSIVE` with reason `COORDINATED_BURST_REVIEW`. All reports remain stored for audit; the public count does not rise with the burst.
- The SMS-analysis API suppresses extracted phone, sender-ID and UPI values so public results do not reveal the reported number.
- These are abuse-detection heuristics. They cannot prove screenshot authenticity, detect every sophisticated edit, or distinguish every legitimate mass campaign from collusion. Use manual review and independent evidence before applying a definitive scam label.
