[Documentation](index.md) / Changelog

# Changelog

Release history of LookSee. Versions follow semantic versioning; entries
under *Unreleased* are on `main` and ship with the next release.

## Unreleased

- Continuous integration runs linting, tests, the migration check, and the
  Studio build on every pull request.
- Test suites for the shared contracts, the API, the inference service, and
  the Enterprise package.
- Repository documentation: README, contributing guide, security policy, code
  of conduct, and issue forms.
- Documentation in English, Russian, Hebrew, and Korean.
- Studio lint tooling fixed for the monorepo layout and a dependency
  advisory resolved.

## 0.1.0

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
