[문서](index.md) / 문제 해결

# 문제 해결

증상, 가능한 원인, 확인할 사항입니다. 대부분의 답은 로그에 있습니다.

```bash
docker compose logs -f api inference mediamtx studio
```

## 카메라가 Pending에 머무름

추론 서비스가 첫 프레임을 받지 못했습니다.

- **서버가 카메라에 접근할 수 없습니다.** 호스트에서 `ffprobe rtsp://user:password@camera/stream`으로 시험합니다. 브라우저가 카메라에 접근된다는 것은 아무것도 증명하지 않습니다. 스트림을 가져오는 것은 서버입니다.
- **카메라가 UDP RTSP만 지원합니다.** MediaMTX는 TCP로 가져옵니다. 카메라에서 TCP를 켜거나 릴레이를 사용합니다.
- URL의 **자격 증명이나 경로가 잘못되었습니다.** MediaMTX가 업스트림 오류를 로그에 남깁니다.
- **첫 프레임이 30초보다 오래 걸렸습니다.** 느린 카메라에는 `FIRST_FRAME_TIMEOUT_SECONDS`를 높입니다.
- **송출자가 없는 Browser webcam 카메라입니다.** 브라우저에서 워크플로의 모니터를 열고 카메라 접근을 허용합니다. 브라우저가 송출하면 카메라가 활성화됩니다.

## 카메라가 Error를 표시함

모니터와 `inference` 로그에 이유가 표시됩니다.

