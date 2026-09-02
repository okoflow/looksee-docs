[Documentation](index.md) / Getting started

# Getting started

This page takes LookSee from a fresh machine to a signed-in Studio with a
detection model ready to use. It uses Docker Compose, which is the supported
way to run LookSee.

## Requirements

- Docker Engine with Docker Compose 2.24 or later. Docker Desktop works on
  macOS and Windows; Linux servers are the usual production target.
- A 64-bit x86 or ARM processor. Detection runs on the CPU by default; a
  CUDA GPU is optional and covered in [Deployment](deployment.md#gpu-inference).
- Free ports `3000` (Studio), `8000` (API), `8554` (RTSP), `8889` (WebRTC),
  and `8189/udp` (WebRTC ICE). PostgreSQL, Valkey, and the MediaMTX control
  API bind to the loopback interface only.

The compose file limits every service. The inference service gets 4 CPUs and
4 GB of memory by default; raise `INFERENCE_CPUS` and `INFERENCE_MEMORY` in
`.env` for more cameras or larger models.

## Install

```bash
git clone https://github.com/okoflow/looksee.git
cd looksee
cp .env.example .env
```

Open `.env` and set two things before the first start:

| Variable | Set it to |
| --- | --- |
| `WEBRTC_HOST_IP` | `127.0.0.1` when the browser runs on the same machine. The machine's address on your network, such as `192.168.1.20`, when you open Studio from another device. Browsers connect to this address for live video. |
| `POSTGRES_PASSWORD`, `MTX_MEDIA_PASSWORD` | Your own values. The example values are placeholders, and the MediaMTX password is visible to browsers that load Studio. |

Then build and start the stack:

```bash
docker compose up -d --build
docker compose ps
```

The first build downloads base images and Python and Node packages and takes
a few minutes. `docker compose ps` shows every service as `healthy` once the
stack is ready: `postgres`, `redis`, `mediamtx`, `api`, `inference`, and
`studio`. The one-shot `api-migrate` and `media-cache-init` services exit
after they finish.

## Create the owner account

Open `http://<WEBRTC_HOST_IP>:3000`. Studio redirects to the setup page while
no account exists. Enter an email address, a display name, and a password of
at least eight characters with one digit and one capital letter. The account
becomes the instance owner and you are signed in.

> [!WARNING]
> Until the owner exists, anyone who can reach port `3000` can claim the
> instance. Create the account right after the first start, and keep the
> stack on a trusted network. [Security](security.md) covers the rest.

![The sign-in page](../images/sign-in.png)

## Add a detection model

LookSee ships without models. A model is a directory under `models/` in the
repository checkout with two files:

```text
models/
└── ppe-helmets/
    ├── manifest.json
    └── model.onnx
```

`model.onnx` is a D-FINE model exported to ONNX. `manifest.json` names the
model and its classes:

```json
{
  "name": "Safety gear (PPE)",
  "labels": ["head", "helmet", "vest"],
  "recommended_confidence_threshold": 0.4
}
```

The `models/` directory is mounted read-only into the API and inference
containers, and the API discovers a new bundle on the next request, so no
restart is needed. Each label becomes an event kind such as
`HELMET_DETECTED`. [Models](models.md) describes the manifest in full and how
to export a model.

## Check the stack

```bash
curl http://127.0.0.1:8000/health
docker compose logs -f api inference
```

The API answers `{"status":"ok"}`. Interactive API documentation is served at
`http://127.0.0.1:8000/docs`.

## Everyday commands

```bash
docker compose logs -f studio api inference mediamtx   # follow logs
docker compose stop                                    # stop, keep data
docker compose up -d                                   # start again
git pull && docker compose up -d --build               # upgrade
docker compose down -v                                 # remove everything, including data
```

Workflows, cameras, credentials, alerts, snapshots, and the signing secret
live in named volumes. `down -v` deletes them; [Deployment](deployment.md)
explains how to back them up.

## Next

[Your first workflow](first-workflow.md) builds a helmet compliance workflow
from the canvas up and shows it running in the monitor.
