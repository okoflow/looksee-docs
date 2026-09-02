[문서](index.md) / 노드

# 노드

워크플로 편집기의 모든 노드에 대한 참조입니다. 노드가 하는 일, 기본값과 제한이 있는 필드, **Run**(실행) 시 검증이 확인하는 내용을 다룹니다. 코드 스타일의 필드 이름은 API에서 쓰는 키이며, 인스펙터는 더 친숙한 레이블로 표시합니다.

## 연결

- **Camera**는 출력이 하나이며 정확히 하나의 **Detect**에 연결됩니다.
- **Detect**는 임의 개수의 필터와 액션에 연결됩니다.
- 모든 필터에는 **If**와 **Else** 두 출력이 있습니다. 통과한 이벤트는 **If**로, 통과하지 못한 이벤트는 **Else**로 나갑니다. 분기가 없는 연결은 **If**로 간주합니다.
- **Snapshot**은 다른 액션으로 이어서 연결되며, 그 액션들은 스냅샷을 받습니다. 다른 액션에는 출력이 없습니다.
- 모든 노드는 이벤트당 최대 한 번 실행됩니다.

팔레트는 노드를 Sources, Detection, Logic, Object, Spatial, Temporal, Actions 그룹으로 나눕니다. Enterprise 노드(Count, Line crossing, Dwell, Slack)는 라이선스 키가 구성되기 전까지 잠금 아이콘을 표시합니다.

## 소스

### Camera — `camera_source`

| 필드 | 기본값 | 제한 | 의미 |
| --- | --- | --- | --- |
| `name` | `Camera` | 1~128자 | **Name**(이름). 모니터와 알림 메시지에 표시됨 |
| `source_type` | `rtsp` | `rtsp`, `rtmp`, `srt`, `webrtc`, `whep`, `file` | **Source**(소스). MediaMTX가 스트림을 수신하는 방식. [카메라](cameras.md) 참고 |
| `url` | 비어 있음 | 최대 2048자 | **URL**. 풀 소스의 스트림 URL, `file`의 자산 키. `webrtc`에서는 사용하지 않음 |

**Run** 시 URL은 소스 유형과 일치해야 합니다. RTSP는 `rtsp://` 또는 `rtsps://`, RTMP는 `rtmp://` 또는 `rtmps://`, SRT는 `srt://`, WHEP는 `http://`, `https://`, `whep://`, `wheps://`, File은 영문자, 숫자, `.`, `_`, `-`, `/`로 이루어진 자산 키입니다.

## 탐지

### Detect — `detect`

| 필드 | 기본값 | 제한 | 의미 |
| --- | --- | --- | --- |
| `model_id` | 없음 | `models/`의 모델 | **Model**(모델). **Run** 시 필수. |
| `event_kinds` | 전체 | 모델이 만드는 종류 중 최대 64개 | **Events**(이벤트). 비어 있으면 모델의 모든 이벤트 종류를 통과시킵니다. |
| `confidence_threshold` | `0.5` | 0~1 | **Confidence**(신뢰도). 이보다 낮은 탐지는 추적 전에 버려집니다. |
| `inference_fps` | `1` | 1~15 | **Checks per second**(초당 검사 횟수). 이 카메라에서 모델을 실행하는 빈도. |

Studio에서 모델을 선택하면 권장 신뢰도가 자동으로 채워집니다.

## 필터

필터는 각 이벤트를 통과시키거나 걸러냅니다. 탐지를 검사하는 필터는 별도 언급이 없는 한 탐지가 없는 이벤트를 통과시킵니다.

### If / Else — `if_else_filter`

이벤트의 속성 하나를 비교합니다. `condition` 객체는 `field`로 속성을 선택합니다.

| `field` | `operator` | `value` | 통과 조건 |
| --- | --- | --- | --- |
| `event_kind` | `is`, `is_not` | 이벤트 종류 | 이벤트의 종류가 일치함(또는 일치하지 않음) |
| `object_class` | `contains`, `not_contains` | 레이블, 최대 256자 | 해당 레이블의 탐지가 있음(또는 없음) |
| `detection_count` | `eq`, `neq`, `gt`, `gte`, `lt`, `lte` | 0~1000, 기본 `gte 1` | 탐지 개수가 선택한 비교를 만족함 |
| `max_confidence` | 위와 같은 여섯 연산자 | 0~1, 기본 `gte 0.5` | 최고 신뢰도가 선택한 비교를 만족함 |

**Run** 시 `event_kind`와 `object_class`에는 값이 필요하며, 그 값은 업스트림 모델이 만들 수 있는 것이어야 합니다.

### Class — `class_filter`

