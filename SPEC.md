# Roku ECP URL-to-Playback Specification

A complete specification for converting public streaming service URLs into Roku External Control Protocol (ECP) playback commands.

---

## 1. Problem Statement

**Input:** A URL string from a public streaming service (Netflix, Disney+, HBO Max, or Prime Video).

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
| `post_launch_key` | string | Key to press after launch: `"Play"` or `"Select"` |

This extraction result is one kind of **content descriptor**. Channels not addressed by URL (e.g. Emby) are never produced by Function 1 — the caller supplies their descriptor. Its `post_launch_key` may be absent for launch-only channels like Emby, plus optional channel-specific fields like `resume_position_ticks`.

### Function 2: `build_playback_command(descriptor) -> playback_command`

Takes a content descriptor. Returns a playback command: a `launch` action, then — if the descriptor has a `post_launch_key` — a `wait` and a `keypress`. Launch-only channels (no `post_launch_key`, e.g. Emby) return just the `launch`.

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

The command is `[launch]`, followed by `[wait 2000ms, keypress]` when the descriptor carries a `post_launch_key`, or nothing for a launch-only channel like Emby. Launch `params` are channel-specific (see the catalog); most channels use `contentId={content_id}&mediaType={media_type}`.

---

## 4. Channel Catalog

Each supported channel is defined by these properties:

| Property | Netflix | Disney+ | HBO Max | Prime Video |
|----------|---------|---------|---------|-------------|
| **Channel ID** | `12` | `291097` | `61322` | `13` |
| **Channel Name** | `Netflix` | `Disney+` | `HBO Max` | `Prime Video` |
| **URL Regex** | `netflix\.com/(?:watch\|title)/(\d+)` | `disneyplus\.com/(?:(?:play|video)/|browse/entity-)([a-f0-9-]+)` | `(?:max\.com|hbomax\.com)/(?:(?:movies|series)/[^/]+/|(?:video/watch|play)/)([^/?]+)` | `(?:amazon\.com\|primevideo\.com)/.*?/([B][A-Z0-9]{9})` |
| **Content ID Format** | Numeric digits | UUID (hex + hyphens) | Alphanumeric + hyphens | ASIN (B + 9 alphanumeric) |
| **Media Type Logic** | `/watch/` in URL = `"movie"`, `/title/` in URL = `"series"` | Always `"movie"` | Always `"movie"` | Always `"movie"` |
| **Post-Launch Key** | `Play` | `Select` | `Select` | `Select` |
| **Public Domain(s)** | `netflix.com` | `disneyplus.com` | `max.com`, `hbomax.com` | `amazon.com`, `primevideo.com` |

Emby (channel `44191`) is addressed by content descriptor rather than URL, and is launch-only (no post-launch key) — see its entry below.

### Channel Details

#### Netflix (Channel ID: 12)

- **URL regex:** `netflix\.com/(?:watch|title)/(\d+)`
- Captures a numeric content ID from either `/watch/{id}` or `/title/{id}` paths
- **Media type detection:** If the URL contains `/watch/`, the media type is `"movie"`. If it contains `/title/`, the media type is `"series"`. This is the only channel with non-trivial media type logic.
- **Post-launch key:** `Play` — Netflix shows a content detail page; pressing Play starts playback
- **Example URLs:**
  - `https://www.netflix.com/watch/81444554` → content_id=`81444554`, media_type=`movie`
  - `https://www.netflix.com/title/80179766` → content_id=`80179766`, media_type=`series`

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
            media_type = channel.determine_media_type(url)
            return {
                channel_id:      channel.channel_id,
                channel_name:    channel.channel_name,
                content_id:      content_id,
                media_type:      media_type,
                post_launch_key: channel.post_launch_key,
            }
    return null
```

**Important:** Use `search` semantics (find pattern anywhere in string), not `match` semantics (match from start of string). The regex patterns do not anchor to the start of the URL.

**Media type determination** is channel-specific:
- Netflix: Check if URL contains `/watch/` (return `"movie"`) or `/title/` (return `"series"`)
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
  Regex: netflix\.com/(?:watch|title)/(\d+)
  Match: "netflix.com/watch/81444554"
  Capture group 1: "81444554"
  URL contains "/watch/" → media_type = "movie"

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
  URL contains "/title/" → media_type = "series"

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

### Example 6: No Match

```
Input:  "https://netflix.com/browse"

Step 1 - URL Extraction:
  Netflix regex: netflix\.com/(?:watch|title)/(\d+) — no match (no /watch/ or /title/ path)
  Disney+ regex: no match (wrong domain)
  HBO Max regex: no match (wrong domain)
  Prime Video regex: no match (wrong domain)

  Result: null
```

### Example 7: Emby (descriptor, launch-only)

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

2. **Which channels do you want?** — Determines which channels to include. Options: Netflix, Disney+, HBO Max (Max), Prime Video (all addressed by URL), and/or Emby (a self-hosted server, addressed by descriptor). Only include channels the user actually has.

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
- Returns an extraction result dict with fields: `channel_id`, `channel_name`, `content_id`, `media_type`, `post_launch_key`
- Returns `null`/`None`/`nil` if the URL does not match any supported URL channel
- Must use `search` (not `match`) regex semantics — the pattern can appear anywhere in the URL

### `build_playback_command(descriptor: dict) -> dict`

- Accepts a content descriptor (the output of `convert_url_to_ecp_command`, or a caller-supplied descriptor for a non-URL channel)
- Returns a playback command dict with fields: `type` (always `"action_sequence"`), `actions` (a `launch`, then a `wait` + `keypress` when the descriptor has a `post_launch_key`, else just the `launch`)
- Launch `params` follow the channel's catalog rule

---

## 10. Test Fixtures

See `test_fixtures.json` for a complete set of test cases:
- **12 valid URLs** covering all channels and edge cases (with/without www, query params, legacy domains)
- **11 invalid URLs** that should return null (browse pages, root pages, search pages, unsupported services)
- **6 playback command cases**: URL channels (wait + keypress) and Emby (launch-only, with and without a resume position)
