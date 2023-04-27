```mdx-code-block
import CodeBlock from '@theme/CodeBlock';
import TOCInline from '@theme/TOCInline';
```

```mermaid
timeline
    title Sample media tracking scenario
    [0s] Video player appeared :  [0s] Call start media tracking : [0s] Track ready event : [30s] Ping event
    [40s] User started playing the video : [40s] Track play event : [60s] Ping event
    [65s] Ad started playing : [65s] Track ad break start : [65s] Track ad start : [70s] Track ad skip : [70s] Track ad break end
    [90s] Video resumed playing after ad : [90s] Ping event : [100s] Track end event
    [120s] Video player disappeared : [120s] Call end media tracking
```

The Snowplow media tracking APIs enable you to track events from media playback on the Web as well as mobile apps.
The trackers provide a set of tracking APIs that enable you to track changes in the media playback (e.g., play, pause, seek events), playback position (ping and percentage progress events), or ad playback events (e.g., ad breaks, ad progress, ad clicks).

While the trackers provide integrations with a few media players, the media tracking APIs are designed to be player agnostic to enable users to implement tracking for any media player.

<TOCInline toc={toc} maxHeadingLevel={4} />

## Tracked events and entities

The media tracking works with a set of out-of-the-box event and entity schemas.
Additionally, you can track custom entities along with the events.

:::info Example app
To illustrate the tracked events and entities, you can visit an example app that showcases the tracked media events and entities live as you watch a video.

