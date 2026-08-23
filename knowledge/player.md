# The player: embedding and controlling it

The player is a hosted page you embed in an iframe. It handles adaptive streaming, subtitles,
chapters, interactions, multiple audio tracks, accessibility and playback reporting — none of which
you have to rewrite.

## Player URLs

```
https://cdn.streamlike.com/play?med_id=MEDIA_ID
https://cdn.streamlike.com/play?permalink=PERMALINK
https://cdn.streamlike.com/play?live_id=STREAM_NAME
https://cdn.streamlike.com/play?str_id=STREAMOUT_ID
```

`med_id` carries the same value as the `media_id` returned by the webservices. Options are added as
query parameters:

```
https://cdn.streamlike.com/play?med_id=MEDIA_ID&autostart=1&muted=1&events=1&active_color=001547
```

A **player configuration** created in the back office bundles those options behind one identifier,
and `&pid=CONFIG_ID` applies it. Prefer it: settings then change without touching your code, and
one account-wide default applies when no `pid` is given.

## Responsive embedding

The iframe has no intrinsic size. Wrap it in a box whose `padding-top` matches the aspect ratio —
`100 / ratio` in percent, with `ratio` read from `metadata.global.ratio` of `/ws/media`:

```html
<div style="position:relative;overflow:hidden;padding-top:56.25%">
  <iframe src="https://cdn.streamlike.com/play?med_id=MEDIA_ID"
          style="position:absolute;top:0;left:0;width:100%;height:100%;border:0"
          allow="autoplay; fullscreen; picture-in-picture"
          allowfullscreen webkitallowfullscreen mozallowfullscreen></iframe>
</div>
```

16:9 is 56.25%, 2:1 is 50%, 21:9 is 42.85%. The `allow` attribute matters: without `fullscreen` the
fullscreen button does nothing, without `autoplay` a muted autostart is refused by the browser.

`setResponsiveIframe()` from `js-streamlike-sdk` does this from a media id, and fetches the ratio
for you — see `knowledge/player.md`.

## Controlling the player from the page

Add `events=1` to the player URL, then talk to the iframe with `postMessage`. Give the iframe an
id:

```html
<iframe id="slplayer" src="https://cdn.streamlike.com/play?med_id=MEDIA_ID&events=1"></iframe>
<script>
  const player = document.getElementById('slplayer');
  player.contentWindow.postMessage('["play"]', '*');
  player.contentWindow.postMessage('["seek",30.4]', '*');
  player.contentWindow.postMessage('["speed",1.9]', '*');
</script>
```

| Command | Effect |
| --- | --- |
| `["play"]` / `["pause"]` | Start / pause playback |
| `["mute"]` / `["unmute"]` | Sound off / on |
| `["fullscreen"]` | Toggle fullscreen |
| `["seek",30.4]` | Jump to a position, in seconds |
| `["speed",1.3]` | Playback rate — below 1 slows down, above 1 speeds up |
| `["volume",0.5]` | Volume, 0 to 1 |

The player pushes back on the same channel, roughly every 250 ms while playing:

```js
window.addEventListener('message', (evt) => {
  // ["sl-progress", 24.363924]  – current position in seconds
  // ["sl-state", "play"]        – "play" | "pause" | "ended"
  console.log(evt.data);
});
```

Three practical notes:

- filter on the origin (`evt.origin`) before trusting a message: your page will receive messages
  from other frames too,
- `sl-state` is how you detect the end of a media to chain the next one,
- the iframe is cross-origin. You cannot read its DOM, and a click inside it sends keystrokes to
  the player, not to your page — keyboard shortcuts bound on your page stop working once the
  viewer has clicked the video.

## Keyboard shortcuts inside the player

| Key | Action |
| --- | --- |
| Space, `k` | Play / pause |
| `m` | Mute |
| `f` | Fullscreen |
| ← / → | ±5 seconds |
| ↑ / ↓ | Volume |
| `>` | Playback speed |
| `c` | Subtitles |
| `p` | Storyboard (mosaic) |
| `d` | Media information |
| `n` | Chapters |
| `0`–`9` | Jump to that tenth of the duration |

## oEmbed

The player is an oEmbed provider, which is what content management systems consume:

```
https://cdn.streamlike.com/oembed?url=<url-encoded player URL>&format=json
```

