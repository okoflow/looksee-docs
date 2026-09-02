[Documentation](index.md) / Enterprise edition

# Enterprise edition

LookSee ships in one set of images with two editions. The Community edition
is the code outside the `ee/` directory of the repository, licensed under
Apache-2.0. The Enterprise edition is the code inside `ee/`, licensed under
the [LookSee Enterprise license](https://github.com/okoflow/looksee/blob/main/ee/LICENSE),
and unlocked with a license key.

## Features

| Feature | Nodes | What it adds |
| --- | --- | --- |
| Measurement filters | Count, Line crossing, Dwell | Counting objects inside a zone against a threshold, counting crossings of a line with a direction, and detecting objects that stay in a zone longer than a limit. All three rely on object tracking across frames. |
| Enterprise integrations | Slack | Messages to a Slack incoming webhook, with the same templates as the other messaging actions. |

In the Community edition these nodes appear in the palette with a lock and
cannot be dragged onto the canvas. A workflow that contains one of them,
for example after importing a graph through the API, is rejected on **Run**
with the error code `feature_not_licensed` and HTTP status 402.

## Enabling

Set the key in `.env` and restart the API:

```bash
LICENSE_KEY=<your key>
```

```bash
docker compose up -d api
```

`GET /entitlements` reports the active edition and features:

```json
{ "edition": "enterprise", "features": ["measurement_filters", "enterprise_integrations"] }
```

Studio reads the same endpoint and unlocks the palette. Workflows that were
saved with Enterprise nodes start working on the next **Run**.

## Licensing terms

The Enterprise license permits copying and modifying the code for development
and testing without a subscription. Production use requires a valid license
for the number of seats. Contributions to `ee/` are accepted under the same
license, as described in the repository's
[CONTRIBUTING.md](https://github.com/okoflow/looksee/blob/main/CONTRIBUTING.md).
