---
layout: post
title: "WIP: Vector OTLP tracing for Codex CLI, Claude Code, and more"
date: 2026-01-28
description: Quick notes on capturing OTLP logs, traces, and metrics with Vector.
tags: [observability, otlp, vector, codex, claude]
---

WIP note: The goal is to track coding model interactions so I can annotate good
vs bad ones and use that data to improve coding CLIs and models over time; I'm
using Vector as a local OTLP sink for CLI tools. I plan to add more
configurability for other tools and explore the data format in a bit. There are
existing tools for this (e.g., `https://langfuse.com`,
`https://phoenix.arize.com`), but I want a lightweight, single-tool setup that
captures all traces and stores them as compressed JSONL for later analysis
instead of standing up a larger stack.

<!--more-->

## Install Vector
- Follow the Vector docs for your OS.
- macOS (Homebrew):
  ```bash
  brew tap vectordotdev/brew && brew install vector
  ```
- Debian/Ubuntu:
  ```bash
  bash -c "$(curl -L https://setup.vector.dev)"
  sudo apt-get install vector
  ```

## Vector config
Save a config like this (e.g. `config.yaml`):

```yaml
sources:
  otlp:
    type: opentelemetry
    grpc:
      address: 0.0.0.0:4317
    http:
      address: 0.0.0.0:4318

sinks:
  file:
    type: file
    inputs: [otlp.logs, otlp.traces, otlp.metrics]
    path: ./vector-%Y-%m-%d.jsonl.zstd
    compression: zstd
    encoding:
      codec: json
    framing:
      method: newline_delimited
```

## Run locally
- Be mindful of the bind IP. Don't use `0.0.0.0` on a public server.
- Adjust the output path to a location you want to keep long-term.
- Run locally with:
  ```bash
  vector -c ./config.yaml
  ```

## Explore logs
- Install zstd tools for `zstdcat`:
  ```bash
  brew install zstd
  ```

## Codex CLI
Add this to `~/.codex/config.toml`:

```toml
[otel]
log_user_prompt=true
exporter = { otlp-grpc = { endpoint = "http://localhost:4317/", headers = { "x-otlp-meta" = "codex-cli" } }}
```

## Claude Code
Set environment variables:

```bash
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_METRICS_EXPORTER=otlp
export OTEL_LOGS_EXPORTER=otlp
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
export OTEL_EXPORTER_OTLP_HEADERS=".." # if necessary
export OTEL_METRIC_EXPORT_INTERVAL=10000
export OTEL_LOGS_EXPORT_INTERVAL=5000
```

## Docs
- Vector quickstart: https://vector.dev/docs/setup/quickstart/
- Codex CLI config: https://developers.openai.com/codex/config-reference/
- Codex CLI observability: https://developers.openai.com/codex/config-advanced#observability-and-telemetry
- Claude Code monitoring: https://code.claude.com/docs/en/monitoring-usage