[Visit the app here.](https://snowplow-media-demo.tomlein.org/media)
Source code for the app is [available here.](https://github.com/snowplow-incubator/snowplow-javascript-tracker-examples/tree/master/react)
:::

:::warning
TODO:
Update the link to the example when it is deployed on Github Pages.
:::

### Media player events

Each media player event is a self-describing event with a unique schema.
The schemas contain a single `label` property:

Request Key | Required | Type/Format | Description
-- | -- | -- | --
label | N | string | A custom identifier for the media player

The schema URIs have the format:
`iglu:com.snowplowanalytics.snowplow/media_player_event_{EVENT_TYPE}/jsonschema/1-0-0`.

You can see the schemas listed under the event tracking methods below.

### Media player entity

The media player entity is attached to all media events and gives information about the current state of the media player.
It contains the current playback position (`currentTime`) as well as the paused or muted state.

<details>
    <summary>Media player entity properties</summary>

| Request Key | Required | Type/Format | Description |
| --- | --- | --- | --- |
| currentTime | Y | number | The current playback time |
| duration | N | number | A double-precision floating-point value indicating the duration of the media in seconds |
| ended | Y | boolean | If playback of the media has ended |
| isLive | Y | boolean | If the media is live |
| loop | Y | boolean | If the video should restart after ending |
| muted | Y | boolean | If the media element is muted |
| paused | Y | boolean | If the media element is paused |
| percentProgress | N | integer | The percent of the way through the media |
| playbackRate | Y | number | Playback rate (1 is normal) |
| volume | Y | integer | Volume percent |
</details>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player/jsonschema/1-0-0`.

### Media player session entity

The media player session entity is used to identify the playback using the `mediaSessionId`.
It also contains statistics about the media playback computed on the tracker (e.g., `timePlayed`, `timeBuffering`, `adsClicked`).

<details>
    <summary>Media player session entity properties</summary>

| Request Key | Required | Type/Format | Description |
| --- | --- | --- | --- |
| mediaSessionId | Y | string | An identifier for the media session (can be provided by the user) |
| startedAt | N | date-time | Date-time timestamp of when the session started. |
| pingInterval | N | number | Interval (seconds) in which the ping events will be sent. Default (10s) is assumed if not specified. |
| timePlayed | N | number | Total seconds user spent playing content (excluding ads). |
| timePlayedMuted | N | number | Total seconds user spent playing content on mute (excluding ads). |
| timePaused | N | number | Total seconds user spent with paused content (excluding linear ads) |
| contentWatched | N | number | Total seconds of the content played. Each part of the content played is counted once (i.e., counts rewinding or rewatching the same content only once). Playback rate does not affect this value. |
| timeBuffering | N | number | Total seconds that playback was buffering during the session. |
| timeSpentAds | N | number | Total seconds that ads played during the session. |
| ads | N | integer | Number of ads played. |
| adsClicked | N | integer | Number of ads that the user clicked on |
| adsSkipped | N | integer | Number of ads that the user skipped |
| adBreaks | N | integer | Number of ad breaks played. |
| avgPlaybackRate | N | number | Average playback rate (1 is normal speed). |
</details>

It is an optional entity that is enabled by default.

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_session/jsonschema/1-0-0`.

### Media player ad and ad break entities

These entities give information about the currently playing ad and ad break.

<details>
    <summary>Media player ad break entity properties</summary>

| Request Key | Required | Type/Format | Description |
| --- | --- | --- | --- |
| name | N | string | Ad break name such as pre-roll, mid-roll, and post-roll. |
| breakId | N | string | ID of the ad break. |
| startTime | Y | number | Playback time in seconds at the start of the ad break. |
| breakType | N | enum: linear, nonlinear, companion | linear  – take full control of the video for a period of time
nonlinear – run concurrently to the video
companion – accompany the video but placed outside the player |
</details>

*Schema for the ad break entity:*
`iglu:com.snowplowanalytics.snowplow/media_player_ad_break/jsonschema/1-0-0`.

<details>
    <summary>Media player ad entity properties</summary>

| Request Key | Required | Type/Format | Description |
| --- | --- | --- | --- |
| name | N | string | Friendly name of the ad |
| adId | Y | string | Unique identifier for the ad. |
| creativeId | N | string | The ID of the ad creative |
| podPosition | N | integer | The number position of the ad within the ad break, starting with 1. |
| percentProgress | Y | number | The percent of the way through the ad |
| duration | Y | number | Length of the video ad in seconds |
| skippable | N | boolean | Indicating whether skip controls are made available to the end user |
</details>

*Schema for the ad entity:*
`iglu:com.snowplowanalytics.snowplow/media_player_ad/jsonschema/1-0-0`.

## Starting and ending media tracking

The tracker keeps track of ongoing media tracking instances in order to manage entities that are tracked along with the media events.

Media tracking instances are identified by an ID (that is tracked in the media player session entity as `mediaSessionId`).
You provide the identifier in the `startMediaTracking` call that initializes the media tracking instance.
All subsequent media track calls will be processed within this media tracking if given the same ID.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`const id = 'XXXXX'; // randomly generated ID
window.snowplow('startMediaTracking', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`import { startMediaTracking } from "@snowplow/browser-plugin-media";
const id = 'XXXXX'; // randomly generated ID
startMediaTracking({ id });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`let id = "XXXXX"
let tracker = Snowplow.defaultTracker()
let mediaTracking = tracker.media.startMediaTracking(id: id, label: nil)
`}
</CodeBlock>)}</>

<>{(props.tracker == 'android-kotlin') && (<CodeBlock language="kotlin">
{`// TODO: not implemented yet for Kotlin`}
</CodeBlock>)}</>

<>{(props.tracker == 'android-java') && (<CodeBlock language="java">
{`// TODO: not implemented yet for Java`}
</CodeBlock>)}</>

Use the `endMediaTracking` call to end media tracking. This will clear the local state for the media tracking and stop any background updates.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('endMediaTracking', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`import { endMediaTracking } from "@snowplow/browser-plugin-media";
endMediaTracking({ id });
`}
</CodeBlock>)}</>

### Configuration

You can provide additional configuration to the `startMediaTracking` call to configure the tracking or give initial information about the media played.

#### Label

