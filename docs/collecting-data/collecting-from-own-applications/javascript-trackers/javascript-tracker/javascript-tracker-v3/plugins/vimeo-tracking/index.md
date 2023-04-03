---
title: "Vimeo Tracking"
date: "2022-01-12"
sidebar_position: 17000
---

```mdx-code-block
import Block5966 from "@site/docs/reusable/javascript-tracker-release-badge-v3/_index.md"

<Block5966/>

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
```

# Snowplow Media Tracking

This plugin will allow the tracking of an embedded Vimeo Iframe or Player.

| Tracker Distribution | Included |
| -------------------- | -------- |
| `sp.js`              | ❌        |
| `sp.lite.js`         | ❌        |

## Download

<table><tbody><tr><td>Download from GitHub Releases (Recommended)</td><td><a href="https://github.com/snowplow/snowplow-javascript-tracker/releases">Github Releases (plugins.umd.zip)</a></td></tr><tr><td>Available on jsDelivr</td><td><a href="https://cdn.jsdelivr.net/npm/@snowplow/browser-plugin-youtube-tracking@latest/dist/index.umd.min.js">jsDelivr</a> (latest)</td></tr><tr><td>Available on unpkg</td><td><a href="https://unpkg.com/@snowplow/browser-plugin-youtube-tracking@latest/dist/index.umd.min.js">unpkg</a> (latest)</td></tr></tbody></table>

## Quick Start

The snippets below show how to get started with the plugin, after [setting up your tracker](/docs/collecting-data/collecting-from-own-applications/javascript-trackers/javascript-tracker/web-quick-start-guide/index.md). 

:::info The plugin's `id` attribute will accept:

- The `id` of an `iframe` element
- An existing instance of `Vimeo.Player`, created with the [Vimeo Player SDK](https://developer.vimeo.com/player/sdk)

:::

<Tabs>
  <TabItem value="iframe" label="Iframe" default>

```html
<!DOCTYPE html>
<html>
  <body>
    <iframe
      id="vimeo-player"
      src="https://player.vimeo.com/video/535907279?h=db7ea8b89c"
    ></iframe>


    <script>
      window.snowplow(
        'addPlugin',
        'https://cdn.jsdelivr.net/npm/@snowplow/browser-plugin-vimeo-tracking@latest/dist/index.umd.min.js',
        ['snowplowVimeoTracking', 'VimeoTrackingPlugin']
      );

      window.snowplow('enableVimeoTracking', {
        id: 'vimeo-player'
      });
    </script>
  </body>
</html>
```

</TabItem>
<TabItem value="player" label="Vimeo.Player Instance">

```html
<!DOCTYPE html>
<html>
  <body>
    <div id=" vimeo-player"></div>

    <script>
      window.snowplow(
        'addPlugin',
        'https://cdn.jsdelivr.net/npm/@snowplow/browser-plugin-vimeo-tracking@latest/dist/index.umd.min.js',
        ['snowplowVimeoTracking', 'VimeoTrackingPlugin']
      );

      const player = new Vimeo.Player(' vimeo-player', {
        videoId: 'zSM4ZyVe8xs'
      });

      window.snowplow('enableVimeoTracking', {
          id: player
      });
    </script>
  </body>
</html>
```

  </TabItem>
</Tabs>

## The enableVimeoTracking function

The `enableVimeoTracking` function takes the form:

```javascript
window.snowplow("enableVimeoTracking", { id, options?: { label?, captureEvents?, boundaries? } })
```

| Parameter               | Type                       | Default             | Description                                                                                                    | Required |
| ----------------------- | -------------------------- | ------------------- | -------------------------------------------------------------------------------------------------------------- | -------- |
| `id`                    | `string` or `Vimeo.Player` | \-                  | The HTML id attribute of the media element, or an instance of `Vimeo.Player`                                   | Yes      |
| `options.label`         | `string`                   | \-                  | An identifiable custom label sent with the event                                                               | No       |
| `options.captureEvents` | `string[]`                 | `["DefaultEvents"]` | The events or Event Group to capture. For a full list of events and groups, check the [section below](#events) | No       |
| `options.boundaries`    | `number[]`                 | `[10, 25, 50, 75]`  | The progress percentages to fire an event at (valid values 1 - 99 inclusive) [[1]](#1)                         | No       |


Below is an example of the full `enableVimeoTracking` function:

```javascript
window.snowplow('enableVimeoTracking', {
  id: ' vimeo-player',
  options: {
    label: 'My Custom Video Label',
    captureEvents: ['play', 'pause', 'ended'],
    boundaries: [20, 80],
  }
})
```

## Events

### Capturable Events

Below is a table of all the events that can be used in `options.captureEvents`

| Name                           | Fire Condition                                                                                                                               | Reference                                                                                             |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| play                           | When the video plays.                                                                                                                        | [link](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/play_event)                  |
| playing                        | After playback is first started, and whenever it is restarted.                                                                               | [link](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/playing_event)               |
| pause                          | When the video is paused.                                                                                                                    | [link](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/pause_event)                 |
| ended                          | When playback reaches the end of a video.                                                                                                    | [link](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/ended_event)                 |
| timeupdate                     | When the playback position of the video changes, generally every 250 ms during playback, but the interval can vary depending on the browser. | [link](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/timeupdate_event)            |
| progress                       | While the video is loading.                                                                                                                  | [link](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/progress_event)              |
| seeking                        | When a seek operation starts                                                                                                                 | [link](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/seeking_event)               |
| seeked                         | When the player seeks to a specific time. A simultaneous timeupdate event also fires.                                                        | [link](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/seeked_event)                |
| texttrackchange                | When the active text track of the captions or subtitles kind changes.                                                                        | [Vimeo](https://developer.vimeo.com/player/sdk/reference#texttrackchange)                             |
| chapterchange                  | When the current chapter changes, or when the active text track changes.                                                                     |                                                                                                       |
| cuepoint                       | When the playback position hits a registered cue point.                                                                                      | [Vimeo](https://developer.vimeo.com/player/sdk/reference#methods-for-cue-points)                      |
| volumechange                   | When the volume in the player changes.                                                                                                       | [link](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/volumechange_event)          |
| playbackratechange             | When the playback rate of the video in the player changes.                                                                                   | [link](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/playbackratechange_event)    |
| bufferstart                    | When buffering starts in the player, during preload, and during seeking.                                                                     | [link](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/bufferstart_event)           |
| bufferend                      | When buffering ends in the player, at the end of preload, and at the end of seeking.                                                         | [link](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/bufferend_event)             |
| error                          | When the player experiences an error                                                                                                         | [Vimeo](https://developer.vimeo.com/player/sdk/reference#error)                                       |
| loaded                         | When a new video is loaded in the player.                                                                                                    | [link](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/loaded_event)                |
| durationchange                 | when the duration attribute has been updated.                                                                                                | [link](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/durationchange_event)        |
| fullscreenchange               | After the browser switches into or out of fullscreen mode.                                                                                   | [link](https://developer.mozilla.org/en-US/docs/Web/API/Document/fullscreenchange_event)              |
| qualitychange                  |                                                                                                                                              |                                                                                                       |
| camerachange                   |                                                                                                                                              |                                                                                                       |
| resize                         |                                                                                                                                              |                                                                                                       |
| enterpictureinpicture          | When the video enters picture-in-picture mode successfully.                                                                                  | [link](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/enterpictureinpicture_event) |
| leavepictureinpicture          | When the video leaves picture-in-picture mode successfully.                                                                                  | [link](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/leavepictureinpicture_event) |
| interactivehotspotclicked      | When a hotspot is clicked.                                                                                                                   | [Vimeo](https://developer.vimeo.com/player/sdk/reference#interactivehotspotclicked)                   |
| interactiveoverlaypanelclicked | When the buttons or images within the interactive overlay panel are clicked.                                                                 | [Vimeo](https://developer.vimeo.com/player/sdk/reference#interactiveoverlaypanelclicked)              |
| percentprogress                | When a percentage boundary set in `options.boundaries` is reached                                                                            |                                                                                                       |

### Event Groups

You can also use a pre-made event group in `options.captureEvents`:

| Name            | Events                                                                                                                                        |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `DefaultEvents` | `["play", "pause", "ended", "seeked", "volumechange", "playbackratechange", "qualitychange", "cuepoint", "chapterchange", "texttrackchange"]` |
| `AllEvents`     | Every event listed in [Capturable Events](#capturable-events)                                                                                 |

It is possible to extend an event group with any event in the Events table above. This could be useful if you want, for example, all the events contained in the "DefaultEvents" group, along with the "error" event. This is expressed in the following way:

```javascript
window.snowplow('enableVimeoTracking', {
  id: " vimeo-player",
  options: {
    captureEvents: ["DefaultEvents", "error"],
  }
})
```

## Schemas and Example Data

# TODO

Three schemas are used with this plugin:

### [An unstructured event with identifying information](https://github.com/snowplow/iglu-central/blob/master/schemas/com.snowplowanalytics.snowplow/media_player_event/jsonschema/1-0-0)

```javascript
{
    "type": "play",
    "label": "Identifying Label"
}
```

### [Snowplow platform-agnostic media context](https://github.com/snowplow/iglu-central/blob/master/schemas/com.snowplowanalytics.snowplow/media_player/jsonschema/1-0-0)

```javascript
{
    "currentTime": 12.32,
    "duration": 20,
    "ended": false,
    "loop": false,
    "muted": true,
    "paused": false,
    "playbackRate": 1,
    "volume": 100
}
```

### [Vimeo player specific context](https://github.com/snowplow/iglu-central/blob/master/schemas/org.whatwg/media_element/jsonschema/1-0-0)

```javascript
{
  "autoPlay": false,
  "avaliablePlaybackRates": [
    0.25, 0.5, 0.75, 1, 1.25, 1.5, 1.75, 2
  ],
  "buffering": false,
  "controls": true,
  "cued": false,
  "loaded": 17,
  "playbackQuality": "hd1080",
  "playerId": "youtube",
  "unstarted": false,
  "url": "https://www.youtube.com/watch?v=zSM4ZyVe8xs",
  "yaw": 0,
  "pitch": 0,
  "roll": 0,
  "fov": 100.00004285756798,
  "avaliableQualityLevels": [
    "hd2160",
    "hd1440",
    "hd1080",
    "hd720",
    "large",
    "medium",
    "small",
    "tiny",
    "auto"
  ]
}
```