```json
{"version":"1.0","type":"video","provider_name":"Streamlike",
 "title":"…","thumbnail_url":"…","width":640,"height":358,
 "html":"<style>…</style><div class=\"sl-resp\"><iframe src=\"…\"></iframe></div>"}
```

The returned `html` is already responsive. `format=xml` is available too.

## Every player parameter

Values are read from the URL, or from the player configuration behind `pid` — the two are
equivalent, and a parameter in the URL overrides the configuration. Booleans are `0` or `1`.
Colours are hexadecimal without `#`. When a parameter accepts either a language code or a boolean,
`1` means "follow the browser language".

The last two columns say whether the parameter applies to a Streamlive (live channel) and to a
Streamout (scheduled broadcast).

| Parameter | Value | Description | Default | Streamlive | Streamout |
| --- | --- | --- | --- | --- | --- |
| `active_color` | RRGGBB | Changes the color of all the player's "active" elements (progress bar, active buttons, etc.). Takes an HTML color without the #. E.g. FF01F8 | - | yes | no |
| `audio_lng` | language code | Forces the audio track language. The -ad suffix targets the language's audio description: audio_lng=en-ad | - | no | no |
| `autostart` | bool | Automatic media start | 0 | yes | no |
| `background_audio` | bool | Switches to the lowest quality available if a video is playing but not visible | 0 | yes | yes |
| `background_color` | RRGGBB | Changes the background color behind the video and the poster. Takes an HTML color without the #. E.g. FF01F8 | - | yes | yes |
| `background_opacity` | integer | Only in combination with a background_color. Overrides the opacity setting of the background color. 0: transparent, 100: opaque | 100 | yes | no |
| `buttons_color` | RRGGBB | Changes the color of the overlaid buttons. Takes an HTML color without the #. E.g. FF01F8 | FFFFFF | yes | no |
| `chapter` | language code / bool | Forces the language or disables the display of chapters. E.g. chapter=fr or chapter=0 | 1 | no | no |
| `chapters_usethumb` | bool | Shows or disables the thumbnail | 1 | no | no |
| `controls` | bool | Whether to show the player controls | 1 | yes | no |
| `cover` | url / bool | URL of the image to use for the cover, or a boolean to enable/disable it | 1 | yes | no |
| `download` | bool / integer | displays a button to download the highest-quality transcode / displays a download button and caps the maximum bitrate of the file that can be downloaded | 0 | no | no |
| `events` | bool | Enables or disables the ability to control the player from outside the iframe | 0 | yes | no |
| `fill_browser` | bool | Enables or disables the playback mode that fills the entire browser window. In this mode, parts of the video may become hidden but it is not distorted. | 0 | yes | yes |
| `forcehd` | bool | Selects only the highest-quality stream (MP4 only) | 0 | no | no |
| `fs` | bool | Enables or disables switching to fullscreen | 1 | yes | yes |
| `fullscreen` | bool | Enables or disables switching to fullscreen | 1 | yes | yes |
| `icons_position` | right left none | Places the panel icons on the right or left, or fully disables the display of panels and icons | right | yes | no |
| `infos` | bool | Shows the title/description/credits block. Setting "nometa" to 1 disables this option | 0 | yes | no |
| `inline_throttling` | integer | Requires the Theo player. Limits the quality of videos available in standard playback but not in fullscreen, by specifying a maximum bitrate (in Kbps) | - | yes | yes |
| `interaction` | language code / bool | Forces the language or disables the display of interactions. E.g. interaction=fr or interaction=0 | 1 | no | no |
| `interface` | bool | Disables all interface elements (buttons, controls, panel, etc.) | 1 | yes | no |
| `landing` | bool | Shows the clickable cover image and a logo in place of the player. The player is loaded after clicking the image. | 0 | no | no |
| `live_dvr` | bool | Enables DVR mode during a live | 1 | yes | no |
| `live_id` | string | Identifier of the live or "stream name" | - | yes | no |
| `logo` | bool | Whether to show the logo | 1 | yes | yes |
| `logo_alpha` | 0-100 | Overrides the transparency setting, only if the logo is defined from a logo_url parameter | 100 | yes | yes |
| `logo_id` | integer | applies the logo referenced by the ID | - | yes | yes |
| `logo_link` | url | Makes the logo clickable and opens the given url in a new window | - | yes | yes |
| `logo_position` | lb lt rt rb | Overrides the position setting (r: right, t: top, b: bottom, l: left), only if the logo is defined from a logo_url parameter | rt | yes | yes |
| `logo_url` | url | Overrides the URL of the logo image | - | yes | yes |
| `loop` | bool | Loops the media | 0 | no | no |
| `max_height` | integer | Sets a height limit for the transcodes made available for adaptive streaming. Combined with inline_throttling, the strictest rule prevails | - | yes | yes |
| `max_width` | integer | Sets a width limit on the transcodes made available for adaptive streaming. Combined with inline_throttling, the strictest rule prevails | - | yes | yes |
| `med_id` | string | encrypted media_id | - | no | no |
| `mosaic` | bool | Shows or disables the mosaic | 1 | no | no |
| `muted` | bool | Mutes the media | 0 | yes | yes |
| `nometa` | bool | No meta tag in the head (title, descriptions, keywords, og, etc.) | 0 | yes | yes |
| `nosharemeta` | bool | Disables the social sharing meta tags (included in nometa) | 0 | yes | yes |
| `nowplaying` | bool | Adds a clickable button to show the number of ongoing playbacks | 0 | yes | no |
| `permalink` | string | Permalink | - | no | no |
| `pid` | encrypted pid | Encrypted ID of the player settings to apply to the media | company default | yes | yes |
| `play_button` | bool | Shows or hides the central "play" button. | 1 | yes | no |
| `playback_speed` | bool | Shows the button to change the media playback speed | 0 | no | no |
| `player` | theo, hlsjs | Forces the player type | hlsjs | yes | yes |
| `preload` | bool | Preloads the media | 0 | no | no |
| `prevent_click` | bool | Blocks clicking on the player | 0 | yes | no |
| `related` | bool or view_id | Shows a grid of media that share keywords and belong to the same view | 0 | no | no |
| `report` | bool | Shows the button to report inappropriate content | 0 | yes | no |
| `share` | bool | Whether to show the sharing block | 0 | yes | no |
| `skin` | id | Plain-text ID of the skin to use | - | yes | no |
| `slider` | bool | Shows or disables the slider | 1 | no | no |
| `sltoken` | string(32) | When token-based security is enabled, must contain the access token created by the API | - | yes | yes |
| `sltoken_duration` | integer | Token validity duration, in seconds | - | yes | yes |
| `sltoken_ip_ids` | string(32) | List of the IDs of the authorized IP groups, separated by commas | - | yes | yes |
| `sltoken_referrer_ids` | string(32) | List of authorized referrer identifiers, separated by commas | - | yes | yes |
| `str_id` | encrypted streamout id | encrypted streamout id | - | no | yes |
| `streamlike_mp_starttc` | integer | Position (in seconds) where the video should start | 0 | no | no |
| `subtitle` | language code / bool | Forces the language or disables the display of subtitles. The -cc suffix targets the language's closed captions. E.g. subtitle=fr, subtitle=fr-cc or subtitle=0 | 1 | no | yes |
| `subtitle_deep_links` | language code / bool | Forces the language or disables the display of subtitle search. E.g. subtitle_deep_links=fr or subtitle_deep_links=0 | 1 | no | yes |
| `subtitles_size` | integer | Font height in em | - | no | yes |
| `swfskin` | id | Plain-text ID of the skin to use | - | no | no |
| `t` | string | Position where the video should start. Accepted formats: hh:mm:ss.000 or shorter | 0 | no | no |
| `tc` | string | Position where the video should start. Accepted formats: hh:mm:ss.000 or shorter | 0 | no | no |
| `throttling` | integer | Limit the quality of available videos by specifying a maximum bitrate (in Kbps); sets a minimum quality threshold if the value is negative | - | yes | yes |
| `timecode` | string | Position where the video should start. Accepted formats: hh:mm:ss.000 or shorter | 0 | no | no |
| `tv` | bool | Removes the controls and the play button and forces automatic playback. Interactions stay visible but are not clickable. Default and non-editable setting for a streamout | 0 | yes | no |
| `user_token` | string(64) | Identifies a specific user by permanently assigning them the same token | - | yes | yes |
| `volume` | float | Volume setting from 0.0 to 1.0 | 1.0 | yes | yes |

