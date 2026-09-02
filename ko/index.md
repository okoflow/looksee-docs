# LookSee 문서

[English](../en/index.md) · [Русский](../ru/index.md) · [עברית](../he/index.md) · **한국어**

LookSee는 자체 호스팅 영상 분석 시스템입니다. 라이브 스트림, 브라우저 웹캠, 비디오 파일을 지켜보며 ONNX 객체 탐지를 실행하고, 탐지 결과를 이벤트로 바꿉니다. 다음에 일어날 일은 워크플로 그래프가 결정합니다. 필터는 이벤트를 중요한 것만 남도록 걸러내고, 액션은 알림, 스냅샷, 메시지를 필요한 사람과 시스템에 전달합니다.

모든 것이 하나의 Docker Compose 파일로 사용자의 하드웨어에서 실행됩니다. 소스 코드는 [okoflow/looksee](https://github.com/okoflow/looksee) 저장소에 Apache-2.0 라이선스로 공개되어 있으며, Enterprise 에디션은 선 통과, 체류 시간, 개수 집계, Slack을 추가합니다.

![헬멧 착용 준수 워크플로가 열린 워크플로 편집기](../images/editor.png)

## 여기서 시작

- [시작하기](getting-started.md) — Docker Compose로 설치하고, 소유자 계정을 만들고, 탐지 모델을 추가합니다.
- [첫 번째 워크플로](first-workflow.md) — 빈 캔버스에서 동작하는 알림까지.

## 가이드

- [개념](concepts.md) — 워크플로, 노드, 이벤트, 카메라, 실행.
- [카메라](cameras.md) — RTSP, RTMP, SRT, 브라우저 웹캠, WHEP, 비디오 파일.
- [모델](models.md) — 모델 번들 형식, 레이블이 이벤트가 되는 방식, D-FINE 모델을 ONNX로 내보내는 방법.
- [노드](nodes.md) — 모든 노드의 필드, 제한, 검증 규칙.
- [액션과 통합](actions-and-integrations.md) — 자격 증명, 메시지 템플릿, Telegram, Discord, 이메일, MQTT, 웹훅, Slack.
- [모니터링과 알림](monitoring-and-alerts.md) — 실시간 보기, 이벤트 피드, 알림 기록, 스냅샷.

## 운영

- [구성](configuration.md) — 모든 환경 변수, 포트, 볼륨.
- [배포](deployment.md) — 네트워크 내 서버, TLS, GPU, 백업, 업그레이드.
- [보안](security.md) — 계정, 세션, 시크릿, 브라우저에 노출되는 정보.
- [문제 해결](troubleshooting.md) — Pending 상태에 머무는 카메라, 영상 없음, 모델 누락, 전달되지 않는 알림.

## 참조

- [API](api.md) — HTTP 엔드포인트, WebSocket 메시지, 오류 형식.
- [Enterprise 에디션](enterprise.md) — 에디션, 기능, 라이선스 키.
- [변경 이력](changelog.md) — 릴리스 기록.

## 기여

문서는 [looksee-docs](https://github.com/okoflow/looksee-docs) 저장소의 Markdown 파일이며, 저장소의 README에 작성 규칙이 있습니다. 코드 기여는 메인 저장소의 [CONTRIBUTING.md](https://github.com/okoflow/looksee/blob/main/CONTRIBUTING.md)를 따르고, 취약점은 [비공개 보고](https://github.com/okoflow/looksee/security/advisories/new)로 제출합니다.