The `label` property is a custom optional identifier that is tracked in the body of each event.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('startMediaTracking', { id, label: 'Video on homepage' });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`startMediaTracking({ id, label: 'Video on homepage' });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`let mediaTracking = tracker.media.startMediaTracking(id: id, label: "Video on homepage")
`}
</CodeBlock>)}</>

#### Media player properties

The `media` property lets you provide information about the media playback that will be used to populate the media player entity tracked with media events.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('startMediaTracking', {
    id,
    media: {
        currentTime: 0, // The current playback time
        duration: 150, // A double-precision floating-point value indicating the duration of the media in seconds
        ended: false // If playback of the media has ended
        isLive: false, // If the media is live
        loop: false, // If the video should restart after ending
        muted: false, // If the media element is muted
        paused: false, // If the media element is paused
        playbackRate: 1.0, // Playback rate (1 is normal)
        volume: 100, // Volume level
    }
});`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`startMediaTracking({
    id,
    media: {
        currentTime: 0, // The current playback time
        duration: 150, // A double-precision floating-point value indicating the duration of the media in seconds
        ended: false // If playback of the media has ended
        isLive: false, // If the media is live
        loop: false, // If the video should restart after ending
        muted: false, // If the media element is muted
        paused: true, // If the media element is paused
        playbackRate: 1.0, // Playback rate (1 is normal)
        volume: 100, // Volume level
    }
});
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`let media = MediaUpdate()
    .currentTime(0) // The current playback time
    .duration(150.0) // A double-precision floating-point value indicating the duration of the media in seconds
    .ended(false) // If playback of the media has ended
    .isLive(false) // If the media is live
    .loop(false) // If the video should restart after ending
    .muted(false) // If the media element is muted
    .paused(true) // If the media element is paused
    .playbackRate(1.0) // Playback rate (1 is normal)
    .volume(100) // Volume level
let mediaTracking = tracker.media.startMediaTracking(id: id, label: nil, media: media)
`}
</CodeBlock>)}</>

#### Ping events

Media ping events (not to be confused with page ping events) are events sent in a regular interval while media tracking is active.
They inform about the current state of the media playback.

By default, ping events are sent every 30 seconds.
This can be configured as follows:

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('startMediaTracking', {
    id,
    pings: { pingInterval: 30 }, // Interval in seconds for sending ping events. Defaults to 30s. 
});`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`startMediaTracking({
    id,
    pings: { pingInterval: 30 }, // Interval in seconds for sending ping events. Defaults to 30s. 
});
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`let configuration = MediaTrackingConfiguration(id: id)
    .pingInterval(30)
let mediaTracking = tracker.media.startMediaTracking(configuration: configuration)
`}
</CodeBlock>)}</>

The ping events are sent in an interval that is unrelated to the media playback.
However, to prevent sending too many events while the player is paused in background (e.g., in a background tab), there is a limit to how many ping events can be sent while the media is paused.
By default, this is set to 1, but it is configurable:

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('startMediaTracking', {
    id,
    pings: { maxPausedPings: 3 }, // Maximum number of consecutive ping events to send when playback is paused. Defaults to 1.
});`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`startMediaTracking({
    id,
    pings: { maxPausedPings: 3 }, // Maximum number of consecutive ping events to send when playback is paused. Defaults to 1.
});
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`let configuration = MediaTrackingConfiguration(id: id)
    .maxPausedPings(3)
let mediaTracking = tracker.media.startMediaTracking(configuration: configuration)
`}
</CodeBlock>)}</>

You can disable ping events as follows:

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('startMediaTracking', { id, pings: false });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`startMediaTracking({ id, pings: false });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`let configuration = MediaTrackingConfiguration(id: id)
    .pings(false)
let mediaTracking = tracker.media.startMediaTracking(configuration: configuration)
`}
</CodeBlock>)}</>

Media ping events have the following schema:
`iglu:com.snowplowanalytics.snowplow/media_player_event_ping/jsonschema/1-0-0`.

#### Media player session

Tracking the media player session entity with all media events is enabled by default, you can disable it as follows:

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('startMediaTracking', { id, session: false });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`startMediaTracking({ id, session: false });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`let configuration = MediaTrackingConfiguration(id: id)
    .session(false)