## Choosing parameters that matter

- **Autoplay**: browsers only allow it muted. `autostart=1&muted=1` is the working pair; the viewer
  unmutes,
- **Bandwidth**: `max_width` / `max_height` cap the renditions offered, `inline_throttling` caps
  the bitrate outside fullscreen (Theo player only), and the strictest rule wins. Useful on mobile
  data or a constrained corporate network,
- **`background_audio=1`** drops to the lowest quality when the video is playing but not visible.
  It is meant for audio-first usage, not for saving bandwidth on a visible player,
- **Chrome**: `interface=0` removes every control for a decorative background video;
  `controls=0`, `play_button=0`, `logo=0`, `icons_position=none` remove pieces one by one,
- **Accessibility**: `audio_lng=en` forces an audio track, `audio_lng=en-ad` its audio
  description, `chapter=fr` and `interaction=fr` force those languages,
- **`user_token`** ties playback events to one viewer, which is what makes resume and per-viewer
  engagement possible — see `knowledge/analytics-and-feeds.md`,
- **`tv=1`** strips the controls and forces playback, the signage mode.

## Legacy helper scripts

Older integrations load `sl5-utils.js` from `assets.streamlike.tv` and call
`new Streamlike().setResponsiveIframe(...)`. That path still works and still appears in the
platform documentation, but new work should use `js-streamlike-sdk`, which is typed, dependency
free and maintained.

