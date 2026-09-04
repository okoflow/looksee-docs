[문서](index.md) / 구성

# 구성

LookSee는 환경 변수로 구성합니다. Docker Compose에서는 `compose.yaml` 옆의 `.env`에서 읽으며, `.env.example`은 설치 시 설정하는 변수를 서비스별로 묶어 나열하고, 나머지는 아래 표에 기본값과 함께 있습니다. 서비스는 시작할 때 변수를 읽으므로 변경 사항은 `docker compose up -d` 뒤에 적용됩니다.

## Compose 스택

이 변수들은 스택 자체를 결정합니다. 이미지, 포트, 비밀번호, 리소스 제한입니다.

| 변수 | 기본값 | 의미 |
| --- | --- | --- |
| `WEBRTC_HOST_IP` | 필수 | 브라우저가 실시간 영상을 위해 MediaMTX에 접근하는 주소. 한 머신에서는 `127.0.0.1`, 서버에서는 LAN 주소. `RUNTIME_*` URL의 기본 호스트로도 쓰입니다. |
| `POSTGRES_PASSWORD` | 필수 | 데이터베이스 비밀번호. |
| `MTX_MEDIA_PASSWORD` | 필수 | MediaMTX 서비스 사용자의 비밀번호. API, 추론 서비스, MediaMTX가 공유합니다. 브라우저로는 전송되지 않습니다. |
| `MTX_MEDIA_USER` | `media` | 읽기와 송출 권한이 있는 MediaMTX 서비스 사용자. |
| `STORAGE_PASSWORD` | 필수 | 내장 비디오 저장소의 시크릿 키. 액세스 키는 `looksee`입니다. |
| `STORAGE_PORT` | `9000` | 저장소 S3 API의 호스트 포트. `127.0.0.1`에 바인딩. |
| `POSTGRES_USER`, `POSTGRES_DB` | `looksee` | 데이터베이스 사용자와 이름. |
| `POSTGRES_PORT` | `5432` | PostgreSQL의 호스트 포트. `127.0.0.1`에 바인딩. |
| `REDIS_PORT` | `6379` | Valkey의 호스트 포트. `127.0.0.1`에 바인딩. |
| `REDIS_MAXMEMORY` | `512mb` | Valkey 메모리 제한. 제거 정책은 `noeviction`. |
| `MTX_RTSP_PORT` | `8554` | RTSP 포트. |
| `MTX_WEBRTC_PORT` | `8889` | WebRTC 시그널링과 재생 포트. |
| `MTX_WEBRTC_ICE_PORT` | `8189` | WebRTC 미디어 포트(UDP). |
| `MTX_LOGLEVEL` | `info` | MediaMTX 로그 수준. |
| `MTX_AUTHHTTPADDRESS` | `http://api:8000/internal/media/auth` | MediaMTX가 허가를 묻는 주소. API가 compose 밖에서 실행될 때만 바꿉니다. |
| `API_PORT` | `8000` | API의 호스트 포트. |
| `WEB_PORT` | `3000` | Studio의 호스트 포트. |
| `INFERENCE_CPUS` | `4.0` | 추론 컨테이너의 CPU 제한. ONNX Runtime 스레드 수도 이에 맞춰 제한됩니다. |
| `INFERENCE_MEMORY` | `4g` | 추론 컨테이너의 메모리 제한. |
| `REGISTRY`, `TAG` | `looksee`, `latest` | 세 애플리케이션 이미지의 이름 접두사와 태그. |

다른 서비스에도 제한이 있습니다. `api`는 CPU 2개와 1 GB, `postgres`는 CPU 2개와 2 GB, `mediamtx`는 CPU 2개와 1 GB, `storage`는 CPU 2개와 1 GB, `redis`는 CPU 1개와 768 MB, `studio`는 CPU 1개와 512 MB입니다. 바꾸려면 `compose.yaml`을 편집하거나 override 파일을 추가합니다.

## API

