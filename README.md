# Roku ECP Reference

A language-agnostic specification for converting public streaming service URLs into Roku ECP (External Control Protocol) playback commands.

## Quickstart

In your favorite LLM, just say "@PROMPT.md"

## What This Is

This directory contains a natural language specification ([SPEC.md](SPEC.md)) detailed enough for an LLM to implement the URL-to-ECP conversion feature in any programming language from a single read. The canonical test fixtures (`test_fixtures.json`) validate correctness of any generated implementation.

## Files

| File | Purpose |
|------|---------|
| `SPEC.md` | Complete specification (the "source code" in natural language) |
| `PROMPT.md` | Initial prompt to pass to an agent |
| `test_fixtures.json` | All test cases as structured JSON |


## Validating a Generated Implementation

To test your own implementation against the fixtures:

1. Implement two functions matching the contract in SPEC.md Section 9
2. Run every case in `test_fixtures.json` against them (speclib consumers get the fixtures materialized into their test suite at sync time)

All 52 cases must pass (27 valid URLs + 16 invalid URLs + 9 playback commands).