---

# `js-streamlike-sdk`

The official browser SDK: TypeScript, MIT, **no runtime dependency**. Current version 3.8.0.

```
https://github.com/Streamlike/js-streamlike-sdk
https://www.npmjs.com/package/js-streamlike-sdk
```

It wraps the webservices and the player, and ships ready-made UI pieces. When the target is a web
front end — including a WebView-based mobile app — start here rather than writing fetch calls.

## Install

```bash
npm install js-streamlike-sdk
```

Or straight from a CDN, no build step:

```html
<div id="my-player"></div>
<script type="module">
  import { generatePlaylistPlayer } from 'https://cdn.jsdelivr.net/npm/js-streamlike-sdk@3.8.0/dist/index.mjs';
  await generatePlaylistPlayer('my-player', { playlistId: 'PLAYLIST_ID' });
</script>
```

A classic script tag works too (`dist/index.global.js`, everything under the `Streamlike` global).
Pin an exact version: `@latest` changes behaviour under pages you no longer control. A CDN also adds
a third party to your pages — integrations that cannot accept that should install through npm and
serve the files themselves.

## What it gives you

**Data — thin wrappers over `/ws/*`:**

| Function | Endpoint |
| --- | --- |
| `getWsMedia(params, options)`, `getMediaFromId(id, …)`, `getMediaFromPermalink(permalink, …)` | `/ws/media` |
| `getMediaMetadata(…)`, `getMediaStatistics(…)` | `/ws/media`, narrowed |
| `getWsPlaylist(…)`, `getMediasFromPlaylist(id, …)`, `getMediasFromView(id, …)`, `getMediasFromCompany(id, …)`, `getPlaylistSize(…)` | `/ws/playlist` |
| `getWsPlaylists(…)`, `getPlaylists(…)` | `/ws/playlists` |
| `getWsRelated(…)`, `getWsNowPlaying(…)`, `getWsResume(…)`, `getWsCountries(…)`, `getWsLanguages(…)` | matching services |
| `getWs(url, debug)` | any endpoint URL |

**UI:**

| Function | What it renders |
| --- | --- |
| `setResponsiveIframe(mediaId, containerId, options)` | A responsive player, ratio fetched from the media |
| `embedPlayerIframe(container, src, ratio, params, debug)` | Same, from a player URL you built |
| `generateThumbnail(target, mediaCustomization, options)` | An interactive preview — `mode: 'animation'` or `'scrubbing'`, `fitMode` cover/contain |
| `generateWords(url, options)` | A live transcript, highlighted and clickable, synchronised with the player |
| `generateTrimmer(target, options)` | A segment selector bound to your inputs |
| `generatePlaylistPlayer(target, options)` | A complete playlist player |

## `setResponsiveIframe`

```js
import { setResponsiveIframe } from 'js-streamlike-sdk';

const response = await setResponsiveIframe('MEDIA_ID', 'player-container', {
  playerParams: { events: 1, autoplay: true, active_color: '293c5a' },
  baseOptions: { debug: true }
});

if (response.res) {
  console.log(response.data.metadata.global.name);
}
```

`playerParams` accepts every player parameter from `knowledge/player.md`. Note the return
shape: metadata lives under `response.data.metadata`, and `response.res` is the success flag.

## `generatePlaylistPlayer`

