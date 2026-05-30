# Live TV Architecture (Roku Client)

This document is an internal technical map of how Live TV works in this app: entry points, views, data loading, playback, recording, and settings.

## Scope

- Covers only the Roku client code in this repository.
- Covers both Live TV screens:
1. Channels list (`LiveTVLibraryView`)
2. TV Guide grid (`Schedule`)

## Core Files

- Scene creation and event wiring:
  - `source/ShowScenes.bs`
  - `source/Main.bs`
  - `source/MainEventHandlers.bs`
- Live TV library shell:
  - `components/Libraries/LiveTVLibraryView.xml`
  - `components/Libraries/LiveTVLibraryView.bs`
- TV Guide and details:
  - `components/liveTv/schedule.xml`
  - `components/liveTv/schedule.bs`
  - `components/liveTv/ProgramDetails.xml`
  - `components/liveTv/ProgramDetails.bs`
  - `components/liveTv/ChannelInfo.xml`
  - `components/liveTv/ChannelInfo.bs`
- Live TV tasks:
  - `components/liveTv/LoadChannelsTask.bs`
  - `components/liveTv/LoadSheduleTask.bs` (file name has typo, component is `LoadScheduleTask`)
  - `components/liveTv/LoadProgramDetailsTask.bs`
  - `components/liveTv/RecordProgramTask.bs`
- Data nodes:
  - `components/data/ChannelData.bs`
  - `components/data/ScheduleProgramData.bs`
- Generic item/video playback pipeline used by Live TV:
  - `components/ItemGrid/LoadItemsTask2.bs`
  - `components/ItemGrid/LoadVideoContentTask.bs`
  - `components/manager/QueueManager.bs`
  - `components/video/VideoPlayerView.bs`
  - `source/api/Items.bs`
  - `source/utils/quickplay.bs`

## Entry Flow

1. User selects a library item with `type = "userview"` and `json.collectiontype = "livetv"` in `onSelectedItemEvent()` (`source/MainEventHandlers.bs`).
2. `CreateLiveTVLibraryView(selectedItem)` is called (`source/ShowScenes.bs`).
3. `LiveTVLibraryView` observes:
   - `selectedItem` (normal selection)
   - `quickPlayNode` (Play button shortcut)
4. Global event loop in `source/Main.bs` dispatches those events back to `onSelectedItemEvent()` or `onQuickPlayEvent()`.

## LiveTVLibraryView Responsibilities

`components/Libraries/LiveTVLibraryView.bs` is the shell that switches between:

1. Channels grid (`itemGrid`)
2. Guide (`Schedule` node inserted dynamically)

It also owns:

- voice search (`VoiceBox`)
- alpha filter (`Alpha`)
- options menu (`ItemGridOptions`) for view/sort/filter/favorite
- user preference persistence for Live TV view state

### Initial Load Behavior

`loadInitialItems()` does the following:

- Reads user settings:
  - `display.livetv.landing`
  - `display.livetv.sortField`
  - `display.livetv.sortAscending`
  - `display.livetv.filter`
- Sets `m.view`:
  - `"livetv"` for channels
  - `"tvGuide"` for guide
- Configures and runs `LoadItemsTask2` in channel mode:
  - `itemType = "TvChannel"`
  - `itemId = " "` (special case for channel query path)
  - `imageDisplayMode = "scaleToFit"` for channel logos/posters
- If landing is guide, calls `showTVGuide()`.

## Channels View Data Path

Channels view uses the generic item loader (`LoadItemsTask2`), not `LoadChannelsTask`.

### Query Path

`LoadItemsTask2.bs` with `itemType = "TvChannel"` goes through:

- URL: `Items`
- Params include:
  - `IncludeItemTypes = "TvChannel"`
  - `UserId`
  - sort/filter/search fields
  - favorites filter if selected (`Filters: "IsFavorite"` + `isFavorite: true`)

Each returned `TvChannel` item is converted to `ChannelData`.

### ChannelData Mapping

`ChannelData.setFields()` maps server JSON to UI fields:

- `id = json.id`
- `type = "TvChannel"`
- `title` based on `tvChannelTitleInfo`:
  - channel number + name
  - channel number only
  - name only
- `live = true`
- poster/logo URLs from image tags

## TV Guide Data Path

Guide view creates `Schedule` once and reuses it.

### Guide Initialization

`showTVGuide()` in `LiveTVLibraryView.bs`:

- creates `Schedule` node
- sends performance beacon `EPGLaunchInitiate`
- binds:
  - `watchChannel` -> `onChannelSelected`
  - `focusedChannel` -> `onChannelFocused`
- passes filter/search into guide
- swaps visible focus from channel grid to guide grid

### Guide Channel and Program Loading

`components/liveTv/schedule.bs`:

1. `LoadChannelsTask` loads channel rows
   - Uses `api.items.Get()` with `IncludeItemTypes: "LiveTvChannel"`
   - Handles filter favorites and search term
2. `LoadScheduleTask` loads guide programs
   - POST `LiveTv/Programs` with:
     - `channelIds`
     - `MinEndDate` / `MaxStartDate`
3. Program rows are appended under each channel in `TimeGrid`.

### Incremental Loading

- Channels are loaded in slices (`channelScheduleLimit = 50`) using `channelIds`.
- As time grid scroll approaches end window, more time is loaded in +24h windows.

### Program Details Loading

When focus changes:

1. `onProgramFocused()` updates details pane with current channel/program.
2. If program is not `fullyLoaded`, `LoadProgramDetailsTask` fetches `LiveTv/Programs/{id}`.
3. Returned node replaces the lightweight grid program node.

## Selection and Playback Triggers

There are two selection paths:

1. `OK` on an item sets `selectedItem`.
2. Play button sets `quickPlayNode`.

