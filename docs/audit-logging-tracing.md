# LightSpeed Audit Logging and Tracing

This document outlines Lightspeed audit logging and tracing configuration and behavior.

## Configuration

### Operator Configuration

The operator reads audit and tracing configuration from the `AgenticOLSConfig` CRD:

```yaml
apiVersion: agentic.openshift.io/v1alpha1
kind: AgenticOLSConfig
metadata:
  name: cluster
  namespace: openshift-lightspeed
spec:
  audit:
    # Enable/disable JSON audit logs to stdout (default: Enabled)
    logging: Enabled  # or Disabled
    
    # OpenTelemetry tracing export
    otel:
      # OTLP gRPC endpoint (e.g., "otel-collector.observability.svc:4317")
      # When empty, no traces are exported (no-op tracer)
      endpoint: "otel-collector.observability.svc:4317"
      # TLS mode: Secure (default) or Insecure
      tlsMode: Insecure
```

**Binary behavior**:
- Reads `AgenticOLSConfig` named `cluster` in the operator namespace at startup
- If CR not found or audit config empty, defaults to: logging enabled, tracing disabled
- Initializes OTEL TracerProvider with OTLP gRPC exporter if endpoint configured
- Creates `ProductionAuditLogger` (JSON logs) or `NoOpAuditLogger` based on logging setting

### Sandbox Configuration

The sandbox uses **environment variables** for configuration (no CRD):

| Variable | Default | Description |
|----------|---------|-------------|
| `LIGHTSPEED_AUDIT_ENABLED` | `false` | Enable JSON audit logs (`"true"` to enable) |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `""` | OTLP endpoint for traces (empty = disabled) |
| `OTEL_SERVICE_NAME` | `lightspeed-agentic-sandbox` | Service name for traces |
| `OTEL_EXPORTER_OTLP_INSECURE` | auto-detect | TLS control (`true`/`false`) |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | `grpc` | Protocol selection (`grpc`/`http/protobuf`) |
| `OTEL_EXPORTER_OTLP_HEADERS` | `""` | Custom headers (URL-encoded, e.g., auth tokens) |
| `OTEL_RESOURCE_ATTRIBUTES` | `""` | Additional resource attributes (`key=value,...`) |
| `OTEL_TRACES_EXPORTER` | `otlp` | Exporter selection (`otlp`/`none`) |

**Binary behavior**:
- `LIGHTSPEED_AUDIT_ENABLED=true` enables `AuditLogger._emit()` JSON output
- `OTEL_EXPORTER_OTLP_ENDPOINT` configures TracerProvider with OTLP exporter
- TLS auto-detected from endpoint scheme (`https://` = TLS) unless `OTEL_EXPORTER_OTLP_INSECURE` is set
- Standard Python logging always enabled

**Example sandbox pod env**:
```yaml
env:
- name: LIGHTSPEED_AUDIT_ENABLED
  value: "true"
- name: OTEL_EXPORTER_OTLP_ENDPOINT
  value: "http://otel-collector.observability.svc:4317"
- name: OTEL_SERVICE_NAME
  value: "my-sandbox"
```

## Telemetry Output

### Operator

The operator emits both JSON audit logs and OTEL span events for each lifecycle event. They contain different data:

| Signal | Content | Purpose |
|--------|---------|---------|
| JSON logs | Full CR payloads (metadata, spec, status) | Compliance audit - complete record |
| Span events | Summarized attributes (truncated, limited counts) | Observability - lightweight tracing |

Example for `audit.analysis.completed`:
- **Log**: Full `analysisResult` CR with all options, diagnoses, proposals
- **Span event**: `proposal.name`, `result.name/uid`, `options.count`, first 3 options only (title, risk)

### Sandbox

The sandbox has **three separate output mechanisms**:

| Mechanism | Format | Purpose |
|-----------|--------|---------|
| Standard logging | `logger.info()` text | Developer debugging |
| JSON audit logs | `print(json.dumps(...))` | Compliance audit |
| OTEL spans | `tracer.start_span()` | Distributed tracing |

**Standard logging**:
```
INFO lightspeed_agentic: [provider:analysis] tool_use: exec_command({"cmd":"kubectl get pods"...})
INFO lightspeed_agentic: [provider:analysis] tool_result: Chunk ID: abc123...
INFO lightspeed_agentic: [provider:analysis] result: cost=$0.0150, tokens=1500
```

