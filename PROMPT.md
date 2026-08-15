# Roku ECP URL-to-Playback: Implementation Prompt

You are implementing a feature that converts public streaming service URLs into Roku ECP (External Control Protocol) playback commands. The complete specification is in SPEC.md in this directory.

## Before You Start

Ask the user these questions:

1. **What programming language should I use?**

2. **What is the IP address of your Roku device?** (Found in Roku Settings > Network > About. Example: `192.168.1.100`)

3. **Which channels do you want to support?** (Netflix, Disney+, HBO Max / Max, Prime Video, Hulu, Apple TV+, YouTube — addressed by URL — and/or Emby, a self-hosted server addressed by descriptor; only include channels the user has)

## What to Build

Read SPEC.md in this directory, then implement exactly two functions:

- `convert_url_to_ecp_command(url)` — Takes a streaming URL string, returns a structured extraction result (channel ID, content ID, media type, post-launch key) or null if the URL doesn't match any supported URL channel.

- `build_playback_command(descriptor)` — Takes a content descriptor, returns an action sequence: a launch action, then a wait + keypress when the descriptor has a post-launch key, or just the launch for a launch-only channel like Emby.

The spec contains the complete channel catalog (regex patterns, channel IDs, media type logic), the algorithm, and worked examples for every channel.

## Validation

Test fixtures are in test_fixtures.json in this directory. Run your implementation against all test cases:
- 28 valid URLs that must return correct extraction results
- 16 invalid URLs that must return null
- 9 playback command cases that must produce the correct action sequence (incl. the launch-only channels: YouTube, Apple TV+, and Emby)

## Key Things to Get Right

- Use regex **search** (find anywhere in string), not match-from-start
- Two channels have non-trivial media type logic, decided from the regex **matched text**, not the full URL (a query string elsewhere in the URL may mention another segment). Netflix: `/watch/` = movie, `/title/` = series (URLs may carry an optional locale prefix like `/us/`, `/en-gb/`). HBO Max: episode pages = "episode" (a second regex, tried first, captures the **last** UUID — the playable video id), `/shows/` or `/series/` = "series", else "movie". All other URL channels always return "movie".
- An HBO Max "series" capture is a show-entity uuid the Roku app cannot play — search-sourcing consumers must resolve it to an episode uuid (SPEC.md §11 *Max Launch Resolution*) before launching, with the mediaType chosen from user intent: `series` resumes the show wherever the account left off, `episode` plays a specifically requested episode.
- Launch params are per-channel: `contentId={id}&mediaType={type}` for URL channels, `Command=PlayNow&ItemIds={id}` for Emby — no URL encoding needed
- Post-launch is driven by the extraction's `post_launch_key`: when present, the command is launch → wait 2000ms → keypress that key; a launch-only channel (no key) is just the launch. Netflix uses `Play`; YouTube and Apple TV+ are **launch-only** — their extraction results omit `post_launch_key`, so no wait and no keypress; the other URL channels use `Select` (Emby, descriptor-addressed, is also launch-only).
- Apple TV+ additionally carries `deep_link: false`: its Roku app ignores deep links entirely (device-verified), so launching only opens the app — surface that to the user.
- If your consumer discovers URLs via a web search API (rather than being handed known-good URLs), also read SPEC.md §11: query shaping (`watch` prefix), the Prime Video ASIN verification probe, the intent-driven Max launch resolution, and the rule that a rejected candidate must not satisfy or block its channel.
