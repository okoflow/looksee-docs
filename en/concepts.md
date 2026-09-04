[Documentation](index.md) / Concepts

# Concepts

This page explains the vocabulary that the rest of the documentation uses:
workflows and nodes, events and how models produce them, cameras and runs,
and the two editions.

## Workflows

A workflow is a graph that describes one analytics task: which camera to
watch, which model to run, which detections matter, and what to do about them.
Workflows are created and edited in Studio and stored by the API. A workflow
is either running (**Active**) or stopped (**Off**); **Run** in the editor
starts it, **Stop** stops it.

A workflow has a name, an optional description, and a graph of nodes and
edges. Canvas comments can be added for documentation and are not executed.

## Nodes and edges

Nodes are the boxes on the canvas. Each node has a kind and a set of fields
that the inspector on the right edits. Kinds fall into four roles:

| Role | Nodes | Purpose |
| --- | --- | --- |
| Source | Camera | Where frames come from |
| Detection | Detect | Which model runs and which of its events pass |
| Filter | If / Else, Class, Size, Zone, Schedule, Debounce; Count, Line crossing, Dwell in the Enterprise edition | Decide whether an event continues |
| Action | Alert, Snapshot, Webhook, Telegram, Email, MQTT, Discord; Slack in the Enterprise edition | Do something with an event |

Edges connect nodes left to right. A Camera connects to exactly one Detect.
Detect fans out to any number of filters and actions. Every filter has two
outputs, **If** and **Else**: an event that passes the filter continues along
**If**, an event that fails continues along **Else**. An edge drawn from a
filter without choosing a branch follows **If**. Snapshot is the one action
with an output: actions connected after it receive the snapshot it took.
Other actions are endpoints.

[Nodes](nodes.md) lists every node with its fields and limits.

## Events

Detection produces events, and events are what flows through the graph. The
inference service reports detections for every processed frame: a label, a
bounding box, a confidence, and a tracking id. The API groups the detections
of a frame by label and derives one event per label, named after it:
`helmet` becomes `HELMET_DETECTED`, `space-empty` becomes
`SPACE_EMPTY_DETECTED`. A model manifest can rename or suppress the event of
a label. There is no fixed list of event kinds; the models you install define
the vocabulary.

An event carries its kind, the camera, the timestamp, the frame size, the
detections behind it, and a count. Filters look at these fields: Zone tests
the centre of each detection against a polygon, Class tests the labels, Size
tests the box area, If / Else compares the kind, the class, the count, or the
highest confidence.

Every event enters the graph; three windows keep repeats in check:

- **Event cooldown** (`EVENT_COOLDOWN_SECONDS`, 2 seconds by default) limits
  how often the live event feed shows the same event kind on the same
  camera. It does not hold events back from the graph.
- **Debounce** is a filter you place where you want it, with its own window.
- **Alert cooldown** is a field on the Alert action and applies to the alert
  history only.

Filter state and cooldowns live in the memory of the API process. They reset
when the API restarts and are not shared between API replicas.

## Cameras and runs

Every Camera node in a workflow becomes a camera record with a status:

| Status | Meaning |
| --- | --- |
| **Pending** | The workflow is running and the inference service has not confirmed the stream yet |
| **Active** | Frames are being processed |
| **Error** | The stream failed; the API retries with a growing delay |
| **Off** | The workflow is stopped |

The monitor shows the same states as **Starting**, **Live**, **Error**, and
**Off**; the API reports them as `pending`, `active`, `error`, and `disabled`.

Starting a workflow asks MediaMTX to expose one path per camera and asks the
inference service to start one worker per camera. Each start is a *run* with
a revision number, so a stopped or reconfigured camera cannot be resurrected
by a late message from an older run. A reconcile loop in the API republishes
the desired state every 30 seconds, which is what brings cameras back after a
restart of any service.

Changing a running workflow's detection settings applies in place. Changing a
camera's source restarts its worker.

[Cameras](cameras.md) covers the source types and how MediaMTX handles them.

## Models

A model is an ONNX file plus a manifest, placed in `models/`. The manifest
declares the labels the model produces and, optionally, a display name, a
recommended confidence threshold, and event overrides. The API discovers
bundles at runtime; Detect nodes pick a model from that catalog.

Detection runs at the rate the Detect node sets (**Checks per second**), not at
the camera frame rate. Objects are tracked between checks so filters that
need identity, such as line crossing and dwell, can follow one object over
time.

[Models](models.md) covers the manifest format and exporting.

## Credentials

Actions that talk to an external service (Telegram, Discord, email, MQTT,
Slack) reference a credential instead of holding a token. Credentials are
stored encrypted, shown only by name and a non-secret summary, and resolved
at delivery time, so rotating a token takes effect immediately.

[Actions and integrations](actions-and-integrations.md) covers each type.

## Editions

The Community edition contains everything outside the `ee/` directory of the
repository and is licensed under Apache-2.0. The Enterprise edition adds the
Count, Line crossing, and Dwell filters and the Slack action, shown with a
lock in the node palette until a license key is configured. Both editions
ship in the same images. [Enterprise edition](enterprise.md) has the details.
