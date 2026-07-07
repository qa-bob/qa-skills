# OTel Test Pipeline Instrumentation

Instrumenting the test pipeline itself with OpenTelemetry so any individual run — especially a failing or unusually slow one — can be diagnosed from its trace, instead of requiring a live reproduction.

---

## What to Trace

```
Test Run (root span)
├── Setup (span): environment provisioning, dependency install, test data seeding
├── Suite: Unit Tests (span)
│   ├── test_foo (span)
│   ├── test_bar (span)
│   └── ...
├── Suite: Integration Tests (span)
│   ├── Container startup (span) — a common, otherwise-invisible source of duration
│   ├── test_baz (span)
│   └── ...
├── Suite: E2E Tests (span)
│   └── ...
└── Teardown (span): environment cleanup, artifact upload
```

The key design decision: trace the *pipeline's own structure* (setup, per-suite, per-test, teardown), not just wrap the whole run in one opaque span. A single root span with no children tells you the run took 14 minutes; a properly structured trace tells you container startup alone took 6 of those minutes, which is the actionable finding.

---

## Instrumentation Pattern (Python / pytest example)

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

trace.set_tracer_provider(TracerProvider())
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter(endpoint="otel-collector:4317"))
)
tracer = trace.get_tracer("test-pipeline")

# pytest hook — wrap each test in its own span automatically
def pytest_runtest_protocol(item, nextitem):
    with tracer.start_as_current_span(item.nodeid) as span:
        span.set_attribute("test.file", str(item.fspath))
        span.set_attribute("test.suite", item.parent.name)
        outcome = yield
        result = outcome.get_result() if outcome else None
        if result and result.failed:
            span.set_attribute("test.outcome", "failed")
            span.record_exception(result.longrepr)
        else:
            span.set_attribute("test.outcome", "passed")
```

Most CI-native test frameworks and popular test runners (pytest, Jest, JUnit) have existing OTel plugins/integrations — prefer those over hand-rolling instrumentation where one already exists and covers the needed attributes.

---

## Trace Context Propagation Across CI Stages

If the pipeline spans multiple discrete CI jobs/stages (e.g. a build job, then a separate test job, then a separate deploy job), propagate the trace context between them so the whole pipeline run appears as one connected trace rather than three disconnected ones:

```yaml
# GitHub Actions example — pass trace context via an environment variable
# or artifact between jobs
- name: Export trace context
  run: echo "TRACEPARENT=$(otel-cli span-context)" >> pipeline_context.env

# Downstream job reads it back to continue the same trace
- name: Import trace context
  run: source pipeline_context.env
```

Without propagation, a slowdown that's actually caused by the build stage will look, from the test stage's own trace, like the test stage started late for no visible reason — reconnecting the full pipeline into one trace is what makes root-causing this kind of cross-stage issue possible at all.

---

## What to Attach as Span Attributes

Beyond basic name/duration/outcome, attach the attributes that make a trace actually useful for diagnosis later, since these are hard to reconstruct after the fact if not captured at the time:

| Attribute | Why it matters |
|---|---|
| Git commit SHA / branch | Correlates a specific trace to the exact code that was tested |
| CI runner/executor ID | Distinguishes "this specific runner was slow" from "the whole pipeline is slow" |
| Retry count | A test that passed on retry #2 is meaningfully different from one that passed on attempt #1 — collapsing these loses the flakiness signal (see `flakiness-and-metrics-dashboards.md`) |
| Resource usage (CPU/memory) at time of test, if available | Helps distinguish "the test got slower" from "the runner was resource-starved by something else running concurrently" |

---

## Checklist

```
- [ ] Trace structure mirrors the pipeline's actual stages (setup/per-suite/per-test/teardown), not one opaque root span
- [ ] Trace context propagates across CI job/stage boundaries so a multi-stage pipeline appears as one connected trace
- [ ] Failed spans record the actual exception/failure detail, not just a pass/fail boolean
- [ ] Retry count is captured per test, not silently collapsed into "eventually passed"
- [ ] Traces are exported somewhere queryable (not just printed to CI logs) so historical runs can actually be compared
```
