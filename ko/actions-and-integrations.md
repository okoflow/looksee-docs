[문서](index.md) / 액션과 통합

# 액션과 통합

액션은 워크플로 끝에 놓이는 노드로, 이벤트를 알림, 스냅샷, 또는 외부 시스템으로 보내는 메시지로 바꿉니다. 외부 서비스와 통신하는 액션은 LookSee에 저장된 자격 증명으로 인증합니다.

## 자격 증명

**Credentials**(자격 증명) 페이지는 저장된 자격 증명을 이름, 유형, 그리고 SMTP 호스트나 웹훅 도메인 같은 비밀이 아닌 요약과 함께 나열합니다. **New credential**(새 자격 증명)은 **Name**(이름), **Type**(유형), 그리고 해당 유형의 필드를 묻습니다. 비밀 값 자체는 인스턴스 시크릿에서 파생한 키로 암호화되며 다시 표시되지 않습니다. 새 비밀 값을 입력하지 않고 자격 증명을 편집하면 저장된 값이 유지됩니다.

| 유형 | 사용하는 액션 | 페이로드 필드 |
| --- | --- | --- |
| `telegram_bot` | Telegram | `bot_token` |
| `discord_webhook` | Discord | `webhook_url` |
| `slack_webhook` | Slack(Enterprise) | `webhook_url` |
| `smtp` | Email | `host`, `port`(기본 587), `username`, `password`, `from_address`, `starttls`(기본 켜짐) |
| `mqtt` | MQTT | `host`, `port`(기본 1883), `username`, `password` |

액션은 자격 증명을 ID로 참조하고 워크플로 시작 시 유형을 확인합니다. SMTP 자격 증명을 가진 Telegram 액션은 **Run**(실행) 시 `credential_type_mismatch`로 실패합니다. 자격 증명은 메시지가 전달될 때 해석되므로, 토큰을 교체하면 실행 중인 워크플로에 즉시 적용됩니다.

![Credentials 페이지](../images/credentials.png)

## 메시지 템플릿

Telegram, Discord, Slack, 이메일, MQTT 액션은 다음 자리 표시자가 있는 템플릿으로 텍스트를 만듭니다.

| 자리 표시자 | 값 |
| --- | --- |
| `{kind}` | 이벤트 종류, 예: `HELMET_DETECTED` |
| `{camera_id}` | 카메라 ID |
| `{ts}` | ISO 8601 형식의 이벤트 시각 |
| `{count}` | 이벤트를 만든 탐지의 개수 |
| `{snapshot_url}` | 앞서 Snapshot이 실행되었으면 스냅샷 경로, 아니면 빈 값 |

기본 메시지 템플릿은 `[{kind}] camera={camera_id} at {ts}`입니다. 알 수 없는 자리 표시자가 있는 템플릿은 작성된 그대로 전송되고 경고가 로그에 남습니다.

## Alert

이벤트를 알림 기록에 기록하고 모니터에 표시합니다. 필드: `severity`(`info`, `warning`, `critical`)와 초 단위 `cooldown_seconds`(재알림 대기 시간, 0~3600, 기본 30). 재알림 대기 시간 안에 같은 카메라에서 반복되는 이벤트는 버려지며, `0`은 모든 이벤트를 기록합니다. Alert는 LookSee 안에 기록이 남는 유일한 액션입니다. [모니터링과 알림](monitoring-and-alerts.md)에서 다룹니다.

## Snapshot

카메라의 최신 프레임을 JPEG로 저장하며, `annotate`가 켜져 있으면(기본값) 탐지를 그려 넣습니다. Snapshot은 출력이 있는 유일한 액션입니다. 사진을 함께 보내야 하는 액션을 그 뒤에 연결합니다. 이 액션들은 템플릿과 페이로드용 `snapshot_url`을 받고, Telegram, Discord, Slack, 이메일은 이미지 자체를 첨부합니다.

## Webhook

선택한 메서드(기본 `POST`, 또는 `GET`이나 `PUT`)로 이벤트를 JSON으로 URL에 전송합니다. 자격 증명은 필요 없습니다. 수신 측이 토큰을 요구하면 URL에 넣습니다. 요청은 5초 후 시간 초과되며, 2xx가 아닌 응답은 로그에 남기고 재시도하지 않습니다. 페이로드는 [API](api.md#웹훅-페이로드)에 문서화되어 있습니다.

## Telegram

채팅에 메시지를 보내고, 스냅샷이 있으면 캡션이 달린 사진을 보냅니다. Telegram의 [@BotFather](https://t.me/botfather)로 봇을 만들고, 토큰을 `telegram_bot` 자격 증명으로 저장하고, 봇을 대상 채팅에 추가한 뒤, 액션에 채팅 ID를 입력합니다. 개인 채팅이라면 먼저 봇에게 메시지를 보내고 `https://api.telegram.org/bot<token>/getUpdates`에서 채팅 ID를 읽습니다.

## Discord

수신 웹훅을 통해 채널에 게시합니다. 채널의 통합 설정에서 웹훅을 만들고, URL을 `discord_webhook` 자격 증명으로 저장하고, 메시지 템플릿을 설정합니다. Discord는 메시지를 2000자로 제한하며, 더 긴 텍스트는 잘립니다.

## Slack

Enterprise 에디션. `slack_webhook` 자격 증명으로 저장한 Slack 수신 웹훅을 통해 채널에 게시하며, 같은 템플릿 자리 표시자를 사용합니다.

## Email

제목 템플릿과 본문 템플릿으로 SMTP를 통해 메시지를 보내고, 스냅샷이 있으면 첨부합니다. 서버를 `smtp` 자격 증명으로 저장합니다. `starttls`는 연결 후 암호화로 전환하며, 포트 587이 기대하는 방식입니다. 발신자는 `from_address`이고, 비어 있으면 `username`입니다. 필드: `to`. 주소 하나 또는 서버가 허용하는 쉼표로 구분된 목록.

## MQTT

`mqtt` 자격 증명으로 저장한 브로커의 토픽에 게시합니다. 필드: `topic`(기본 `looksee/events`)과 `payload_template`. 템플릿이 비어 있으면 웹훅과 같은 JSON을 게시하고, 템플릿이 있으면 렌더링된 텍스트를 게시합니다. 연결은 5초 후 시간 초과됩니다.

## 전달 동작

- 액션은 자신에게 도달한 모든 이벤트에 대해 그래프 순서로 실행됩니다. 실패한 액션은 오류를 로그에 남기며 다른 액션을 멈추지 않습니다.
- 두 분기가 같은 액션으로 이어지더라도 각 액션은 이벤트당 최대 한 번 실행됩니다.
- 시크릿은 로그에 나타나지 않습니다. 전달 실패는 액션과 상태만 기록됩니다.
- 외부로 나가는 요청은 하나의 연결 풀을 공유하며 API 컨테이너에서 직접 나갑니다. 프록시나 방화벽을 그에 맞게 구성하십시오.
