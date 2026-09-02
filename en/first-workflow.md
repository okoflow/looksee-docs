[Documentation](index.md) / Your first workflow

# Your first workflow

This tutorial builds a helmet compliance workflow: a camera watching a loading
dock, a model that recognizes heads, helmets, and vests, a zone that limits
detection to the dock floor, a schedule for working hours, and an alert with a
photo whenever a bare head appears. It assumes a running instance from
[Getting started](getting-started.md) and a model bundle whose labels include
`head`.

## Create the workflow

Open **Workflows** and click **New workflow**. The dialog offers a **Blank
workflow** and ready-made scenarios such as *Helmet compliance* and *Fire &
smoke*. A scenario builds the graph for you with a few parameters to fill in;
this tutorial starts from **Blank workflow** to show every step. Give the
workflow a name and create it. The editor opens on an empty canvas.

![The New workflow dialog with the built-in scenarios](../images/new-workflow.png)

## Add the camera

Drag **Camera** from the palette onto the canvas and select it. In the
inspector, give it a **Name**, keep **Source** at **RTSP**, and enter the
stream **URL** of the camera, for example
`rtsp://user:password@192.168.1.30:554/stream1`. For a quick test without a
camera, choose **Browser webcam** as the source instead; it needs no URL.

## Add detection

Drag **Detect** onto the canvas and connect the camera's output to it. Select
the node and choose the model under **Model**. **Events** lists the event
kinds the model produces; select **Head** so only bare heads continue into
the graph. Keep the suggested **Confidence** and set **Checks per second** to
`2`.

![The editor with the Detect node selected](../images/editor.png)

## Limit detection to the dock

Drag **Zone** onto the canvas and connect **Detect** to it. Draw the polygon
in the inspector by clicking corners, or use **Edit on preview** to draw on a
live frame once the workflow runs. Only detections whose centre lies inside
the polygon pass through the zone's **If** output.

![Drawing a zone polygon in the inspector](../images/editor-zone.png)

## Add working hours

Drag **Schedule** onto the canvas and connect the zone's **If** output to it.
Set **Start hour** to `6` and **End hour** to `22`, and deselect **Sat** and
**Sun** under **Weekdays**. Leave **Outside schedule** off. Events outside
the window leave through **Else**, which stays unconnected here, so they are
dropped.

Schedules use the `EVENT_TIMEZONE` of the API, UTC by default;
[Configuration](configuration.md) shows how to set the site's time zone.

## Take a photo and alert

Drag **Snapshot** onto the canvas and connect the schedule's **If** output to
it. Then drag **Alert**, connect **Snapshot** to it, set **Severity** to
**Critical**, and set **Cooldown seconds** to `60` so one worker without a
helmet produces one alert per minute, not one per check.

The Snapshot comes first so the alert carries the photo. Any other action
connected after **Snapshot** receives the same image: add **Telegram** with a
bot credential to notify a supervisor, or **Webhook** to notify another
system.

## Run it

Click **Save**, then **Run**. LookSee validates the graph and points at the
node to fix when something is missing: a camera without a URL, a Detect
without a model, an action without a credential. When the graph is valid the
workflow status turns **Active** and the camera goes from **Pending** to
**Active** within a few seconds.

Switch to **Monitor**. The camera plays live with the zone drawn over it,
detections appear as boxes, the **Live** tab lists events as they enter the
graph, and the **Alerts** tab shows each alert with its snapshot.

![The monitor with the zone overlay and detections](../images/monitor.png)

**Stop** in the header stops the workflow; the camera goes to **Off** and the
alert history stays.

## Next

- [Nodes](nodes.md) lists every field of the nodes used here and the rest of
  the palette.
- [Actions and integrations](actions-and-integrations.md) explains how to
  connect Telegram, Discord, email, MQTT, and webhooks.
- [Cameras](cameras.md) covers the other source types and what to do when a
  camera stays **Pending**.