| 변수 | 기본값 | 의미 |
| --- | --- | --- |
| `LICENSE_KEY` | 설정 안 함 | Enterprise 라이선스 키. 설정하지 않거나 비어 있으면 Community 에디션으로 실행됩니다. |
| `DATABASE_URL` | compose가 설정 | SQLAlchemy URL, `postgresql+asyncpg://user:password@host:5432/looksee`. |
| `REDIS_URL` | compose가 설정 | Valkey URL, `redis://host:6379/0`. |
| `MEDIAMTX_API_URL` | compose가 설정 | MediaMTX 제어 API, `http://mediamtx:9997`. |
| `MODELS_DIR` | `/app/models` | 모델 번들 디렉터리. |
| `SNAPSHOTS_DIR` | `/data/snapshots` | Snapshot 액션이 JPEG 파일을 쓰는 곳. `/snapshots`에서 제공됩니다. |
| `SECRET_KEY` | 설정 안 함 | 세션 서명과 자격 증명 암호화의 루트 시크릿. 설정하지 않으면 첫 시작 시 시크릿을 생성해 `SECRET_KEY_FILE`에 저장합니다. |
| `SECRET_KEY_FILE` | `/data/keys/secret.key` | 생성된 시크릿의 위치. `api_keys` 볼륨에 있습니다. |
| `AUTH_COOKIE_SECURE` | `false` | 세션 쿠키에 `Secure`를 표시합니다. HTTPS 뒤에서는 `true`로 설정합니다. |
| `CORS_ORIGIN_REGEX` | `^https?://(localhost\|127\.0\.0\.1)(:\d+)?$` | 브라우저에서 API를 호출할 수 있는 오리진. 다른 오리진에서 오는 상태 변경 요청과 WebSocket 연결은 거부됩니다. Studio가 localhost가 아니면 Studio 오리진으로 설정합니다. |
| `EVENT_COOLDOWN_SECONDS` | `2` | 실시간 피드에서 같은 카메라의 같은 종류 `event` 메시지 두 개 사이의 최소 간격. `0`은 재알림 대기 시간을 비활성화합니다. 그래프는 모든 이벤트를 받습니다. |
| `EVENT_TIMEZONE` | `UTC` | Schedule 필터의 시간대. `Europe/Berlin` 같은 IANA 이름. |
| `RECONCILE_INTERVAL_SECONDS` | `30` | API가 원하는 카메라 상태를 다시 전파하고 실패한 카메라를 재시도하는 주기. |
| `CONSUMER_GROUP` | `api-workers` | 탐지 프레임을 위한 Valkey 스트림 컨슈머 그룹. |
| `S3_ENDPOINT_URL`, `S3_BUCKET` | `http://storage:9000`, `looksee` | File 카메라용 자산 라이브러리. compose는 내장 저장소를 가리키게 합니다. 외부 S3 호환 버킷을 쓰려면 둘 다 설정합니다. 네이티브 실행에서는 둘 다 설정해야 라이브러리가 활성화됩니다. |
| `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY` | `looksee`, `STORAGE_PASSWORD` | 자산 라이브러리 자격 증명. |
| `S3_REGION` | `auto` | 외부 버킷의 리전. Cloudflare R2는 `auto`. |
| `S3_PREFIX` | 비어 있음 | 버킷 안의 키 접두사. |
| `MEDIA_CACHE_DIR` | compose가 설정 | 재생을 위해 내려받은 자산을 캐시하는 곳. |
| `MEDIA_MOUNT_DIR` | `/media` | MediaMTX가 같은 캐시를 보는 경로. |

## 추론

