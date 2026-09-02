[문서](index.md) / API

# API

API는 포트 `8000`의 FastAPI 서비스입니다. Studio가 유일한 기본 클라이언트이며, Studio가 하는 모든 일은 스크립트와 통합에서도 할 수 있습니다. 요청과 응답 스키마가 포함된 대화형 문서는 `/docs`에서, OpenAPI 문서는 `/openapi.json`에서 제공됩니다.

## 인증

`/health`, `/auth/*`, `/docs`, `/openapi.json`을 제외한 모든 엔드포인트는 `looksee_session` 쿠키를 요구합니다. 로그인해서 쿠키를 얻고 7일 동안 재사용합니다.

```bash
curl -c cookies.txt -H 'content-type: application/json' \
  -d '{"email":"owner@example.com","password":"..."}' \
  http://127.0.0.1:8000/auth/login

curl -b cookies.txt http://127.0.0.1:8000/workflows
```

## 오류

오류는 `detail` 메시지가 있는 JSON을 반환합니다. 워크플로 그래프 오류는 클라이언트가 해당 노드에 포커스를 맞출 수 있도록 안정적인 `code`와 문제가 된 노드의 `node_id`를 추가합니다.

```json
{ "detail": "camera 'camera' reaches no detect node", "code": "detect_node_missing", "node_id": "camera" }
```

| 상태 코드 | 의미 |
| --- | --- |
| `401` | 세션이 없거나 잘못됨. 이메일이나 비밀번호가 틀림 |
| `402` | 그래프가 라이선스 없이 Enterprise 노드를 사용함(`feature_not_licensed`) |
| `404` | 알 수 없는 워크플로, 알림, 자격 증명, 자산 |
| `409` | 소유자 계정이 이미 존재함 |
| `422` | 검증 실패. 필드가 범위를 벗어났거나 그래프를 실행할 수 없음(`code` 설정됨) |
| `503` | 자산 라이브러리가 구성되지 않음 |