The one function that turns a playlist into a working interface: player, previous/next, clickable
list, information panel, auto-advance, paging, shareable timecoded links.

```js
const controller = await generatePlaylistPlayer('playlist-player', {
  playlistId: 'PLAYLIST_ID',
  info:     { title: true, position: true, duration: true, views: false, description: false },
  listItem: { thumbnail: true, index: true, title: true, duration: true },
  listPosition: 'right',      // 'right' | 'left' | 'bottom' | 'top'
  pageSize: 10,               // medias per request
  autoNext: true,
  hideTokenized: true,        // drop medias that cannot be played
  fullscreen: false,
  autostart: false,
  loop: false,
  labels: { previous: 'Previous', next: 'Next' },
  onMediaChange: (media, index) => {},
  onPlaylistEnd: () => {}
});
```

The controller exposes `play`, `pause`, `seek`, `next`, `previous`, `playIndex`, `playMedia`,
`loadMore`, `getCurrentIndex`, `getCurrentMedia`, `getMedias`, `getTotal`, `getCurrentTime`,
`isFullscreen`, `toggleFullscreen`, `getShareUrl`, `destroy`.

Behaviour worth knowing before you fight it:

- **paging is automatic.** Medias load `pageSize` at a time; nearing the end of the loaded set
  fetches the next page, a "load more" button appears while medias remain, and the counter reads
  `20 / 330`. `getTotal()` is the real size,
- **restricted medias are handled.** Token-protected medias are hidden by default and removed from
  the counts, so positions stay consistent; medias secured by IP or referrer keep their player and
  only show a notice if playback actually fails. The player probes the URL with a `HEAD` request
  rather than guessing,
- **`fullscreen: true` fullscreens the container**, not the iframe — which is what keeps playback
  running when the media changes. It disables the player's own fullscreen button, and steps aside
  on iPhone where Safari only fullscreens a native video element,
- **`shareParams: { enabled: true }`** reads `?media=…&t=…` from the page URL and
  `controller.getShareUrl()` builds such a link for the current position,
- **styling** is class-based, every element prefixed with `classPrefix` (`sl-playlist` by default),
  single-class selectors so your CSS wins. `injectStyles: false` starts from nothing,
- `listItem.interactiveThumbnail` downloads one storyboard per entry — fine for a few dozen items,
  not for hundreds.

## Transcript and trimmer

`generateWords(wordsUrl, { wordsContainer, iframePlayer, autoScroll })` needs the words file of a
subtitle track, found at `media.metadata.subtitles[0].subtitle.url.words`, plus the player iframe —
so call it after the player exists. `generateTrimmer` binds a start/end selector to your own number
inputs, for editing tools.

## When not to use it

The SDK is browser-side, so it only reaches what the webservices expose. Anything that writes —
creating medias, editing metadata, issuing playback tokens — goes through your backend and the REST
API. There is no JavaScript SDK for the REST API, and there should not be one in a browser.

---

# Playback: streams, tracks and native players

Most integrations should embed the player (`knowledge/player.md`) and stop reading here: it
already handles adaptive streaming, tracks, subtitles, resume, reporting and browser quirks. Drive
the streams yourself only when the player cannot be embedded — a native mobile player, a set-top
box, a media pipeline.

## Three ways to reach the files

**1. The player.** `https://cdn.streamlike.com/play?med_id=…` — everything included.

**2. A direct file URL, resolved by the platform.** A redirect endpoint picks the best encoded file
for a target size:

```
https://cdn.streamlike.com/html5/{type}/media_id/{media_id}/width/1280/height/720
https://cdn.streamlike.com/html5/{type}/permalink/{permalink}
```

`{type}` is `hls`, `idevicev2`, `idevicev1`, `mp4`, `mp4low`, `webm` for video, `mp3` or `aac` for
audio. The endpoint answers `302` to the chosen file on the CDN. Without `width`/`height` it
returns the largest rendition — which for `hls` means one rendition, not the adaptive master.

**3. The file manifest.** `/ws/media` returns
`html5_sources[].html5_source.manifest`, the URL of a JSON index of every encoded file:

```json
{"mp4":       [{"globalbitrate": 1408, "width": 2090, "height": 1962, "url": "//cfcdn…/x_1280_720….mp4"}],
 "mp4low":    [ … ],
 "idevicev2": [{"globalbitrate": 0, "url": "//cfcdn…/idevicev2/index.m3u8"},
               {"globalbitrate": 320, "width": 240, "height": 176, "url": "//cfcdn…/….m3u8"}],
 "idevicev1": [ … ]}
```