### Normal Selection Path (`selectedItem`)

`onSelectedItemEvent()` handles Live TV types:

- `tvchannel`, `video`, or `program`:
  - Shows loading dialog
  - If `program`, rewrites `selectedItem.id = selectedItem.json.ChannelId`
  - If resumable ticks exist, shows resume popup
  - Otherwise clears queue, pushes selected item, plays queue

### Quick Play Path (`quickPlayNode`)

`onQuickPlayEvent()` routes by type:

- `tvchannel` -> `quickplay.tvChannel()`
- `program` -> `quickplay.program()`

Both create a queue item with `type = "video"` and push it to queue, then `playQueue()` is called by event handler.

## Queue and Video Player Flow (Live TV)

1. Queue receives item (`tvchannel` direct path or `video` wrapper path).
2. `QueueManager.playQueue()` selects `CreateVideoPlayerView()` for `video`/`tvchannel`.
3. `VideoPlayerView` launches `LoadVideoContentTask`.
4. `LoadVideoContentTask` calls `ItemMetaData()` and `ItemPostPlaybackInfo()`.

### Live Detection in LoadVideoContentTask

`LoadItems_AddVideoContent()` marks stream as live when:

- type is `recording`, or
- metadata has `ChannelId` (Live TV/Program-like data)

When live:

- sets `video.content.live = true`
- sets `video.content.StreamFormat = "hls"`
- requests playback info with empty `mediaSourceId` (`meta.live` path)
- stores `transcodeParams` with `MediaSourceId` and `LiveStreamId` for playstate updates

### Live Playback Error Recovery

`VideoPlayerView.onState()` has Live TV specific retry logic:

- On playback error before play starts:
  - up to `REPLAY_RETRIES` (3)
  - sets queue force transcode to `FORCELIVETVREMUX`
  - reloads metadata/playback URL
- If still failing after retries:
  - shows `"Live TV Playback Failed"` dialog
  - exits player

### Live Playstate Reporting

`ReportPlayback()` appends for live content:

- `MediaSourceId`
- `LiveStreamId`

This keeps server-side live session state accurate.

### Live Rewind Window

`updateLiveTvTrackers()` uses `pauseBufferEnd` and `liveTvRewindLimit`:

- updates live progress start offset
- prevents seeking/pausing beyond allowed rewind window
- resumes playback if paused outside window

## Recording Flow

Recording is handled inside guide details only (`ProgramDetails` + `Schedule`).

### Record Permission Gate

- User ability loaded from policy in `LoadUserAbilities()` (`source/api/userauth.bs`):
  - `session.user.Policy.EnableLiveTvManagement` -> `livetv.canrecord`
- `ProgramDetails.setupLabels()` hides Record/Record Series buttons when `livetv.canrecord = false`.

### Record Actions

On record button press:

1. `Schedule` creates `RecordProgramTask`.
2. Task behavior:
   - If no timer exists:
     - GET `LiveTv/Timers/Defaults?programId=...`
     - POST to:
       - `LiveTv/Timers` (single)
       - or `LiveTv/SeriesTimers` (series)
   - If timer exists:
     - DELETE:
       - `LiveTv/Timers/{TimerId}`
       - or `LiveTv/SeriesTimers/{SeriesTimerId}`
3. UI updates red recording icon (`hdSmallIconUrl = "pkg:/images/red.png"`).

## Favorites in Live TV

- Favorite toggle is shared `ItemGridOptions` behavior.
- Uses `FavoriteItemsTask`:
  - `api.users.MarkFavorite()`
  - `api.users.UnmarkFavorite()`
- In guide mode, options menu targets currently focused channel (`m.channelFocused`) if available.

## User Settings that Affect Live TV

- `display.livetv.landing`
  - `"guide"` or `"channels"` (maps from web custom pref `landing-livetv`; Roku only supports these two)
- `display.livetv.sortField`
- `display.livetv.sortAscending`
- `display.livetv.filter`
- `tvChannelTitleInfo`
  - `numberName`, `number`, `name`
- `tvGuideChannelDisplay`
  - `logoTitle`, `logo`, `title`
- `ui.guide.indicator.live`
- `ui.guide.indicator.repeat`
- `livetv.canrecord` (derived from user policy)
- `playback.media.forceTranscodeLiveTV`
- `attemptDirectPlay` does not apply to live TV fallback path in same way as VOD; live has its own remux retry path.

## API Endpoints Used by Live TV Paths

- Browse/channels:
  - `GET Items` with `IncludeItemTypes=TvChannel` (channels grid)
  - `GET Items` with `IncludeItemTypes=LiveTvChannel` (guide channels task)
- Guide:
  - `POST LiveTv/Programs` (schedule window)
  - `GET LiveTv/Programs/{id}` (full program details)
- Playback:
  - `GET Items/{id}` metadata
  - `POST Items/{id}/PlaybackInfo` with `AutoOpenLiveStream=true`
- Recording:
  - `GET LiveTv/Timers/Defaults`
  - `POST LiveTv/Timers`
  - `POST LiveTv/SeriesTimers`
  - `DELETE LiveTv/Timers/{id}`
  - `DELETE LiveTv/SeriesTimers/{id}`
- Favorites:
  - `POST userfavoriteitems/{id}`
  - `DELETE userfavoriteitems/{id}`

## Important Behavior Notes

- `LiveTVScreen` enum only contains `channels` and `guide` (`source/enums/LiveTVScreen.bs`), even though web may expose more Live TV landing variants.
- `LoadSheduleTask.bs` is misspelled in filename, but component and usage are consistent (`LoadScheduleTask`).
- `quickplay.program()` validates `json.ChannelId` exists but pushes `itemNode.id` as playback id. The normal selection flow rewrites `id` to channel id first, so that path is safe.
