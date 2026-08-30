# Benchmarking

Reusable [SparkRun](https://sparkrun.dev/) benchmark profiles for testing inference recipes on the DGX Spark / ASUS Ascent GX10.

These profiles provide consistent workloads for comparing models, quantizations, and recipe changes. Benchmark results are generated and managed by SparkRun rather than stored manually in this directory.

## Profiles

### `quick.yaml`

Fast single-user throughput check.

Use this for quick comparisons after changing a recipe, runtime setting, quantization, or model.

```bash
sparkrun benchmark @blee/qwen38-27b-nvfp4 \
  --profile @blee/quick \
  --solo
```

Typical uses:

* Compare quantizations
* Compare runtime/recipe changes
* Check for performance regressions
* Establish basic decode throughput

### `agent.yaml`

Heavier workload intended to better approximate interactive coding-agent use.

Uses a larger prompt and tests performance with an established context rather than measuring only an empty KV cache.

```bash
sparkrun benchmark @blee/qwen38-27b-nvfp4 \
  --profile @blee/agent \
  --solo
```

For models with larger context windows, additional depths can be supplied at runtime:

```bash
sparkrun benchmark @blee/qwen38-27b-nvfp4 \
  --profile @blee/agent \
  --solo \
  -b depth=0,16384,65536,131072,196608
```

This is useful for identifying performance changes as long-running coding sessions fill the context window.

### `concurrency.yaml`

Tests throughput and latency with multiple simultaneous requests.

Primarily intended for models serving chat, vision, and coding subagents where several requests may execute concurrently.

```bash
sparkrun benchmark @blee/qwen36-35b-a3b-vision \
  --profile @blee/concurrency \
  --solo
```

Use this to evaluate the tradeoff between per-request latency and aggregate throughput as concurrency increases.

## Overriding Benchmark Parameters

Profile settings can be changed without modifying the profile itself using `-b`:

```bash
sparkrun benchmark @blee/qwen38-27b-nvfp4 \
  --profile @blee/quick \
  --solo \
  -b tg=1024
```

Multiple benchmark parameters can be overridden:

```bash
sparkrun benchmark @blee/qwen38-27b-nvfp4 \
  --profile @blee/agent \
  --solo \
  -b depth=0,65536,131072 \
  -b tg=1024
```

## Testing Recipe Changes

Recipe parameters can be overridden with `-o`, allowing configuration changes to be benchmarked without editing the stored recipe.

For example, test a lower vLLM memory allocation:

```bash
sparkrun benchmark @blee/qwen38-27b-nvfp4 \
  --profile @blee/agent \
  --solo \
  -o gpu_memory_utilization=0.45
```

This makes it easy to experiment with settings while keeping the committed recipe as the known-good default.

## Guidelines

Use the same benchmark profile when comparing models or configurations.

Prefer changing one significant variable at a time when tuning a recipe.

The `quick` profile is intended for rapid iteration. Use `agent` when a change may behave differently with larger prompts or populated context.

Use `concurrency` when evaluating models intended to serve parallel chat or subagent workloads.

Model-specific context limits should generally be supplied as benchmark overrides rather than baked into shared profiles.
