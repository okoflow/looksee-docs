[Documentation](index.md) / Configuration

# Configuration

LookSee is configured through environment variables. With Docker Compose they
come from `.env` next to `compose.yaml`; `.env.example` lists the variables
an installation sets, grouped by service, and the tables below cover the
rest with their defaults. Services read their variables at start, so a
change takes effect after `docker compose up -d`.

## Compose stack

These variables shape the stack itself: images, ports, passwords, and
resource limits.

| Variable | Default | Meaning |
| --- | --- | --- |
| `WEBRTC_HOST_IP` | required | Address browsers use to reach MediaMTX for live video: `127.0.0.1` on one machine, the LAN address on a server. Also the default host in the `RUNTIME_*` URLs. |
| `POSTGRES_PASSWORD` | required | Database password. |
| `MTX_MEDIA_PASSWORD` | required | Password of the MediaMTX service user, shared by the API, the inference service, and MediaMTX. Never sent to browsers. |
| `MTX_MEDIA_USER` | `media` | MediaMTX service user with read and publish rights. |
| `STORAGE_PASSWORD` | required | Secret key of the bundled video storage; the access key is `looksee`. |
| `STORAGE_PORT` | `9000` | Host port for the storage's S3 API, bound to `127.0.0.1`. |
| `POSTGRES_USER`, `POSTGRES_DB` | `looksee` | Database user and name. |
| `POSTGRES_PORT` | `5432` | Host port for PostgreSQL, bound to `127.0.0.1`. |
| `REDIS_PORT` | `6379` | Host port for Valkey, bound to `127.0.0.1`. |
| `REDIS_MAXMEMORY` | `512mb` | Valkey memory limit; the eviction policy is `noeviction`. |
| `MTX_RTSP_PORT` | `8554` | RTSP port. |
| `MTX_WEBRTC_PORT` | `8889` | WebRTC signalling and playback port. |
| `MTX_WEBRTC_ICE_PORT` | `8189` | WebRTC media port (UDP). |
| `MTX_LOGLEVEL` | `info` | MediaMTX log level. |
| `MTX_AUTHHTTPADDRESS` | `http://api:8000/internal/media/auth` | Where MediaMTX asks for authorization. Change it only when the API runs outside compose. |
| `API_PORT` | `8000` | Host port for the API. |
| `WEB_PORT` | `3000` | Host port for Studio. |
| `INFERENCE_CPUS` | `4.0` | CPU limit of the inference container; also caps ONNX Runtime threads. |
| `INFERENCE_MEMORY` | `4g` | Memory limit of the inference container. |
| `REGISTRY`, `TAG` | `looksee`, `latest` | Image name prefix and tag for the three application images. |

Other services are limited too: `api` 2 CPUs and 1 GB, `postgres` 2 CPUs and
2 GB, `mediamtx` 2 CPUs and 1 GB, `storage` 2 CPUs and 1 GB, `redis` 1 CPU
and 768 MB, `studio` 1 CPU and 512 MB. Edit `compose.yaml` or add an override file to change them.

## API

| Variable | Default | Meaning |
| --- | --- | --- |
| `LICENSE_KEY` | unset | Enterprise license key. Unset or blank runs the Community edition. |
| `DATABASE_URL` | set by compose | SQLAlchemy URL, `postgresql+asyncpg://user:password@host:5432/looksee`. |
| `REDIS_URL` | set by compose | Valkey URL, `redis://host:6379/0`. |
| `MEDIAMTX_API_URL` | set by compose | MediaMTX control API, `http://mediamtx:9997`. |
| `MODELS_DIR` | `/app/models` | Model bundle directory. |
| `SNAPSHOTS_DIR` | `/data/snapshots` | Where the Snapshot action writes JPEG files; served at `/snapshots`. |
| `SECRET_KEY` | unset | Root secret for session signing and credential encryption. Unset, a secret is generated on first start and stored in `SECRET_KEY_FILE`. |
| `SECRET_KEY_FILE` | `/data/keys/secret.key` | Location of the generated secret, on the `api_keys` volume. |
| `AUTH_COOKIE_SECURE` | `false` | Mark the session cookie `Secure`. Set to `true` behind HTTPS. |
| `CORS_ORIGIN_REGEX` | `^https?://(localhost\|127\.0\.0\.1)(:\d+)?$` | Origins allowed to call the API from a browser. Requests that change state and WebSocket connections from other origins are rejected. Set it to the Studio origin when it is not localhost. |
| `EVENT_COOLDOWN_SECONDS` | `2` | Minimum gap between two `event` messages of the same kind on the same camera in the live feed. `0` disables the cooldown. The graph receives every event. |
| `EVENT_TIMEZONE` | `UTC` | Time zone for the Schedule filter, an IANA name such as `Europe/Berlin`. |
| `RECONCILE_INTERVAL_SECONDS` | `30` | How often the API republishes the desired camera state and retries failed cameras. |
| `CONSUMER_GROUP` | `api-workers` | Valkey stream consumer group for detection frames. |
| `S3_ENDPOINT_URL`, `S3_BUCKET` | `http://storage:9000`, `looksee` | Asset library for File cameras. Compose points them at the bundled storage; set both to use an external S3-compatible bucket. Natively, both must be set to enable the library. |
| `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY` | `looksee`, `STORAGE_PASSWORD` | Asset library credentials. |
| `S3_REGION` | `auto` | Region of an external bucket; `auto` for Cloudflare R2. |
| `S3_PREFIX` | empty | Key prefix inside the bucket. |
| `MEDIA_CACHE_DIR` | set by compose | Where downloaded assets are cached for playback. |
| `MEDIA_MOUNT_DIR` | `/media` | Where MediaMTX sees the same cache. |