| 필드 | 기본값 | 제한 | 통과 조건 |
| --- | --- | --- | --- |
| `classes` | 비어 있음 | 최대 64개 레이블 | 어느 탐지라도 이 레이블 중 하나를 가짐. 비어 있으면 모두 통과. |

레이블은 업스트림 모델에 속해야 합니다. 탐지가 없는 이벤트는 이 필터를 통과하지 못합니다.

### Size — `size_filter`

| 필드 | 기본값 | 제한 | 통과 조건 |
| --- | --- | --- | --- |
| `min_area` | `0` | 0~1 | 어느 탐지의 상자든 프레임에서 최소 이 비율을 차지하고… |
| `max_area` | `1` | 0~1, `min_area` 이상 | …최대 이 비율을 차지함 |

멀리 있는 객체나 몇 픽셀 크기의 오탐지를 무시할 때 유용합니다.

### Zone — `zone_filter`

| 필드 | 기본값 | 제한 | 통과 조건 |
| --- | --- | --- | --- |
| `polygon` | 비어 있음 | 3~100개 점, 프레임 기준 0~1 상대 좌표 | 어느 탐지의 상자 중심점이든 다각형 안에 있음 |

다각형은 인스펙터에서 그리거나 **Edit on preview**(미리보기에서 편집)로 실시간 프레임 위에 그립니다. **Run** 시 다각형에는 점이 셋 이상 필요합니다.

### Schedule — `time_window_filter`

| 필드 | 기본값 | 제한 | 의미 |
| --- | --- | --- | --- |
| `start_hour` | `0` | 0~23 | **Start hour**(시작 시각), 포함 |
| `end_hour` | `23` | 0~23 | **End hour**(종료 시각), 포함. `start_hour`보다 작으면 자정을 넘겨 이어짐 |
| `weekdays` | 월요일~금요일 | 0(월요일)~6(일요일), **Run** 시 하나 이상 | 시간 범위가 적용되는 **Weekdays**(요일) |
| `invert` | 꺼짐 | | **Outside schedule**(일정 외): 근무 시간 외 시나리오를 위해 범위 밖에서 대신 통과시킴 |

시각은 `EVENT_TIMEZONE` 기준으로 평가됩니다.

### Debounce — `debounce_filter`

| 필드 | 기본값 | 제한 | 의미 |
| --- | --- | --- | --- |
| `seconds` | `30` | 1~3600 | 이벤트가 하나 통과한 뒤 이 시간 동안 같은 카메라의 이후 이벤트는 통과하지 못함 |

시간 창은 노드별, 카메라별로 유지되므로 한 워크플로의 여러 Debounce 노드는 서로 독립적입니다.

### Count — `count_threshold_filter` (Enterprise)

| 필드 | 기본값 | 제한 | 통과 조건 |
| --- | --- | --- | --- |
| `polygon` | 비어 있음 | 최대 100개 점. 3개 미만이면 프레임 전체를 셈 | |
| `operator` | `gte` | `gte`, `lte` | 다각형 안의 탐지 개수가 최소(또는 최대)… |
| `count` | `1` | 0~1000 | …이 값임 |

### Line crossing — `line_crossing_filter` (Enterprise)

| 필드 | 기본값 | 제한 | 통과 조건 |
| --- | --- | --- | --- |
| `line` | 비어 있음 | **Run** 시 정확히 2개 점 | 추적 중인 객체의 중심점이 이전 검사 이후 선의 한쪽에서 다른 쪽으로 이동함 |
| `direction` | `any` | `any`, `in`, `out` | `in`은 첫 점에서 둘째 점을 바라볼 때 선의 왼쪽에서 오른쪽으로 넘어가는 객체를, `out`은 반대 방향을 셈 |

추적 ID에 의존합니다. 빠른 객체에는 **Checks per second**를 높이십시오.

### Dwell — `dwell_filter` (Enterprise)

| 필드 | 기본값 | 제한 | 통과 조건 |
| --- | --- | --- | --- |
| `polygon` | 비어 있음 | 3~100개 점 | |
| `min_seconds` | `60` | 1~3600 | 추적 중인 객체가 다각형 안에 최소 이 시간 동안 머무름 |

## 액션

자격 증명 유형이 표시된 필드에는 해당 유형의 저장된 자격 증명이 필요합니다. 각 통합은 [액션과 통합](actions-and-integrations.md)에서 설명합니다.

### Alert — `log_alert_action`

| 필드 | 기본값 | 제한 |
| --- | --- | --- |
| `severity` | `warning` | **Severity**(심각도): `info`, `warning`, `critical` |
| `cooldown_seconds` | `30` | **Cooldown seconds**(재알림 대기 시간, 초), 0~3600. `0`은 모든 이벤트를 기록 |

