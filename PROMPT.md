# Roku ECP URL-to-Playback: Implementation Prompt

You are implementing a feature that converts public streaming service URLs into Roku ECP (External Control Protocol) playback commands. The complete specification is in SPEC.md in this directory.

## Before You Start

Ask the user these questions:

1. **What programming language should I use?**

2. **What is the IP address of your Roku device?** (Found in Roku Settings > Network > About. Example: `192.168.1.100`)

3. **Which streaming services do you have installed on your Roku?** (Netflix, Disney+, HBO Max / Max, Prime Video — only include channels the user has)

## What to Build

Read SPEC.md in this directory, then implement exactly two functions:

- `convert_url_to_ecp_command(url)` — Takes a streaming URL string, returns a structured extraction result (channel ID, content ID, media type, post-launch key) or null if the URL doesn't match any supported channel.

- `build_playback_command(extraction)` — Takes an extraction result, returns a full playback command (launch channel, wait 2000ms, press key).

The spec contains the complete channel catalog (regex patterns, channel IDs, media type logic), the algorithm, and worked examples for every channel.

## Validation

Test fixtures are in test_fixtures.json in this directory. Run your implementation against all test cases:
- 12 valid URLs that must return correct extraction results
- 11 invalid URLs that must return null
- 4 playback command cases that must produce the correct action sequence

## Key Things to Get Right

- Use regex **search** (find anywhere in string), not match-from-start
- Netflix is the only channel with non-trivial media type logic: `/watch/` = movie, `/title/` = series. All other channels always return "movie".
- The params string is always `contentId={id}&mediaType={type}` — no URL encoding needed
- The post-launch delay is always 2000 milliseconds
- Netflix uses `Play` as the post-launch key; all others use `Select`