| 변수 | 기본값 | 의미 |
| --- | --- | --- |
| `REDIS_URL` | compose가 설정 | Valkey URL. |
| `MEDIAMTX_RTSP_URL` | `rtsp://mediamtx:8554` | 카메라 경로를 읽는 곳. |
| `MTX_MEDIA_USER`, `MTX_MEDIA_PASSWORD` | `.env`에서 | 카메라 경로를 읽기 위한 자격 증명. |
| `RTSP_TRANSPORT` | `tcp` | MediaMTX에서 읽을 때 `tcp` 또는 `udp`. |
| `MODELS_DIR` | `/app/models` | 모델 번들 디렉터리. |
| `FIRST_FRAME_TIMEOUT_SECONDS` | `30` | 오류를 보고하기 전에 첫 프레임을, 그리고 실행 중에는 다음 프레임을 기다리는 시간. |
| `LAST_FRAME_TTL_SECONDS` | `10` | 각 카메라의 최신 JPEG를 Snapshot 액션이 쓸 수 있도록 유지하는 시간. |

## Studio

Studio는 요청마다 서버에서 구성을 읽고 공개 부분을 브라우저에 전달하므로, 이미지를 다시 빌드하지 않고도 다른 주소를 가리키도록 바꿀 수 있습니다.

| 변수 | 기본값 | 의미 |
| --- | --- | --- |
| `RUNTIME_API_URL` | `http://<WEBRTC_HOST_IP>:<API_PORT>` | 브라우저에서 보는 API 기본 URL. |
| `RUNTIME_WS_URL` | `ws://<WEBRTC_HOST_IP>:<API_PORT>` | 브라우저에서 보는 WebSocket 기본 URL. |
| `RUNTIME_MEDIAMTX_WEBRTC_URL` | `http://<WEBRTC_HOST_IP>:<MTX_WEBRTC_PORT>` | 브라우저에서 보는 MediaMTX WebRTC URL. |
| `RUNTIME_DOCS_URL` | `https://github.com/okoflow/looksee-docs` | 사이드바 **Documentation**(문서) 링크의 대상. compose는 문서 저장소로 고정합니다. |
| `RUNTIME_GITHUB_URL` | `https://github.com/okoflow/looksee` | **GitHub** 링크의 대상. |
| `SERVER_API_URL` | `http://api:8000` | Studio 서버 자체가 로그인 확인에 쓰는 API 주소. 브라우저에는 전송되지 않습니다. |

리버스 프록시 뒤에서는 세 `RUNTIME_*` URL을 공개 주소로 설정합니다. [배포](deployment.md)에 예제가 있습니다.

## 포트

| 포트 | 서비스 | 바인딩 | 용도 |
| --- | --- | --- | --- |
| `3000` | studio | 모든 인터페이스 | 웹 인터페이스 |
| `8000` | api | 모든 인터페이스 | HTTP API와 WebSocket |
| `8554` | mediamtx | 모든 인터페이스 | RTSP |
| `8889` | mediamtx | 모든 인터페이스 | WebRTC 시그널링과 재생 |
| `8189/udp` | mediamtx | 모든 인터페이스 | WebRTC 미디어 |
| `9997` | mediamtx | `127.0.0.1` | MediaMTX 제어 API |
| `9000` | storage | `127.0.0.1` | 비디오 저장소의 S3 API |
| `5432` | postgres | `127.0.0.1` | PostgreSQL |
| `6379` | redis | `127.0.0.1` | Valkey |

## 볼륨

| 볼륨 | 서비스 | 내용 |
| --- | --- | --- |
| `postgres_data` | postgres | 워크플로, 카메라, 자격 증명, 알림, 사용자, 대기 중인 전달 |
| `storage_data` | storage | File 카메라용으로 업로드한 비디오 파일 |
| `redis_data` | redis | 탐지 스트림과 명령 채널. 잃어도 안전함 |
| `api_snapshots` | api | 스냅샷 JPEG 파일 |
| `api_keys` | api | 생성된 `secret.key` |
| `media-cache` | api, mediamtx | File 카메라용으로 캐시된 비디오 파일 |
| `./models`(바인드) | api, inference | 모델 번들, 읽기 전용 |

`postgres_data`, `api_keys`, `storage_data`가 백업할 가치가 있는 상태를 담습니다. 시크릿이 없으면 저장된 자격 증명을 해독할 수 없고 모든 세션이 로그아웃됩니다. [배포](deployment.md#백업)에서 백업 절차를 설명합니다.
