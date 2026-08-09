# Roku ECP URL-to-Playback Specification

A complete specification for converting public streaming service URLs into Roku External Control Protocol (ECP) playback commands.

---

## 1. Problem Statement

**Input:** A URL string from a public streaming service (Netflix, Disney+, HBO Max, Prime Video, Hulu, Apple TV+, or YouTube).

**Output:** A structured playback command that, when executed against a Roku device, launches the correct streaming app and starts playing the content. Returns `null`/`None` if the URL does not match any supported channel.

**Example:**
```
Input:  "https://www.netflix.com/watch/81444554"
Output: ActionSequence [
  Launch channel 12 with params "contentId=81444554&mediaType=movie",
  Wait 2000ms,
  Press "Play"
]
```

---

## 2. Roku ECP Protocol Essentials

Roku devices expose an HTTP API on port **8060** called the External Control Protocol (ECP). Only two endpoints are needed:

### Launch a Channel

```
POST http://{roku_ip}:8060/launch/{channel_id}?{params}
```

- `{roku_ip}` — The Roku device's IP address on the local network
- `{channel_id}` — Numeric string identifying the Roku channel (app)
- `{params}` — Query string with channel-specific parameters
- The POST body is empty
- Standard deep link params: `contentId={id}&mediaType={type}`

### Send a Keypress

```
POST http://{roku_ip}:8060/keypress/{key_name}
```

- `{key_name}` — One of: `Home`, `Up`, `Down`, `Left`, `Right`, `Select`, `Back`, `Play`, `Pause`, `Rev`, `Fwd`, `InstantReplay`, `Info`, `Search`, `Backspace`
- The POST body is empty

### Timing

After launching a channel, the app needs time to load before it can receive keypresses. A **2000 millisecond** delay is required between the launch command and any subsequent keypress.

---

## 3. Output Data Model

The system exposes two functions. Here are their input/output contracts as JSON schemas:

### Function 1: `convert_url_to_ecp_command(url) -> extraction_result | null`

Takes a URL string. Returns an extraction result dict if the URL matches a supported channel, or null if not.

