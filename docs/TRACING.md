# Distributed Tracing for llm-d-kv-cache-manager

This document describes how to enable and use OpenTelemetry distributed tracing in the llm-d KV Cache Manager.

## Overview

The KV Cache Manager implements manual OpenTelemetry instrumentation with custom spans at critical operations:

- **`kvcache.manager.get_scores`**: Main scoring operation (SERVER span)
- **`kvcache.storage.lookup`**: Storage backend lookup (INTERNAL span)
- **`kvcache.scorer.compute`**: Scoring algorithm execution (INTERNAL span)

## Configuration

Tracing is configured via environment variables:

```bash
# OTLP collector endpoint (required)
export OTEL_EXPORTER_OTLP_ENDPOINT="http://otel-collector:4317"

# Sampling strategy (optional, default: parentbased_traceidratio)
export OTEL_TRACES_SAMPLER="parentbased_traceidratio"

# Sampling ratio (optional, default: 0.1 for 10%)
export OTEL_TRACES_SAMPLER_ARG="0.1"
```

## Initialization

In your application code, initialize tracing before using the KV Cache Manager:

```go
import (
    "context"
    "github.com/llm-d/llm-d-kv-cache-manager/pkg/telemetry"
)

func main() {
    ctx := context.Background()

    // Initialize tracing
    shutdown, err := telemetry.InitTracing(ctx)
    if err != nil {
        log.Fatalf("Failed to initialize tracing: %v", err)
    }
    defer shutdown(ctx)

    // ... rest of your application
}
```

## Spans and Attributes

### kvcache.manager.get_scores (SERVER)

**Attributes:**
- `gen_ai.request.model` (string): Model identifier
- `kvcache.pod_count` (int): Number of pods being evaluated
- `kvcache.block_keys.count` (int): Number of block keys generated
- `kvcache.total_blocks_available` (int): Total blocks available across all pods
- `kvcache.hit_ratio` (float): Cache hit ratio (pods with hits / total pods)
- `kvcache.pods_with_hits` (int): Number of pods with non-zero scores

**Child Spans:** `kvcache.storage.lookup`, `kvcache.scorer.compute`

### kvcache.storage.lookup (INTERNAL)

**Attributes:**
- `kvcache.lookup.block_count` (int): Number of blocks to lookup
- `kvcache.lookup.pod_filter_count` (int): Number of pods in filter set
- `kvcache.lookup.cache_hit` (bool): Whether any blocks were found
- `kvcache.lookup.blocks_found` (int): Number of blocks found

### kvcache.scorer.compute (INTERNAL)

**Attributes:**
- `kvcache.scorer.algorithm` (string): Scoring algorithm used (e.g., "LongestPrefix")
- `kvcache.scorer.key_count` (int): Number of keys being scored
- `kvcache.score.max` (int): Maximum score assigned
- `kvcache.score.avg` (float): Average score across all pods
- `kvcache.scorer.pods_scored` (int): Number of pods that received scores

## Example Trace

A complete trace for a `GetPodScores` call will show:

```
kvcache.manager.get_scores (10.5ms)
├── kvcache.storage.lookup (8.2ms)
└── kvcache.scorer.compute (1.1ms)
```

## Viewing Traces

### With Jaeger

1. Deploy Jaeger and OpenTelemetry Collector:
```bash
# docker-compose.yml example
version: '3'
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # Jaeger UI
      - "14250:14250"  # gRPC collector

  otel-collector:
    image: otel/opentelemetry-collector:latest
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      - "4317:4317"  # OTLP gRPC
```

2. Configure the KV Cache Manager:
```bash
export OTEL_EXPORTER_OTLP_ENDPOINT="http://localhost:4317"
```

3. View traces at http://localhost:16686

### Query Examples

**Find all KV cache operations:**
```
service="llm-d-kv-cache-manager" AND operation="kvcache.manager.get_scores"
```

**Find slow lookups (>100ms):**
```
operation="kvcache.storage.lookup" AND duration > 100ms
```

**Find low cache hit ratios:**
```
kvcache.hit_ratio < 0.5
```

## Security Considerations

The instrumentation follows metadata-only tracing:

**Captured:**
- ✅ Model identifier
- ✅ Pod counts and block counts
- ✅ Cache hit ratios and scores
- ✅ Timing metrics

**NOT Captured:**
- ❌ Prompts or actual tokens
- ❌ Pod identifiers (only counts)
- ❌ Sensitive user data

## Performance Impact

- Overhead when tracing disabled: ~0% (no-op tracer)
- Overhead at 10% sampling: <2% latency increase
- Span creation is lightweight (microseconds)
- Use parent-based sampling to respect upstream decisions

## Troubleshooting

### No traces appearing

1. Check OTLP endpoint is reachable:
```bash
curl http://otel-collector:4317
```

2. Verify tracing initialization:
```bash
# Look for log message
"OpenTelemetry tracing initialized successfully"
```

3. Check sampling rate (set to 100% for testing):
```bash
export OTEL_TRACES_SAMPLER_ARG="1.0"
```

### High overhead

- Reduce sampling rate: `OTEL_TRACES_SAMPLER_ARG="0.01"` (1%)
- Use batch exporter (already enabled by default)
- Check collector performance

## Integration with Gateway

When the KV Cache Manager is called by the gateway, trace context is automatically propagated via gRPC metadata. Ensure the gateway injects trace context into outgoing requests:

```go
// Gateway side - inject trace context
import "go.opentelemetry.io/otel"

// In gRPC client interceptor
md := metadata.New(nil)
otel.GetTextMapPropagator().Inject(ctx, &MetadataCarrier{md})
ctx = metadata.NewOutgoingContext(ctx, md)
```

The KV Cache Manager automatically extracts trace context from incoming gRPC calls.

## References

- [OpenTelemetry Go Documentation](https://opentelemetry.io/docs/languages/go/)
- [GenAI Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
- [llm-d Distributed Tracing Proposal](../../llm-d/docs/proposals/distributed-tracing.md)
