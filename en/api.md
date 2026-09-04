[Documentation](index.md) / API

# API

The API is a FastAPI service on port `8000`. Studio is its only built-in
client, and everything Studio does is available to scripts and integrations.
Interactive documentation with request and response schemas is served at
`/docs`, and the OpenAPI document at `/openapi.json`.

## Authentication

All endpoints except `/health`, `/auth/*`, `/docs`, `/openapi.json`, and
the MediaMTX callback `/internal/media/auth` require the `looksee_session`
cookie. Obtain it by signing in and reuse it for seven days; `/auth/login`
and `/auth/setup` accept 20 attempts per client and 100 in total per
minute, then answer `429`:

```bash
curl -c cookies.txt -H 'content-type: application/json' \
  -d '{"email":"owner@example.com","password":"..."}' \
  http://127.0.0.1:8000/auth/login

curl -b cookies.txt http://127.0.0.1:8000/workflows
```

## Errors

Errors return JSON with a `detail` message. Workflow graph errors add a
stable `code` and the `node_id` of the offending node so a client can focus
it:

```json
{ "detail": "camera 'camera' reaches no detect node", "code": "detect_node_missing", "node_id": "camera" }
```

| Status | Meaning |
| --- | --- |
| `401` | Missing or invalid session; wrong email or password |
| `402` | The graph uses an Enterprise node without a license (`feature_not_licensed`) |
| `403` | The `Origin` header matches neither `CORS_ORIGIN_REGEX` nor the API's own origin |
| `404` | Unknown workflow, camera, alert, credential, delivery, or asset |
| `409` | The owner account already exists; a retry of a delivery that has not failed |
| `422` | Validation failed: a field is out of range, or the graph cannot run (`code` set) |
| `429` | Too many sign-in attempts; wait for `Retry-After` |
| `503` | Video storage is not configured or unreachable; sign-in while Valkey is unavailable |

