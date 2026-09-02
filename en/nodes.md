[Documentation](index.md) / Nodes

# Nodes

Reference for every node in the workflow editor: what it does, its fields
with defaults and limits, and what the validation on **Run** checks. Field
names in code style are the keys used in the API; the inspector shows them
with friendlier labels.

## Connections

- A **Camera** has one output and connects to exactly one **Detect**.
- **Detect** connects to any number of filters and actions.
- Every filter has two outputs, **If** and **Else**. An event that passes
  leaves through **If**; one that fails leaves through **Else**. An edge
  without a branch counts as **If**.
- **Snapshot** connects onward to other actions, which then receive the
  snapshot. Other actions have no output.
- Every node runs at most once per event.

The palette groups nodes into Sources, Detection, Logic, Object, Spatial,
Temporal, and Actions. Enterprise nodes (Count, Line crossing, Dwell, Slack)
show a lock until a license key is configured.

## Sources

### Camera — `camera_source`

| Field | Default | Limits | Meaning |
| --- | --- | --- | --- |
| `name` | `Camera` | 1 to 128 characters | **Name**. Shown in the monitor and in alert messages |
| `source_type` | `rtsp` | `rtsp`, `rtmp`, `srt`, `webrtc`, `whep`, `file` | **Source**. How MediaMTX ingests the stream; see [Cameras](cameras.md) |
| `url` | empty | up to 2048 characters | **URL**. Stream URL for pull sources, asset key for `file`; unused for `webrtc` |

On **Run** the URL must match the source type: `rtsp://` or `rtsps://` for
RTSP, `rtmp://` or `rtmps://` for RTMP, `srt://` for SRT, `http://`,
`https://`, `whep://`, or `wheps://` for WHEP, and an asset key made of
letters, digits, `.`, `_`, `-`, and `/` for File.

## Detection

### Detect — `detect`

| Field | Default | Limits | Meaning |
| --- | --- | --- | --- |
| `model_id` | none | a model from `models/` | **Model**. Required on **Run**. |
| `event_kinds` | all | up to 64 kinds the model produces | **Events**. Empty passes every event kind of the model. |
| `confidence_threshold` | `0.5` | 0 to 1 | **Confidence**. Detections below it are dropped before tracking. |
| `inference_fps` | `1` | 1 to 15 | **Checks per second**. How often the model runs on this camera. |

Selecting a model in Studio fills in its recommended confidence.

## Filters

A filter passes or fails each event. Filters that inspect detections pass
an event with no detections, except where noted.

### If / Else — `if_else_filter`

Compares one property of the event. The `condition` object selects the
property with `field`:

| `field` | `operator` | `value` | Passes when |
| --- | --- | --- | --- |
| `event_kind` | `is`, `is_not` | an event kind | The event's kind matches (or does not) |
| `object_class` | `contains`, `not_contains` | a label, up to 256 characters | A detection with that label is present (or absent) |
| `detection_count` | `eq`, `neq`, `gt`, `gte`, `lt`, `lte` | 0 to 1000, default `gte 1` | The number of detections compares as chosen |
| `max_confidence` | the same six operators | 0 to 1, default `gte 0.5` | The highest confidence compares as chosen |

On **Run**, `event_kind` and `object_class` need a value, and it must be one
the upstream model can produce.

### Class — `class_filter`

| Field | Default | Limits | Passes when |
| --- | --- | --- | --- |
| `classes` | empty | up to 64 labels | Any detection has one of the labels. Empty passes everything. |

Labels must belong to the upstream model.

### Size — `size_filter`

| Field | Default | Limits | Passes when |
| --- | --- | --- | --- |
| `min_area` | `0` | 0 to 1 | Any detection's box covers at least this fraction of the frame… |
| `max_area` | `1` | 0 to 1, at least `min_area` | …and at most this fraction |

Useful to ignore distant objects or false detections the size of a few
pixels.

### Zone — `zone_filter`

| Field | Default | Limits | Passes when |
| --- | --- | --- | --- |
| `polygon` | empty | 3 to 100 points, coordinates 0 to 1 relative to the frame | The centre of any detection's box lies inside the polygon |

Draw the polygon in the inspector or on a live frame with **Edit on
preview**. On **Run** the polygon needs at least three points.

### Schedule — `time_window_filter`

| Field | Default | Limits | Meaning |
| --- | --- | --- | --- |
| `start_hour` | `0` | 0 to 23 | **Start hour**, inclusive |
| `end_hour` | `23` | 0 to 23 | **End hour**, inclusive; smaller than `start_hour` wraps past midnight |
| `weekdays` | Monday to Friday | 0 (Monday) to 6 (Sunday), at least one on **Run** | **Weekdays** the window applies |
| `invert` | off | | **Outside schedule**: pass outside the window instead, for after-hours scenarios |

Times are evaluated in `EVENT_TIMEZONE`.

### Debounce — `debounce_filter`

| Field | Default | Limits | Meaning |
| --- | --- | --- | --- |
| `seconds` | `30` | 1 to 3600 | After an event passes, further events from the same camera fail for this long |

The window is kept per node and per camera, so several Debounce nodes in one
workflow are independent.

