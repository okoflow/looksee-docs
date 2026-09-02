[문서](index.md) / 모델

# 모델

LookSee는 ONNX로 내보낸 객체 탐지 모델을 실행합니다. 모델은 *모델 번들*로 설치합니다. 모델 번들은 `models/` 아래에 ONNX 파일과 매니페스트를 담은 디렉터리입니다. API는 실행 시점에 모델 번들을 발견하고, Detect 노드는 그중에서 선택하며, 추론 서비스는 모델을 한 번 로드해 카메라 간에 공유합니다.

## 모델 번들 구조

```text
models/
├── ppe-helmets/
│   ├── manifest.json
│   └── model.onnx
└── road-objects/
    ├── manifest.json
    └── model.onnx
```

디렉터리 이름이 모델 ID입니다. 소문자 영문자, 숫자, `-`, `_`를 최대 64자까지 쓸 수 있습니다. 모델 번들은 변경 불가로 취급합니다. 실행 중인 워커는 로드한 버전을 계속 유지하므로, 모델을 교체할 때는 파일을 덮어쓰지 말고 새 디렉터리를 추가합니다.

`models/`는 `api`와 `inference` 컨테이너에 읽기 전용으로 바인드 마운트됩니다. 새 모델 번들은 다음 요청에서 Studio에 나타나며 재시작이 필요 없습니다. 모델 파일은 `.gitignore`에 의해 저장소에서 제외됩니다.

## 매니페스트

```json
{
  "name": "Safety gear (PPE)",
  "labels": ["head", "helmet", "vest"],
  "recommended_confidence_threshold": 0.4,
  "events": {
    "head": "NO_HELMET_DETECTED"
  }
}
```

| 필드 | 필수 | 의미 |
| --- | --- | --- |
| `labels` | 예 | 모델이 출력하는 클래스. 클래스 ID 순서의 목록이거나, 클래스 ID(`"0"`, `"1"`, …)를 레이블에 대응시키는 객체. ID는 0부터 연속해야 하고 레이블은 고유해야 합니다. |
| `name` | 아니요 | Studio에 표시되는 이름, 1~128자. 기본값은 ID에서 `-`와 `_`를 공백으로 바꾸고 단어 첫 글자를 대문자로 만든 값입니다. |
| `recommended_confidence_threshold` | 아니요 | 모델을 선택했을 때 Studio가 제안하는 0과 1 사이의 값. |
| `events` | 아니요 | 레이블이 만드는 이벤트의 재정의. 문자열은 이벤트 종류 이름이고, `null`은 해당 레이블이 이벤트를 만들지 않음을 뜻합니다. 키는 `labels`에 있는 레이블이어야 합니다. |

알 수 없는 필드는 거부되므로, 필드 이름의 오타는 무시되지 않고 보고됩니다.

## 레이블에서 이벤트로

매니페스트에서 달리 정하지 않으면 모든 레이블이 이벤트 종류가 됩니다. 레이블을 대문자로 바꾸고, 영문자와 숫자가 아닌 문자의 연속을 `_`로 바꾸고, 숫자로 시작하는 레이블에는 `CLASS_` 접두사를 붙인 뒤, `_DETECTED`를 덧붙입니다.

| 레이블 | 이벤트 종류 |
| --- | --- |
| `helmet` | `HELMET_DETECTED` |
| `space-empty` | `SPACE_EMPTY_DETECTED` |
| `trafficLight-Red` | `TRAFFICLIGHT_RED_DETECTED` |
| `9mm` | `CLASS_9MM_DETECTED` |

이벤트 종류는 Detect 노드의 **Events** 필드와 If / Else 조건의 **event kind**가 제시하는 항목입니다. 레이블에 더 명확한 이름을 붙이거나 `other` 같은 배경 클래스를 침묵시키려면 매니페스트의 `events`를 사용합니다.

```json
"events": { "other": null, "head": "NO_HELMET_DETECTED" }
```