**JSON audit logs**:
```json
{"timestamp": "...", "level": "audit", "event": "audit.agent.tool.call", "trace_id": "...", "phase": "analysis", "tool_name": "exec_command", "tool_input": "..."}
```

**OTEL spans**:
- `tool.{name}` span with attributes: `tool.name`, `tool.input`, `tool.output`

| Event | Standard Log | JSON Audit | OTEL Span |
|-------|--------------|------------|-----------|
| Agent started | None | `audit.agent.started` | None |
| Tool call | `tool_use: name(input)` (500 chars) | `audit.agent.tool.call` (full input) | Creates `tool.{name}` span (300 chars) |
| Tool result | `tool_result: output` (1000 chars) | `audit.agent.tool.result` (full output) | Sets `tool.output` attr (500 chars), ends span |
| Thinking | `thinking: ...` (2000 chars) | `audit.agent.thinking` (full text) | None |
| Text output | None | `audit.agent.text` (full text) | None |
| Result | `result: cost=$X, tokens=Y` | `audit.agent.completed` | None |

**Key differences**:
- **Spans only cover tool execution** — no spans for thinking, text, started, completed events
- **Audit logs have full payloads** — spans truncate (input: 300 chars, output: 500 chars)
- **Spans capture duration** — audit logs only have point-in-time timestamps
- **Spans have hierarchy** — parent-child relationships; audit logs are flat with just `trace_id`

## Span Attributes

### Operator Spans

The operator creates workflow/lifecycle spans that track the Kubernetes proposal lifecycle.

**`proposal.lifecycle` span** (root span):
| Attribute | Value |
|-----------|-------|
| `proposal.name` | proposal name |
| `proposal.namespace` | namespace |
| `proposal.uid` | UID |

**`proposal.analyze` / `proposal.execute` / `proposal.verify` / `proposal.escalate` spans**:
| Attribute | Value |
|-----------|-------|
| `proposal.name` | proposal name |
| `proposal.namespace` | namespace |
| `retry_index` | (on execute/verify, if retrying) |

**`proposal.human_approval` span** (measures human decision time):
| Attribute | Value |
|-----------|-------|
| `proposal.name` | proposal name |
| `proposal.namespace` | namespace |
| `approver.uid` | approver UID (on end) |
| `approver.username` | approver username (on end) |
| `approval.decision` | decision (on end) |

**`proposal.terminal` span**:
| Attribute | Value |
|-----------|-------|
| `proposal.name` | proposal name |
| `proposal.namespace` | namespace |
| `phase` | terminal phase |
| `reason` | terminal reason |

**Span events** (attached to phase spans):

| Event | CR Type | Attributes |
|-------|---------|------------|
| `audit.analysis.completed` | AnalysisResult | `result.name`, `result.uid`, `options.count`, `option.{i}.title`, `option.{i}.risk` |
| `audit.execution.completed` | ExecutionResult | `result.name`, `result.uid`, `actions_taken.count`, `failure_reason`, `action.{i}.type`, `action.{i}.description` |
| `audit.verification.completed` | VerificationResult | `result.name`, `result.uid`, `summary`, `checks.count`, `check.{i}.name`, `check.{i}.result` |
| `audit.verification.retry` | VerificationResult | `result.name`, `summary`, `retry_count`, `checks.count` |
| `audit.escalation.completed` | EscalationResult | (logged, attributes TBD) |

Note: Span events truncate/limit data (e.g., first 3 options, first 5 actions/checks) while JSON audit logs contain full CR payloads.

### Sandbox Spans

**`agent.run` span**:
| Attribute | Value |
|-----------|-------|
| `model` | model name |
| `provider` | provider name |

**`tool.{name}` span** (e.g., `tool.exec_command`):
| Attribute | Value |
|-----------|-------|
| `tool.name` | tool name |
| `tool.input` | input JSON (truncated 300 chars) |
| `tool.output` | output (truncated 500 chars) |

## Trace Hierarchy

When the operator calls the sandbox, traces are propagated via W3C `traceparent` header, creating a unified hierarchy:

```
proposal.lifecycle
└── proposal.analyze
    └── agent.run
        ├── tool.exec_command
        ├── tool.kubectl
        └── ...
```