[Nodes](nodes.md#validation) lists every graph error code.

## Endpoints

### Health and setup

| Method and path | Description |
| --- | --- |
| `GET /health` | Returns `{"status":"ok"}`; unauthenticated |
| `GET /auth/status` | `{"requires_setup": true}` until the owner exists |
| `POST /auth/setup` | Create the owner: `email`, `name`, `password`. Sets the session cookie. |
| `POST /auth/login` | `email`, `password`. Sets the session cookie. |
| `POST /auth/logout` | Clears the cookie |
| `GET /auth/me` | The signed-in user |
| `GET /entitlements` | `{"edition": "community", "features": []}` or the Enterprise features |

### Media

| Method and path | Description |
| --- | --- |
| `POST /cameras/{camera_id}/media-access` | `action` of `read` or `publish`; returns a `token` for MediaMTX, valid for five minutes. Publishing is granted for Browser webcam cameras only. |
| `POST /internal/media/auth` | Called by MediaMTX for every connection; answers `204` or `401`. Not for clients. |

### Models

| Method and path | Description |
| --- | --- |
| `GET /models` | Discovered bundles: `id`, `name`, `classes` (`class_id`, `label`, `event_kind`), `recommended_confidence_threshold` |

### Workflows

| Method and path | Description |
| --- | --- |
| `GET /workflows` | All workflows with their cameras and statuses, newest first |
| `POST /workflows` | Create: `name`, optional `description`, optional `graph`. Returns `201`. |
| `GET /workflows/{id}` | One workflow |
| `PATCH /workflows/{id}` | Partial update of `name`, `description`, `enabled`, `graph`. Setting `enabled` to `true` validates the graph and starts the cameras. |
| `DELETE /workflows/{id}` | Stop and delete; returns `204` |

A graph is `{"nodes": [...], "edges": [...], "comments": [...]}`. Each node
has an `id`, a `position` with `x` and `y`, and `data` whose `kind` selects
the node type and whose other fields are listed in [Nodes](nodes.md). Each
edge has an `id`, `source`, `target`, and an optional `branch` of `if` or
`else` for edges leaving a filter.

```json
{
  "name": "Fire watch",
  "graph": {
    "nodes": [
      {"id": "camera", "position": {"x": 0, "y": 80}, "data": {"kind": "camera_source", "name": "Warehouse", "source_type": "rtsp", "url": "rtsp://192.168.1.30/stream1"}},
      {"id": "detect", "position": {"x": 240, "y": 80}, "data": {"kind": "detect", "model_id": "fire-smoke", "event_kinds": ["FIRE_DETECTED", "SMOKE_DETECTED"], "confidence_threshold": 0.3, "inference_fps": 1}},
      {"id": "alert", "position": {"x": 480, "y": 80}, "data": {"kind": "log_alert_action", "severity": "critical", "cooldown_seconds": 0}}
    ],
    "edges": [
      {"id": "e1", "source": "camera", "target": "detect"},
      {"id": "e2", "source": "detect", "target": "alert"}
    ]
  }
}
```

### Alerts

| Method and path | Description |
| --- | --- |
| `GET /alerts` | Newest first. Filters: `camera_id`, `workflow_id`, `severity` (`info`, `warning`, `critical`), `limit` (1 to 500, default 100) |
| `DELETE /alerts/{id}` | Delete one alert |
| `DELETE /alerts` | Delete alerts, optionally scoped by `camera_id` or `workflow_id` |

An alert has `id`, `workflow_id`, `camera_id`, `kind`, `severity`,
`message`, `payload` (timestamp, detections, metadata, `snapshot_url` when a
Snapshot ran before the Alert), and `created_at`.

### Credentials

| Method and path | Description |
| --- | --- |
| `GET /credentials` | `id`, `name`, `type`, `summary`, timestamps. Payloads are never returned. |
| `POST /credentials` | `name`, `type`, `payload`. Payload fields per type are in [Actions and integrations](actions-and-integrations.md#credentials). |
| `PATCH /credentials/{id}` | Partial update of `name` and `payload`; omitting `payload` keeps the stored secret |
| `DELETE /credentials/{id}` | Delete |

### Deliveries

Webhook, Telegram, Discord, Slack, Email, and MQTT messages queued by the
graph; [Monitoring and alerts](monitoring-and-alerts.md#deliveries) explains
the retry policy.

| Method and path | Description |
| --- | --- |
| `GET /deliveries` | Newest first: `id`, `status` (`pending`, `processing`, `sent`, `failed`), `attempts`, `available_at`, `last_error`, `created_at`. Filters: `status`, `limit` (1 to 100, default 50) |
| `POST /deliveries/{id}/retry` | Queue a failed delivery again; returns `204`, or `409` when it has not failed |

### Assets

Available when the asset library is configured, which the compose stack does
by default; every call returns `503` while the storage is unreachable.

| Method and path | Description |
| --- | --- |
| `GET /assets` | Objects in the bucket |
| `POST /assets` | Multipart upload with a `file` field; returns `201` |
| `GET /assets/{key}/content` | The cached file, with range requests for playback |
| `DELETE /assets/{key}` | Delete the object |

### Snapshots

`GET /snapshots/<file>.jpg` serves images written by the Snapshot action. The
request needs the session cookie.

## WebSocket

`ws://<host>:8000/ws/cameras/{camera_id}` streams live messages for one
camera to a client that presents the session cookie. The socket is one-way;
inbound frames are ignored. Every message is JSON with a `type`:

| Type | Fields | When |
| --- | --- | --- |
| `detections` | `ts`, `frame_width`, `frame_height`, `detections[]` | Every processed frame |
| `event` | `kind`, `ts`, `frame_width`, `frame_height`, `detections[]` | Every event the graph receives, at most one per kind and camera within `EVENT_COOLDOWN_SECONDS` |
| `worker` | `status`, `ts`, `reason` | The camera changes status |
| `alert` | `id`, `kind`, `severity`, `message`, `ts`, `snapshot_url` | An Alert action fires |

A detection has `label`, `bounding_box` as `[x_min, y_min, x_max, y_max]` in
frame pixels, `confidence`, `class_id`, and `tracker_id` (or `null`).

## Webhook payload

The Webhook and MQTT actions deliver this JSON for every event:

```json
{
  "camera_id": "01a061c1-ff68-7611-af72-436d9d5ba907",
  "kind": "HELMET_DETECTED",
  "ts": "2026-09-02T10:58:20.983914+00:00",
  "metadata": { "count": 2, "model_id": "ppe-helmets" },
  "snapshot_url": "/snapshots/20260902-105820-01a061c1-3f9a2c1e.jpg"
}
```

`snapshot_url` is present only when a Snapshot action ran earlier in the
graph, and is relative to the API origin.
