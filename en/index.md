# LookSee documentation

**English** · [Русский](../ru/index.md) · [עברית](../he/index.md) · [한국어](../ko/index.md)

LookSee is a self-hosted video analytics system. It watches live streams,
browser webcams, and video files, runs ONNX object detection on them, and turns
detections into events. A workflow graph decides what happens next: filters
narrow the events down to what matters, and actions deliver alerts, snapshots,
and messages to the people and systems that need them.

Everything runs on your hardware from one Docker Compose file. The source code
lives in the [okoflow/looksee](https://github.com/okoflow/looksee) repository
under Apache-2.0; an Enterprise edition adds line crossing, dwell time,
counting, and Slack.

![The workflow editor with a helmet compliance workflow](../images/editor.png)

## Start here

- [Getting started](getting-started.md) — install with Docker Compose, create
  the owner account, add a detection model.
- [Your first workflow](first-workflow.md) — from an empty canvas to a running
  alert.

## Guide

- [Concepts](concepts.md) — workflows, nodes, events, cameras, and runs.
- [Cameras](cameras.md) — RTSP, RTMP, SRT, browser webcams, WHEP, and video
  files.
- [Models](models.md) — the bundle format, how labels become events, and how
  to export a D-FINE model to ONNX.
- [Nodes](nodes.md) — every node with its fields, limits, and validation
  rules.
- [Actions and integrations](actions-and-integrations.md) — credentials,
  message templates, Telegram, Discord, email, MQTT, webhooks, and Slack.
- [Monitoring and alerts](monitoring-and-alerts.md) — the live view, the event
  feed, alert history, and snapshots.

## Operate

- [Configuration](configuration.md) — every environment variable, port, and
  volume.
- [Deployment](deployment.md) — a server on your network, TLS, GPUs, backups,
  and upgrades.
- [Security](security.md) — accounts, sessions, secrets, and what browsers can
  see.
- [Troubleshooting](troubleshooting.md) — cameras stuck in pending, no video,
  missing models, undelivered alerts.

## Reference

- [API](api.md) — HTTP endpoints, WebSocket messages, and the error format.
- [Enterprise edition](enterprise.md) — editions, features, and the license
  key.
- [Changelog](changelog.md) — release history.

## Contribute

The documentation is Markdown in the
[looksee-docs](https://github.com/okoflow/looksee-docs) repository; its README
lists the writing conventions. Code contributions follow
[CONTRIBUTING.md](https://github.com/okoflow/looksee/blob/main/CONTRIBUTING.md)
in the main repository, and vulnerabilities go through
[private reporting](https://github.com/okoflow/looksee/security/advisories/new).
