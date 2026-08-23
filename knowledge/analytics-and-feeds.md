# Analytics: counting playbacks and engagement

The Streamlike player reports everything by itself. A custom or native player reports nothing
unless you make it — which is the most common cause of "the console says nobody watches our
videos".

## Counting a playback from your own player

One `GET` per playback, when playback actually starts:

```
https://cdn.streamlike.com/o.k?m=MEDIA_ID&t=TIMESTAMP&s=hls&p=myapp
```

| Parameter | Required | Meaning |
| --- | --- | --- |
| `m` | yes | `media_id` |
| `t` | recommended | Cache buster: a timestamp or random number |
| `s` | recommended | Stream type played: `mp4` or `hls` |
| `p` | no | Name of your player, to tell sources apart in the console |

Fire it on the first `playing` event, once per playback, and unbind afterwards — every call counts
as a view.

## Reporting engagement from your own player

Engagement is the ratio between the average duration played per visit and the total length of the
media. It can exceed 1 when a viewer replays parts. To feed it, report **the segments actually
watched** to `eng.k`:

```
https://cdn.streamlike.com/eng.k?m=MEDIA_ID&d=137&t=hls&q=720&p=myapp
  &u=USER&s=SESSION&f=FINGERPRINT&ts=1529597356&rs=0&re=57
```

| Parameter | Required | Meaning |
| --- | --- | --- |
| `m` | yes | `media_id` |
| `d` | yes | Total media duration, seconds |
| `t` | yes | Stream type: `hls`, `mp4`, `mp3` |
| `q` | yes | Video height being played, pixels |
| `p` | yes | Player name |
| `ts` | yes | Client timestamp at the end of the segment, POSIX seconds |
| `rs` | yes | Segment start, seconds, between 0 and `re` |
| `re` | yes | Segment end, seconds, between `rs` and `d` |
| `s` | no | Session id, unique per player display |
| `u` | no | User id — the same value you pass as `user_token` elsewhere |
| `f` | no | Client fingerprint |

Call it at short, regular intervals so an abrupt close loses almost nothing, and again on every
seek, since a jump ends the current segment and starts another. Overlapping segments are expected
and are what makes replays visible.

## Identifying a viewer: `user_token`

Pass the same `user_token` for a given viewer, everywhere:

- to the player: `&user_token=…` (up to 64 characters, your own value),
- to `eng.k` as `u`,
- to `/ws/resume?media_id=…&user_token=…` to read back the last position seen.

That is what turns anonymous counts into per-viewer figures — resume, per-viewer engagement
(`GET /analytics/userstats/{media_id}/{user_token}/{from}/{to}`), and token statistics
(`GET /analytics/tokenstats/{from}/{to}`).

Choose the value carefully: it is a pseudonymous identifier of a person. Use an internal account id
or a random per-account value, never an email address or anything readable, and document it in your
privacy notice.

## Ratings

`/ws/vote?company_id=…&media_id=…&value=0..5` stores a rating. `/ws/media` returns
`statistics.rating_hits` and `statistics.rating_totalvalue`; the average is their quotient.

The service requires a whitelisted IP that cannot be waived, and the platform explicitly leaves
vote deduplication and abuse prevention to you: keep a "this user already voted" record in your own
storage, and rate-limit before calling.

For a like/dislike feed, the rating is a reasonable transport (`5` for a like), but the durable
state — who liked what — belongs in your database. The platform stores an aggregate, not a per-user
history.

## Reading figures back, server side

The API exposes the same numbers as the console, all on a `{from}/{to}` date range:

| Endpoint | Figures |
| --- | --- |
| `GET /analytics/playback/{from}/{to}` | Playbacks per type |
| `GET /analytics/playback/top/popular/{from}/{to}` | Most watched medias |
| `GET /analytics/playback/location/{countries\|cities\|continents}/{from}/{to}` | Geography |
| `GET /analytics/playback/client/{devices\|browsers\|os}/{from}/{to}` | Devices |
| `GET /analytics/playback/referers/{from}/{to}` | Where playbacks come from |
| `GET /analytics/engagement/{media_id}/connections/{from}/{to}` | Engagement timeline of one media |
| `GET /analytics/engagement/{media_id}/qualities/{from}/{to}` | Engagement per video quality |
| `GET /analytics/viewership/{from}/{to}` | Viewership |
| `GET /analytics/medias/{from}/{to}` | Per-media table |
| `GET /analytics/userstats/{media_id}/{user_token}/{from}/{to}` | One viewer on one media |
| `GET /analytics/transfer`, `/storage`, `/encoding`, `/greenhousegas` `/{from}/{to}` | Consumption, including CO₂e |

