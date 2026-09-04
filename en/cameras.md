[Documentation](index.md) / Cameras

# Cameras

A Camera node tells LookSee where frames come from. MediaMTX ingests every
source, exposes it to the inference service as one RTSP path per camera, and
serves the same stream to browsers over WebRTC for the live monitor.

## Source types

| Source type | Field | How it is ingested |
| --- | --- | --- |
| **RTSP** | `rtsp://` or `rtsps://` URL | MediaMTX pulls the stream on demand. Most IP cameras and NVRs. |
| **RTMP** | `rtmp://` or `rtmps://` URL | MediaMTX pulls the stream on demand. |
| **SRT** | `srt://` URL | MediaMTX pulls the stream on demand. |
| **Browser webcam** | none | Publisher mode. Studio publishes the browser's camera over WebRTC (WHIP) to MediaMTX. |
| **WHEP** | `http://`, `https://`, `whep://`, or `wheps://` URL | MediaMTX pulls a remote WebRTC endpoint. |
| **File** | an asset key | A video from the asset library is cached locally and looped as a live stream. |

Pull sources are on demand: MediaMTX connects to the upstream when the
inference service or a browser asks for the path and disconnects ten seconds
after the last reader leaves. Credentials for the upstream go into the URL,
`rtsp://user:password@192.168.1.30:554/stream1`, and stay on the server.

## RTSP, RTMP, and SRT

Enter the URL your camera or encoder publishes. Notes for common setups:

- Use the camera's substream (720p or lower) rather than the main stream.
  Detection runs at a few frames per second and does not benefit from 4K,
  while decoding it costs CPU.
- H.264 decodes everywhere. H.265 works for detection but browsers do not
  play it over WebRTC, so the live monitor stays black while alerts still
  arrive.
- MediaMTX pulls RTSP over TCP. If a camera only offers UDP, put it behind a
  relay that speaks TCP.
- A camera on another network needs to be reachable from the machine running
  LookSee; the browser never connects to the camera directly.

## Browser webcam

A camera with source type **Browser webcam** has no URL. When the workflow
runs and the editor or the monitor is open, Studio asks for permission to use
the webcam and publishes it to MediaMTX from the browser. The monitor shows a
**Publish** and **Stop publish** button for these cameras. Closing the tab
stops the stream; the camera goes back to **Pending** until a browser
publishes again.

The browser publishes to `WEBRTC_HOST_IP` on port `8889` and needs the ICE
port `8189/udp` open; see [Networking](#networking).

## Video files

A **File** camera plays an object from the asset library on a loop, which is
useful for testing a workflow against a recording. The asset library is an
S3-compatible bucket. The compose stack bundles one, the `storage` service
backed by RustFS with the `storage_data` volume, and creates the bucket on
start; set the `S3_*` variables to use an external endpoint such as
Cloudflare R2 or MinIO instead. Without a configured bucket the **Assets**
picker and the `/assets` endpoints are disabled; when the bucket is
configured but unreachable, they answer `503`.

Files are uploaded through Studio or `POST /assets`. On start the API
downloads the object into the media cache volume, which MediaMTX reads to
loop the file with ffmpeg. Object keys are restricted to letters, digits,
`.`, `_`, `-`, and `/` because they end up inside that ffmpeg command.

## MediaMTX

MediaMTX runs as the `mediamtx` service with the configuration in
`docker/mediamtx/mediamtx.yml`. LookSee manages its paths through the control
API on port `9997`; the file itself only sets protocols and authentication:

- RTSP on `8554` over TCP; WebRTC on `8889` with ICE on `8189/udp`. RTMP,
  HLS, and SRT listeners are off, which does not affect pulling from RTMP or
  SRT upstreams.
- Authentication is delegated to the API: MediaMTX calls
  `/internal/media/auth` for every reader, publisher, and control API
  request. The inference service and the API use the service user,
  `MTX_MEDIA_USER` with `MTX_MEDIA_PASSWORD`; browsers present a
  short-lived, camera-scoped grant issued by the API.
  [Security](security.md#media-access) has the details.
- The control API is open to loopback and private network ranges without
  authentication; compose binds it to `127.0.0.1`.

Every camera has the path name of its camera id. The inference service reads
`rtsp://mediamtx:8554/<camera id>` and browsers play
`http://<WEBRTC_HOST_IP>:8889/<camera id>/whep`.

## Networking

Live video reaches the browser over WebRTC, which needs more than an open
web port:

| Port | Protocol | Who connects |
| --- | --- | --- |
| `8889` | TCP | Browsers, for WebRTC signalling and playback (WHEP) and webcam publishing (WHIP) |
| `8189` | UDP | Browsers, for the WebRTC media (ICE) |
| `8554` | TCP | Encoders that push RTSP into the path of a **Browser webcam** camera (publisher mode); not needed for pull sources |

`WEBRTC_HOST_IP` must be the address browsers can reach: `127.0.0.1` on a
laptop, the LAN address on a server. MediaMTX advertises it in ICE
candidates; a wrong value results in a monitor that connects and then shows
**Connection disconnected**. [Deployment](deployment.md) covers reverse
proxies and TLS.

## Camera status

The monitor shows the status badge next to the camera name and the workflow
list shows it per workflow.

- **Pending** (**Starting** in the monitor): the worker is starting. For pull sources the first frame
  usually arrives within a few seconds; the inference service gives up after
  `FIRST_FRAME_TIMEOUT_SECONDS` (30 seconds) and reports an error.
- **Active** (**Live** in the monitor): frames are flowing.
- **Error**: the stream failed or stalled for 30 seconds. The API retries
  with a delay that doubles from 30 seconds up to 5 minutes. The reason is
  shown in the monitor and in the `api` and `inference` logs.
- **Off**: the workflow is stopped.

[Troubleshooting](troubleshooting.md) lists the common causes of **Pending**
and **Error**.
