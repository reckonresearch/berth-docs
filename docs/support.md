# Support

## Something is wrong with a number

This is the most useful thing you can report, and it takes precedence over
everything else.

Run the harness on your hardware, then open an issue with your `traces.jsonl`
and the report it produced:

```bash
python -m bench.sounding --base-url http://localhost:8000 \
    --silicon <key> --model <key> --model-id <hf-id> --out traces.jsonl
python -m bench.fit_overhead traces.jsonl
python -m bench.validate     traces.jsonl --prefill-overhead-ms <fitted>
```

[Open an issue](https://github.com/reckonresearch/berth/issues) with those
attached. Disputes that arrive with traces outrank everything else, and
unfavourable results publish.

A trace carries no prompt content, only cell coordinates and two timing
numbers, so it is safe to share.

## Something is broken

[GitHub issues](https://github.com/reckonresearch/berth/issues). Include the
berth version (`pip show berth-placement`), your Python version, and the full
traceback.

Before filing, check the two things that account for most reports:

**The prefill floor.** It belongs to one accelerator, driver, server and config,
and is not predictable from a spec sheet. Fit your own with
`python -m bench.fit_overhead`; do not reuse a value from these docs.

**Mock traces.** If validation reports something implausible, check the
provenance line in the header. A fit over mock data recovers the model that
generated it.

## An accelerator or model is missing

Open an issue with the specification sheet. Adding a silicon profile is a few
lines, and it ships tagged `prior` until someone measures it.

## Contributing code

See [CONTRIBUTING.md](https://github.com/reckonresearch/berth/blob/main/CONTRIBUTING.md).
Two rules specific to this project:

- A new check needs a negative test. A rule that only passes its own example
  proves nothing.
- Do not make the implementation stricter than the specification it implements.
  If the specification is wrong, amend the specification.

## Security

Report privately per
[SECURITY.md](https://github.com/reckonresearch/berth/blob/main/SECURITY.md).
Do not open a public issue for a vulnerability.

## Everything else

[hello@reckonresearch.com](mailto:hello@reckonresearch.com)

berth is built and maintained by [Reckon Research](https://reckonresearch.com),
publisher of the placement-premium index. Apache-2.0.