`space-empty`와 `space_empty`처럼 같은 이벤트 종류로 정규화되는 두 레이블은 하나의 이벤트로 합쳐집니다. 둘 중 하나에 명시적인 이름을 지정하십시오.

## 지원되는 ONNX 내보내기 형식

추론 서비스는 파일 이름이 아니라 ONNX 그래프의 입력과 출력 이름을 보고 어댑터를 선택합니다. 두 가지 D-FINE 내보내기 레이아웃을 지원합니다.

| 내보내기 | 입력 | 출력 | 참고 |
| --- | --- | --- | --- |
| Deploy | `images`(float, N×3×H×W), `orig_target_sizes`(int64, N×2) | `labels`, `boxes`, `scores` | 권장 내보내기 방식. 프레임을 모델의 정사각형 입력에 맞게 레터박스 처리하고 상자를 프레임 픽셀 좌표로 되돌립니다. |
| Raw | 이미지 입력 하나 | `logits`, `pred_boxes` | 학습 그래프 내보내기. 후처리는 공식 D-FINE 후처리기를 그대로 따릅니다. |

다른 시그니처의 모델은 워커 시작 시 거부되고, 카메라는 *unsupported ONNX signature* 이유와 함께 **Error** 상태가 됩니다.

D-FINE 체크포인트를 내보내려면 [D-FINE 저장소](https://github.com/Peterande/D-FINE)의 내보내기 도구를 deploy 그래프를 켠 상태로 사용하고, 기본 입력·출력 이름을 유지한 뒤, 결과 파일을 `model.onnx`로 둡니다. D-FINE의 어떤 크기든 동작하며, 카메라 몇 대 규모에서는 medium 모델이 정확도와 CPU 비용의 균형이 좋습니다.

## 실행 공급자

ONNX Runtime은 CUDA, CoreML, CPU 순서로 사용할 수 있는 첫 공급자를 선택합니다. 공개된 추론 이미지에는 CPU 빌드가 들어 있습니다. NVIDIA GPU를 쓰려면 [배포](deployment.md#gpu-추론)의 설명대로 `gpu` extra로 이미지를 빌드합니다.

CPU 할당량이 있는 컨테이너에서는 추론 서비스가 ONNX Runtime의 스레드 수를 할당량에 맞게 제한해, 여러 워커가 호스트를 초과 점유하지 않도록 합니다. `.env`의 `INFERENCE_CPUS`가 그 할당량을 정합니다.

## 탐지와 추적

각 카메라 워커는 스트림을 디코딩하고 최신 프레임을 유지하며, Detect 노드의 **Checks per second**(초당 검사 횟수) 속도로 모델을 실행합니다. 노드의 **Confidence**(신뢰도)보다 낮은 탐지는 추적 전에 버려집니다. 그다음 ByteTrack이 추적 ID를 부여하는데, 이 ID는 객체가 약 3초 동안 시야에서 사라져도 검사 사이에 유지되므로 Line crossing과 Dwell이 객체를 따라갈 수 있습니다.

두 Detect 설정에 대한 지침:

- **Confidence**: 매니페스트의 권장값에서 시작합니다. 값을 낮추면 더 많이 잡아내지만 오탐도 늘어납니다. 그림자나 반사에 알림이 울리면 값을 높입니다.
- **Checks per second**: 존재 여부나 구역 시나리오에는 1~2로 충분합니다. 선을 넘는 사람이나 빠른 차량을 세려면 4~8이 필요합니다. 검사 한 번은 CPU나 GPU에서 모델 실행 한 번을 소모합니다.

## 모델을 구하는 곳

LookSee는 모델 없이 배포되며, 저장소도 모델을 제공하지 않습니다. 원하는 클래스로 D-FINE 모델을 학습하거나 미세 조정하고, 위 설명대로 내보낸 뒤, 매니페스트를 작성합니다. 모델 번들은 일반 디렉터리이므로 머신 간에 복사하거나 자체 저장소에 보관할 수 있습니다.