let mediaTracking = tracker.media.startMediaTracking(configuration: configuration)
`}
</CodeBlock>)}</>

#### Percentage progress events

Percentage progress events are tracked when the playback reaches some percentage boundaries.
To send percentage progress events, set the percentage boundaries when they should be tracked:

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('startMediaTracking', { id, boundaries: [10, 25, 50, 75] });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`startMediaTracking({ id, boundaries: [10, 25, 50, 75] });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`let configuration = MediaTrackingConfiguration(id: id)
    .boundaries([10, 25, 50, 75])
let mediaTracking = tracker.media.startMediaTracking(configuration: configuration)
`}
</CodeBlock>)}</>

Percentage progress events have the following schema:
`iglu:com.snowplowanalytics.snowplow/media_player_event_percent_progress/jsonschema/1-0-0`.

#### Filter captured events

In case you want to discard some of the events that are tracked during the media tracking, you can set the `captureEvents` property to a list of event types that are allowed (all other event types will be ignored):

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('startMediaTracking', { id, captureEvents: ['play'] });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`import { MediaPlayerEventType }from "@snowplow/browser-plugin-media";
startMediaTracking({ id, captureEvents: [MediaPlayerEventType.Play] });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`let configuration = MediaTrackingConfiguration(id: id)
    .captureEvents([.play])
let mediaTracking = tracker.media.startMediaTracking(configuration: configuration)
`}
</CodeBlock>)}</>

#### Add context entities

You can provide custom context entities to describe the media playback.
They will be added to all events tracked in the media tracking.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('startMediaTracking', {
    id,
    context: [
        {
            schema: 'iglu:org.schema/video/jsonschema/1-0-0',
            data: { creativeId: '1234' }
        }
    ]
});`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`startMediaTracking({
    id,
    context: [
        {
            schema: 'iglu:org.schema/video/jsonschema/1-0-0',
            data: { creativeId: '1234' }
        }
    ]
});
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`let configuration = MediaTrackingConfiguration(id: id)
    .entities([
        SelfDescribingJson(
            schema: "iglu:org.schema/video/jsonschema/1-0-0",
            andData: ["creativeId": "1234"]
        )
    ])
let mediaTracking = tracker.media.startMediaTracking(configuration: configuration)
`}
</CodeBlock>)}</>

## Updating playback properties

Updates stored attributes of the media player such as the current playback.
Use this function to continually update the player attributes so that they can be sent in the background ping events.
We recommend updating the playback position every 1 second.


<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('updateMediaPlayer', {
    id,
    media: { currentTime: 10 }
});`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`import { updateMediaPlayer } from "@snowplow/browser-plugin-media";
updateMediaPlayer({
    id,
    media: { currentTime: 10 }
});
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.update(media: MediaUpdate().currentTime(10.0))
`}
</CodeBlock>)}</>

## Tracking media events

Having started a media tracking instance, you can use it to track media events as you receive them from the media player.

