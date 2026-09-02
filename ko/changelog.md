[문서](index.md) / 변경 이력

# 변경 이력

LookSee의 릴리스 기록입니다. 버전은 시맨틱 버전 관리를 따르며, *Unreleased* 아래 항목은 `main`에 있고 다음 릴리스에 포함됩니다.

## Unreleased

- 지속적 통합이 모든 풀 리퀘스트에서 린트, 테스트, 마이그레이션 검사, Studio 빌드를 실행합니다.
- 공유 계약, API, 추론 서비스, Enterprise 패키지의 테스트 스위트.
- 저장소 문서: README, 기여 가이드, 보안 정책, 행동 강령, 이슈 양식.
- 영어, 러시아어, 히브리어, 한국어 문서.
- 모노레포 구조에 맞게 Studio 린트 도구를 수정하고 의존성 보안 권고를 해결했습니다.

## 0.1.0

첫 릴리스.

- Camera, Detect, If / Else, Class, Size, Zone, Schedule, Debounce 노드와 Alert, Snapshot, Webhook, Telegram, Email, MQTT, Discord 액션이 있는 워크플로 편집기. Enterprise 에디션에는 Count, Line crossing, Dwell, Slack.
- MediaMTX를 통해 수신하는 RTSP, RTMP, SRT, WHEP, 브라우저 웹캠, S3 호환 자산 라이브러리의 비디오 파일 카메라 소스.
- CPU, CUDA, CoreML에서 ByteTrack 추적과 함께 동작하는 D-FINE 내보내기용 ONNX 추론.
- WebRTC 재생, 탐지와 구역 오버레이, 이벤트 피드, 스냅샷이 포함된 알림 기록을 갖춘 실시간 모니터.
- 암호화된 자격 증명 저장소, 세션 쿠키를 사용하는 소유자 계정, PostgreSQL과 Valkey가 포함된 Docker Compose 배포.
