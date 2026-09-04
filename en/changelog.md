[Documentation](index.md) / Changelog

# Changelog

Release history of LookSee. Versions follow semantic versioning, and dates
are release dates. Changes on `main` that are not released yet are listed
under *Unreleased*.

## [Unreleased]

## [0.2.0] - 2026-09-05

Upgrading from 0.1.0:

- Set `STORAGE_PASSWORD` in `.env`. `POSTGRES_PASSWORD` and
  `MTX_MEDIA_PASSWORD` no longer have example values and must be set as
  well; compose refuses to start while any of the three is blank.
- Rotate `MTX_MEDIA_PASSWORD`: 0.1.0 sent it to every browser that loaded
  Studio.
- Run `docker compose up -d --build` and let `api-migrate` apply migration
  `0002` and `storage-init` create the video bucket. Update the API, the
  inference service, Studio, and MediaMTX together: MediaMTX now authorizes
  every connection through the API.
- Events no longer wait out `EVENT_COOLDOWN_SECONDS` before entering the
  graph. Where the cooldown used to suppress repeated actions, add a
  Debounce filter or an Alert cooldown.
- The `docs` compose profile and `DOCS_PORT` are gone; the documentation
  lives in this repository.

### Added

- Bundled S3-compatible video storage: the `storage` service (RustFS) with
  the `storage_data` volume, created and provisioned by compose, so File
  cameras work without an external bucket. The `S3_*` variables still point
  the API at an external one.
- Delivery outbox: Webhook, Telegram, Discord, Slack, Email, and MQTT
  messages are stored in PostgreSQL and retried after transient failures
  with a delay that doubles up to five minutes, for at most eight attempts.
  `GET /deliveries` lists the queue and `POST /deliveries/{id}/retry`
  re-queues a failed delivery. Webhook requests carry an `Idempotency-Key`
  header.
- Camera-scoped media grants: `POST /cameras/{id}/media-access` issues a
  five-minute token for reading or publishing one camera, and MediaMTX
  authorizes every connection through the API.
- Sign-in and setup rate limit: 20 attempts per client and 100 in total per
  minute, answered with `429` and `Retry-After`.
- Origin check: state-changing requests and WebSocket connections whose
  `Origin` matches neither `CORS_ORIGIN_REGEX` nor the API origin are
  rejected with `403`.
- Continuous integration runs linting, tests, the migration check, the
  Studio build, and a compose configuration check on every pull request,
  with test suites for the shared contracts, the API, the inference
  service, and the Enterprise package.
- Repository documentation: README, contributing guide, security policy,
  code of conduct, and issue forms. This documentation in English, Russian,
  Hebrew, and Korean.

### Changed

- The MediaMTX media password is a backend secret shared by the API, the
  inference service, and MediaMTX; Studio no longer receives it.
- `EVENT_COOLDOWN_SECONDS` throttles only the `event` messages of the live
  feed; every event enters the graph.
- Storage failures answer `503` with a readable message instead of an
  internal error.
- Detection frames left unacknowledged by a stopped API process are
  processed again after one minute, and frames are processed one at a time.
- `.env.example` is grouped by service and lists only the settings an
  installation sets; required passwords are blank.
- Studio bundles its font instead of loading it from Google Fonts, keeps
  edits made while a save is in flight, and links **Documentation** to this
  repository.
- MediaMTX 1.20.1.

### Fixed

- Switching a camera from RTSP to Browser webcam left MediaMTX pulling the
  old stream; the path configuration is now replaced instead of patched.
- WebRTC sessions are closed when leaving the monitor, and saving a
  workflow no longer races with edits made during the save.
- Studio lint tooling for the monorepo layout and a dependency advisory in
  Studio.

## [0.1.0] - 2026-08-06

First release.

- Workflow editor with Camera, Detect, If / Else, Class, Size, Zone,
  Schedule, and Debounce nodes and the Alert, Snapshot, Webhook, Telegram,
  Email, MQTT, and Discord actions; Count, Line crossing, Dwell, and Slack in
  the Enterprise edition.
- Camera sources over RTSP, RTMP, SRT, WHEP, browser webcams, and video
  files from an S3-compatible asset library, ingested through MediaMTX.
- ONNX inference for D-FINE exports with ByteTrack tracking on CPU, CUDA, or
  CoreML.
- Live monitor with WebRTC playback, detection and zone overlays, an event
  feed, and alert history with snapshots.
- Encrypted credential store, owner account with session cookies, and a
  Docker Compose deployment with PostgreSQL and Valkey.

[Unreleased]: https://github.com/okoflow/looksee/compare/v0.2.0...HEAD
[0.2.0]: https://github.com/okoflow/looksee/compare/v0.1.0...v0.2.0
[0.1.0]: https://github.com/okoflow/looksee/releases/tag/v0.1.0