Typically, you would subscribe to notifications from the media player (e.g., user clicks play, volume changes, content is buffering) with callbacks that would track the Snowplow events.
For an example, see the code that subscribes for events from an HTML5 media player [here](https://github.com/snowplow-incubator/snowplow-javascript-tracker-examples/blob/issue/media_plugin_example/react/src/components/video.jsx).

:::warning
TODO:
Update the link to the example code once we merge it into master.
:::

### Providing additional information

This section explains how to update information in the media entities along with tracked events.

#### Update media player properties

You can update properties for the media player entity as events are tracked.
The updated properties will apply for the current and all following events.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackMediaVolumeChange', {
    id,
    media: { volume: 33 }
});`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`import { trackMediaVolumeChange } from "@snowplow/browser-plugin-media";
trackMediaVolumeChange({
    id,
    media: { volume: 33 }
});
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.volumeChange, media: MediaUpdate().volume(33))
`}
</CodeBlock>)}</>

#### Update ad and ad break properties

When tracking ad events, you can attach information about the currently playing ad or ad break.
It is only necessary to attach this information once – at the start of the ad break for ad break properties, or at the start of an ad for ad properties.
The tracker will remember the information and attach it to all following ad events until the next ad complete or ad break end event.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackMediaAdStart', {
    id,
    ad: {
        name: 'Podcast Ad', // Friendly name of the ad
        adId: '1234', // Unique identifier for the ad
        creativeId: '4321', // The ID of the ad creative
        duration: 15, // Length of the video ad in seconds
        skippable: true, // Indicating whether skip controls are made available to the end user
    }
});`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`import { trackMediaAdStart } from "@snowplow/browser-plugin-media";
trackMediaAdStart({
    id,
    ad: {
        name: 'Podcast Ad', // Friendly name of the ad
        adId: '1234', // Unique identifier for the ad
        creativeId: '4321', // The ID of the ad creative
        duration: 15, // Length of the video ad in seconds
        skippable: true, // Indicating whether skip controls are made available to the end user
    }
});
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`let ad = MediaAdUpdate()
    .name("Podcast Ad") // Friendly name of the ad
    .adId("1234") // Unique identifier for the ad
    .creativeId("4321") // The ID of the ad creative
    .duration("15") // Length of the video ad in seconds
    .skippable(true) // Indicating whether skip controls are made available to the end user
mediaTracking.track(.adStart, ad: ad)
`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackMediaAdBreakStart', {
    id,
    adBreak: {
        name: 'pre-roll', // Ad break name
        breakId: '2345', // An identifier for the ad break
        breakType: 'linear', // Type of ads within the break
    }
});`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`import { trackMediaAdBreakStart, MediaPlayerAdBreakType } from "@snowplow/browser-plugin-media";
trackMediaAdBreakStart({
    id,
    adBreak: {
        name: 'pre-roll', // Ad break name
        breakId: '2345', // An identifier for the ad break
        breakType: MediaPlayerAdBreakType.Linear, // Type of ads within the break
    }
});
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`let adBreak = MediaAdBreakUpdate()
    .name("pre-roll") // Ad break name
    .breakId("2345") // An identifier for the ad break
    .breakType(.linear) // Type of ads within the break
mediaTracking.track(.adBreakStart, adBreak: adBreak)
`}
</CodeBlock>)}</>

#### Add context entities

You can add custom context entities to tracked events.
This will only apply to the currently tracked event.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackMediaPlay', {
    id,
    context: [
        {
            schema: 'iglu:org.schema/video/jsonschema/1-0-0',
            data: { creativeId: '1234' }
        }
    ]
});`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`import { trackMediaPlay } from "@snowplow/browser-plugin-media";
trackMediaPlay({
    id,
    context: [
        {
            schema: 'iglu:org.schema/video/jsonschema/1-0-0',
            data: { creativeId: '1234' }
        }
    ]
});
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(event: MediaEvent(.play, entities: [
    SelfDescribingJson(
        schema: "iglu:org.schema/video/jsonschema/1-0-0",
        andData: [
            "creativeId": "1234"
        ]
    )
]))
`}

</CodeBlock>)}</>

### Available event types

#### Events for controlling the playback

##### Ready

Tracks a media player ready event that is fired when the media tracking is successfully attached to the player and can track events.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackMediaReady', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`trackMediaReady({ id });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.ready)`}
</CodeBlock>)}</>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_event_ready/jsonschema/1-0-0`.

##### Play

Tracks a media player play event sent when the player changes state to playing from previously being paused.

Tracking this event will automatically set the `paused` property in the media player entity to `false`.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackMediaPlay', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`trackMediaPlay({ id });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.play)`}
</CodeBlock>)}</>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_event_play/jsonschema/1-0-0`.

##### Pause

Tracks a media player pause event sent when the user pauses the playback.

