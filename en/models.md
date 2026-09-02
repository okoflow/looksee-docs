[Documentation](index.md) / Models

# Models

LookSee runs object detection models exported to ONNX. A model is installed
as a *bundle*: a directory under `models/` with the ONNX file and a manifest.
The API discovers bundles at runtime, Detect nodes choose among them, and the
inference service loads a model once and shares it between cameras.

## Bundle layout

```text
models/
├── ppe-helmets/
│   ├── manifest.json
│   └── model.onnx
└── road-objects/
    ├── manifest.json
    └── model.onnx
```

The directory name is the model id. It may contain lowercase letters,
digits, `-`, and `_`, up to 64 characters. Bundles are treated as immutable:
replace a model by adding a new directory rather than overwriting files in
place, because running workers keep the version they loaded.

`models/` is bind-mounted read-only into the `api` and `inference`
containers. A new bundle appears in Studio on the next request; no restart is
required. Model files are excluded from the repository by `.gitignore`.

## Manifest

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

| Field | Required | Meaning |
| --- | --- | --- |
| `labels` | Yes | The classes the model outputs, either as a list in class-id order or as an object mapping class ids (`"0"`, `"1"`, …) to labels. Ids must be contiguous from zero and labels unique. |
| `name` | No | Display name in Studio, 1 to 128 characters. Defaults to the id with `-` and `_` replaced by spaces and title-cased. |
| `recommended_confidence_threshold` | No | A value between 0 and 1 that Studio suggests when the model is selected. |
| `events` | No | Overrides for the event a label produces. A string names the event kind; `null` means the label produces no event. Keys must be labels from `labels`. |

Unknown fields are rejected, so a typo in a field name is reported instead of
ignored.

## From labels to events

Every label becomes an event kind unless the manifest says otherwise. The
label is upper-cased, runs of characters other than letters and digits become
`_`, and `_DETECTED` is appended:

| Label | Event kind |
| --- | --- |
| `helmet` | `HELMET_DETECTED` |
| `space-empty` | `SPACE_EMPTY_DETECTED` |
| `trafficLight-Red` | `TRAFFICLIGHT_RED_DETECTED` |
| `9mm` | `CLASS_9MM_DETECTED` |

Event kinds are what the Detect node's **Events** field and the If / Else
condition **event kind** offer. Use `events` in the manifest to give a label a
clearer name, or to silence background classes such as `other`:

```json
"events": { "other": null, "head": "NO_HELMET_DETECTED" }
```

Two labels that normalize to the same event kind, such as `space-empty` and
`space_empty`, merge into one event; give one of them an explicit name.

## Supported ONNX exports

The inference service selects an adapter by looking at the ONNX graph's input
and output names, not at the file name. Two D-FINE export layouts are
supported:

| Export | Inputs | Outputs | Notes |
| --- | --- | --- | --- |
| Deploy | `images` (float, N×3×H×W), `orig_target_sizes` (int64, N×2) | `labels`, `boxes`, `scores` | The recommended export. Frames are letterboxed to the model's square input and boxes mapped back to frame pixels. |
| Raw | one image input | `logits`, `pred_boxes` | The training-graph export. Post-processing mirrors the official D-FINE post-processor. |

A model with any other signature is rejected when the worker starts, and the
camera goes to **Error** with an *unsupported ONNX signature* reason.

To export a D-FINE checkpoint, use the export tooling of the
[D-FINE repository](https://github.com/Peterande/D-FINE) with the deploy
graph enabled, keep the default input and output names, and place the
resulting file as `model.onnx`. Any D-FINE size works; medium models balance
accuracy and CPU cost well for a few cameras.

## Execution providers

ONNX Runtime picks the first available provider in this order: CUDA, CoreML,
CPU. The published inference image contains the CPU build. For NVIDIA GPUs,
build the image with the `gpu` extra as described in
[Deployment](deployment.md#gpu-inference).

In a container with a CPU quota, the inference service caps ONNX Runtime's
thread count to the quota so several workers do not oversubscribe the host.
`INFERENCE_CPUS` in `.env` sets that quota.

## Detection and tracking

Each camera worker decodes the stream, keeps the newest frame, and runs the
model at the Detect node's **Checks per second**. Detections below the
node's **Confidence** are dropped before tracking. ByteTrack then assigns a
tracking id that persists across checks for about three seconds of lost
sight, which lets Line crossing and Dwell follow an object.

Guidance for the two Detect settings:

- **Confidence**: start from the manifest's recommendation. Lower values catch
  more and false-alarm more; raise it when alerts fire on shadows or
  reflections.
- **Checks per second**: 1 to 2 is enough for presence and zone scenarios; 4
  to 8 for counting people crossing a line or fast vehicles. Each check costs
  one model run on the CPU or GPU.

## Where to get models

LookSee ships without models, and the repository does not host any. Train or
fine-tune a D-FINE model on your classes, export it as described above, and
write a manifest for it. A bundle is a plain directory, so it can be copied
between machines or kept in your own storage.
