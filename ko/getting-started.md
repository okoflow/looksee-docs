[문서](index.md) / 시작하기

# 시작하기

이 페이지는 새 머신에서 시작해 탐지 모델이 준비된 상태로 Studio에 로그인하기까지의 과정을 다룹니다. LookSee를 실행하는 공식 지원 방식인 Docker Compose를 사용합니다.

## 요구 사항

- Docker Compose 2.24 이상이 포함된 Docker Engine. macOS와 Windows에서는 Docker Desktop이 동작하며, 운영 환경은 보통 Linux 서버입니다.
- 64비트 x86 또는 ARM 프로세서. 탐지는 기본적으로 CPU에서 실행됩니다. CUDA GPU는 선택 사항이며 [배포](deployment.md#gpu-추론)에서 다룹니다.
- 사용 가능한 포트 `3000`(Studio), `8000`(API), `8554`(RTSP), `8889`(WebRTC), `8189/udp`(WebRTC ICE). PostgreSQL, Valkey, 비디오 저장소, MediaMTX 제어 API는 루프백 인터페이스에만 바인딩됩니다.

compose 파일은 모든 서비스에 리소스 제한을 둡니다. 추론 서비스는 기본적으로 CPU 4개와 메모리 4 GB를 받습니다. 카메라를 더 연결하거나 더 큰 모델을 쓰려면 `.env`에서 `INFERENCE_CPUS`와 `INFERENCE_MEMORY`를 높입니다.

## 설치

```bash
git clone https://github.com/okoflow/looksee.git
cd looksee
cp .env.example .env
```

첫 시작 전에 `.env`를 열어 다음 값을 설정합니다.

| 변수 | 설정값 |
| --- | --- |
| `WEBRTC_HOST_IP` | 브라우저가 같은 머신에서 실행되면 `127.0.0.1`. 다른 기기에서 Studio를 열려면 `192.168.1.20`처럼 네트워크에서 이 머신에 접근하는 주소. 브라우저는 실시간 영상을 위해 이 주소에 연결합니다. |
| `POSTGRES_PASSWORD`, `MTX_MEDIA_PASSWORD`, `STORAGE_PASSWORD` | 직접 정한 비공개 값. `.env.example`에는 비어 있으며, 설정하기 전까지 스택은 시작을 거부합니다. |

그런 다음 스택을 빌드하고 시작합니다.

```bash
docker compose up -d --build
docker compose ps
```

첫 빌드는 베이스 이미지와 Python, Node 패키지를 내려받느라 몇 분이 걸립니다. 스택이 준비되면 `docker compose ps`가 `storage`, `postgres`, `redis`, `mediamtx`, `api`, `inference`, `studio` 서비스를 모두 `healthy`로 표시합니다. 일회성 서비스인 `storage-init`, `api-migrate`, `media-cache-init`는 작업을 마치면 종료됩니다.

## 소유자 계정 만들기

`http://<WEBRTC_HOST_IP>:3000`을 엽니다. 계정이 없는 동안 Studio는 초기 설정 페이지로 리디렉션합니다. 이메일 주소, 표시 이름, 그리고 숫자 하나와 대문자 하나를 포함한 8자 이상의 비밀번호를 입력합니다. 이 계정이 인스턴스의 소유자가 되며 바로 로그인됩니다.

> [!WARNING]
> 소유자가 생기기 전까지는 포트 `3000`에 접근할 수 있는 누구나 인스턴스를 차지할 수 있습니다. 첫 시작 직후 계정을 만들고, 스택은 신뢰할 수 있는 네트워크에 두십시오. 나머지는 [보안](security.md)에서 다룹니다.

![로그인 페이지](../images/sign-in.png)

## 탐지 모델 추가

LookSee는 모델 없이 배포됩니다. 모델은 저장소 체크아웃의 `models/` 아래에 두 파일을 담은 디렉터리입니다.

```text
models/
└── ppe-helmets/
    ├── manifest.json
    └── model.onnx
```

`model.onnx`는 ONNX로 내보낸 D-FINE 모델입니다. `manifest.json`은 모델의 이름과 클래스를 정의합니다.

```json
{
  "name": "Safety gear (PPE)",
  "labels": ["head", "helmet", "vest"],
  "recommended_confidence_threshold": 0.4
}
```

`models/` 디렉터리는 API와 추론 컨테이너에 읽기 전용으로 마운트되고, API는 다음 요청에서 새 모델 번들을 발견하므로 재시작이 필요 없습니다. 각 레이블은 `HELMET_DETECTED`와 같은 이벤트 종류가 됩니다. 매니페스트의 전체 설명과 모델 내보내기 방법은 [모델](models.md)에 있습니다.

## 스택 확인

```bash
curl http://127.0.0.1:8000/health
docker compose logs -f api inference
```

API는 `{"status":"ok"}`로 응답합니다. 대화형 API 문서는 `http://127.0.0.1:8000/docs`에서 제공됩니다.

## 자주 쓰는 명령

```bash
docker compose logs -f studio api inference mediamtx   # follow logs
docker compose stop                                    # stop, keep data
docker compose up -d                                   # start again
git pull && docker compose up -d --build               # upgrade
docker compose down -v                                 # remove everything, including data
```

워크플로, 카메라, 자격 증명, 알림, 스냅샷, 업로드한 비디오, 서명 시크릿은 명명된 볼륨에 저장됩니다. `down -v`는 이들을 삭제합니다. 백업 방법은 [배포](deployment.md)에서 설명합니다.

## 다음 단계

[첫 번째 워크플로](first-workflow.md)에서 헬멧 착용 준수 워크플로를 캔버스부터 만들어 보고 모니터에서 동작하는 모습을 확인합니다.
