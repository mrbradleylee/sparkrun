# SparkRun Recipes

Personal [SparkRun](https://sparkrun.dev/) recipe registry for running and tuning local models on NVIDIA GB10 / DGX Spark-class hardware.

The repository contains known-good model serving configurations and reusable benchmark profiles. Recipes are intended to be directly runnable through SparkRun while keeping model-specific runtime configuration under version control.

## Repository Layout

```text
.
├── .sparkrun/
│   └── registry.yaml
├── recipes/
│   ├── qwen38-27b-nvfp4.yaml
│   └── qwen36-35b-a3b-nvfp4.yaml
├── benchmarking/
│   ├── README.md
│   ├── quick.yaml
│   ├── agent.yaml
│   └── concurrency.yaml
└── README.md
```

## Registry

The registry is named `blee`.

Add it to SparkRun:

```bash
sparkrun registry add \
  https://github.com/YOUR_GITHUB_USERNAME/sparkrun.git \
  --trust
```

Update the local registry after changes are pushed:

```bash
sparkrun registry update
```

List available recipes:

```bash
sparkrun list
```

Inspect a recipe:

```bash
sparkrun show @blee/qwen38-27b-nvfp4
```

## Models

### Qwen3.8 27B NVFP4

```text
@blee/qwen38-27b-nvfp4
```

Primary long-context model for coding, orchestration, and heavier workloads.

Default configuration:

| Setting                | Default                     |
| ---------------------- | --------------------------- |
| Model                  | `unsloth/Qwen3.8-27B-NVFP4` |
| Port                   | `8000`                      |
| Context                | `262144`                    |
| GPU memory utilization | `0.50`                      |
| KV cache               | `fp8`                       |
| Tensor parallel        | `1`                         |
| Max sequences          | `2`                         |
| Reasoning parser       | `qwen3`                     |
| Tool parser            | `qwen3_coder`               |
| Auto tool choice       | Enabled                     |
| Speculative decoding   | MTP, 3 tokens               |

Launch:

```bash
sparkrun run @blee/qwen38-27b-nvfp4 --solo
```

The recipe intentionally allocates approximately half of the available unified GPU memory so a second model can remain resident.

### Qwen3.6 35B-A3B NVFP4

```text
@blee/qwen36-35b-a3b-nvfp4
```

Fast multimodal model for chat, vision, coding subagents, and parallel worker tasks.

Default configuration:

| Setting                | Default                         |
| ---------------------- | ------------------------------- |
| Model                  | `unsloth/Qwen3.6-35B-A3B-NVFP4` |
| Port                   | `8001`                          |
| Context                | `131072`                        |
| GPU memory utilization | `0.27`                          |
| KV cache               | `fp8`                           |
| Max sequences          | `4`                             |
| MM processor cache     | `0.5 GiB`                       |
| CUTE DSL architecture  | `sm_121a`                       |
| Reasoning parser       | `qwen3`                         |
| Tool parser            | `qwen3_coder`                   |
| Auto tool choice       | Enabled                         |

Launch:

```bash
sparkrun run @blee/qwen36-35b-a3b-nvfp4 --solo
```

vLLM automatically selects a compatible MoE backend for the model and hardware.

## Default Dual-Model Layout

The recipes are designed to coexist on a single 128 GB GB10 system.

```text
128 GB Unified Memory
│
├── Qwen3.8 27B NVFP4
│   ├── port 8000
│   ├── 262k context
│   ├── 50% vLLM memory budget
│   └── primary / long-context
│
├── Qwen3.6 35B-A3B NVFP4
│   ├── port 8001
│   ├── 128k context
│   ├── 27% vLLM memory budget
│   └── fast / vision / subagents
│
└── 23% unallocated
    └── OS / runtime / headroom
```

Memory utilization values are deliberately conservative defaults and can be tuned as actual dual-model memory requirements are established.

## Recipe Overrides

SparkRun recipe defaults can be overridden without modifying the committed recipe.

### Context Length

Run Qwen3.8 at 128k instead of its default 262k:

```bash
sparkrun run @blee/qwen38-27b-nvfp4 \
  --solo \
  --max-model-len 131072
```

### GPU Memory

Test Qwen3.8 with a smaller memory allocation:

```bash
sparkrun run @blee/qwen38-27b-nvfp4 \
  --solo \
  --gpu-mem 0.45
```

### Recipe-Specific Settings

Arbitrary recipe defaults can be overridden with `-o`:

```bash
sparkrun run @blee/qwen38-27b-nvfp4 \
  --solo \
  -o max_num_seqs=1
```

Multiple overrides can be combined:

```bash
sparkrun run @blee/qwen38-27b-nvfp4 \
  --solo \
  --max-model-len 131072 \
  --gpu-mem 0.45 \
  -o max_num_seqs=1
```

## Benchmarking

Reusable SparkRun benchmark profiles are stored in `benchmarking/`.

### Quick

Basic single-user throughput test:

```bash
sparkrun benchmark @blee/qwen38-27b-nvfp4 \
  --profile @blee/quick \
  --solo
```

### Agent

Heavier prompt/context workload:

```bash
sparkrun benchmark @blee/qwen38-27b-nvfp4 \
  --profile @blee/agent \
  --solo
```

Long-context depths can be supplied when benchmarking the 262k model:

```bash
sparkrun benchmark @blee/qwen38-27b-nvfp4 \
  --profile @blee/agent \
  --solo \
  -b depth=0,16384,65536,131072,196608
```

### Concurrency

Useful for testing subagent/worker workloads:

```bash
sparkrun benchmark @blee/qwen36-35b-a3b-nvfp4 \
  --profile @blee/concurrency \
  --solo
```

See [`benchmarking/README.md`](benchmarking/README.md) for profile details and additional examples.

## Benchmarking Recipe Overrides

Recipe tuning can be tested without changing the stored defaults.

For example:

```bash
sparkrun benchmark @blee/qwen38-27b-nvfp4 \
  --profile @blee/agent \
  --solo \
  -o gpu_memory_utilization=0.45
```

Benchmark parameters can be changed independently using `-b`:

```bash
sparkrun benchmark @blee/qwen38-27b-nvfp4 \
  --profile @blee/quick \
  --solo \
  -b tg=1024
```

This keeps committed recipes as known-good configurations while allowing temporary experiments from the command line.

## Hugging Face Authentication

Hugging Face credentials are intentionally **not stored in this repository**.

Set `HF_TOKEN` in the host/runtime environment used by SparkRun.

Do not commit:

```text
HF_TOKEN
.env
API keys
credentials
```

The repository `.gitignore` should include:

```gitignore
.env
.env.*
*.log
.DS_Store
```

## Updating Recipes

Typical workflow:

```bash
git pull

# Edit a recipe
nvim recipes/qwen38-27b-nvfp4.yaml

# Validate/test as appropriate

git add .
git commit -m "Tune Qwen3.8 recipe"
git push
```

Refresh SparkRun after pushing:

```bash
sparkrun registry update
```

The updated recipe can then be inspected:

```bash
sparkrun show @blee/qwen38-27b-nvfp4
```

and launched normally:

```bash
sparkrun run @blee/qwen38-27b-nvfp4 --solo
```

## Design Goals

Keep recipes:

* Reproducible
* Model-specific
* Easy to override for experiments
* Free of credentials
* Tuned for GB10 / Blackwell
* Conservative enough for simultaneous model serving

Committed recipes should represent known-good defaults. Experimental settings should generally be tested with SparkRun overrides before replacing those defaults.
