# Roku ECP Reference

A language-agnostic specification for converting public streaming service URLs into Roku ECP (External Control Protocol) playback commands.

## Quickstart

In your favorite LLM, just say "@PROMPT.md"

## What This Is

This directory contains a natural language specification ([SPEC.md](SPEC.md)) detailed enough for an LLM to implement the URL-to-ECP conversion feature in any programming language from a single read. A Python reference implementation and test harness validate correctness.

## Files

| File | Purpose |
|------|---------|
| `SPEC.md` | Complete specification (the "source code" in natural language) |
| `PROMPT.md` | Initial prompt to pass to an agent |
| `test_fixtures.json` | All test cases as structured JSON |


## Running Tests

## Validating a Generated Implementation

To test your own implementation against the fixtures:

1. Implement two functions matching the contract in SPEC.md Section 9
2. Update the import in `test_reference.py` to point to your module
3. Run `pytest test_reference.py -v`

All 27 tests must pass (12 valid URLs + 11 invalid URLs + 4 playback commands).