Read it like this:

- `idevicev2` is the HLS group, `idevicev1` the older one, `mp4` / `mp4low` progressive files,
- **the entry whose `globalbitrate` is `0` is the adaptive master** (`index.m3u8`). That is the URL
  to hand to a native player; the others are single renditions,
- URLs are protocol-relative (`//cfcdn…`): prefix `https:`,
- `width` and `height` on a rendition are encoder-side values, not the display resolution — read
  the resolution from the HLS master if you need it exactly.

`/ws/manifest?media_id=…` returns the same information through the webservice, and needs a
whitelisted IP. The manifest URL from `/ws/media` does not, which usually makes it the easier
route.

## Inside the HLS master

```
#EXTM3U
#EXT-X-MEDIA:TYPE=SUBTITLES,GROUP-ID="subs",NAME="fr",LANGUAGE="fr",URI="/c/…/subtitles/fr/….m3u8"
#EXT-X-STREAM-INF:PROGRAM-ID=1,BANDWIDTH=1503232,RESOLUTION=1024x576,SUBTITLES="subs"
….m3u8?sltoken=
```

- subtitle tracks are declared as `EXT-X-MEDIA` with their language,
- variant URIs carry `?sltoken=` — empty for an open media, filled for a protected one. Keep the
  query string when you rewrite URLs, or playback stops at the first segment,
- some URIs are relative to the CDN root rather than to the manifest. Resolve them against the
  manifest URL, and test with a player that is strict about it.

Two pitfalls that cost real debugging time:

- **never pin a player to an audio-only variant** when you mean to show video. A media can publish
  one, and a player parked on it plays sound with no picture — and reports no video progress,
- an adaptive player left to itself starts low and climbs. If your app measures "is it playing" by
  bitrate, measure playback position instead.

## Multiple audio tracks

A media carrying several audio languages is flagged `metadata.global.is_multiple_audio` in
`/ws/media`, and `/ws/playlist?multiple_audio=1` filters the catalog on it.

- in the player, `audio_lng=en` selects a track, and the `-ad` suffix its audio description:
  `audio_lng=en-ad`,
- in a native player, the tracks are in the HLS master as `EXT-X-MEDIA:TYPE=AUDIO` entries with
  their `LANGUAGE` — select by language code, not by index; the order is not a contract,
- server side, `GET /medias/{media_id}/audio-tracks` lists the declared tracks and the API can add,
  replace, rename or promote them — see `knowledge/api.md`. `GET /medias/{media_id}` also returns
  `source.audio_tracks` for a read-only view, on a single media only,
- those endpoints need API 5.31 (5.37 for `source.audio_tracks`) and the `multiple_audio` filter
  webservices 5.20. Check `scripts/openapi_lookup.py show …` against the published description
  before relying on them: an older server answers `404`, or accepts the filter and ignores it.

## Subtitles and transcripts

`/ws/media` returns each subtitle track in four formats:

```json
"subtitles": [{"subtitle": {"language_id": "fr",
   "url": {"dfxp": "…", "vtt": "…", "srt": "…", "m3u8": "…"}}}]
```

- `vtt` for a browser `<track>`, `m3u8` for HLS, `srt` for downloads, `dfxp` for legacy tooling,
- the player displays them on its own; `c` toggles them, `subtitle=fr` forces a language,
  `subtitle=fr-cc` its closed-caption variant, `subtitle=0` disables them,
- tracks produced by speech-to-text also expose a word-level file, which is what
  `generateWords()` from the JS SDK renders as a live transcript.

## Live and Streamout

- a **live** channel plays through the same player with `live_id=STREAM_NAME`; `live_dvr=1` enables
  the DVR window,
- a **Streamout** is a scheduled broadcast — a playlist played on a timetable — embedded with
  `str_id=STREAMOUT_ID`. Several player parameters do not apply to it (the table in
  `knowledge/player.md` has a column for this),
- `/ws/nowplaying?media_id=…` gives the concurrent viewer count, for a live badge.

## Downloads

`download=1` on the player shows a download button for the highest quality; an integer caps the
bitrate of the downloadable file. Availability also depends on the media being marked downloadable
(`metadata.global.is_downloadable`, set through the API or the back office).
