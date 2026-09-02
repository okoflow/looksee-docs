[Documentation](index.md) / Troubleshooting

# Troubleshooting

Symptoms, likely causes, and what to check. Most answers are in the logs:

```bash
docker compose logs -f api inference mediamtx studio
```

## A camera stays Pending

The inference service has not received a first frame.

- **The server cannot reach the camera.** Test from the host with
  `ffprobe rtsp://user:password@camera/stream`. The browser reaching the
  camera proves nothing; the server pulls it.
- **The camera only speaks RTSP over UDP.** MediaMTX pulls over TCP. Enable
  TCP on the camera or relay it.
- **Wrong credentials or path** in the URL. MediaMTX logs the upstream error.
- **The first frame took longer than 30 seconds.** Raise
  `FIRST_FRAME_TIMEOUT_SECONDS` for slow cameras.
- **A Browser webcam camera without a publisher.** Open the workflow's
  monitor in a browser and allow camera access; the camera activates when a
  browser publishes.

## A camera shows Error

The monitor and the `inference` log show the reason.

- **no frames from stream for 30s**: the stream stalled. The camera
  reconnects on its own and the API retries with a growing delay up to five
  minutes; fix the network or camera if it repeats.
- **unsupported ONNX signature**: the model export is not one of the
  supported D-FINE layouts. See [Models](models.md#supported-onnx-exports).
- **model not found**: the bundle disappeared or the `models/` mount is
  missing.

## The monitor shows no video

Alerts arrive but the picture is black or shows **Connection disconnected**.

- **`WEBRTC_HOST_IP` is wrong.** It must be the address the browser can reach.
  The value is baked into WebRTC candidates; `127.0.0.1` only works on the
  server itself.
- **UDP port `8189` is blocked** between the browser and the server. WebRTC
  media does not go through the HTTP proxy.
- **The stream is H.265.** Browsers do not play it over WebRTC. Switch the
  camera to H.264 or use the substream; detection keeps working either way.
- **No active stream** for a Browser webcam camera means nothing is
  publishing yet.
- **Different origins.** If Studio and MediaMTX are served under different
  schemes (`https` Studio, `http` MediaMTX), the browser blocks the
  connection. Proxy both, as in [Deployment](deployment.md#tls-with-a-reverse-proxy).

## Webcam access is denied

Browsers allow `getUserMedia` only on `localhost` or over HTTPS. Serve Studio
over TLS for remote users, and check the browser's site permissions after a
denied prompt.

## A model is missing from the Detect node

- The bundle directory must contain both `manifest.json` and `model.onnx`.
- The manifest failed validation: unknown field, duplicate labels, gaps in
  class ids, or an `events` key that is not a label. The `api` log names the
  problem.
- The directory name contains characters outside lowercase letters, digits,
  `-`, and `_`.

## Run fails

**Run** shows a message and focuses the node it refers to.

- **Status 402, `feature_not_licensed`**: the graph contains Count, Line
  crossing, Dwell, or Slack without a license. See
  [Enterprise edition](enterprise.md).
- **Status 422** with a code: the graph cannot run as drawn. The
  [validation table](nodes.md#validation) explains each code; the common ones
  are a camera without a Detect node, a Detect node without a model, and an
  action without a credential.

## Alerts are not delivered

- **The Alert action fires but Telegram, Discord, email, MQTT, or Slack stay
  silent.** The credential is missing, of the wrong type, or the service
  rejected the request. The `api` log records every delivery failure without
  the secret.
- **Telegram** needs the chat id of a chat the bot is a member of; send the
  bot a message first.
- **Email** needs a server that accepts the `from_address`; many providers
  require STARTTLS on port 587.
- **Webhook** targets must answer within five seconds.
- **Nothing fires at all.** Check that the event kind selected in Detect is
  one the model produces, and that a filter is not sending everything to
  **Else**.

## Too many or too few alerts

- The **event cooldown** (`EVENT_COOLDOWN_SECONDS`) drops repeats of the same
  kind within two seconds by default.
- The Alert action's **cooldown** suppresses repeats per camera; `0` records
  everything.
- A **Debounce** filter gives any branch its own window.
- Raise **Confidence** when shadows or reflections trigger detections.

## Snapshots are missing

- The Snapshot action must come before the actions that use it in the graph.
- The latest frame of a camera is kept for `LAST_FRAME_TTL_SECONDS` (10
  seconds). A camera that just stopped has no frame to snapshot.
- Snapshot files live in the `api_snapshots` volume; the URL in an alert is
  relative to the API origin.

## Schedule filters fire at the wrong time

Schedules use `EVENT_TIMEZONE`, which defaults to UTC. Set it to the site's
IANA time zone, such as `America/Chicago`, and restart the API.

## Forgotten owner password

There is no reset flow. Remove the owner row and run setup again; workflows,
credentials, and alerts are kept.

```bash
docker compose exec postgres psql -U looksee -d looksee -c "DELETE FROM users;"
```

> [!WARNING]
> Until you complete the setup page again, anyone who can reach Studio can
> claim the instance.

## High CPU usage

- Lower **Checks per second** on Detect nodes.
- Use camera substreams at 720p or lower; decoding 4K costs more than
  detection.
- Give the inference container more CPUs with `INFERENCE_CPUS`, or move to a
  GPU as described in [Deployment](deployment.md#gpu-inference).

## A port is already in use

Change the host port in `.env`: `WEB_PORT`, `API_PORT`, `POSTGRES_PORT`,
`REDIS_PORT`, `MTX_RTSP_PORT`, `MTX_WEBRTC_PORT`, `MTX_WEBRTC_ICE_PORT`. When
`MTX_WEBRTC_PORT` changes, set `RUNTIME_MEDIAMTX_WEBRTC_URL` to match.
