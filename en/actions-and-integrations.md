[Documentation](index.md) / Actions and integrations

# Actions and integrations

Actions are the nodes at the end of a workflow: they turn an event into an
alert, a snapshot, or a message to an external system. Actions that talk to
a service authenticate with a credential stored in LookSee.

## Credentials

The **Credentials** page lists stored credentials with a name, a type, and a
non-secret summary such as the SMTP host or the webhook's domain. **New
credential** asks for a **Name**, a **Type**, and the fields of that type. The secret
itself is encrypted with a key derived from the instance secret and is never
shown again; editing a credential without entering a new secret keeps the
stored one.

| Type | Used by | Payload fields |
| --- | --- | --- |
| `telegram_bot` | Telegram | `bot_token` |
| `discord_webhook` | Discord | `webhook_url` |
| `slack_webhook` | Slack (Enterprise) | `webhook_url` |
| `smtp` | Email | `host`, `port` (default 587), `username`, `password`, `from_address`, `starttls` (default on) |
| `mqtt` | MQTT | `host`, `port` (default 1883), `username`, `password` |

Actions reference a credential by id and check its type when the workflow
starts: a Telegram action with an SMTP credential fails **Run** with
`credential_type_mismatch`. Credentials are resolved when a message is
delivered, so replacing a token applies to running workflows immediately.

![The Credentials page](../images/credentials.png)

## Message templates

Telegram, Discord, Slack, email, and MQTT actions format their text from a
template with these placeholders:

| Placeholder | Value |
| --- | --- |
| `{kind}` | The event kind, such as `HELMET_DETECTED` |
| `{camera_id}` | The camera id |
| `{ts}` | The event time in ISO 8601 |
| `{count}` | How many detections produced the event |
| `{snapshot_url}` | The snapshot path when a Snapshot ran earlier, otherwise empty |

The default message template is `[{kind}] camera={camera_id} at {ts}`. A
template with an unknown placeholder is sent as written and a warning is
logged.

## Alert

Records the event in the alert history and shows it in the monitor. Fields:
**severity** (`info`, `warning`, `critical`) and **cooldown** in seconds (0
to 3600, default 30). Within the cooldown, repeats on the same camera are
dropped; `0` records every event. Alerts are the only action with a history
inside LookSee; [Monitoring and alerts](monitoring-and-alerts.md) covers it.

## Snapshot

Saves the camera's latest frame as a JPEG, drawn with the detections when
**annotate** is on (the default). Snapshot is the one action that has an
output: connect the actions that should carry the picture after it. They
receive `snapshot_url` for templates and payloads, and Telegram, Discord,
Slack, and email attach the image itself.

## Webhook

Sends the event as JSON to a URL with the chosen method (`POST` by default,
`GET` or `PUT`). No credential is needed; put a token into the URL if the
receiver requires one. The request times out after five seconds, and a
non-2xx response is logged, not retried. The payload is documented in
[API](api.md#webhook-payload).

## Telegram

Sends a message, or a photo with a caption when a snapshot is available, to a
chat. Create a bot with Telegram's [@BotFather](https://t.me/botfather), store
its token as a `telegram_bot` credential, add the bot to the target chat, and
enter the chat id in the action. For a private chat, message the bot first
and read the chat id from `https://api.telegram.org/bot<token>/getUpdates`.

## Discord

Posts to a channel through an incoming webhook. Create the webhook in the
channel's integration settings, store its URL as a `discord_webhook`
credential, and set the message template. Discord caps messages at 2000
characters; longer texts are truncated.

## Slack

Enterprise edition. Posts to a channel through a Slack incoming webhook
stored as a `slack_webhook` credential, with the same template placeholders.

## Email

Sends a message through SMTP with a subject and a body template and attaches
the snapshot when one is available. Store the server as an `smtp` credential.
`starttls` upgrades the connection after connecting, which is what port 587
expects; the sender is `from_address`, or `username` when it is empty.
Field: **to**, one address or a comma-separated list accepted by your server.

## MQTT

Publishes to a topic on a broker stored as an `mqtt` credential. Fields:
**topic** (default `looksee/events`) and **payload template**. An empty
template publishes the same JSON as the webhook; a template publishes its
rendered text. Connections time out after five seconds.

## Delivery behaviour

- Actions run in graph order for every event that reaches them. An action
  that fails logs the error and does not stop the other actions.
- Each action runs at most once per event, even when two branches lead to it.
- Secrets never appear in logs; delivery failures are logged with the action
  and the status only.
- Outbound requests share one connection pool and go directly from the API
  container; configure the proxy or firewall accordingly.