[노드](nodes.md#검증)에 모든 그래프 오류 코드가 나열되어 있습니다.

## 엔드포인트

### 상태 확인과 초기 설정

| 메서드와 경로 | 설명 |
| --- | --- |
| `GET /health` | `{"status":"ok"}`를 반환. 인증 불필요 |
| `GET /auth/status` | 소유자가 생기기 전까지 `{"requires_setup": true}` |
| `POST /auth/setup` | 소유자 생성: `email`, `name`, `password`. 세션 쿠키를 설정합니다. |
| `POST /auth/login` | `email`, `password`. 세션 쿠키를 설정합니다. |
| `POST /auth/logout` | 쿠키를 지움 |
| `GET /auth/me` | 로그인한 사용자 |
| `GET /entitlements` | `{"edition": "community", "features": []}` 또는 Enterprise 기능 |

### 모델

| 메서드와 경로 | 설명 |
| --- | --- |
| `GET /models` | 발견된 모델 번들: `id`, `name`, `classes`(`class_id`, `label`, `event_kind`), `recommended_confidence_threshold` |

### 워크플로

| 메서드와 경로 | 설명 |
| --- | --- |
| `GET /workflows` | 모든 워크플로와 카메라, 상태. 최신순 |
| `POST /workflows` | 생성: `name`, 선택적 `description`, 선택적 `graph`. `201`을 반환합니다. |
| `GET /workflows/{id}` | 워크플로 하나 |
| `PATCH /workflows/{id}` | `name`, `description`, `enabled`, `graph`의 부분 업데이트. `enabled`를 `true`로 설정하면 그래프를 검증하고 카메라를 시작합니다. |
| `DELETE /workflows/{id}` | 중지하고 삭제. `204`를 반환 |

그래프는 `{"nodes": [...], "edges": [...], "comments": [...]}`입니다. 각 노드에는 `id`, `x`와 `y`가 있는 `position`, 그리고 `kind`로 노드 유형을 선택하는 `data`가 있으며, `data`의 나머지 필드는 [노드](nodes.md)에 나열되어 있습니다. 각 연결에는 `id`, `source`, `target`이 있고, 필터에서 나가는 연결에는 선택적으로 `if` 또는 `else` 값의 `branch`가 있습니다.

```json
{
  "name": "Fire watch",
  "graph": {
    "nodes": [
      {"id": "camera", "position": {"x": 0, "y": 80}, "data": {"kind": "camera_source", "name": "Warehouse", "source_type": "rtsp", "url": "rtsp://192.168.1.30/stream1"}},
      {"id": "detect", "position": {"x": 240, "y": 80}, "data": {"kind": "detect", "model_id": "fire-smoke", "event_kinds": ["FIRE_DETECTED", "SMOKE_DETECTED"], "confidence_threshold": 0.3, "inference_fps": 1}},
      {"id": "alert", "position": {"x": 480, "y": 80}, "data": {"kind": "log_alert_action", "severity": "critical", "cooldown_seconds": 0}}
    ],
    "edges": [
      {"id": "e1", "source": "camera", "target": "detect"},
      {"id": "e2", "source": "detect", "target": "alert"}
    ]
  }
}
```

### 알림

| 메서드와 경로 | 설명 |
| --- | --- |
| `GET /alerts` | 최신순. 필터: `camera_id`, `workflow_id`, `severity`(`info`, `warning`, `critical`), `limit`(1~500, 기본 100) |
| `DELETE /alerts/{id}` | 알림 하나 삭제 |
| `DELETE /alerts` | 알림 삭제. 선택적으로 `camera_id` 또는 `workflow_id`로 범위 지정 |

알림에는 `id`, `workflow_id`, `camera_id`, `kind`, `severity`, `message`, `payload`(타임스탬프, 탐지, 메타데이터, Alert 앞에 Snapshot이 실행된 경우 `snapshot_url`), `created_at`이 있습니다.

### 자격 증명

| 메서드와 경로 | 설명 |
| --- | --- |
| `GET /credentials` | `id`, `name`, `type`, `summary`, 타임스탬프. 페이로드는 결코 반환되지 않습니다. |
| `POST /credentials` | `name`, `type`, `payload`. 유형별 페이로드 필드는 [액션과 통합](actions-and-integrations.md#자격-증명)에 있습니다. |
| `PATCH /credentials/{id}` | `name`과 `payload`의 부분 업데이트. `payload`를 생략하면 저장된 비밀 값이 유지됨 |
| `DELETE /credentials/{id}` | 삭제 |

### 자산

자산 라이브러리가 구성된 경우에 사용할 수 있습니다. 아니면 모든 호출이 `503`을 반환합니다.

| 메서드와 경로 | 설명 |
| --- | --- |
| `GET /assets` | 버킷의 객체 |
| `POST /assets` | `file` 필드가 있는 멀티파트 업로드. `201`을 반환 |
| `GET /assets/{key}/content` | 캐시된 파일. 재생을 위한 범위 요청 지원 |
| `DELETE /assets/{key}` | 객체 삭제 |

### 스냅샷

`GET /snapshots/<file>.jpg`는 Snapshot 액션이 쓴 이미지를 제공합니다. 요청에는 세션 쿠키가 필요합니다.

## WebSocket

`ws://<host>:8000/ws/cameras/{camera_id}`는 세션 쿠키를 제시한 클라이언트에 카메라 하나의 실시간 메시지를 스트리밍합니다. 소켓은 단방향이며 들어오는 프레임은 무시됩니다. 모든 메시지는 `type`이 있는 JSON입니다.

| 유형 | 필드 | 시점 |
| --- | --- | --- |
| `detections` | `ts`, `frame_width`, `frame_height`, `detections[]` | 처리된 모든 프레임 |
| `event` | `kind`, `ts`, `frame_width`, `frame_height`, `detections[]` | 이벤트 재알림 대기 시간을 통과한 모든 이벤트 |
| `worker` | `status`, `ts`, `reason` | 카메라 상태가 바뀔 때 |
| `alert` | `id`, `kind`, `severity`, `message`, `ts`, `snapshot_url` | Alert 액션이 실행될 때 |

탐지에는 `label`, 프레임 픽셀 단위의 `[x_min, y_min, x_max, y_max]` 형식인 `bounding_box`, `confidence`, `class_id`, `tracker_id`(또는 `null`)가 있습니다.

## 웹훅 페이로드

Webhook과 MQTT 액션은 모든 이벤트에 대해 이 JSON을 전달합니다.

```json
{
  "camera_id": "01a061c1-ff68-7611-af72-436d9d5ba907",
  "kind": "HELMET_DETECTED",
  "ts": "2026-09-02T10:58:20.983914+00:00",
  "metadata": { "count": 2, "model_id": "ppe-helmets" },
  "snapshot_url": "/snapshots/20260902-105820-01a061c1-3f9a2c1e.jpg"
}
```

`snapshot_url`은 그래프에서 앞서 Snapshot 액션이 실행된 경우에만 존재하며, API 오리진에 대한 상대 경로입니다.