Tracking this event will automatically set the `paused` property in the media player entity to `true`.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackMediaPause', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`trackMediaPause({ id });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.pause)`}
</CodeBlock>)}</>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_event_pause/jsonschema/1-0-0`.

##### End

Tracks a media player end event sent when playback stops when end of the media is reached or because no further data is available.

Tracking this event will automatically set the `ended` and `paused` properties in the media player entity to `true`.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackMediaEnd', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`trackMediaEnd({ id });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.end)`}
</CodeBlock>)}</>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_event_end/jsonschema/1-0-0`.

##### SeekStart

Tracks a media player seek start event sent when a seek operation begins.

If multiple seek start events are tracked after each other (without a seek end event), the tracker tracks only the first one.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackSeekStart', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`trackSeekStart({ id });
`}

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.seekStart)`}
</CodeBlock>)}</>

</CodeBlock>)}</>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_event_seek_start/jsonschema/1-0-0`.

##### SeekEnd

Tracks a media player seek end event sent when a seek operation completes.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackSeekEnd', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`trackSeekEnd({ id });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.seekEnd)`}
</CodeBlock>)}</>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_event_seek_end/jsonschema/1-0-0`.

#### Events for changes in playback settings

##### PlaybackRateChange

Tracks a media player playback rate change event sent when the playback rate has changed.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackPlaybackRateChange', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`trackPlaybackRateChange({ id });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.playbackRateChange)`}
</CodeBlock>)}</>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_event_playback_rate_change/jsonschema/1-0-0`.

##### VolumeChange

Tracks a media player volume change event sent when the volume has changed.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackVolumeChange', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`trackVolumeChange({ id });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.volumeChange)`}
</CodeBlock>)}</>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_event_volume_change/jsonschema/1-0-0`.

##### FullscreenChange

Tracks a media player fullscreen change event fired immediately after the browser switches into or out of full-screen mode.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackFullscreenChange', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`trackFullscreenChange({ id });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.fullscreenChange)`}
</CodeBlock>)}</>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_event_fullscreen_change/jsonschema/1-0-0`.

##### PictureInPictureChange

Tracks a media player picture-in-picture change event fired immediately after the browser switches into or out of picture-in-picture mode.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackPictureInPictureChange', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`trackPictureInPictureChange({ id });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.pictureInPictureChange)`}
</CodeBlock>)}</>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_event_picture_in_picture_change/jsonschema/1-0-0`.

#### Events for ad events

##### AdBreakStart

Tracks a media player ad break start event that signals the start of an ad break.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackAdBreakStart', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`trackAdBreakStart({ id });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.adBreakStart)`}
</CodeBlock>)}</>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_event_ad_break_start/jsonschema/1-0-0`.

##### AdBreakEnd

Tracks a media player ad break end event that signals the end of an ad break.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackAdBreakEnd', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`trackAdBreakEnd({ id });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.adBreakEnd)`}
</CodeBlock>)}</>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_event_ad_break_end/jsonschema/1-0-0`.

##### AdStart

Tracks a media player ad start event that signals the start of an ad.

Tracking this event will increase the counter of `ads` in the session entity.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackAdStart', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`trackAdStart({ id });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.adStart)`}
</CodeBlock>)}</>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_event_ad_start/jsonschema/1-0-0`.

##### AdSkip

Tracks a media player ad skip event fired when the user activated a skip control to skip the ad creative.

Tracking this event will increase the counter of `adsSkipped` in the session entity.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackAdSkip', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`trackAdSkip({ id });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.adSkip)`}
</CodeBlock>)}</>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_event_ad_skip/jsonschema/1-0-0`.

##### AdFirstQuartile

Tracks a media player ad first quartile played event fired when a quartile of ad is reached after continuous ad playback at normal speed.

