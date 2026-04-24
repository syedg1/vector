# RFC 25329 - 2026-04-29 - Internal Trace Data Model

This RFC replaces the inner representation of Vector's `TraceEvent`, today a thin newtype over
`LogEvent`, with a strongly-typed container that mirrors the wire-level batching of OTLP and
Datadog APM traces. Each `TraceEvent` carries one `Resource`, one `Scope`, one Datadog-specific
`ChunkContext`, and the `Vec<Span>` belonging to that grouping, plus the existing `EventMetadata`.
The container shape yields zero-loss `OTLP -> Vector -> OTLP` and `Datadog -> Vector -> Datadog`
round trips, including across Vector's disk buffers, and gives transforms a uniform typed surface
across the two source formats.

## Context

- [RFC 11851 -- OpenTelemetry traces source](2022-03-15-11851-ingest-opentelemetry-traces.md) was
  accepted on the condition that an internal trace model be established before the work was
  completed.
- [RFC 9572 -- Accept Datadog traces](2021-10-15-9572-accept-datadog-traces.md) introduced the
  `datadog_agent` trace ingest path, which the `datadog_traces` sink can consume but which does
  not have a well-defined internal representation.
- An earlier draft of an internal trace model is available at
  [2024-03-22-20170-trace-data-model](https://github.com/hdost/vector/blob/add-trace-data-model/rfcs/2024-03-22-20170-trace-data-model.md);
  this RFC supersedes that draft.
- The current implementation in
  [`lib/vector-core/src/event/trace.rs`](../lib/vector-core/src/event/trace.rs) is
  `TraceEvent(LogEvent)` -- a thin newtype with no type structure. Transforms depend on the
  ingesting source's key layout, and cross-format conversions are ad-hoc per sink.
- [vectordotdev/vector#22659 -- Transform between opentelemetry and datadog traces](https://github.com/vectordotdev/vector/issues/22659).

## Glossary

This RFC references several trace data formats. The two Vector targets are OTLP and the Datadog
agent-to-backend protobuf; everything else is informational.

- **OTLP (OpenTelemetry Protocol)**: the wire format the OpenTelemetry project defines for traces,
  metrics, and logs. The traces schema lives in
  [`opentelemetry/proto/trace/v1/trace.proto`](https://github.com/open-telemetry/opentelemetry-proto/blob/main/opentelemetry/proto/trace/v1/trace.proto),
  with shared value types in
  [`common/v1/common.proto`](https://github.com/open-telemetry/opentelemetry-proto/blob/main/opentelemetry/proto/common/v1/common.proto)
  and resource types in
  [`resource/v1/resource.proto`](https://github.com/open-telemetry/opentelemetry-proto/blob/main/opentelemetry/proto/resource/v1/resource.proto).
  When this document says "OTLP" it means that wire schema and the data model it defines
  (`ResourceSpans`, `ScopeSpans`, `Span`, `AnyValue`, etc.).
- **OpenTelemetry**: the broader project under which OTLP is one component. References in this RFC
  to "OpenTelemetry" name the project's non-wire artefacts: the
  [specification](https://github.com/open-telemetry/opentelemetry-specification) and the
  [semantic conventions](https://github.com/open-telemetry/semantic-conventions) (the registry of
  attribute keys such as `service.name` and `http.request.method`).
- **Datadog APM trace format**: Vector targets exactly one hop in the Datadog tracing pipeline --
  the agent-to-backend protobuf served at `/api/v0.2/traces`. When this RFC says "Datadog"
  unqualified, it means that format. The schema lives in three protobuf files in the Datadog
  Agent repository:
  - [`agent_payload.proto`](https://github.com/DataDog/datadog-agent/blob/main/pkg/proto/datadog/trace/agent_payload.proto)
    -- `AgentPayload` (`tracerPayloads[]`, agent-level `tags`, `agentVersion`, `targetTPS`,
    `errorTPS`).
  - [`tracer_payload.proto`](https://github.com/DataDog/datadog-agent/blob/main/pkg/proto/datadog/trace/tracer_payload.proto)
    -- `TracerPayload` (`chunks[]`, tracer-level fields) and `TraceChunk`
    (`priority`/`origin`/`droppedTrace`/`tags`, `spans[]`).
  - [`span.proto`](https://github.com/DataDog/datadog-agent/blob/main/pkg/proto/datadog/trace/span.proto)
    -- the per-span shape (`service`, `name`, `resource`, `traceID`, `spanID`, `parentID`,
    `start`, `duration`, `error`, `meta`, `metrics`, `type`, `meta_struct`).

  The [Datadog Agent's OTLP ingest](https://github.com/DataDog/datadog-agent/blob/main/pkg/trace/api/otlp.go)
  is cited as the reference implementation for OTLP-to-Datadog field mappings adopted here.
- **Datadog tracer-to-agent API** (informational): tracer SDKs send traces to the Datadog Agent
  over a separate set of HTTP endpoints (`/v0.3/traces`, `/v0.4/traces`, `/v0.5/traces`,
  `/v0.7/traces`) using JSON, msgpack, or protobuf. These are upstream of the agent-to-backend
  hop Vector consumes; the public guide
  [Send traces to the Agent by API](https://docs.datadoghq.com/tracing/guide/send_traces_to_agent_by_api/)
  documents the legacy v0.3 JSON shape and is cited only as a reference for per-span field
  semantics. Vector does not consume these endpoints directly.
- **W3C Trace Context** (informational): the W3C recommendation defining the
  [`traceparent` and `tracestate` HTTP headers](https://www.w3.org/TR/trace-context/). The
  proposed `TraceFlags` and `TraceState` types correspond to these headers; the size bounds
  quoted in the `TraceState` rationale (32 entries, 512 bytes total) come from this spec.
- **Zipkin v2, Jaeger, OpenTracing** (informational): other trace data models referenced in passing
  for context. None are targeted by this RFC and they are not constraints on the design. Zipkin v2
  is documented at the [Zipkin API](https://zipkin.io/zipkin-api/#/default/get_spans); Jaeger at
  [jaegertracing.io](https://www.jaegertracing.io/docs/latest/architecture/#span); OpenTracing at
  the [OpenTracing spec](https://github.com/opentracing/specification/blob/master/specification.md).

## Cross cutting concerns

- First-class OpenTelemetry signal support
  ([vectordotdev/vector#1444](https://github.com/vectordotdev/vector/issues/1444)).
- APM stats aggregation in the `datadog_traces` sink, today reading magic keys from `TraceEvent`,
  will read typed fields after this RFC lands.
- VRL trace-specific semantics on the new typed surface (`.resource.service`, `.chunk.priority`,
  `.spans[i].name`, etc.).

## Scope

### In scope

- Define `TraceEvent` as an array of spans plus supporting resource data, replacing the current
  `TraceEvent(LogEvent)`.
- Specify the bidirectional mapping between `TraceEvent` and the OTLP wire format.
- Specify the bidirectional mapping between `TraceEvent` and the Datadog agent-to-backend protobuf.
- Guarantee zero-loss round-trip through Vector for both formats when the pipeline does not
  otherwise mutate the data: `OTLP -> Vector -> OTLP` and `Datadog -> Vector -> Datadog` produce
  output that the OTLP or Datadog backend ingests as the same data as the original. This is
  effective equivalence once re-ingested, not byte-level equivalence; details that the backend
  does not observe (e.g. span order within a chunk, the specific chunk grouping a span ends up
  in when chunk-scoped metadata is preserved) may differ. The Datadog guarantee is also
  conditional on existing producer-side disjointness conventions; see "Datadog attribute
  partitions".

### Out of scope

- VRL function additions for trace-specific operations (e.g. `decode_trace_state`).
- New trace sources/sinks (Zipkin, Jaeger, etc.).
- APM stats computation semantics (already covered by RFC 9862).
- Zero-loss cross-format round-trip (`Datadog -> OTLP -> Datadog`,
  `OTLP -> Datadog -> OTLP`). Cross-format conversion is supported as a one-way operation;
  best-effort encoding under reserved keys is described in the mapping sections.

## Pain

- Transforms written against today's `TraceEvent` depend on the exact key layout the ingesting
  source produced. A remap that works for `datadog_agent` traces does not work for OTLP traces,
  even when the semantic intent is identical. This is the opposite of how `Metric` behaves and
  is the primary blocker to useful trace transforms.
- Cross-format routing (e.g. `opentelemetry` source -> `datadog_traces` sink) requires bespoke
  translation reading undocumented magic keys. Each new sink duplicates this work.
- `TraceEvent` currently corrupts numeric ID precision (`trace_id as i64` in both the
  `datadog_agent` source and the `datadog_traces` sink, see
  [#14687](https://github.com/vectordotdev/vector/issues/14687)). A typed model fixes this by
  construction.
- VRL programs authoring spans without typed events, links, or status can produce structurally
  invalid output that is only discovered at sink encoding time.

## Proposal

### User Experience

A `TraceEvent` carries one `Resource`, one `Scope`, one `ChunkContext`, and a `Vec<Span>`. VRL
accesses these directly:

```coffee
# Route by resource service.
if .resource.service == "checkout" { ... }

# Read a Datadog chunk-scoped tag (no-op for OTLP-sourced events).
.decision_maker = .chunk.tags."_dd.p.dm"

# Filter health-check spans across the whole event.
.spans = filter(.spans, |_, span| { span.name != "GET /health" })

# Mark slow DB spans as errors.
.spans = map_values(.spans, |span| {
    if span.span_type == "db" && span.duration > 1.0 {
        span.status.code = "error"
        span.status.message = "slow query"
    }
    span
})

# Read a semantic-convention attribute on the root span, falling back to a
# Datadog-native key.
.user_id = .spans[0].attributes."user.id" ?? .spans[0].attributes."usr.id"
```

Datadog's three span-level partitions (`meta`/`metrics`/`meta_struct`) are merged into
`Span.attributes`; the two resource-level tag scopes (`AgentPayload.tags`, `TracerPayload.tags`) are
preserved as `.resource.attributes."_dd.payload"` and `.resource.attributes."_dd.tracer"`; and
chunk-scoped state lives on `.chunk`. Each is specified under its own subsection below.

The `trace_to_log` transform is retained, but its output shape changes from a source-defined
key layout to a uniform, source-independent one; the migration guide provides a
field-by-field mapping.

### Implementation

#### `TraceEvent`

```rust
pub struct TraceEvent {
    resource: Resource,
    scope:    Scope,
    /// Datadog-only chunk-scoped state. Default-empty when the event is
    /// OTLP-sourced.
    chunk:    ChunkContext,
    /// Spans belonging to this resource/scope/chunk grouping.
    spans:    Vec<Span>,
    metadata: EventMetadata,
}
```

Each `TraceEvent` corresponds to one wire-level grouping:

- OTLP: one `ScopeSpans` (with its enclosing `ResourceSpans` providing `Resource`).
- Datadog: one `(TracerPayload, distinct Span.service, TraceChunk)` triple.

#### `Span`

```rust
pub struct Span {
    pub trace_id:       TraceId,
    pub span_id:        SpanId,
    pub parent_span_id: Option<SpanId>,
    pub trace_state:    TraceState,
    pub flags:          TraceFlags,

    pub name:           KeyString,
    pub kind:           SpanKind,

    pub start_time:     DateTime<Utc>,
    /// Span duration with nanosecond precision. The VRL surface exposes
    /// `.spans[i].duration` as float seconds.
    pub duration:       Duration,
    pub status:         SpanStatus,

    /// Datadog-native, no OTLP equivalent: human-readable identifier of the
    /// resource being traced (URL, handler, SQL statement).
    pub resource_name:  Option<KeyString>,

    /// Datadog-native, no OTLP equivalent: free-form span type
    /// (web, db, cache, http, ...).
    pub span_type:      Option<KeyString>,

    /// Per-span attribute map. On OTLP ingest this is `Span.attributes`
    /// verbatim; on Datadog ingest it is the union of the wire-level
    /// `meta`/`metrics`/`meta_struct` maps distinguished by `Value` variant
    /// (see "Datadog attribute partitions").
    pub attributes:     Attributes,

    pub events:         Vec<SpanEvent>,
    pub links:          Vec<SpanLink>,

    pub dropped_attributes_count: u32,
    pub dropped_events_count:     u32,
    pub dropped_links_count:      u32,
}
```

#### `Resource` and `Scope`

```rust
pub struct Resource {
    pub service:     Option<KeyString>,   // service.name
    pub environment: Option<KeyString>,   // deployment.environment
    pub host:        Option<KeyString>,   // host.name
    pub attributes:  Attributes,
    pub schema_url:  Option<KeyString>,
    pub dropped_attributes_count: u32,
}

pub struct Scope {
    pub name:       KeyString,
    pub version:    KeyString,
    pub attributes: Attributes,
    pub schema_url: Option<KeyString>,
    pub dropped_attributes_count: u32,
}
```

#### Identifiers

```rust
pub struct TraceId(NonZeroU128);
pub struct SpanId(NonZeroU64);

impl TraceId {
    /// Low 64 bits, used as Datadog's `trace_id`.
    pub fn low_u64(self)  -> u64 { self.0.get() as u64 }
    /// High 64 bits, stored in Datadog's `_dd.p.tid` meta tag.
    pub fn high_u64(self) -> u64 { (self.0.get() >> 64) as u64 }
}
```

Conversions to and from `u128`/`u64` and OTLP's 16/8-byte big-endian representations are provided
as cheap copies via `From` (when the source is statically non-zero) and `TryFrom` (otherwise).

#### Status, kind, chunk context

```rust
pub enum SpanKind { Unspecified, Internal, Server, Client, Producer, Consumer }

pub enum SpanStatus {
    Unset,
    Ok,
    Error(String),
}

/// Datadog `TraceChunk`-scoped state. Default-empty for OTLP-sourced events.
pub struct ChunkContext {
    pub priority: Option<SamplingPriority>,
    pub origin:   Option<KeyString>,
    pub dropped:  bool, /// `TraceChunk.droppedTrace`
    pub tags:     Attributes,
}

pub enum SamplingPriority {
    UserReject, // -1
    AutoReject, //  0
    AutoKeep,   //  1
    UserKeep,   //  2
    /// Out-of-range value. Datadog tracing libraries may uncommonly emit these.
    Other(i32),
}
```

#### `TraceFlags` and `TraceState`

`TraceFlags` is the W3C trace-flags bitfield. `SAMPLED` is the only bit defined today; sources
construct via `TraceFlags::from_bits_retain(byte)` and sinks read the raw value via
`flags.bits()`, so unknown bits round-trip unchanged.

```rust
bitflags::bitflags! {
    #[derive(Clone, Copy, Debug, Default, PartialEq, Eq, Hash)]
    pub struct TraceFlags: u8 {
        const SAMPLED = 0x01;
    }
}
```

`TraceState` stores the W3C `tracestate` header verbatim and exposes map-like accessors that
parse on demand. Sources copy the header in unchanged; sinks emit it unchanged unless a
transform mutated it.

```rust
#[derive(Clone, Default, Eq, PartialEq)]
pub struct TraceState(String);

impl TraceState {
    pub fn from_raw(s: impl Into<String>)          -> Self;
    pub fn as_str(&self)                           -> &str;
    pub fn is_empty(&self)                         -> bool;

    pub fn get(&self, key: &str)                   -> Option<&str>;
    pub fn insert(&mut self, key: &str, val: &str);
    pub fn remove(&mut self, key: &str)            -> bool;
    pub fn iter(&self)                             -> impl Iterator<Item = (&str, &str)> + '_;
}
```

`insert` rewrites the underlying string in-place, preserving entry order and inserting new
entries at the head, per the W3C spec.

#### Events and links

```rust
pub struct SpanEvent {
    pub name: KeyString,
    pub time: DateTime<Utc>,
    pub attributes: Attributes,
    pub dropped_attributes_count: u32,
}

pub struct SpanLink {
    pub trace_id:    TraceId,
    pub span_id:     SpanId,
    pub trace_state: TraceState,
    pub flags:       TraceFlags,
    pub attributes:  Attributes,
    pub dropped_attributes_count: u32,
}
```

#### `Attributes`

```rust
pub struct Attributes(ObjectMap);
```

A newtype over `ObjectMap` (re-exported from `vrl::value`). `Value` carries `Bytes`, `Float`,
`Integer`, `String`, `Boolean`, `Timestamp`, `Array`, and `Object` variants. Keys follow
OpenTelemetry semantic conventions; snake_case is preferred for new internally-generated keys.

Datadog attributes are always scalars on the wire, so the nested-value capability is unused on
the Datadog egress path; the Datadog sink stringifies any non-scalar value via JSON.

#### Datadog attribute partitions: convention versus invariant

Datadog spans carry attributes in three independent wire-level maps:

- `meta`: keys to UTF-8 strings.
- `metrics`: keys to IEEE-754 doubles.
- `meta_struct`: keys to opaque bytes (msgpack-encoded structured payloads).

The Datadog `Span` proto defines all three as separate `map<string, ...>` fields. The three
maps are merged into `Span.attributes` on ingest (`meta` -> `Value::String`,
`metrics` -> `Value::Float`, `meta_struct` -> `Value::Bytes`) and reconstructed by `Value`
variant on egress.

If a non-conforming producer emits a cross-partition collision (the same key in more than one
partition), the Datadog source resolves it deterministically (last-loaded partition wins, in
the order `meta` -> `metrics` -> `meta_struct`) and emits a `DatadogAttributeCollision`
internal event that increments `component_errors_total` and writes a rate-limited `warn!` log.

Datadog egress: the sink scans `Span.attributes` and partitions by `Value` variant.
`Value::Integer` is coerced to `f64` and routed to `metrics`. Variants with no native Datadog
partition (`Value::Boolean`, `Value::Timestamp`, `Value::Array`, `Value::Object`) are
stringified (JSON for composite variants) into `meta`. The result is one entry per key in
exactly one wire partition.

OTLP egress (cross-format): the merged entries flow through OTLP egress identically to
OTLP-sourced attributes; their `Value` variants are valid OTLP `AnyValue` shapes. Cross-format
round-trip through OTLP is out of scope; the OTLP source has no inverse mapping that would
re-partition these entries on a subsequent Datadog egress.

#### Datadog resource tag scopes

Datadog's `AgentPayload.tags` and `TracerPayload.tags` are preserved verbatim as two reserved
top-level entries in `Resource.attributes`. Each entry's value is an `Object` holding the
wire-level map:

| Wire scope            | `Resource.attributes` key | Value shape |
| --------------------- | ------------------------- | ----------- |
| `AgentPayload.tags`   | `_dd.payload`             | `Object`    |
| `TracerPayload.tags`  | `_dd.tracer`              | `Object`    |

VRL access:

```coffee
.agent_apm_mode  = .resource.attributes."_dd.payload"."_dd.apm_mode"
.tracer_apm_mode = .resource.attributes."_dd.tracer"."_dd.apm_mode"
```

The two keys live under the `_dd.*` namespace alongside other Datadog-internal keys
(`_dd.apm_mode`, `_dd.tags.container`, `_dd.tags.process`, `_dd.p.dm`, `_dd.p.tid`,
`_dd.error_tracking_*`, `_dd.otel.gateway`). OpenTelemetry semantic conventions do not emit
`_dd.*`-prefixed keys, so OTLP-sourced and transform-generated resource attributes cannot
collide with the reserved names.

OTLP egress (cross-format): the two scope objects have no OTLP-native home. The OTLP sink
emits them as best-effort `Resource.attributes` entries of `KvlistValue` type under the
reserved keys `datadog.tags.payload` and `datadog.tags.tracer`.

#### Datadog chunk context

Datadog `TraceChunk.priority`, `origin`, `droppedTrace`, and `tags` apply uniformly to every
span in the chunk. Each `TraceEvent` corresponds to exactly one chunk by construction (see
"Datadog mapping"), so these fields live on `TraceEvent.chunk` directly. OTLP-sourced events
carry a default-empty `ChunkContext` (no Datadog wire concept).

VRL access:

```coffee
.priority       = .chunk.priority
.origin         = .chunk.origin
.dropped        = .chunk.dropped
.decision_maker = .chunk.tags."_dd.p.dm"
```

OTLP egress (cross-format): the four chunk fields are emitted as best-effort `Span.attributes`
entries on every span of the event under the reserved keys `datadog.chunk.priority`,
`datadog.chunk.origin`, `datadog.chunk.dropped`, and `datadog.chunk.tags` (`KvlistValue`).

#### OTLP mapping

Each OTLP `ScopeSpans` is one `TraceEvent`. The containing `ResourceSpans.resource` populates
`TraceEvent.resource`; the `ScopeSpans.scope` populates `TraceEvent.scope`; the spans inside
populate `TraceEvent.spans`; `TraceEvent.chunk` is default-empty.

| OTLP                                                              | Internal                                        |
| ----------------------------------------------------------------- | ----------------------------------------------- |
| `ResourceSpans.resource.attributes["service.name"]`               | `Resource.service`                              |
| `ResourceSpans.resource.attributes["deployment.environment"]`     | `Resource.environment`                          |
| `ResourceSpans.resource.attributes["host.name"]`                  | `Resource.host`                                 |
| `ResourceSpans.resource.attributes` (others)                      | `Resource.attributes`                           |
| `ResourceSpans.schema_url`                                        | `Resource.schema_url`                           |
| `ScopeSpans.scope.*`                                              | `Scope.*`                                       |
| `Span.trace_id`, `Span.span_id`, `Span.parent_span_id`            | same (all-zero rejected per OTLP)               |
| `Span.trace_state`                                                | `Span.trace_state` (verbatim)                   |
| `Span.flags`                                                      | `Span.flags`                                    |
| `Span.name`, `Span.kind`                                          | `Span.name`, `Span.kind`                        |
| `Span.start_time_unix_nano`, `end_time_unix_nano` [^otlp-time]    | `Span.start_time`, `Span.duration` (ns-exact)   |
| `Span.attributes`                                                 | `Span.attributes`                               |
| `Span.events`, `Span.links`                                       | `Span.events`, `Span.links`                     |
| `Span.status.{code,message}`                                      | `Span.status.{code,message}`                    |
| `Span.dropped_*_count`                                            | `Span.dropped_*_count`                          |

[^otlp-time]: Computed as `end_time_unix_nano − start_time_unix_nano` on ingress; reconstructed as `start_time_unix_nano + duration` on egress. Both are integer nanoseconds; the round trip is bit-exact.

On OTLP egress, `TraceEvent`s sharing a `Resource` are gathered into one `ResourceSpans`; each event
becomes one `ScopeSpans`. Datadog-native fields (`Span.resource_name`, `Span.span_type`,
`TraceEvent.chunk.*`, `Resource.attributes."_dd.payload"`, `Resource.attributes."_dd.tracer"`) have
no OTLP-native home; the OTLP sink emits them best-effort under the reserved keys documented in
"Datadog resource tag scopes" and "Datadog chunk context".

`Span.duration` converts to `end_time_unix_nano = start_time_unix_nano + duration.as_nanos()`
on egress. Both quantities are integer nanoseconds in memory and on the wire; the round trip
is bit-exact, with no rounding.

OTLP defines several "field absent" / "field default-valued" pairs as semantically equivalent
at the spec level, in which case the model represents both forms as the default value and
the round-trip preserves spec-defined semantic equivalence even when the wire bytes differ:

- `ResourceSpans.resource` (proto comment: "If this field is not set then no resource info
  is known") and `ScopeSpans.scope` (proto comment: "Semantically when InstrumentationScope
  isn't set, it is equivalent with an empty instrumentation scope name (unknown)") -- absent
  on the wire is spec-equivalent to a default-valued message. The model carries
  `TraceEvent.resource` and `TraceEvent.scope` as values rather than `Option`, and egress
  emits the field unconditionally.
- `Span.status` (proto comment: "Semantically when Status isn't set, it means span's status
  code is unset, i.e. assume STATUS_CODE_UNSET (code = 0)") -- the model represents this as
  `SpanStatus::Unset` and egress emits the corresponding zero-coded `Status`.
- `Span.kind` -- `SPAN_KIND_UNSPECIFIED = 0` is the proto3 default; absent and zero-valued
  are byte-identical anyway.

#### Datadog mapping

An `AgentPayload` expands into one `TraceEvent` per
`(TracerPayload, distinct Span.service, TraceChunk)` triple. The grouping rules are:

- Each `TraceChunk` becomes one `TraceEvent`. A chunk whose spans use more than one
  `Span.service` is split into one event per distinct service; egress re-coalesces such
  events back into a single chunk (see below).
- The enclosing `TracerPayload`'s metadata (`hostname`, `env`, `containerID`, `languageName`,
  `tracerVersion`, etc.) populates the event's `Resource`. Per-span `Span.service` populates
  `Resource.service`.
- `AgentPayload.tags` and `TracerPayload.tags` populate `Resource.attributes."_dd.payload"`
  and `Resource.attributes."_dd.tracer"` respectively (see "Datadog resource tag scopes").
- `TraceChunk.{priority, origin, droppedTrace, tags}` populate `TraceEvent.chunk`.
- `Scope` is left default; Datadog has no scope concept.

| Datadog                                                       | Internal                                              |
| ------------------------------------------------------------- | ----------------------------------------------------- |
| `TracerPayload.hostname`                                      | `Resource.host`                                       |
| `TracerPayload.env`                                           | `Resource.environment`                                |
| `Span.service` (per span)                                     | `Resource.service` of the event holding the span      |
| `AgentPayload.tags`                                           | `Resource.attributes."_dd.payload"`                   |
| `TracerPayload.tags`                                          | `Resource.attributes."_dd.tracer"`                    |
| `TraceChunk.{priority, origin, droppedTrace, tags}`           | `TraceEvent.chunk`                                    |
| `TracerPayload` fields, `AgentPayload.agentVersion` [^dd-tp]  | `Resource.attributes` under defined keys (see note)   |
| `Span.traceID` (u64)                                          | `Span.trace_id.low_u64`                               |
| `Span.meta["_dd.p.tid"]` (hex u64) if present                 | `Span.trace_id.high_u64`                              |
| `Span.spanID`, `Span.parentID`                                | `Span.span_id`, `Span.parent_span_id`                 |
| `Span.name`                                                   | `Span.name`                                           |
| `Span.resource`                                               | `Span.resource_name`                                  |
| `Span.type`                                                   | `Span.span_type`                                      |
| `Span.start`, `Span.duration` (ns int64)                      | `Span.start_time`, `Span.duration` (ns-exact)         |
| `Span.error` and `Span.meta["error.message"]`                 | `Span.status` [^dd-err]                               |
| `Span.meta`                                                   | `Span.attributes` (`Value::String`)                   |
| `Span.metrics`                                                | `Span.attributes` (`Value::Float`)                    |
| `Span.meta_struct`                                            | `Span.attributes` (`Value::Bytes`)                    |

[^dd-tp]: `TracerPayload` fields mapped to `Resource.attributes` under semantic convention keys:
    `containerID`, `languageName`, `languageVersion`, `tracerVersion`, `runtimeID`, `appVersion`;
    plus `AgentPayload.agentVersion`.

[^dd-err]: `Span.error == 1` maps to `Error(meta["error.message"].cloned().unwrap_or_default())`,
    else `Unset`. The `error.*` meta entries also flow into `Span.attributes` per the meta merge
    rule, keeping `error.type`/`error.stack` accessible alongside the typed status.

The precise OpenTelemetry semantic convention keys for tracer/runtime/app/agent metadata in
`Resource.attributes`, and the OTLP-`kind`-to-Datadog-`Span.type` derivation used on egress
when `Span.span_type` is absent, follow the [Datadog Agent's OTLP ingest
mapping](https://github.com/DataDog/datadog-agent/blob/main/pkg/trace/api/otlp.go) and are
deferred to implementation PRs.

On Datadog egress, the sink:

- Sets each wire `Span.error = 1` if `Span.status` is `Error(_)`, else `0`.
- Re-partitions each `Span.attributes` into `meta`/`metrics`/`meta_struct` by `Value` variant
  per "Datadog attribute partitions" above.
- For spans where `Span.status = Error(message)` and `Span.attributes."error.message"` is
  absent, sets `meta["error.message"] = message` so OTLP-sourced spans (whose status was
  populated from `Span.status.message` rather than `meta`) emit Datadog-conventional error
  detail. Spans whose `Span.attributes."error.message"` is already set retain that value
  unchanged.
- Gathers events sharing a non-service `Resource` into one `TracerPayload`, with each span's
  `Span.service` reconstructed from its event's `Resource.service`.
- Within each `TracerPayload`, groups spans across events by `(ChunkContext, trace_id)` and
  emits one `TraceChunk` per group. A multi-service wire chunk that was split into multiple
  events on ingest re-coalesces into one chunk on egress; a non-conforming multi-trace chunk
  produces one egress chunk per `trace_id`. Both shapes are equivalent to the input as
  observed by the Datadog backend (see "Scope").
- Gathers `TracerPayload`s into one `AgentPayload`. Agent-payload-level fields
  (`agentVersion`, `targetTPS`, `errorTPS`, `tags`) come from Vector configuration plus
  `Resource.attributes."_dd.payload"`.

#### Retention of `TraceEvent` and `Event::Trace`

The `Event::Trace(TraceEvent)` variant on the outer `Event` enum is retained. Only the inner
representation changes:

```rust
pub enum Event {
    Log(LogEvent),
    Metric(Metric),
    Trace(TraceEvent),
}
```

#### Migration: coexistence of `LogEvent` and typed representations

During the migration, `TraceEvent` is an enum:

```rust
pub enum TraceEvent {
    /// Pre-migration source output: an untyped `LogEvent` whose key layout is
    /// defined by the producing source. `LogEvent` already carries its own
    /// `EventMetadata`, which is reused as-is.
    Legacy(LogEvent),
    /// Post-migration typed container.
    Typed {
        resource: Resource,
        scope:    Scope,
        chunk:    ChunkContext,
        spans:    Vec<Span>,
        metadata: EventMetadata,
    },
}
```

The end-state `struct TraceEvent { resource, scope, chunk, spans, metadata }` shown above is
reached by deleting the `Legacy` arm once every component has migrated; the `Typed` arm's
fields become the struct's fields verbatim.

Both accessor families coexist on `TraceEvent` and dispatch on the variant:

- `metadata()` / `metadata_mut()` and finalizer methods return the inner `LogEvent`'s
  metadata when `Legacy`, and the typed `metadata` field when `Typed`. Callers see no
  behaviour change.
- The existing untyped accessors (`get(path)`, `insert(path, value)`, `as_map()`, etc.)
  operate on the `Legacy` form only; on a `Typed` variant they return a deterministic error.
- The new typed accessors (`resource()`, `spans()`, `chunk()`, etc., plus their `_mut`
  variants) operate on the `Typed` form. On a `Legacy` variant they convert on demand by
  running the source-specific `Legacy -> Typed` shim attached to the producing component.
- Explicit `to_typed(&mut self)` rewrites a `Legacy` variant in place into `Typed` via the
  same shim. There is no symmetric `to_legacy`.

Per-component shims are unidirectional (`Legacy -> Typed` only). The `datadog_agent` source
ships with a shim that knows the source's `LogEvent` key layout and produces a typed
container; the OTLP source ships with the equivalent shim for its layout. Trace-aware sinks
and transforms accept `Typed` input directly and convert `Legacy` input on demand via the
source-specific shim of the producing component.

After every source, sink, and transform has been migrated, the `Legacy` variant and the shims
are deleted, leaving only the typed struct.

## Rationale

### Architectural choices

- The container shape mirrors the wire-level batching of both OTLP and Datadog: each
  `TraceEvent` is one `(resource, scope, chunk)` grouping. Source ingest and sink egress are
  pure mechanical translations between the wire shape and the container.
- Sharing `Resource`/`Scope`/`ChunkContext` across sibling spans is structural (a struct
  field), not pointer-based (an `Arc`). Disk-buffer serialization preserves the sharing for
  free; no `Arc` reconstruction or read-side interning is needed.
- The shape a user sees is the same whether the event arrived via OTLP or from the Datadog
  Agent. Source-native attribute maps are preserved on the appropriate typed level; nothing is
  copied into a parallel "extensions" map. Transforms can be written once and applied
  uniformly.
- Typed fields let transforms be written once. `Metric` demonstrates this model in Vector's
  architecture; extending to traces gives them parity and unblocks RFC 11851.
- Keeping the outer `Event::Trace(TraceEvent)` variant unchanged minimises churn at every
  call site that dispatches on `Event` (topology, buffers, finalizers, etc.); only the inner
  representation changes.

### Per-type design choices

- `Resource` promotes only the three semantic-convention fields both wire formats agree on
  (`service.name`, `deployment.environment`, `host.name`); other resource attributes stay in
  `Resource.attributes` under standard semantic convention keys. Promoting more would force Vector
  to track upstream semantic convention evolution or ossify a stale subset; promoting fewer would
  force every cross-format transform to read source-specific keys for common metadata.
- Encoding `TraceId`/`SpanId` non-zero invariants in the type itself eliminates a class of
  malformed values by construction. OTLP defines all-zero IDs as invalid, and Datadog uses
  zero only as the "no parent" sentinel (already represented as `None`). Using unsigned
  integer types fixes the existing `i64`-coercion precision bug
  ([#14687](https://github.com/vectordotdev/vector/issues/14687)).
- `TraceFlags` preserves unknown bits verbatim via `bitflags::from_bits_retain` so future
  W3C Trace Context additions (e.g. the Level 2 `random` flag) round-trip without changing
  the type or its serialization.
- `Span.duration` is stored as `std::time::Duration` (nanosecond integer), matching the wire
  domain of both OTLP (`fixed64` nanoseconds) and Datadog (`int64` nanoseconds) exactly. The
  VRL surface exposes `.spans[i].duration` as float seconds for ergonomic comparisons.
- The `Attributes(ObjectMap)` newtype delegates typed storage to VRL's existing `Value`,
  which already covers the full set of types either wire format can transmit. The newtype
  exists so future invariants (key validation, size bounds) can be added without requiring a
  migration. Nested `Array`/`Object` values are supported because OTLP's `AnyValue` is
  recursive and the OpenTelemetry semantic conventions define attributes whose values are
  arrays of strings or structured records; flat-scalar storage would require lossy flattening
  on ingest and reconstruction on egress.

### Datadog-specific design choices

- The `meta`/`metrics`/`meta_struct` merge into `Span.attributes` relies on a producer-side
  disjointness convention rather than a wire-format invariant. The Datadog `Span` proto does
  not constrain keysets across the three maps, but every examined Datadog SDK and the trace
  agent maintain disjointness by construction. The model treats this as a contract the
  Datadog source asserts; the merge is lossless when each wire key has a single value type
  (regardless of which partition carried it) and lossy only when the same key holds multiple
  value types simultaneously, which no examined producer emits. If the convention ever
  ceases to hold for production traffic, the contained fallbacks (`Value::Bag` variant or a
  separate `Span.datadog_attributes` field) are documented under "Alternatives".
- The two resource-level tag scopes (`AgentPayload.tags`, `TracerPayload.tags`) are kept as
  separate sub-objects inside `Resource.attributes` rather than merged because the two
  scopes collide on known keys. The Datadog Agent's trace writer
  ([`pkg/trace/writer/trace.go`](https://github.com/DataDog/datadog-agent/blob/main/pkg/trace/writer/trace.go))
  writes `_dd.apm_mode` into `AgentPayload.tags` from its own configuration, and the Agent's
  processing pipeline ([`pkg/trace/agent/agent.go`](https://github.com/DataDog/datadog-agent/blob/main/pkg/trace/agent/agent.go))
  writes the same key into `TracerPayload.tags` from a span's `Meta`. The two values are
  semantically distinct (Agent's claimed mode versus tracer-reported mode) and appear in the
  same payload; merging would lose data, while preserving them as separate sub-objects is
  structural and convention-independent.
- The Datadog egress chunk-grouping rule `(ChunkContext, trace_id)` relies on a producer-side
  convention parallel to the `meta`/`metrics`/`meta_struct` story: the `TraceChunk` proto
  describes a chunk as "a list of spans with the same trace ID", and Datadog producers honor
  this by construction. For the conforming case, multi-service chunks split on ingest re-
  coalesce into one egress chunk and single-service chunks pass through unchanged; for a
  non-conforming multi-trace chunk, egress emits one chunk per `trace_id`. Both shapes are
  effectively equivalent at the Datadog backend, since chunk grouping is an ingestion-time
  transport detail rather than a semantic primitive.

### Migration approach

- The migration uses an `enum TraceEvent { Legacy, Typed }` so each trace source, sink, and
  transform can migrate in its own PR while the rest of the system continues to operate
  against the representation it expects. See "Wholesale migration" under Alternatives for
  why a single atomic replacement was rejected.
- Per-component shims convert `Legacy -> Typed` only, never the reverse: a `Typed` event has
  no source provenance on which to base a back-conversion to a source-specific `LogEvent`
  shape. This forces the migration sequencing in the Plan of Attack -- trace-aware
  consumers (sinks, transforms, VRL programs) must accept `Typed` input before any source
  flips to emitting `Typed` natively. Untyped accessors (`get(path)`, `as_map()`, etc.) on a
  `Typed` variant return a deterministic error rather than synthesizing a `Legacy` shape,
  so cases where a non-migrated consumer encounters a `Typed` event surface as test
  failures rather than silent data corruption.

## Drawbacks

- Breaking change for VRL configurations against today's `TraceEvent` key layout. Users must
  migrate to typed paths.
- The `trace_to_log` transform's output also changes; downstream VRL programs against its
  output must update.
- Topology granularity is coarser than per-span: each event carries up to a chunk's worth of
  spans (typically tens to hundreds, larger in deep call trees). Buffer-size limits expressed
  in events bound span counts less directly than the previous `LogEvent`-per-span design.
- Per-span operations (filter, sample, mutate one span) require VRL iteration over `.spans`
  rather than per-event treatment. A topology-level expand-on-input/collapse-on-output shim
  could let single-span transforms operate unchanged; that mechanism is deferred to
  implementation.
- The Datadog round-trip guarantee depends on a producer-side disjointness convention for
  `meta`/`metrics`/`meta_struct`. Every Datadog SDK examined keeps the partitions disjoint by
  construction; the model treats the convention as a contract the source asserts.
- Every trace source and sink must be rewritten to produce/consume the typed container. The
  Plan of Attack sequences this so each component migrates independently, but it is non-trivial
  work.

## Prior Art

- [OTLP traces protocol](https://github.com/open-telemetry/opentelemetry-proto/blob/main/opentelemetry/proto/trace/v1/trace.proto)
  -- the primary shape this RFC adopts. The container `TraceEvent` is structurally one
  `ScopeSpans` plus its `Resource` and the Datadog-only `ChunkContext`.
- [Datadog APM agent-to-backend
  protobuf](https://github.com/DataDog/datadog-agent/tree/main/pkg/proto/datadog/trace) -- the
  second native format Vector targets.
- [Datadog Agent OTLP
  ingest](https://github.com/DataDog/datadog-agent/blob/main/pkg/trace/api/otlp.go) --
  reference implementation for the OTLP-to-Datadog field mappings adopted here.
- [2024-03-22-20170 draft](https://github.com/hdost/vector/blob/add-trace-data-model/rfcs/2024-03-22-20170-trace-data-model.md)
  -- an earlier draft that modelled the event as a `ResourceSpans` (batch of multiple
  scope/spans groupings). The current RFC adopts a similar container shape but at finer
  granularity (one event per `ScopeSpans` rather than per `ResourceSpans`).

## Alternatives

### One span per event (`TraceEvent { span: Span, metadata }`)

An earlier draft of this RFC carried a single span per event. This, however, requires the
`Resource`, `Scope`, and `ChunkContext` to either be duplicated for each span or to be shared via
`Arc`. Rejected because Vector's disk buffers serialize each event as one record: `Arc` sharing
collapses on serialization, every span on disk gets a full inline copy of resource/scope/chunk, and
on read every span gets an independent allocation, thus costing Vector both extra costs in
serialization and deserialization as well as the associated memory expansion and sink-level
reassembly mechanics. The container shape eliminates the inflation by aligning the event boundary
with the wire-batching boundary, so the shared context appears once per grouping on disk and in
memory regardless of how the path is buffered, and `Arc` machinery is not needed.

The per-span shape did offer two ergonomic advantages: the internal memory usage of a single span
(with the resources shared) is more consistent and granular, and per-span operations (filter,
sample, mutate one span) work directly without iteration. Recovering the latter in the container
shape is a topology-level shim concern, deferred to implementation.

### Parallel `Event` variants for new and old trace formats

Introduce `Event::NewTrace` alongside `Event::Trace`, leaving the existing `TraceEvent` untouched.
Rejected because it splits trace handling across two `Event` variants for the duration of the
migration, forcing every topology-level dispatch site to handle both. The tagged-inner approach
contains the duality inside `TraceEvent`, leaving `Event::Trace` as the single dispatch arm.

### Discriminated union (`Trace::{Otel, Datadog}` or `Span::{Otel, Datadog})

Carry each format as-is and dispatch at every consumer. Rejected because it directly inverts the
stated pain -- every transform and every cross-format sink would handle two shapes with the
possibility of more later. This is effectively the status quo over `LogEvent` just with predefined
fields.

### Separate `Span.datadog_attributes` field preserving the three wire partitions verbatim

Carry a `DatadogAttributes { meta, metrics, meta_struct }` field on `Span` alongside the
canonical `attributes`, populated only on Datadog ingest. This represents the wire format
exactly and preserves any cross-partition collision. Rejected because it splits the attribute
surface in two, forces every attribute-aware component to handle both, and is paid against a
collision case no examined Datadog SDK or agent emits. Listed as the contained mechanical
fallback if the producer-side disjointness convention ever ceases to hold for production
traffic; the change is local to `Span`, the Datadog source, the Datadog sink, and a unified
read helper, with no impact on the OTLP side.

### `Value::Bag` variant for cross-partition collisions

Add a `Value::Bag(SmallVec<[Value; 2]>)` variant carrying multiple values per key. Rejected
for the same reason as the separate-field alternative; further, more intrusive because `Value`
is shared across `LogEvent`, `Metric`, and `Span`.

### Namespace-prefixed unified map for span partitions

Encode Datadog's three span-attribute partitions inside `Span.attributes` itself by prefixing
each key with its partition name (`dd.meta.<k>`, `dd.metrics.<k>`, `dd.meta_struct.<k>`).
Rejected because the prefixes leak Datadog-specific encoding into every transform regardless
of source: an OTLP-only pipeline has to know about the namespace to avoid colliding with it,
and an OTLP-sourced attribute that happens to use a `dd.meta.*` key is silently misclassified
on egress. The `Value`-variant routing achieves the same egress mapping without imposing any
naming constraint.

### Bare top-level resource scope keys

Use `payload` and `tracer` as the two reserved top-level keys in `Resource.attributes`
instead of the namespaced `_dd.payload` / `_dd.tracer` adopted in the proposal. The contents
are identical in both forms. Rejected because the bare names are plausible attribute keys
that legitimate OTLP-sourced or transform-generated resource attributes can already use:
OpenTelemetry semantic conventions are uniformly dotted, but user-set resource attributes
(via `OTEL_RESOURCE_ATTRIBUTES=payload=...`), transform-generated attributes
(`.resource.attributes.payload = ...`), and future OpenTelemetry additions are not bound by
that convention. A collision under the bare-keys design is silent on Datadog egress: the
sink would either misclassify a legitimate user attribute as Datadog wire data, or drop it
as ill-typed. The namespaced form eliminates the collision class entirely because no
non-Datadog-internal source emits `_dd.*`-prefixed keys.

### Single merged `attributes` map with richer typed fields

Promote additional concepts (service, env, host *and* all semantic-convention equivalents) to typed
fields. Rejected because the semantic convention space is large and evolving; fixing it in typed
fields either forces Vector to track upstream releases or ossifies a stale subset. The proposal
types only the three resource fields both formats agree on; the rest stay in source-native
attributes where users already expect them.

### Timing as `start_time` + `end_time` (OTLP-native)

OTLP stores `start_time_unix_nano` and `end_time_unix_nano` as two independent `fixed64`s.
Datadog stores `start` plus `duration`. The driving factor for adopting `start + duration` is
that `duration` is more useful than `end_time` in realistic transforms (filtering slow spans,
computing percentiles, classifying long-running requests), so the chosen representation also
matches transform access patterns.

### Duration as `f64` seconds

Storing `duration` as `f64` was considered for VRL ergonomics. Rejected because both OTLP
(`fixed64` nanoseconds) and Datadog (`int64` nanoseconds) carry duration as integer
nanoseconds, and `f64`'s 53-bit mantissa cannot exactly represent every integer nanosecond
beyond `2^53 ns` (about 104 days). Storing `duration` as `std::time::Duration` preserves the
wire domain exactly. The VRL surface exposes float seconds at the boundary; a complementary
integer-nanosecond view (`.spans[i].duration_nanos`) is documented under "Future Improvements".

### `ChunkContext.priority` as a raw `i32`

Datadog's wire representation is a signed integer with four well-known values
(`UserReject = -1`, `AutoReject = 0`, `AutoKeep = 1`, `UserKeep = 2`). Storing the raw `i32`
directly is simpler. Rejected because transforms that condition on priority then have to
compare against magic numbers, and there is no way to surface "this is a non-standard value"
to the user. A strict enum with an `Other(i32)` escape hatch keeps typed ergonomics for the
common path while preserving any out-of-range value.

### `TraceFlags` via `enumflags2`

[`enumflags2`](https://crates.io/crates/enumflags2) was considered as the bitfield generator
for `TraceFlags`. Rejected because `enumflags2` rejects undefined bits at construction time,
which would silently lose forward-compatibility data on the W3C trace-flags wire byte (e.g. the
Trace Context Level 2 `random` flag once defined would be discarded when read by an unaware
Vector build). [`bitflags`](https://crates.io/crates/bitflags) supports `from_bits_retain`,
which preserves the full byte intact, so the same Vector build round-trips spans with
not-yet-defined flag bits without modification.

### Parsed `TraceState`

Storing `TraceState` as `IndexMap<KeyString, KeyString>` would let transforms operate on
entries without an accessor layer. Rejected because every source and sink would have to invoke
the parser/serializer even for pure-relay pipelines, and because the W3C-imposed bounds (32
entries, 512 bytes total) and typical real-world headers (a single short entry) mean per-entry
allocation costs more than re-parsing the raw header per accessor call.

### Wholesale migration

Replace `TraceEvent(LogEvent)` with the typed container in one PR. Rejected because the resulting PR
would touch every trace source, every trace sink, the APM stats aggregator, every trace-aware
transform, and a large body of tests simultaneously. The chosen `enum TraceEvent { Legacy, Typed }`
coexistence design lets each component migrate in its own PR, subject to a partial-order
constraint that consumers migrate before producers (see "Plan Of Attack").

### Feature-flagged switch

Gate the new representation behind a Cargo feature or runtime flag until all components are
migrated, then flip the default. Rejected because feature combinations proliferate quickly
across every trace source/sink and VRL, and because a runtime flag would require duplicate
code paths in performance-sensitive components.

## Outstanding Questions

- N/A.

## Plan Of Attack

Each step below is intended to land as an independent PR. The
`enum TraceEvent { Legacy, Typed }` coexistence is what makes the sequence possible. The
sequencing rule is for trace-aware consumers (sinks, transforms, VRL programs) to migrate to
`Typed`-native input before any source flips to emitting `Typed` natively, because per-component
shims are unidirectional (`Legacy -> Typed` only) and a `Typed` event has no source provenance
on which to base a `Typed -> Legacy` conversion.

- [ ] Convert `TraceEvent` to the migration enum: `Legacy(LogEvent)` and
  `Typed { resource, scope, chunk, spans, metadata }`, demonstrating the new structure and the
  supporting types. Every component still produces and consumes `Legacy`; nothing functionally
  changes. All accessor methods dispatch on the variant. Add the typed accessor API and a default
  `Legacy -> Typed` shim that errors if invoked before any source-specific shim has been
  registered, so accidental mixed access is loud.
- [ ] Add VRL typed-path support for `.resource.*`, `.scope.*`, `.chunk.*`, and `.spans[*].*`
  on `VrlTarget`. Untyped VRL paths against `Typed` events return a deterministic error.
- [ ] Migration guide for users:
  - field-by-key VRL programs against the old `TraceEvent` (must move to typed paths;
    legacy paths break against `Typed` events);
  - field-by-key VRL programs against the old `trace_to_log` output (must move to the
    new uniform layout).
- [ ] OTLP `Legacy -> Typed` shim and `Typed -> OTLP-wire` conversion in
  `lib/opentelemetry-proto`. The shim is registered by the `opentelemetry` source for events it
  produces in `Legacy` form; the `Typed -> wire` conversion is what the eventual native source
  emission and any OTLP sink will use.
- [ ] Datadog `Legacy -> Typed` shim and `Typed <-> Datadog-wire` conversions in
  `src/sources/datadog_agent/traces.rs` and `src/sinks/datadog/traces/`, similarly registered by
  the `datadog_agent` source for its `Legacy` events.
- [ ] Property-based round-trip unit tests for `OTLP -> Vector -> OTLP` and
  `Datadog -> Vector -> Datadog`, asserting effective equivalence (per the "Scope" definition)
  against generated payloads. The Datadog suite must cover the multi-service single-trace case
  (split-on-ingest, merge-on-egress) and a non-conforming multi-trace chunk.
- [ ] Migrate the `datadog_traces` sink to consume `Typed` natively (converting `Legacy`
  inputs via the shim); update APM stats aggregation to read typed fields.
- [ ] Migrate the `sample` transform (and tests) to typed access.
- [ ] Migrate the `trace_to_log` transform to operate on `Typed` (converting `Legacy` inputs via
  the shim) and emit a uniform, source-independent `LogEvent` layout. Document the new key
  layout.
- [ ] Migrate the `opentelemetry` source to produce `Typed` natively. By this point every
  trace-aware downstream component is `Typed`-capable.
- [ ] Migrate the `datadog_agent` source to produce `Typed` natively.
- [ ] Collapse the `TraceEvent` enum to a struct with only the typed variant's fields. Remove
  the per-component shims.

## Future Improvements

- Topology-level per-span shim: a transform mode that fans out a `TraceEvent` into per-span events,
  runs a downstream transform once per span, and collapses results back into the container. Lets
  single-span transforms be authored without explicit iteration while keeping the wire-aligned event
  shape as the source of truth.
- VRL helpers for trace-state parsing/encoding: `parse_trace_state`, `encode_trace_state`,
  `merge_span_attributes`, `decode_otlp_span`, `decode_datadog_span`.
- Lossless integer-nanosecond view for span duration: `.spans[i].duration` is exposed as float
  seconds, exact for any duration under `2^53 ns` (about 104 days). Workloads needing access to
  durations beyond that limit can have a complementary `.spans[i].duration_nanos` view added without
  affecting the underlying data model.
- Link-based routing: a trace-aware router transform that emits to different sinks based on
  `SpanLink` targets.
- Stateful trace-aggregator transforms: tail-based sampling, per-trace APM-stats aggregation, and
  similar trace-scoped operations expressed as transforms over the wire-aligned container shape.
