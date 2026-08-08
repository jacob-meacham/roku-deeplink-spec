# Design: v1.4.0 — Add YouTube Channel (First Launch-Only URL Channel)

**Date:** 2026-08-08
**Status:** Approved

## Goal

Add YouTube as the seventh URL-addressed channel in the Roku ECP spec, and rev the
spec to v1.4.0. YouTube is the first URL channel whose deep link auto-plays, so it
is also the first URL channel with no `post_launch_key` — the playback command is a
bare `launch`, the path previously exercised only by descriptor-addressed Emby.

## Decisions

- **Channel:** YouTube, Roku channel ID `837`.
- **URL scope:** `youtube.com/watch?v={id}` and `youtu.be/{id}` only. No shorts,
  embed, or m.youtube.com variants (YAGNI — these aren't what people share).
- **Post-launch:** Auto-plays on deep link (confirmed on-device). Extraction result
  omits `post_launch_key`; command is launch-only.
- **Launch params (Approach A):** Standard `contentId={content_id}&mediaType=movie`.
  YouTube's app ignores `mediaType`; keeping it preserves the rule that all URL
  channels use standard params — only Emby has channel-specific params.
- **Version:** v1.4.0 minor bump, matching the v1.3.0 precedent for channel
  additions. Single commit on `main`.

## Catalog Entry

| Property | Value |
|----------|-------|
| Channel ID | `837` |
| Channel Name | `YouTube` |
| URL Regex | `(?:youtube\.com/watch\?(?:[^#\s]*&)?v=\|youtu\.be/)([A-Za-z0-9_-]{11})` |
| Content ID Format | 11 chars: letters, digits, `-`, `_` |
| Media Type Logic | Always `"movie"` |
| Post-Launch Key | none (launch-only) |
| Public Domains | `youtube.com`, `youtu.be` |

Regex notes:

- Captures from a **query parameter** (`v=`) — a first for this spec; every prior
  channel captures from the path.
- `(?:[^#\s]*&)?` allows `v` to appear after other params
  (`watch?app=desktop&v=…`) while refusing to match inside another param name
  (the segment before `v=` must end in `&`, so `sv=` cannot match).
- The strict `{11}` ID class is what makes malformed IDs (e.g. `abc123`) return
  null; `youtu.be` tracking params (`?si=…`) fall outside the capture naturally.

## Data-Model Touch

`post_launch_key` becomes officially **optional on extraction results**. Today §3
allows absence only on caller-supplied descriptors (Emby). `build_playback_command`
already branches on its presence, so the algorithm is unchanged — only wording:

- §3: extraction-result field table + prose.
- §5: extraction pseudocode sets `post_launch_key` only when the channel has one.
- §9: function contract for `convert_url_to_ecp_command`.

## Fixtures (`test_fixtures.json`)

New totals: **24 valid / 15 invalid / 9 playback = 48 cases.**

Add 4 valid:

1. `https://www.youtube.com/watch?v=dQw4w9WgXcQ`
2. `https://youtu.be/dQw4w9WgXcQ`
3. `https://www.youtube.com/watch?app=desktop&v=jNQXAC9IVRw` (v not first)
4. `https://youtu.be/jNQXAC9IVRw?si=AbCdEfGh123` (share link with tracking)

Add 3 invalid:

1. `https://www.youtube.com/` (root page)
2. `https://www.youtube.com/watch?list=PLabc123` (watch URL with no `v` param)
3. `https://www.youtube.com/@somecreator` (channel page)

Correct 1 existing invalid fixture: `https://www.youtube.com/watch?v=abc123` is
currently described as "Unsupported streaming service". YouTube is now supported;
the URL stays null because `abc123` is not a valid 11-char video ID. Reword the
description accordingly.

Add 1 playback command: YouTube extraction result (no `post_launch_key`) →
`{launch 837, params "contentId=dQw4w9WgXcQ&mediaType=movie"}` and nothing else —
first launch-only case fed from a URL extraction.

## Docs Ripple

- **SPEC.md:** §1 channel list; §4 catalog table column + YouTube details
  subsection; §6 worked example with launch-only ECP call trace; §7 channel
  question; §10 fixture counts.
- **PROMPT.md:** channel list in question 3; fixture counts; "Key Things to Get
  Right" bullet: YouTube is launch-only — no wait/keypress.
- **README.md:** total case count (48).

## Testing

The fixtures are the test suite. Sanity-check the new regex against every new
valid/invalid fixture (plus the corrected one) with a quick throwaway script
before committing.