The tracker will automatically set the `percentProgress` property of the ad entity to 25% when this event is tracked.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackAdFirstQuartile', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`trackAdFirstQuartile({ id });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.adFirstQuartile)`}
</CodeBlock>)}</>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_event_ad_quartile/jsonschema/1-0-0`.

##### AdMidpoint

Tracks a media player ad midpoint played event fired when a midpoint of ad is reached after continuous ad playback at normal speed.

The tracker will automatically set the `percentProgress` property of the ad entity to 50% when this event is tracked.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackAdMidpoint', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`trackAdMidpoint({ id });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.adMidpoint)`}
</CodeBlock>)}</>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_event_ad_quartile/jsonschema/1-0-0`.

##### AdThirdQuartile

Tracks media player ad third quartile played event fired when a quartile of ad is reached after continuous ad playback at normal speed.

The tracker will automatically set the `percentProgress` property of the ad entity to 75% when this event is tracked.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackAdThirdQuartile', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`trackAdThirdQuartile({ id });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.adThirdQuartile)`}
</CodeBlock>)}</>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_event_ad_quartile/jsonschema/1-0-0`.

##### AdComplete

Tracks a media player ad complete event that signals the ad creative was played to the end at normal speed.

The tracker will automatically set the `percentProgress` property of the ad entity to 100% when this event is tracked.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackAdComplete', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`trackAdComplete({ id });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.adComplete)`}
</CodeBlock>)}</>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_event_ad_quartile/jsonschema/1-0-0`.

##### AdClick

Tracks a media player ad click event fired when the user clicked on the ad.

Tracking this event will increase the counter of `adsClicked` in the session entity.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackAdClick', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`trackAdClick({ id });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.adClick)`}
</CodeBlock>)}</>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_event_ad_click/jsonschema/1-0-0`.

##### AdPause

Tracks a media player ad pause event fired when the user clicked the pause control and stopped the ad creative.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackAdPause', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`trackAdPause({ id });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.adPause)`}
</CodeBlock>)}</>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_event_ad_pause/jsonschema/1-0-0`.

##### AdResume

Tracks a media player ad resume event fired when the user resumed playing the ad creative after it had been stopped or paused.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackAdResume', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`trackAdResume({ id });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.adResume)`}
</CodeBlock>)}</>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_event_ad_resume/jsonschema/1-0-0`.

#### Events for data quality

##### BufferStart

Tracks a media player buffering start event fired when the player goes into the buffering state and begins to buffer content.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackBufferStart', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`trackBufferStart({ id });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.bufferStart)`}
</CodeBlock>)}</>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_event_buffer_start/jsonschema/1-0-0`.

##### BufferEnd

Tracks a media player buffering end event fired when the the player finishes buffering content and resumes playback.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackBufferEnd', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`trackBufferEnd({ id });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.bufferEnd)`}
</CodeBlock>)}</>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_event_buffer_end/jsonschema/1-0-0`.

##### QualityChange

Tracks a media player quality change event tracked when the video playback quality changes automatically.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackQualityChange', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`trackQualityChange({ id });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.qualityChange)`}
</CodeBlock>)}</>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_event_quality_change/jsonschema/1-0-0`.

##### UserUpdateQuality

Tracks a media player user update quality event tracked when the video playback quality changes as a result of user interaction (choosing a different quality setting).

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackUserUpdateQuality', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`trackUserUpdateQuality({ id });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.userUpdateQuality)`}
</CodeBlock>)}</>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_event_user_update_quality/jsonschema/1-0-0`.

##### Error

Tracks a media player error event tracked when the resource could not be loaded due to an error.

<>{(props.tracker == 'js-tag') && (<CodeBlock language="javascript">
{`window.snowplow('trackError', { id });`}
</CodeBlock>)}</>

<>{(props.tracker == 'js-browser') && (<CodeBlock language="javascript">
{`trackError({ id });
`}
</CodeBlock>)}</>

<>{(props.tracker == 'ios') && (<CodeBlock language="swift">
{`mediaTracking.track(.error)`}
</CodeBlock>)}</>

*Schema:*
`iglu:com.snowplowanalytics.snowplow/media_player_event_error/jsonschema/1-0-0`.
