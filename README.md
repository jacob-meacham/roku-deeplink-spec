# Roku ECP Reference

A language-agnostic specification for converting public streaming service URLs into Roku ECP (External Control Protocol) playback commands.

## What This Is

This directory contains a natural language specification ([SPEC.md](SPEC.md)) detailed enough for an LLM to implement the URL-to-ECP conversion feature in any programming language from a single read. A Python reference implementation and test harness validate correctness.

## Files

| File | Purpose |
|------|---------|
| `SPEC.md` | Complete specification (the "source code" in natural language) |
| `test_fixtures.json` | All test cases as structured JSON |
| `reference.py` | Python reference implementation |
| `test_reference.py` | Pytest test runner |

## Running Tests

Prerequisites: Python 3.10+, pytest

```bash
cd roku-ecp-reference
pip install pytest
pytest test_reference.py -v
```

## Validating a Generated Implementation

To test your own implementation against the fixtures:

1. Implement two functions matching the contract in SPEC.md Section 9
2. Update the import in `test_reference.py` to point to your module
3. Run `pytest test_reference.py -v`

All 27 tests must pass (12 valid URLs + 11 invalid URLs + 4 playback commands).
