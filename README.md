# berth documentation

MkDocs-Material site for berth, structured after the Tinker docs information
architecture and adapted for an open-source local library (no hosted service,
no pricing, no API keys).

## Pages
- index.md ................ overview + division-of-labor table + one-line pitch
- quickstart.md ........... install and first estimate
- silicon-and-models.md ... the fleet and model registry, with provenance status
- the-physics.md .......... every roofline term, auditable
- validation-p0.md ........ P0 hardware validation, passes and failures
- placement-premium.md .... the headline quantity and how to read it

## Build and publish
    pip install mkdocs-material
    mkdocs serve            # local preview at http://127.0.0.1:8000
    mkdocs build            # static site into ./site
    mkdocs gh-deploy        # publish to GitHub Pages

## Adding to reckonresearch.com
Two clean options:
1. Subpath: build into the main site repo under /docs and link "Docs" from the
   navbar. Simplest; keeps one domain.
2. Subdomain: docs.reckonresearch.com as its own GitHub Pages site (a CNAME in
   the docs repo). Mirrors the Tinker setup (tinker-docs.thinkingmachines.ai).

Either way the content is these six markdown files; only the hosting wrapper
differs.

## Red-team corrections applied
- Quickstart used invented functions premium()/place() and field usd_per_mtok.
  Corrected to the real API: Estimate.cost_per_mtok, PlacementClient.place with
  PlacementPolicy(objective=min_cost). All code blocks now execute against v8.
- SUPERSEDED at v9: the CLI now exists. pyproject declares
  `[project.scripts] berth = "berth.cli:main"` with estimate/premium/list
  subcommands, so the CLI documentation is correct and index.md's "on the
  roadmap" line has been removed.
- The distributed package is named `berth-placement`, not `berth`. `pip install
  berth` installs an unrelated project. Fix every install instruction, including
  the one on reckonresearch.com, before publishing to PyPI.
- Added a caveat that place() may return a prior (unmeasured) accelerator.

## Second red-team pass (applied)
- placement-premium.md defined a 1.6x premium as "60 percent less". A 1.6x
  premium is 37.5 percent less, or 60 percent more from the other end. Corrected
  and both directions now stated.
- The measured premium table was presented as a result "at equal p99" while
  comparing 45.3 against 93.9 tokens per second per stream. It is a matched-batch
  comparison. The condition is now stated on index, placement-premium and the
  walkthrough, with the SLO inversion spelled out.
- placement-premium.md still called `premium()`, which does not exist. Replaced
  with `berth premium` and `client.estimate`.
- verify-and-contribute.md applied the H100 PCIe fitted floor (54.6 ms) to an
  `h100-sxm` example, which is the cross-setup promotion the same page forbids.
  Example moved to h100-pcie and `bench.fit_overhead` documented.
- Trace schema had no field distinguishing mock from measured output, while the
  page invited contributions by pull request. Requires the code change in
  PATCHES.md; documented as schema 2 with a `source` field.
- Peak TFLOPS column carries sparsity ratings, roughly 2x dense. Harmless on
  measured silicon (absorbed by fitted MFU), optimistic on prior silicon for
  compute-bound predictions. Disclosed.

## Provenance rule (carried from the project)
Every silicon profile and every predicted number is tagged MEASURED or prior.
Do not edit a page in a way that presents a spec-sheet prior as a measurement.