**Extraction Result:**
```json
{
  "channel_id": "12",
  "channel_name": "Netflix",
  "content_id": "81444554",
  "media_type": "movie",
  "post_launch_key": "Play"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `channel_id` | string | Roku channel ID (numeric string) |
| `channel_name` | string | Human-readable channel name |
| `content_id` | string | Content identifier extracted from the URL |
| `media_type` | string | One of: `"movie"`, `"series"` |
| `post_launch_key` | string (optional) | Key to press after launch: `"Play"` or `"Select"`. **Absent** for launch-only channels (e.g. YouTube), whose deep link starts playback with no keypress |

This extraction result is one kind of **content descriptor**. Channels not addressed by URL (e.g. Emby) are never produced by Function 1 — the caller supplies their descriptor. A descriptor's `post_launch_key` is likewise absent for launch-only channels like Emby, and it may carry optional channel-specific fields like `resume_position_ticks`.

### Function 2: `build_playback_command(descriptor) -> playback_command`

Takes a content descriptor. Returns a playback command: a `launch` action, then — if the descriptor has a `post_launch_key` — a `wait` and a `keypress`. Launch-only channels (no `post_launch_key`, e.g. YouTube, Emby) return just the `launch`.

**Playback Command:**
```json
{
  "type": "action_sequence",
  "actions": [
    {"type": "launch", "channel_id": "12", "params": "contentId=81444554&mediaType=movie"},
    {"type": "wait", "milliseconds": 2000},
    {"type": "keypress", "key": "Play", "count": 1}
  ]
}
```

**Action types:**

| Action | Fields | Description |
|--------|--------|-------------|
| `launch` | `channel_id`, `params` | Launch the channel with its params (always the first, sometimes the only, action) |
| `wait` | `milliseconds` | Delay before the next action |
| `keypress` | `key`, `count` | Press a remote key `count` times (`count` ≥ 1) |

The command is `[launch]`, followed by `[wait 2000ms, keypress]` when the descriptor carries a `post_launch_key`, or nothing for a launch-only channel like YouTube or Emby. Launch `params` are channel-specific (see the catalog); most channels use `contentId={content_id}&mediaType={media_type}`.

---

## 4. Channel Catalog

Each supported channel is defined by these properties:

| Property | Netflix | Disney+ | HBO Max | Prime Video | Hulu | Apple TV+ | YouTube |
|----------|---------|---------|---------|-------------|------|-----------|---------|
| **Channel ID** | `12` | `291097` | `61322` | `13` | `2285` | `551012` | `837` |
| **Channel Name** | `Netflix` | `Disney+` | `HBO Max` | `Prime Video` | `Hulu` | `Apple TV+` | `YouTube` |
| **URL Regex** | `netflix\.com/(?:\w{2}(?:-\w{2})?/)?(?:watch\|title)/(\d+)` | `disneyplus\.com/(?:(?:play|video)/|browse/entity-)([a-f0-9-]+)` | `(?:max\.com|hbomax\.com)/(?:(?:movies|series)/[^/]+/|(?:video/watch|play)/)([^/?]+)` | `(?:amazon\.com\|primevideo\.com)/.*?/([B][A-Z0-9]{9})` | `hulu\.com/(?:series\|watch\|movie)/(?:[a-z0-9-]+-)?([0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12})` | `tv\.apple\.com/(?:\w{2}/)?(?:show\|movie\|episode)/[^/]+/(umc\.cmc\.[a-z0-9]+)` | `(?:youtube\.com/watch\?(?:[^#\s]*&)?v=\|youtu\.be/)([A-Za-z0-9_-]{11})` |
| **Content ID Format** | Numeric digits | UUID (hex + hyphens) | Alphanumeric + hyphens | ASIN (B + 9 alphanumeric) | UUID (hex + hyphens) | `umc.cmc.` + alphanumeric | 11 chars: letters, digits, `-`, `_` |
| **Media Type Logic** | `/watch/` in matched text = `"movie"`, `/title/` in matched text = `"series"` | Always `"movie"` | Always `"movie"` | Always `"movie"` | Always `"movie"` | Always `"movie"` | Always `"movie"` |
| **Post-Launch Key** | `Play` | `Select` | `Select` | `Select` | `Select` | `Select` | none (launch-only) |
| **Public Domain(s)** | `netflix.com` | `disneyplus.com` | `max.com`, `hbomax.com` | `amazon.com`, `primevideo.com` | `hulu.com` | `tv.apple.com` | `youtube.com`, `youtu.be` |

Emby (channel `44191`) is addressed by content descriptor rather than URL, and is launch-only (no post-launch key) — see its entry below.

### Channel Details

#### Netflix (Channel ID: 12)

- **URL regex:** `netflix\.com/(?:\w{2}(?:-\w{2})?/)?(?:watch|title)/(\d+)`
- Captures a numeric content ID from either `/watch/{id}` or `/title/{id}` paths
- An optional locale segment may precede the path type: two letters (`/us/`) or two-letter pairs (`/en-gb/`). Locale-prefixed URLs are common in search engine results.
- **Media type detection:** Examine the **matched text** (regex capture group 0), not the full URL: if the match contains `/watch/`, the media type is `"movie"`; if it contains `/title/`, it is `"series"`. Exactly one of the two appears in any match, so this is deterministic even when the rest of the URL (e.g. a query parameter) mentions the other segment. This is the only channel with non-trivial media type logic.
- **Post-launch key:** `Play` — Netflix shows a content detail page; pressing Play starts playback
- **Example URLs:**
  - `https://www.netflix.com/watch/81444554` → content_id=`81444554`, media_type=`movie`
  - `https://www.netflix.com/title/80179766` → content_id=`80179766`, media_type=`series`
  - `https://www.netflix.com/us/title/80179766` → content_id=`80179766`, media_type=`series`
  - `https://netflix.com/en-gb/watch/81444554` → content_id=`81444554`, media_type=`movie`
  - `https://www.netflix.com/watch/81444554?next=/title/80179766` → content_id=`81444554`, media_type=`movie` (the `/title/` in the query string is outside the matched text)

#### Disney+ (Channel ID: 291097)

- **URL regex:** `disneyplus\.com/(?:(?:play|video)/|browse/entity-)([a-f0-9-]+)`
- Captures UUID-format content IDs from `/play/{id}`, `/video/{id}`, or `/browse/entity-{id}` paths
- Content IDs are lowercase hex with hyphens (e.g., `f63db666-b097-4c61-99c1-b778de2d4ae1`)
- **Media type:** Always `"movie"`
- **Post-launch key:** `Select` — Disney+ shows a profile selection screen; pressing Select chooses the default profile, then content auto-plays
- **Example URL:**
  - `https://www.disneyplus.com/play/f63db666-b097-4c61-99c1-b778de2d4ae1` → content_id=`f63db666-b097-4c61-99c1-b778de2d4ae1`, media_type=`movie`

#### HBO Max (Channel ID: 61322)

- **URL regex:** `(?:max\.com|hbomax\.com)/(?:(?:movies|series)/[^/]+/|(?:video/watch|play)/)([^/?]+)`
- Matches both `max.com` (current domain) and `hbomax.com` (legacy domain)
- Captures content ID from `/movies/{slug}/{id}`, `/series/{slug}/{id}`, `/video/watch/{id}`, or `/play/{id}` paths
- Content ID stops at the first `/` or `?` character (captured by `[^/?]+`)
- **Important:** For URLs like `/video/watch/{id1}/{id2}`, only the first ID is captured. Do NOT use `/movie/{id}` URLs — those IDs don't work for deep linking.
- **Media type:** Always `"movie"`
- **Post-launch key:** `Select` — profile selection, then auto-play
- **Example URLs:**
  - `https://www.max.com/video/watch/bd43b2a4-1639-4197-96d4-2ec14eb45e9e` → content_id=`bd43b2a4-1639-4197-96d4-2ec14eb45e9e`, media_type=`movie`
  - `https://www.hbomax.com/video/watch/legacy-id` → content_id=`legacy-id`, media_type=`movie`

#### Prime Video (Channel ID: 13)

- **URL regex:** `(?:amazon\.com|primevideo\.com)/.*?/([B][A-Z0-9]{9})`
- Matches both `amazon.com` and `primevideo.com` domains
- Captures Amazon ASIN identifiers: exactly 10 characters, starting with `B`, followed by 9 uppercase alphanumeric characters
- The `/.*?/` allows any path structure before the ASIN (e.g., `/gp/video/detail/`, `/dp/`, `/detail/`)
- **Media type:** Always `"movie"`
- **Post-launch key:** `Select` — profile selection, then auto-play
- **Example URLs:**
  - `https://www.amazon.com/gp/video/detail/B0DKTFF815` → content_id=`B0DKTFF815`, media_type=`movie`
  - `https://amazon.com/dp/B0FQM41JFJ/ref=xyz` → content_id=`B0FQM41JFJ`, media_type=`movie`
  - `https://www.primevideo.com/detail/B0EXAMPL12` → content_id=`B0EXAMPL12`, media_type=`movie`

#### Hulu (Channel ID: 2285)

- **URL regex:** `hulu\.com/(?:series|watch|movie)/(?:[a-z0-9-]+-)?([0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12})`
- Matches `/series/`, `/watch/`, and `/movie/` paths; an optional human-readable slug prefix (e.g. `the-bear-`) is skipped, capturing only the trailing UUID
- Content IDs are lowercase-hex UUIDs (e.g., `565d8976-9e52-4f30-a6f5-a47e7fe1abd4`)
- **Media type:** Always `"movie"`
- **Post-launch key:** `Select` — press Select once the app loads to begin playback
- **Example URLs:**
  - `https://www.hulu.com/series/the-bear-565d8976-9e52-4f30-a6f5-a47e7fe1abd4` → content_id=`565d8976-9e52-4f30-a6f5-a47e7fe1abd4`, media_type=`movie`
  - `https://www.hulu.com/watch/565d8976-9e52-4f30-a6f5-a47e7fe1abd4` → content_id=`565d8976-9e52-4f30-a6f5-a47e7fe1abd4`, media_type=`movie`

#### Apple TV+ (Channel ID: 551012)

- **URL regex:** `tv\.apple\.com/(?:\w{2}/)?(?:show|movie|episode)/[^/]+/(umc\.cmc\.[a-z0-9]+)`
- Matches `/show/`, `/movie/`, and `/episode/` paths, with an optional two-letter region segment (e.g. `/us/`) before the type
- Content IDs are Apple `umc.cmc.` identifiers followed by lowercase alphanumerics (e.g., `umc.cmc.1srk2goyh2q2zdxcx605w8vtx`)
- **Media type:** Always `"movie"`
- **Post-launch key:** `Select` — press Select once the app loads to begin playback
- **Example URLs:**
  - `https://tv.apple.com/us/show/severance/umc.cmc.1srk2goyh2q2zdxcx605w8vtx` → content_id=`umc.cmc.1srk2goyh2q2zdxcx605w8vtx`, media_type=`movie`
  - `https://tv.apple.com/movie/killers-of-the-flower-moon/umc.cmc.5x1fg9gl9mwn7qzd3s6ztph5p` → content_id=`umc.cmc.5x1fg9gl9mwn7qzd3s6ztph5p`, media_type=`movie`

#### YouTube (Channel ID: 837)

- **URL regex:** `(?:youtube\.com/watch\?(?:[^#\s]*&)?v=|youtu\.be/)([A-Za-z0-9_-]{11})`
- Captures the 11-character video ID from `youtube.com/watch?v={id}` or `youtu.be/{id}`. No shorts, embed, or `m.youtube.com` variants.
- This is the only channel that captures from a **query parameter** rather than the path. The `(?:[^#\s]*&)?` allows `v` to appear after other params (`watch?app=desktop&v=…`) while refusing to match inside another param name — the segment before `v=` must end in `&`, so `sv=` cannot match.
- The strict `{11}` ID class (letters, digits, `-`, `_`) is what rejects malformed IDs like `abc123`; `youtu.be` tracking params (`?si=…`) fall outside the capture naturally.
- **Media type:** Always `"movie"` — YouTube's app ignores `mediaType`, but keeping the standard `contentId={id}&mediaType=movie` params preserves the rule that all URL channels use standard params (only Emby has channel-specific params).
- **Post-launch key:** none — the deep link auto-plays (confirmed on-device). The extraction result **omits** `post_launch_key`, and the playback command is a single `launch` with no wait/keypress. First launch-only URL channel; previously only descriptor-addressed Emby exercised this path.
- **Example URLs:**
  - `https://www.youtube.com/watch?v=dQw4w9WgXcQ` → content_id=`dQw4w9WgXcQ`, media_type=`movie`
  - `https://youtu.be/dQw4w9WgXcQ` → content_id=`dQw4w9WgXcQ`, media_type=`movie`
  - `https://www.youtube.com/watch?app=desktop&v=jNQXAC9IVRw` → content_id=`jNQXAC9IVRw`, media_type=`movie`

#### Emby (Channel ID: 44191)

- **Not addressed by public URL** (no `url_pattern`). Emby is a self-hosted media server; content is discovered through the caller's Emby library search (out of scope), which yields an item ID used as `content_id`.
- **Launch params:** `Command=PlayNow&ItemIds={content_id}` (append `&StartPositionTicks={resume_position_ticks}` if the descriptor provides it). `PlayNow` begins playback directly.
- **Post-launch key:** none — the command is a single `launch` (no wait/keypress). ~2–3s to playback vs ~12s for a navigate-and-press sequence.

---

## 5. Algorithm

### Step 1: URL Extraction

```
function convert_url_to_ecp_command(url):
    for each channel in CHANNEL_CATALOG where channel has a url_pattern:
        match = regex_search(channel.url_pattern, url)
        if match:
            content_id = match.capture_group(1)
            media_type = channel.determine_media_type(match.matched_text)
            result = {
                channel_id:   channel.channel_id,
                channel_name: channel.channel_name,
                content_id:   content_id,
                media_type:   media_type,
            }
            if channel.post_launch_key is present:
                result.post_launch_key = channel.post_launch_key
            return result
    return null
```

**Important:** Use `search` semantics (find pattern anywhere in string), not `match` semantics (match from start of string). The regex patterns do not anchor to the start of the URL.

**Media type determination** is channel-specific:
- Netflix: Check the **matched text** (the substring the regex matched, i.e. capture group 0 — not the full URL) for `/watch/` (return `"movie"`) or `/title/` (return `"series"`). Exactly one appears in any match; checking the full URL instead is wrong, because the other segment can appear elsewhere (e.g. in a query parameter).
- All other channels: Always return `"movie"`

### Step 2: Build Playback Command

```
function build_playback_command(descriptor):
    channel = CHANNEL_CATALOG[descriptor.channel_id]
    actions = [ { type: "launch",
                  channel_id: descriptor.channel_id,
                  params: channel.build_launch_params(descriptor) } ]
    if descriptor.post_launch_key is present:
        actions += [ { type: "wait", milliseconds: 2000 },
                     { type: "keypress", key: descriptor.post_launch_key, count: 1 } ]
    return { type: "action_sequence", actions: actions }
```

`build_launch_params` follows the channel's catalog rule: most channels use `contentId={content_id}&mediaType={media_type}`; Emby uses `Command=PlayNow&ItemIds={content_id}` (+ optional `&StartPositionTicks={resume_position_ticks}`). No URL encoding is needed. The `wait` + `keypress` are appended only when the descriptor carries a `post_launch_key`.

---

## 6. Concrete Examples

### Example 1: Netflix Movie

```
Input:  "https://www.netflix.com/watch/81444554"

Step 1 - URL Extraction:
  Regex: netflix\.com/(?:\w{2}(?:-\w{2})?/)?(?:watch|title)/(\d+)
  Match: "netflix.com/watch/81444554"
  Capture group 1: "81444554"
  Matched text contains "/watch/" → media_type = "movie"

  Result: {
    channel_id: "12",
    channel_name: "Netflix",
    content_id: "81444554",
    media_type: "movie",
    post_launch_key: "Play"
  }

Step 2 - Playback Command:
  {
    type: "action_sequence",
    actions: [
      {type: "launch", channel_id: "12", params: "contentId=81444554&mediaType=movie"},
      {type: "wait", milliseconds: 2000},
      {type: "keypress", key: "Play", count: 1}
    ]
  }

ECP HTTP calls (given roku_ip = "192.168.1.100"):
  1. POST http://192.168.1.100:8060/launch/12?contentId=81444554&mediaType=movie
  2. sleep(2000ms)
  3. POST http://192.168.1.100:8060/keypress/Play
```

### Example 2: Netflix Series

```
Input:  "https://www.netflix.com/title/80179766"

Step 1 - URL Extraction:
  Regex match: "netflix.com/title/80179766"
  Capture group 1: "80179766"
  Matched text contains "/title/" → media_type = "series"

  Result: {
    channel_id: "12",
    channel_name: "Netflix",
    content_id: "80179766",
    media_type: "series",
    post_launch_key: "Play"
  }

Step 2 - Playback Command:
  {
    type: "action_sequence",
    actions: [
      {type: "launch", channel_id: "12", params: "contentId=80179766&mediaType=series"},
      {type: "wait", milliseconds: 2000},
      {type: "keypress", key: "Play", count: 1}
    ]
  }
```

### Example 3: Disney+

```
Input:  "https://www.disneyplus.com/play/f63db666-b097-4c61-99c1-b778de2d4ae1"

Step 1 - URL Extraction:
  Regex match: "disneyplus.com/play/f63db666-b097-4c61-99c1-b778de2d4ae1"
  Capture group 1: "f63db666-b097-4c61-99c1-b778de2d4ae1"
  media_type = "movie" (always for Disney+)

  Result: {
    channel_id: "291097",
    channel_name: "Disney+",
    content_id: "f63db666-b097-4c61-99c1-b778de2d4ae1",
    media_type: "movie",
    post_launch_key: "Select"
  }

Step 2 - Playback Command:
  {
    type: "action_sequence",
    actions: [
      {type: "launch", channel_id: "291097", params: "contentId=f63db666-b097-4c61-99c1-b778de2d4ae1&mediaType=movie"},
      {type: "wait", milliseconds: 2000},
      {type: "keypress", key: "Select", count: 1}
    ]
  }
```

### Example 4: HBO Max

```
Input:  "https://www.max.com/video/watch/bd43b2a4-1639-4197-96d4-2ec14eb45e9e"

Step 1 - URL Extraction:
  Regex match: "max.com/video/watch/bd43b2a4-1639-4197-96d4-2ec14eb45e9e"
  Capture group 1: "bd43b2a4-1639-4197-96d4-2ec14eb45e9e"
  media_type = "movie" (always for HBO Max)

  Result: {
    channel_id: "61322",
    channel_name: "HBO Max",
    content_id: "bd43b2a4-1639-4197-96d4-2ec14eb45e9e",
    media_type: "movie",
    post_launch_key: "Select"
  }
```

### Example 5: Prime Video

```
Input:  "https://www.amazon.com/gp/video/detail/B0DKTFF815"

Step 1 - URL Extraction:
  Regex match: "amazon.com/gp/video/detail/B0DKTFF815"
  Capture group 1: "B0DKTFF815"
  media_type = "movie" (always for Prime Video)

  Result: {
    channel_id: "13",
    channel_name: "Prime Video",
    content_id: "B0DKTFF815",
    media_type: "movie",
    post_launch_key: "Select"
  }
```

### Example 6: YouTube (launch-only)

```
Input:  "https://www.youtube.com/watch?v=dQw4w9WgXcQ"

Step 1 - URL Extraction:
  Regex: (?:youtube\.com/watch\?(?:[^#\s]*&)?v=|youtu\.be/)([A-Za-z0-9_-]{11})
  Match: "youtube.com/watch?v=dQw4w9WgXcQ"
  Capture group 1: "dQw4w9WgXcQ"
  media_type = "movie" (always for YouTube)

  Result: {
    channel_id: "837",
    channel_name: "YouTube",
    content_id: "dQw4w9WgXcQ",
    media_type: "movie"
  }
  (no post_launch_key — YouTube is launch-only)

Step 2 - Playback Command:
  {
    type: "action_sequence",
    actions: [
      {type: "launch", channel_id: "837", params: "contentId=dQw4w9WgXcQ&mediaType=movie"}
    ]
  }

ECP HTTP calls (given roku_ip = "192.168.1.100"):
  1. POST http://192.168.1.100:8060/launch/837?contentId=dQw4w9WgXcQ&mediaType=movie
  (no sleep, no keypress — the deep link auto-plays)
```

### Example 7: No Match

```
Input:  "https://netflix.com/browse"

Step 1 - URL Extraction:
  Netflix regex: netflix\.com/(?:\w{2}(?:-\w{2})?/)?(?:watch|title)/(\d+) — no match (no /watch/ or /title/ path)
  Disney+ regex: no match (wrong domain)
  HBO Max regex: no match (wrong domain)
  Prime Video regex: no match (wrong domain)

  Result: null
```

### Example 8: Emby (descriptor, launch-only)

```
Input (descriptor from a library search, not a URL):
  { channel_id: "44191", channel_name: "Emby", content_id: "3f9a1c" }

Step 2 - Playback Command (catalog: Emby params + post-launch []):
  {
    type: "action_sequence",
    actions: [
      {type: "launch", channel_id: "44191", params: "Command=PlayNow&ItemIds=3f9a1c"}
    ]
  }

With a resume position, the launch params become:
  "Command=PlayNow&ItemIds=3f9a1c&StartPositionTicks=12000000000"
```

---

## 7. Questions to Ask the User

Before implementing, gather these from the user:

1. **Roku device IP address** — Required to construct ECP URLs. The Roku must be on the same local network. Users can find this in Roku Settings > Network > About.

2. **Which channels do you want?** — Determines which channels to include. Options: Netflix, Disney+, HBO Max (Max), Prime Video, Hulu, Apple TV+, YouTube (all addressed by URL), and/or Emby (a self-hosted server, addressed by descriptor). Only include channels the user actually has.

3. **Do you need the full playback command or just URL extraction?** — Some use cases only need to identify the channel and content ID from a URL, without generating the full ECP action sequence.

---

## 8. Adding a New Channel

To add support for a new streaming service, you need these 6 pieces of information:

1. **Roku Channel ID** — Install the channel on your Roku, then query: `GET http://{roku_ip}:8060/query/apps`. Find the channel in the XML response; the `id` attribute is the channel ID.

2. **URL Pattern** — Visit the streaming service's website, navigate to a piece of content, and examine the URL. Identify which part of the URL path contains the content identifier. Write a regex that captures it.

3. **Content ID Format** — What format is the content ID? Numeric, UUID, ASIN, slug, etc. This determines the regex capture group.

4. **Media Type Logic** — Does the URL distinguish between movies and series? Most channels use a single URL format for all content (always return `"movie"`). Some (like Netflix) use different URL paths.

5. **Post-Launch Key** — After launching, does the channel need a keypress to start playback (`Play` for a content detail page, `Select` for profile selection), or do the launch params start playback directly (launch-only, no key — like Emby's `Command=PlayNow`)? Test by running:
   ```
   curl -X POST "http://{roku_ip}:8060/launch/{channel_id}?contentId={id}&mediaType=movie"
   ```
   If a profile selection screen appears, use `Select`. If a content detail page appears, use `Play`.

6. **Channel Name** — Human-readable name for display purposes.

Add the new channel to the channel catalog following the same structure as existing entries.

---

## 9. Function Signature Contract

Any implementation must expose exactly these two functions:

### `convert_url_to_ecp_command(url: string) -> dict | null`

- Accepts a single URL string
- Returns an extraction result dict with fields: `channel_id`, `channel_name`, `content_id`, `media_type`, and — only for channels that have one — `post_launch_key` (launch-only channels like YouTube omit it)
- Returns `null`/`None`/`nil` if the URL does not match any supported URL channel
- Must use `search` (not `match`) regex semantics — the pattern can appear anywhere in the URL

### `build_playback_command(descriptor: dict) -> dict`

- Accepts a content descriptor (the output of `convert_url_to_ecp_command`, or a caller-supplied descriptor for a non-URL channel)
- Returns a playback command dict with fields: `type` (always `"action_sequence"`), `actions` (a `launch`, then a `wait` + `keypress` when the descriptor has a `post_launch_key`, else just the `launch`)
- Launch `params` follow the channel's catalog rule

---

## 10. Test Fixtures

See `test_fixtures.json` for a complete set of test cases:
- **24 valid URLs** covering all channels and edge cases (with/without www, query params, legacy domains, Netflix locale prefixes, media-type adversarial cases, YouTube query-param capture)
- **15 invalid URLs** that should return null (browse pages, root pages, search pages, malformed video IDs, unsupported services)
- **9 playback command cases**: URL channels (wait + keypress), YouTube (launch-only from a URL extraction), and Emby (launch-only descriptor, with and without a resume position)