- **no frames from stream for 30s**: 스트림이 멈췼습니다. 카메라는 스스로 재연결하고 API는 최대 5분까지 늘어나는 간격으로 재시도합니다. 반복되면 네트워크나 카메라를 점검합니다.
- **unsupported ONNX signature**: 모델 내보내기가 지원되는 D-FINE 레이아웃이 아닙니다. [모델](models.md#지원되는-onnx-내보내기-형식)을 참고하십시오.
- **model not found**: 모델 번들이 사라졌거나 `models/` 마운트가 없습니다.

## 모니터에 영상이 없음

알림은 도착하지만 화면이 검거나 **Connection disconnected**(연결 끊김)가 표시됩니다.

- **`WEBRTC_HOST_IP`가 잘못되었습니다.** 브라우저가 접근할 수 있는 주소여야 합니다. 이 값은 WebRTC 후보에 그대로 들어가며, `127.0.0.1`은 서버 자체에서만 동작합니다.
- 브라우저와 서버 사이에서 **UDP 포트 `8189`가 막혀 있습니다.** WebRTC 미디어는 HTTP 프록시를 거치지 않습니다.
- **스트림이 H.265입니다.** 브라우저는 WebRTC로 이를 재생하지 못합니다. 카메라를 H.264로 바꾸거나 서브스트림을 사용합니다. 어느 쪽이든 탐지는 계속 동작합니다.
- Browser webcam 카메라의 **No active stream**(활성 스트림 없음)은 아직 아무것도 송출하고 있지 않다는 뜻입니다.
- **오리진이 다릅니다.** Studio와 MediaMTX가 다른 스킴(`https` Studio, `http` MediaMTX)으로 제공되면 브라우저가 연결을 차단합니다. [배포](deployment.md#리버스-프록시를-통한-tls)처럼 둘 다 프록시합니다.

## 웹캠 접근이 거부됨

브라우저는 `localhost` 또는 HTTPS에서만 `getUserMedia`를 허용합니다. 원격 사용자에게는 Studio를 TLS로 제공하고, 권한 요청을 거부한 뒤에는 브라우저의 사이트 권한을 확인합니다.

## Detect 노드에 모델이 없음

- 모델 번들 디렉터리에는 `manifest.json`과 `model.onnx`가 모두 있어야 합니다.
- 매니페스트가 검증에 실패했습니다. 알 수 없는 필드, 중복 레이블, 클래스 ID의 빈틈, 레이블이 아닌 `events` 키 등입니다. `api` 로그가 문제를 알려 줍니다.
- 디렉터리 이름에 소문자 영문자, 숫자, `-`, `_` 이외의 문자가 있습니다.

## Run이 실패함

**Run**(실행)이 메시지를 표시하고 해당 노드에 포커스를 맞춥니다.

- **상태 코드 402, `feature_not_licensed`**: 그래프에 라이선스 없이 Count, Line crossing, Dwell, Slack이 있습니다. [Enterprise 에디션](enterprise.md)을 참고하십시오.
- 코드가 있는 **상태 코드 422**: 그래프를 그려진 대로 실행할 수 없습니다. [검증 표](nodes.md#검증)가 각 코드를 설명합니다. 흔한 경우는 Detect 노드가 없는 카메라, 모델이 없는 Detect 노드, 자격 증명이 없는 액션입니다.

## 알림이 전달되지 않음

- **Alert 액션은 실행되지만 Telegram, Discord, 이메일, MQTT, Slack이 조용합니다.** 자격 증명이 없거나, 유형이 잘못되었거나, 서비스가 요청을 거부했습니다. `api` 로그는 모든 전달 실패를 시크릿 없이 기록합니다.
- **Telegram**은 봇이 속한 채팅의 채팅 ID가 필요합니다. 먼저 봇에게 메시지를 보내십시오.
- **Email**은 `from_address`를 받아 주는 서버가 필요합니다. 많은 공급자가 포트 587에서 STARTTLS를 요구합니다.
- **Webhook** 대상은 5초 안에 응답해야 합니다.
- **아무것도 실행되지 않습니다.** Detect에서 선택한 이벤트 종류가 모델이 만드는 것인지, 그리고 어떤 필터가 모든 것을 **Else**로 보내고 있지는 않은지 확인합니다.

## 알림이 너무 많거나 너무 적음

- **이벤트 재알림 대기 시간**(`EVENT_COOLDOWN_SECONDS`)은 기본적으로 2초 안에 같은 종류가 반복되면 버립니다.
- Alert 액션의 **cooldown**(재알림 대기 시간)은 카메라별 반복을 억제합니다. `0`은 모두 기록합니다.
- **Debounce** 필터는 어떤 분기에든 자체 시간 창을 부여합니다.
- 그림자나 반사가 탐지를 유발하면 **Confidence**(신뢰도)를 높입니다.

## 스냅샷이 없음

- Snapshot 액션은 그래프에서 이를 사용하는 액션보다 앞에 와야 합니다.
- 카메라의 최신 프레임은 `LAST_FRAME_TTL_SECONDS`(10초) 동안 유지됩니다. 방금 멈춘 카메라에는 스냅샷으로 찍을 프레임이 없습니다.
- 스냅샷 파일은 `api_snapshots` 볼륨에 있으며, 알림의 URL은 API 오리진에 대한 상대 경로입니다.

## Schedule 필터가 잘못된 시간에 동작함

일정은 `EVENT_TIMEZONE`을 사용하며 기본값은 UTC입니다. `America/Chicago`처럼 현장의 IANA 시간대로 설정하고 API를 재시작합니다.

## 소유자 비밀번호를 잊음

재설정 절차는 없습니다. 소유자 행을 삭제하고 초기 설정을 다시 실행합니다. 워크플로, 자격 증명, 알림은 유지됩니다.

```bash
docker compose exec postgres psql -U looksee -d looksee -c "DELETE FROM users;"
```

> [!WARNING]
> 초기 설정 페이지를 다시 완료하기 전까지는 Studio에 접근할 수 있는 누구나 인스턴스를 차지할 수 있습니다.

## CPU 사용량이 높음

- Detect 노드의 **Checks per second**(초당 검사 횟수)를 낮춥니다.
- 720p 이하의 카메라 서브스트림을 사용합니다. 4K 디코딩은 탐지보다 비용이 큽니다.
- `INFERENCE_CPUS`로 추론 컨테이너에 CPU를 더 주거나, [배포](deployment.md#gpu-추론)의 설명대로 GPU로 옮깁니다.

## 포트가 이미 사용 중

`.env`에서 호스트 포트를 바꿉니다. `WEB_PORT`, `API_PORT`, `POSTGRES_PORT`, `REDIS_PORT`, `MTX_RTSP_PORT`, `MTX_WEBRTC_PORT`, `MTX_WEBRTC_ICE_PORT`. `MTX_WEBRTC_PORT`를 바꾸면 `RUNTIME_MEDIAMTX_WEBRTC_URL`도 맞게 설정합니다.