### Snapshot — `snapshot_action`

| 필드 | 기본값 |
| --- | --- |
| `annotate` | 켜짐: 이미지에 탐지를 그림(**Annotate**) |

### Webhook — `webhook_action`

| 필드 | 기본값 | 제한 |
| --- | --- | --- |
| `url` | 비어 있음 | **URL**: **Run** 시 `http://` 또는 `https://` URL |
| `method` | `POST` | **Method**(메서드): `POST`, `GET`, `PUT` |

### Telegram — `telegram_action`

| 필드 | 기본값 | 제한 |
| --- | --- | --- |
| `credential_id` | 없음 | `telegram_bot` 자격 증명 |
| `chat_id` | 비어 있음 | 1~64자 |
| `message_template` | `[{kind}] camera={camera_id} at {ts}` | 최대 4096자 |

### Email — `email_action`

| 필드 | 기본값 | 제한 |
| --- | --- | --- |
| `credential_id` | 없음 | `smtp` 자격 증명 |
| `to` | 비어 있음 | 최대 320자 |
| `subject_template` | `[{kind}] on camera {camera_id}` | 최대 998자 |
| `body_template` | `{kind} at {ts}`, 카메라, 스냅샷 | 최대 8192자 |

### MQTT — `mqtt_action`

| 필드 | 기본값 | 제한 |
| --- | --- | --- |
| `credential_id` | 없음 | `mqtt` 자격 증명 |
| `topic` | `looksee/events` | 1~512자 |
| `payload_template` | 비어 있음: 이벤트를 JSON으로 전송 | 최대 8192자 |

### Discord — `discord_action`

| 필드 | 기본값 | 제한 |
| --- | --- | --- |
| `credential_id` | 없음 | `discord_webhook` 자격 증명 |
| `message_template` | `[{kind}] camera={camera_id} at {ts}` | 최대 2000자 |

### Slack — `slack_action` (Enterprise)

| 필드 | 기본값 | 제한 |
| --- | --- | --- |
| `credential_id` | 없음 | `slack_webhook` 자격 증명 |
| `message_template` | `[{kind}] camera={camera_id} at {ts}` | 최대 4096자 |

## 제한

그래프 하나는 최대 200개 노드, 400개 연결, 100개 주석을 담습니다. 노드 ID는 최대 128자, 연결 ID는 최대 256자입니다.

## 검증

그래프를 저장하면 구조를 확인합니다. **Run**은 추가로 모든 노드가 실행 가능한지 확인합니다. Studio는 오류가 가리키는 노드에 포커스를 맞추고, API는 `code` 필드에 코드를 담아 상태 코드 422로, 라이선스 오류는 402로 반환합니다.

| 코드 | 의미 |
| --- | --- |
| `duplicate_node_ids`, `duplicate_edge_id` | 두 노드 또는 두 연결이 같은 ID를 가짐 |
| `edge_node_missing` | 연결이 존재하지 않는 노드를 가리킴 |
| `branch_on_non_filter` | 연결에 If/Else 분기가 있지만 필터가 아닌 노드에서 나감 |
| `no_camera_source` | 워크플로에 Camera가 없음 |
| `graph_cycle` | 연결이 순환을 이룸 |
| `node_not_runnable` | 필수 필드가 비어 있거나 잘못됨. 메시지가 필드 이름을 알려 줌 |
| `feature_not_licensed` | 라이선스 키 없이 Enterprise 노드를 사용함(상태 코드 402) |
| `detect_node_missing`, `multiple_detect_nodes` | Camera가 Detect에 도달하지 않거나 둘 이상에 도달함 |
| `model_not_selected`, `model_unavailable` | Detect에 모델이 없거나, 모델이 `models/`에 없음 |
| `unsupported_event_kinds` | Detect가 모델이 만들지 않는 이벤트를 선택함 |
| `unsupported_classes` | Class 필터가 업스트림 모델이 만들지 않는 레이블을 지정함 |
| `unsupported_condition_event`, `unsupported_condition_class` | If / Else 조건이 업스트림 모델이 만들지 않는 이벤트나 레이블을 지정함 |
| `asset_store_not_configured`, `asset_unavailable` | 자산 라이브러리 없는 File 카메라, 또는 객체가 없는 File 카메라 |
| `credential_unavailable`, `credential_type_mismatch` | 액션의 자격 증명이 없거나 다른 유형임 |

어느 Camera에도 도달하지 않는 노드는 허용되며 실행 시 무시됩니다. Studio는 이런 노드에 경고를 표시합니다.