`/ws/media` also carries a cheap playback counter (`statistics.media_access`) if all you need is
"how many views" next to a thumbnail.

## Third-party tracking

Tracking accounts (Google Analytics and others) are configured in the back office and attached to
medias, so playback events reach your own analytics without any code in your integration. The API
manages them under `/trackings` and
`POST /medias/{media_ids}/actions/integration/tracking`.

## Where the numbers will not match

- a media embedded with a non-Streamlike player and no `o.k` call is invisible in the console,
  while its traffic still shows in transfer consumption — that gap is also how bandwidth theft is
  spotted,
- viewers on mobile networks change IP, so IP-based restrictions and geography can disagree,
- playbacks are counted on playback start; engagement on watched segments. Comparing the two counts
  as if they measured the same thing leads nowhere.

---

# Feeds, sharing and packaging

Ready-made outputs that save writing a generator.

## WebTV profiles: where the links point

Feeds contain links, and the platform has to know what a link to one of your medias looks like. A
**WebTV profile** holds two base URLs — one for a media, one for a playlist:

```
media URL:    https://www.example.com/media.php?permalink=
playlist URL: https://www.example.com/playlist.php?playlist_id=
```

Create it in the back office (Rendering → WebTV profiles) or through `/profiles` in the API, then
pass `profile_id=` to a feed. A profile marked as default applies with no parameter. Without a
profile, links point at the Streamlike player.

## mRSS

```
https://cdn.streamlike.com/ws/rss?playlist_id=PLAYLIST_ID&profile_id=PROFILE_ID
```

An mRSS 2.0 feed of a playlist or a company, filtered like `/ws/playlist` (`company_id`,
`playlist_id`, `lng`, `query`, `orderby`, `sortorder`, `page`, `pagesize`). Only online medias are
included. Specification: `http://www.rssboard.org/media-rss`.

## Podcast

```
https://cdn.streamlike.com/ws/podcast?playlist_id=PLAYLIST_ID&lng=fr&orderby=releasedate
```

A podcast feed built from a playlist of audio or video medias, provided the HTML5 option is enabled
on the account. Only `playlist_id`, `lng` and `orderby` apply. Per-playlist podcast metadata (name,
author, category, cover) comes from the platform and is returned by `/ws/playlists`.

## Google video sitemap

```
https://cdn.streamlike.com/ws/videositemap?playlist_id=ID1|ID2|ID3&profile_id=PROFILE_ID
```

Generates the sitemap search engines expect: for each media a `<loc>` built from the profile, plus
title, description, large thumbnail, file URL, player URL, duration, publication date and keywords.
Several playlists are joined with `|`; `company_id` covers the whole account. `no_content_loc`
leaves out the direct file URL when you do not want it exposed.

Regenerate it whenever you publish — a sitemap is only useful while it matches the site.

## QR codes

```
https://cdn.streamlike.com/ws/qr?media_id=MEDIA_ID&size=6&level=M
```

Despite the name, the service does not return the image: it answers an HTML `<img>` tag pointing at
a generated PNG on the CDN.

```html
<img src="http://cfcdn.streamlike.com/qr/2445c6da7e4a2744b3ac89b6fea0767d.png" alt=""/>
```

Parse out the `src` if you need the file itself; the PNG is stable and cacheable. `size` sets the
module size, `level` the error correction level. Handy for print and signage.

## Short URLs

`GET /tools/shorturl` in the API turns a URL into a Streamlike short link and generates its QR
code. `Streamlink` endpoints (`/streamlink/urls`, `POST /streamlink/urls/media/{media_id}`) create
and manage short URLs bound to a media, when you need one that keeps working as the media changes.

## SCORM

```
GET /medias/{media_id}/scorm
```

Returns a SCORM 1.2 package (ZIP) wrapping the media in an iframe — the shape an LMS expects for
e-learning catalogs.

## Social platforms

The platform can push a media to linked YouTube and Dailymotion accounts:
`GET /medias/{media_id}/social` lists linkable accounts,
`POST /medias/{media_id}/social/{account_id}` links one,
`POST /medias/{media_id}/social/{account_id}/push` forces the push. Accounts themselves are managed
under `/social/accounts`.
