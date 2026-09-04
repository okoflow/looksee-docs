[문서](index.md) / 변경 이력

# 변경 이력

LookSee의 릴리스 기록입니다. 버전은 시맨틱 버전 관리를 따르고, 날짜는 릴리스 날짜입니다. `main`에 있지만 아직 릴리스되지 않은 변경은 *Unreleased* 아래에 나열됩니다.

## [Unreleased]

## [0.2.0] - 2026-09-05

0.1.0에서 업그레이드:

- `.env`에 `STORAGE_PASSWORD`를 설정합니다. `POSTGRES_PASSWORD`와 `MTX_MEDIA_PASSWORD`도 더 이상 예제 값이 없으므로 설정해야 합니다. 세 값 중 하나라도 비어 있으면 compose는 시작을 거부합니다.
- `MTX_MEDIA_PASSWORD`를 교체합니다. 0.1.0은 Studio를 불러온 모든 브라우저에 이 값을 보냈습니다.
- `docker compose up -d --build`를 실행해 `api-migrate`가 마이그레이션 `0002`를 적용하고 `storage-init`가 비디오 버킷을 만들게 합니다. API, 추론 서비스, Studio, MediaMTX를 함께 업데이트합니다. MediaMTX는 이제 모든 연결을 API를 통해 허가합니다.
- 이벤트는 더 이상 그래프에 들어가기 전에 `EVENT_COOLDOWN_SECONDS`를 기다리지 않습니다. 재알림 대기 시간으로 반복 액션을 억제하던 곳에는 Debounce 필터나 Alert 재알림 대기 시간을 추가합니다.
- compose의 `docs` 프로필과 `DOCS_PORT`가 사라졌습니다. 문서는 이 저장소에 있습니다.

### 추가

- 내장 S3 호환 비디오 저장소: compose가 만들고 준비하는 `storage` 서비스(RustFS)와 `storage_data` 볼륨. File 카메라가 외부 버킷 없이 동작합니다. `S3_*` 변수로는 여전히 API를 외부 버킷으로 향하게 할 수 있습니다.
- 전달 아웃박스: Webhook, Telegram, Discord, Slack, Email, MQTT 메시지가 PostgreSQL에 저장되고, 일시적 실패 뒤에는 최대 5분까지 두 배씩 늘어나는 간격으로 최대 8회 재시도됩니다. `GET /deliveries`가 큐를 나열하고 `POST /deliveries/{id}/retry`가 실패한 전달을 다시 큐에 넣습니다. Webhook 요청은 `Idempotency-Key` 헤더를 담습니다.
- 카메라별 미디어 그랜트: `POST /cameras/{id}/media-access`가 카메라 하나를 읽거나 송출하는 5분짜리 토큰을 발급하고, MediaMTX는 모든 연결을 API를 통해 허가합니다.
- 로그인과 초기 설정 속도 제한: 클라이언트당 분당 20회, 전체 분당 100회이며, 초과하면 `Retry-After`와 함께 `429`로 응답합니다.
- Origin 검사: `Origin`이 `CORS_ORIGIN_REGEX`에도 API 오리진에도 맞지 않는 상태 변경 요청과 WebSocket 연결은 `403`으로 거부됩니다.
- 지속적 통합이 모든 풀 리퀘스트에서 린트, 테스트, 마이그레이션 검사, Studio 빌드, compose 구성 검사를 실행하며, 공유 계약, API, 추론 서비스, Enterprise 패키지의 테스트 스위트를 포함합니다.
- 저장소 문서: README, 기여 가이드, 보안 정책, 행동 강령, 이슈 양식. 이 문서의 영어, 러시아어, 히브리어, 한국어 판.

### 변경

- MediaMTX 미디어 비밀번호는 API, 추론 서비스, MediaMTX가 공유하는 백엔드 시크릿입니다. Studio는 더 이상 이 값을 받지 않습니다.
- `EVENT_COOLDOWN_SECONDS`는 실시간 피드의 `event` 메시지만 제한합니다. 모든 이벤트가 그래프에 들어갑니다.
- 저장소 실패는 내부 오류 대신 읽을 수 있는 메시지와 함께 `503`으로 응답합니다.
- 중지된 API 프로세스가 확인하지 못한 탐지 프레임은 1분 뒤 다시 처리되며, 프레임은 한 번에 하나씩 처리됩니다.
- `.env.example`은 서비스별로 묶이고 설치 시 설정하는 값만 나열합니다. 필수 비밀번호는 비어 있습니다.
- Studio는 Google Fonts에서 글꼴을 내려받는 대신 글꼴을 내장하고, 저장 중에 한 편집을 유지하며, **Documentation** 링크를 이 저장소로 연결합니다.
- MediaMTX 1.20.1.

### 수정

- 카메라를 RTSP에서 Browser webcam으로 바꾸면 MediaMTX가 이전 스트림을 계속 가져오던 문제. 경로 구성을 패치하는 대신 교체합니다.
- 모니터를 떠날 때 WebRTC 세션이 닫히고, 워크플로 저장이 저장 중의 편집과 충돌하지 않습니다.
- 모노레포 구조에 맞춘 Studio 린트 도구와 Studio 의존성의 보안 권고.

## [0.1.0] - 2026-08-06

첫 릴리스.

- Camera, Detect, If / Else, Class, Size, Zone, Schedule, Debounce 노드와 Alert, Snapshot, Webhook, Telegram, Email, MQTT, Discord 액션이 있는 워크플로 편집기. Enterprise 에디션에는 Count, Line crossing, Dwell, Slack.
- MediaMTX를 통해 수신하는 RTSP, RTMP, SRT, WHEP, 브라우저 웹캠, S3 호환 자산 라이브러리의 비디오 파일 카메라 소스.
- CPU, CUDA, CoreML에서 ByteTrack 추적과 함께 동작하는 D-FINE 내보내기용 ONNX 추론.
- WebRTC 재생, 탐지와 구역 오버레이, 이벤트 피드, 스냅샷이 포함된 알림 기록을 갖춘 실시간 모니터.
- 암호화된 자격 증명 저장소, 세션 쿠키를 사용하는 소유자 계정, PostgreSQL과 Valkey가 포함된 Docker Compose 배포.

[Unreleased]: https://github.com/okoflow/looksee/compare/v0.2.0...HEAD
[0.2.0]: https://github.com/okoflow/looksee/compare/v0.1.0...v0.2.0
[0.1.0]: https://github.com/okoflow/looksee/releases/tag/v0.1.0
