# Declaring what to watch

`.berth/classes.yaml`, in **your** repository. Pilot reads it and writes
nothing outside the paths it names.

```yaml
version: 1
repo:
  allowed_paths:
    - deploy/voice.yaml
    - deploy/embed.yaml
classes:
  - name: voice-agent-prod
    model_id: NousResearch/Meta-Llama-3-8B    # what to watch in the registry
    model: llama3-8b                          # the berth registry key
    running_on: h100-pcie                     # what it runs on today
    config_path: deploy/voice.yaml
    slo:
      metric: p99_ttft_ms
      bound_ms: 800
    workload:
      concurrency: 8
      prompt_tokens: 512
      output_tokens: 128
      mtok_per_hour: 12.0                     # optional, enables the ledger
```

## Declared axes and chosen axes

Three things are **declared** by you and define the problem:

| | |
| --- | --- |
| `model` | drives the physics: bytes per token, cache geometry, precision |
| `workload` | token lengths, concurrency, arrival shape |
| `slo` | the bound, applied as a feasibility gate |

Three are **chosen** by the engine and are the search space: **chips**,
**providers**, and **price basis**.

Declaring one of the chosen axes is refused:

```
classes[0] carries 'provider' as a declared axis. Chips, providers and price
bases are chosen by the engine, not declared. If you need to narrow the
search, put it in a `constraints:` block on this class and say why.
```

The reason is that the engine cannot otherwise tell a preference from a
physical property. `running_on` is not a pin: it says what runs today, so a
recommendation has something to beat.

## Narrowing the search

A constraint is legitimate and it goes somewhere else:

```yaml
    constraints:
      providers: [aws]
      price_bases: [on-demand, reserved]
      interruption_tolerant: false
      exclude_silicon: [mi300x]
      reason: data residency, EU only, and AMD quota is not approved
```

**A reason is required.** A constraint costs money, and the reason is what
lets anyone tell later whether it is still worth paying. The status page shows
what the answer would have been without it.

`interruption_tolerant: false` is the default and it is not a narrowing. It
means spot has no price for this workload rather than a low one: an evicted
request does not deliver a late token, it delivers nothing.

## Every field is refused rather than defaulted

Missing fields raise, named in the order you read them down the file. Duplicate
class names raise, because the name is the key state is tracked against. An
empty `allowed_paths` means none, deliberately: an agent that can write
anywhere is one nobody grants access to.

`mtok_per_hour` is the exception, and it is optional. Without it the class
appears on the status page and not in the ledger, because a percentage
improvement is not money until somebody says how much work there is, and
inferring it would let the reported saving be adjusted by changing an
assumption.

## Parsing

```python
from berth.declaration import load_yaml

decl = load_yaml(open(".berth/classes.yaml").read(), repo="acme/infra")
repo = decl.repo_target("acme", "infra")   # the same paths bound the client
```

PyYAML is used when present and a minimal reader handles this format when it
is not. The core has no dependencies and you should not install one to be
read.
