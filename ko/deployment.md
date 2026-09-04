[문서](index.md) / 배포

# 배포

[시작하기](getting-started.md)는 한 머신에서 LookSee를 실행합니다. 이 페이지는 다른 사람들이 사용하는 서버를 다룹니다. 주소 지정, 리버스 프록시를 통한 TLS, GPU 추론, 백업, 업그레이드입니다.

## 네트워크 내 서버

`.env`의 `WEBRTC_HOST_IP`를 사용자가 서버에 접근하는 주소, 예를 들어 `192.168.1.20`으로 설정합니다. 브라우저는 실시간 영상을 위해 이 주소에 연결하고, 기본 `RUNTIME_*` URL은 Studio를 이 주소로 향하게 합니다. 예제 비밀번호를 바꾼 뒤 스택을 시작합니다.

```bash
docker compose up -d --build
```

Studio는 `http://192.168.1.20:3000`에 있습니다. 이 구성에서는 브라우저 웹캠이 동작하지 않습니다. 브라우저는 `localhost` 또는 HTTPS에서만 카메라 접근을 허용하므로, 웹캠 워크플로에는 아래의 TLS 설정이 필요합니다.

## 리버스 프록시를 통한 TLS

Studio, API, MediaMTX의 WebRTC 포트 앞에 둔 리버스 프록시에서 TLS를 종단합니다. 예제는 [Caddy](https://caddyserver.com/)를 사용합니다. Caddy는 추가 구성 없이 인증서를 발급받고 WebSocket을 프록시합니다. WebSocket 업그레이드를 전달하는 프록시라면 무엇이든 동작합니다.

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

`.env`에서 Studio와 API를 공개 이름으로 향하게 합니다.

```bash
WEBRTC_HOST_IP=203.0.113.10          # the server's public address, for WebRTC media
RUNTIME_API_URL=https://api.example.com
RUNTIME_WS_URL=wss://api.example.com
RUNTIME_MEDIAMTX_WEBRTC_URL=https://media.example.com
CORS_ORIGIN_REGEX=^https://studio\.example\.com$
AUTH_COOKIE_SECURE=true
```

세 이름은 등록 가능 도메인 `example.com`을 공유하므로 API가 설정한 세션 쿠키가 Studio의 요청에 함께 전송됩니다. WebRTC 시그널링은 프록시를 거치지만, 미디어 자체는 UDP 포트 `8189`를 통해 `WEBRTC_HOST_IP`로 직접 흐릅니다. 방화벽에서 이 포트를 열고 프록시하지 마십시오.

`.env`를 편집한 뒤 영향을 받는 서비스를 재시작합니다.

```bash
docker compose up -d
```

## GPU 추론

공개된 추론 이미지는 CPU에서 모델을 실행합니다. ONNX Runtime은 CUDA가 있으면 이를 우선 사용하며, `looksee-inference` 패키지에는 `onnxruntime` 대신 `onnxruntime-gpu`를 설치하는 `gpu` extra가 있습니다. Dockerfile은 이를 위해 두 빌드 인수를 제공합니다.

| 인수 | 기본값 | 용도 |
| --- | --- | --- |
| `TARGET` | `cpu` | 설치할 extra: `cpu` 또는 `gpu` |
| `BASE_IMAGE` | `python:3.12-slim-trixie` | 베이스 이미지. GPU 빌드에는 ONNX Runtime 릴리스와 맞는 CUDA 런타임과 cuDNN, 그리고 Python 3.12가 포함된 이미지가 필요합니다 |

GPU를 통과시키는 compose override로 빌드하고 실행합니다.

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

호스트에는 NVIDIA 드라이버와 NVIDIA Container Toolkit이 필요합니다. 시작 시 추론 로그가 ONNX Runtime이 찾은 실행 공급자를 나열하며, `CUDAExecutionProvider`가 있으면 GPU가 사용 중임을 확인할 수 있습니다.

## 용량 산정

탐지 비용은 모든 카메라에 걸친 초당 모델 실행 횟수, 즉 모든 Detect 노드의 **Checks per second**(초당 검사 횟수) 합입니다. 디코딩 비용은 탐지와 무관하게 스트림 해상도와 프레임 속도에 따라 달라집니다. CPU 배포를 여유 있게 유지하는 두 가지 방법:

- 720p 이하의 카메라 서브스트림을 사용합니다.
- 시나리오가 더 필요로 하지 않는 한 **Checks per second**를 1~2로 유지합니다.

카메라를 추가할 때마다 `INFERENCE_CPUS`와 `INFERENCE_MEMORY`를 높입니다. API, PostgreSQL, Valkey는 그에 비해 가볍습니다. API 복제본은 하나만 실행합니다. 필터 상태와 재알림 대기 시간이 API의 메모리에 있기 때문입니다.

## 백업

상태는 백업할 가치가 있는 세 볼륨 `postgres_data`, `api_keys`, 그리고 업로드한 비디오가 있는 `storage_data`에 있으며, 증거 이미지를 보관한다면 `api_snapshots`도 포함됩니다. Compose는 볼륨 이름 앞에 프로젝트 이름을 붙이며, 기본값은 `looksee`입니다.

```bash
# Database
docker compose exec -T postgres pg_dump -U looksee looksee > looksee.sql

# Signing and encryption secret, and snapshots
docker run --rm -v looksee_api_keys:/data -v "$PWD":/backup alpine \
  tar czf /backup/api_keys.tgz -C /data .
docker run --rm -v looksee_api_snapshots:/data -v "$PWD":/backup alpine \
  tar czf /backup/api_snapshots.tgz -C /data .

# Uploaded videos
docker run --rm -v looksee_storage_data:/data -v "$PWD":/backup alpine \
  tar czf /backup/storage_data.tgz -C /data .
```

새 스택으로 복원하려면 `postgres`만 먼저 시작하고, `psql`로 덤프를 불러오고, 같은 방식으로 아카이브를 볼륨에 풀어 넣은 뒤, 나머지를 시작합니다. 생성된 파일에 의존하는 대신 `.env`에 `SECRET_KEY`를 설정하면 키가 구성 백업의 일부가 됩니다.

## 업그레이드

```bash
git pull
docker compose up -d --build
```

`api-migrate` 서비스가 API 시작 전에 데이터베이스 마이그레이션을 적용합니다. API, 추론 서비스, Studio, MediaMTX는 계약이 함께 바뀌므로 같은 버전으로 실행해야 합니다. 먼저 [변경 이력](changelog.md)을 읽으십시오. 릴리스가 `.env`에 필수 변수를 추가할 수 있으며, 그 변수 없이는 compose가 시작을 거부합니다. 이후 `docker compose ps`를 확인합니다.

## 네이티브 서비스

개발할 때는 인프라를 컨테이너에서 실행하고 서비스는 소스 트리에서 실행할 수 있습니다. 저장소의 [CONTRIBUTING.md](https://github.com/okoflow/looksee/blob/main/CONTRIBUTING.md#development-setup)가 그 설정을 설명합니다.
