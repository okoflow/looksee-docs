[문서](index.md) / Enterprise 에디션

# Enterprise 에디션

LookSee는 하나의 이미지 세트에 두 에디션을 담아 배포됩니다. Community 에디션은 저장소의 `ee/` 디렉터리 밖의 코드로, Apache-2.0으로 라이선스됩니다. Enterprise 에디션은 `ee/` 안의 코드로, [LookSee Enterprise 라이선스](https://github.com/okoflow/looksee/blob/main/ee/LICENSE)로 라이선스되며 라이선스 키로 잠금 해제됩니다.

## 기능

| 기능 | 노드 | 추가되는 것 |
| --- | --- | --- |
| 측정 필터 | Count, Line crossing, Dwell | 구역 안의 객체 수를 임계값과 비교하고, 방향이 있는 선 통과를 세고, 구역에 제한 시간보다 오래 머무는 객체를 탐지합니다. 셋 모두 프레임 간 객체 추적에 의존합니다. |
| Enterprise 통합 | Slack | Slack 수신 웹훅으로 보내는 메시지. 다른 메시징 액션과 같은 템플릿을 사용합니다. |

Community 에디션에서는 이 노드들이 팔레트에 잠금 아이콘과 함께 표시되며 캔버스로 끌어다 놓을 수 없습니다. 예를 들어 API로 그래프를 가져온 뒤처럼 이 노드 중 하나를 포함한 워크플로는 **Run**(실행) 시 오류 코드 `feature_not_licensed`와 HTTP 상태 코드 402로 거부됩니다.

## 활성화

`.env`에 키를 설정하고 API를 재시작합니다.

```bash
LICENSE_KEY=<your key>
```

```bash
docker compose up -d api
```

`GET /entitlements`는 활성 에디션과 기능을 보고합니다.

```json
{ "edition": "enterprise", "features": ["measurement_filters", "enterprise_integrations"] }
```

Studio도 같은 엔드포인트를 읽어 팔레트의 잠금을 해제합니다. Enterprise 노드와 함께 저장된 워크플로는 다음 **Run**부터 동작합니다.

## 라이선스 조건

Enterprise 라이선스는 구독 없이 개발과 테스트 목적으로 코드를 복사하고 수정하는 것을 허용합니다. 운영 환경에서 사용하려면 시트 수에 맞는 유효한 라이선스가 필요합니다. `ee/`에 대한 기여는 저장소의 [CONTRIBUTING.md](https://github.com/okoflow/looksee/blob/main/CONTRIBUTING.md)에 설명된 대로 같은 라이선스로 받습니다.