### Count — `count_threshold_filter` (Enterprise)

| Field | Default | Limits | Passes when |
| --- | --- | --- | --- |
| `polygon` | empty | up to 100 points; fewer than 3 counts the whole frame | |
| `operator` | `gte` | `gte`, `lte` | The number of detections inside the polygon is at least (or at most)… |
| `count` | `1` | 0 to 1000 | …this value |

### Line crossing — `line_crossing_filter` (Enterprise)

| Field | Default | Limits | Passes when |
| --- | --- | --- | --- |
| `line` | empty | exactly 2 points on **Run** | A tracked object's centre moved from one side of the line to the other since the previous check |
| `direction` | `any` | `any`, `in`, `out` | `in` counts an object crossing from the left side of the line to its right side when looking from the first point to the second; `out` counts the opposite direction |

Relies on tracking ids; raise **Checks per second** for fast objects.

### Dwell — `dwell_filter` (Enterprise)

| Field | Default | Limits | Passes when |
| --- | --- | --- | --- |
| `polygon` | empty | 3 to 100 points | |
| `min_seconds` | `60` | 1 to 3600 | A tracked object has stayed inside the polygon for at least this long |

## Actions

Fields marked with a credential type need a stored credential of that type;
[Actions and integrations](actions-and-integrations.md) describes each
integration.

### Alert — `log_alert_action`

| Field | Default | Limits |
| --- | --- | --- |
| `severity` | `warning` | **Severity**: `info`, `warning`, `critical` |
| `cooldown_seconds` | `30` | **Cooldown seconds**, 0 to 3600; `0` records every event |

### Snapshot — `snapshot_action`

| Field | Default |
| --- | --- |
| `annotate` | on: draw detections onto the image (**Annotate**) |

### Webhook — `webhook_action`

| Field | Default | Limits |
| --- | --- | --- |
| `url` | empty | **URL**: an `http://` or `https://` URL on **Run** |
| `method` | `POST` | **Method**: `POST`, `GET`, `PUT` |

### Telegram — `telegram_action`

| Field | Default | Limits |
| --- | --- | --- |
| `credential_id` | none | a `telegram_bot` credential |
| `chat_id` | empty | 1 to 64 characters |
| `message_template` | `[{kind}] camera={camera_id} at {ts}` | up to 4096 characters |

### Email — `email_action`

| Field | Default | Limits |
| --- | --- | --- |
| `credential_id` | none | an `smtp` credential |
| `to` | empty | up to 320 characters |
| `subject_template` | `[{kind}] on camera {camera_id}` | up to 998 characters |
| `body_template` | `{kind} at {ts}`, camera, snapshot | up to 8192 characters |

### MQTT — `mqtt_action`

| Field | Default | Limits |
| --- | --- | --- |
| `credential_id` | none | an `mqtt` credential |
| `topic` | `looksee/events` | 1 to 512 characters |
| `payload_template` | empty: send the event as JSON | up to 8192 characters |

### Discord — `discord_action`

| Field | Default | Limits |
| --- | --- | --- |
| `credential_id` | none | a `discord_webhook` credential |
| `message_template` | `[{kind}] camera={camera_id} at {ts}` | up to 2000 characters |

### Slack — `slack_action` (Enterprise)

| Field | Default | Limits |
| --- | --- | --- |
| `credential_id` | none | a `slack_webhook` credential |
| `message_template` | `[{kind}] camera={camera_id} at {ts}` | up to 4096 characters |

## Limits

A graph holds up to 200 nodes, 400 edges, and 100 comments. Node ids are up
to 128 characters and edge ids up to 256.

## Validation

Saving a graph checks its structure. **Run** additionally checks that every
node can execute. Studio focuses the node an error refers to; the API returns
the code in the `code` field with status 422, or 402 for a licensing error.

| Code | Meaning |
| --- | --- |
| `duplicate_node_ids`, `duplicate_edge_id` | Two nodes or edges share an id |
| `edge_node_missing` | An edge points at a node that does not exist |
| `branch_on_non_filter` | An edge has an If/Else branch but leaves a node that is not a filter |
| `no_camera_source` | The workflow has no Camera |
| `graph_cycle` | Edges form a loop |
| `node_not_runnable` | A required field is empty or invalid; the message names the field |
| `feature_not_licensed` | An Enterprise node without a license key (status 402) |
| `detect_node_missing`, `multiple_detect_nodes` | A Camera reaches no Detect, or more than one |
| `model_not_selected`, `model_unavailable` | A Detect has no model, or its model is not in `models/` |
| `unsupported_event_kinds` | Detect selects events the model does not produce |
| `unsupported_classes` | A Class filter names labels the upstream model does not produce |
| `unsupported_condition_event`, `unsupported_condition_class` | An If / Else condition names an event or label the upstream model does not produce |
| `asset_store_not_configured`, `asset_unavailable` | A File camera without an asset library, or with a missing object |
| `credential_unavailable`, `credential_type_mismatch` | An action's credential is missing or of another type |

Nodes that no Camera reaches are allowed and ignored at run time; Studio
marks them with a warning.
