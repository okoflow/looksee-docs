[Documentation](index.md) / Deployment

# Deployment

[Getting started](getting-started.md) runs LookSee on one machine. This page
covers a server that other people use: addressing, TLS through a reverse
proxy, GPU inference, backups, and upgrades.

## A server on your network

Set `WEBRTC_HOST_IP` in `.env` to the address users reach the server by,
such as `192.168.1.20`. Browsers connect to that address for live video, and
the default `RUNTIME_*` URLs point Studio at it. Change the example
passwords, then start the stack:

```bash
docker compose up -d --build
```

Studio is at `http://192.168.1.20:3000`. Browser webcams do not work in this
setup: browsers allow camera access only on `localhost` or over HTTPS, so a
webcam workflow needs the TLS setup below.

## TLS with a reverse proxy

Terminate TLS in a reverse proxy in front of Studio, the API, and MediaMTX's
WebRTC port. The example uses [Caddy](https://caddyserver.com/), which
obtains certificates and proxies WebSockets without extra configuration; any
proxy that forwards WebSocket upgrades works.

```caddyfile
studio.example.com {
    reverse_proxy 127.0.0.1:3000
}

api.example.com {
    reverse_proxy 127.0.0.1:8000
}

media.example.com {
    reverse_proxy 127.0.0.1:8889
}
```

Point Studio and the API at the public names in `.env`:

```bash
WEBRTC_HOST_IP=203.0.113.10          # the server's public address, for WebRTC media
RUNTIME_API_URL=https://api.example.com
RUNTIME_WS_URL=wss://api.example.com
RUNTIME_MEDIAMTX_WEBRTC_URL=https://media.example.com
CORS_ORIGIN_REGEX=^https://studio\.example\.com$
AUTH_COOKIE_SECURE=true
```

The three names share the registrable domain `example.com`, so the session
cookie set by the API is sent with Studio's requests. WebRTC signalling goes
through the proxy; the media itself flows over UDP port `8189` straight to
`WEBRTC_HOST_IP`, so open that port on the firewall and do not proxy it.

Restart the affected services after editing `.env`:

```bash
docker compose up -d
```

## GPU inference

The published inference image runs models on the CPU. ONNX Runtime prefers
CUDA when it is available, and the `looksee-inference` package has a `gpu`
extra that installs `onnxruntime-gpu` instead of `onnxruntime`. The
Dockerfile exposes two build arguments for this:

| Argument | Default | Purpose |
| --- | --- | --- |
| `TARGET` | `cpu` | Which extra to install: `cpu` or `gpu` |
| `BASE_IMAGE` | `python:3.12-slim-trixie` | Base image; a GPU build needs one with the CUDA runtime and cuDNN that match the ONNX Runtime release, plus Python 3.12 |

Build and run with a compose override that passes the GPU through:

```yaml
# compose.gpu.yaml
services:
  inference:
    build:
      args:
        TARGET: gpu
        BASE_IMAGE: <cuda base image with python 3.12>
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
```

```bash
docker compose -f compose.yaml -f compose.gpu.yaml up -d --build inference
```

The host needs the NVIDIA driver and the NVIDIA Container Toolkit. On
startup the inference log lists the execution providers ONNX Runtime found;
`CUDAExecutionProvider` confirms the GPU is in use.

## Sizing

Detection cost is the number of model runs per second across all cameras:
the sum of every Detect node's **Checks per second**. Decoding cost depends on
the stream resolution and frame rate, independent of detection. Two levers
keep a CPU deployment comfortable:

- Use camera substreams at 720p or lower.
- Keep **Checks per second** at 1 to 2 unless the scenario needs more.

Raise `INFERENCE_CPUS` and `INFERENCE_MEMORY` as cameras are added. The API,
PostgreSQL, and Valkey are light by comparison. Run one API replica: event
cooldowns and tracking state live in the API's memory.

## Backups

State lives in two volumes worth backing up, `postgres_data` and `api_keys`,
plus `api_snapshots` if you keep evidence images. Compose prefixes volume
names with the project name, `looksee` by default.

```bash
# Database
docker compose exec -T postgres pg_dump -U looksee looksee > looksee.sql

# Signing and encryption secret, and snapshots
docker run --rm -v looksee_api_keys:/data -v "$PWD":/backup alpine \
  tar czf /backup/api_keys.tgz -C /data .
docker run --rm -v looksee_api_snapshots:/data -v "$PWD":/backup alpine \
  tar czf /backup/api_snapshots.tgz -C /data .
```

Restore into a fresh stack by starting `postgres` alone, loading the dump
with `psql`, extracting the archives into the volumes the same way, and then
starting the rest. Setting `SECRET_KEY` in `.env` instead of relying on the
generated file makes the key part of your configuration backup.

## Upgrades

```bash
git pull
docker compose up -d --build
```

The `api-migrate` service applies database migrations before the API starts,
and the API and inference service must run the same version because their
message contracts change together. Check `docker compose ps` afterwards; the
[changelog](changelog.md) lists changes that need operator action.

## Native services

For development, the infrastructure can run in containers while the
services run from the source tree. The repository's
[CONTRIBUTING.md](https://github.com/okoflow/looksee/blob/main/CONTRIBUTING.md#development-setup)
describes that setup.