## Inference

| Variable | Default | Meaning |
| --- | --- | --- |
| `REDIS_URL` | set by compose | Valkey URL. |
| `MEDIAMTX_RTSP_URL` | `rtsp://mediamtx:8554` | Where camera paths are read from. |
| `MTX_MEDIA_USER`, `MTX_MEDIA_PASSWORD` | from `.env` | Credentials for reading camera paths. |
| `RTSP_TRANSPORT` | `tcp` | `tcp` or `udp` for reading from MediaMTX. |
| `MODELS_DIR` | `/app/models` | Model bundle directory. |
| `FIRST_FRAME_TIMEOUT_SECONDS` | `30` | How long to wait for the first frame, and for the next frame once running, before reporting an error. |
| `LAST_FRAME_TTL_SECONDS` | `10` | How long the latest JPEG of each camera stays available for the Snapshot action. |

## Studio

Studio reads its configuration on the server for every request and passes the
public part to the browser, so an image can be repointed without a rebuild.

| Variable | Default | Meaning |
| --- | --- | --- |
| `RUNTIME_API_URL` | `http://<WEBRTC_HOST_IP>:<API_PORT>` | API base URL as seen from the browser. |
| `RUNTIME_WS_URL` | `ws://<WEBRTC_HOST_IP>:<API_PORT>` | WebSocket base URL as seen from the browser. |
| `RUNTIME_MEDIAMTX_WEBRTC_URL` | `http://<WEBRTC_HOST_IP>:<MTX_WEBRTC_PORT>` | MediaMTX WebRTC URL as seen from the browser. |
| `RUNTIME_DOCS_URL` | `https://github.com/okoflow/looksee-docs` | Target of the **Documentation** link in the sidebar; compose pins it to the documentation repository. |
| `RUNTIME_GITHUB_URL` | `https://github.com/okoflow/looksee` | Target of the **GitHub** link. |
| `SERVER_API_URL` | `http://api:8000` | API address used by the Studio server itself for the sign-in guard. Never sent to the browser. |

Behind a reverse proxy, set the three `RUNTIME_*` URLs to the public
addresses; [Deployment](deployment.md) has an example.

## Ports

| Port | Service | Bound to | Purpose |
| --- | --- | --- | --- |
| `3000` | studio | all interfaces | Web interface |
| `8000` | api | all interfaces | HTTP API and WebSocket |
| `8554` | mediamtx | all interfaces | RTSP |
| `8889` | mediamtx | all interfaces | WebRTC signalling and playback |
| `8189/udp` | mediamtx | all interfaces | WebRTC media |
| `9997` | mediamtx | `127.0.0.1` | MediaMTX control API |
| `9000` | storage | `127.0.0.1` | S3 API of the video storage |
| `5432` | postgres | `127.0.0.1` | PostgreSQL |
| `6379` | redis | `127.0.0.1` | Valkey |

## Volumes

| Volume | Service | Contents |
| --- | --- | --- |
| `postgres_data` | postgres | Workflows, cameras, credentials, alerts, users, queued deliveries |
| `storage_data` | storage | Uploaded video files for File cameras |
| `redis_data` | redis | Detection stream and command channels; safe to lose |
| `api_snapshots` | api | Snapshot JPEG files |
| `api_keys` | api | The generated `secret.key` |
| `media-cache` | api, mediamtx | Cached video files for File cameras |
| `./models` (bind) | api, inference | Model bundles, read-only |

`postgres_data`, `api_keys`, and `storage_data` hold the state worth backing
up; without the secret, stored credentials cannot be decrypted and every
session is signed out. [Deployment](deployment.md#backups) describes a backup routine.
