# Roku ECP URL-to-Playback: Implementation Prompt

You are implementing a feature that converts public streaming service URLs into Roku ECP (External Control Protocol) playback commands. The complete specification is in SPEC.md in this directory.

## Before You Start

Ask the user these questions:

1. **What programming language should I use?**

2. **What is the IP address of your Roku device?** (Found in Roku Settings > Network > About. Example: `192.168.1.100`)

3. **Which channels do you want to support?** (Netflix, Disney+, HBO Max / Max, Prime Video — addressed by URL — and/or Emby, a self-hosted server addressed by descriptor; only include channels the user has)

## What to Build

Read SPEC.md in this directory, then implement exactly two functions:

- `convert_url_to_ecp_command(url)` — Takes a streaming URL string, returns a structured extraction result (channel ID, content ID, media type) or null if the URL doesn't match any supported URL channel.

- `build_playback_command(descriptor)` — Takes a content descriptor, returns an action sequence: a launch action followed by the channel's post-launch actions (which may be none, as for Emby).

The spec contains the complete channel catalog (regex patterns, channel IDs, media type logic), the algorithm, and worked examples for every channel.

## Validation

Test fixtures are in test_fixtures.json in this directory. Run your implementation against all test cases:
- 12 valid URLs that must return correct extraction results
- 11 invalid URLs that must return null
- 6 playback command cases that must produce the correct action sequence (incl. Emby, launch-only)

## Key Things to Get Right

- Use regex **search** (find anywhere in string), not match-from-start
- Netflix is the only channel with non-trivial media type logic: `/watch/` = movie, `/title/` = series. All other URL channels always return "movie".
- Launch params are per-channel: `contentId={id}&mediaType={type}` for the public channels, `Command=PlayNow&ItemIds={id}` for Emby — no URL encoding needed
- Post-launch actions are per-channel, not fixed: Netflix presses `Play`; Disney+/HBO Max/Prime press `Select` after a 2000ms wait; Emby has none. Read the list from the catalog — don't hard-code one wait + one press.
