# Provider-Neutral Python API Client — Architecture and Documentation Blueprint

A reusable blueprint for designing a **robust, typed, synchronous/asynchronous Python client** without requiring a particular model, API vendor, cloud platform, or hosted product. It preserves the structure and technical depth of a production SDK README while separating portable design rules from provider adapters.

**Status:** documentation and interface-design reference. These four supplied Markdown files do **not** contain an installable SDK or prove that any code shown here has been implemented. Examples using `your_sdk` and placeholder names are **illustrative contracts**, not executable imports until a real package is built.

**Attribution:** The organization and several engineering concerns were inspired by the [OpenAI Python SDK](https://github.com/openai/openai-python) and its [documentation](https://github.com/openai/openai-python/blob/main/README.md). This is an independent adaptation, not an official OpenAI release or an assertion of feature parity. Preserve the original copyright and license conditions wherever code or other licensed material is reused. The historical changelogs supplied alongside this file belong to the upstream repository, not this project.

The intended separation of concerns is: **configuration → authentication → transport → API operations → typed parsing**, with optional adapters for provider-specific conventions. Observability, security, cancellation, validation, and compatibility span every layer.

## Documentation

Use this README as the public-facing design map. A working repository should eventually link to its **own** API reference, schema, migration notes, examples, threat model, release policy, and contribution guide. Do not point users to upstream documentation as if it documented independent implementations.

For each implemented feature, document its supported request/response contract, authentication, errors, retries, timeout/cancellation behavior, and both synchronous and asynchronous use where applicable. Label experimental or unimplemented interfaces explicitly.

## Installation

Choose a package/distribution name only when the actual project exists and the name is available. Publish verified wheels/sdists with a declared Python compatibility range, signed or traceable artifacts where available, and a reproducible dependency lock for development.

```sh
# Placeholder only; replace after the package exists and has been published.
python -m pip install your-package-name
```

Do not install a third-party package merely because its name resembles the placeholder. State optional transport, signing, and cloud-auth dependencies separately from the minimal core.

## Usage

A proposed API surface is shown below. It is **not** a claim that the `your_sdk` module exists:

```python
import os
from your_sdk import Client  # illustrative interface, not a published import

with Client(
    base_url=os.environ["SERVICE_API_BASE_URL"],
    token=os.environ.get("SERVICE_API_TOKEN"),
    timeout=30.0,
) as client:
    result = client.request("GET", "/resources", params={"limit": 10})
    print(result)
```

**Composition rules:** accept the endpoint as configuration; validate scheme/origin; make auth replaceable; make retries explicit and safe; expose request IDs; avoid silently changing JSON shapes; separate per-request overrides from immutable client settings; close connections deterministically. Resource helpers can wrap the request layer without obscuring underlying HTTP semantics.

Maintain a compatibility contract for typed request/response models; never imply every provider implements the same resource names, parameters, or streaming formats. Use capability detection or explicit adapter configuration.

### Workload Identity Authentication

A portable authentication interface obtains credentials at request time without tying the core client to a single identity provider. It should support refresh, expiration, scope/audience binding, and safe concurrency. Never serialize credentials into logs or send them to unapproved origins.

For example, a token-provider **contract** can be expressed without a proprietary SDK:

```python
from collections.abc import Callable

# An adapter may receive a provider returning short-lived access tokens.
TokenProvider = Callable[[], str]
```

Adapters must define who owns refresh, caching, cancellation, and retry behavior. A token suitable for one audience or origin may be invalid for another.

#### Service-account tokens

In orchestrated environments, read a projected token from a configured, protected file only when the environment supports it. Keep file permissions restrictive; bind the audience to the destination; handle token rotation and reject stale or empty values. Do not hardcode orchestrator paths in the transport layer.

#### Managed workload identity

For managed cloud workloads, delegate token acquisition to a pluggable credential provider. Do not assume a specific cloud metadata endpoint, tenant, environment variable, or CLI is available. Validate the requested audience and avoid falling back silently to user credentials.

#### Identity tokens from metadata endpoints

Metadata-backed credentials can be useful but also create SSRF and credential-exfiltration risks. Allow only well-defined metadata destinations, never send arbitrary user-provided URLs to metadata services, and document platform-specific protections in the corresponding adapter.

#### Custom subject token provider

Accept a caller-supplied callable or credential object with an explicit lifetime policy. Make its exception behavior deterministic; refresh before expiry without stampeding requests; isolate state across clients. An adapter must not share tokens across unrelated scopes or configured services.

#### X.509 workload identity (mutual TLS)

Where supported, bind client certificates and trust roots to a dedicated TLS context and restricted origin. Keep private keys out of the package and logs, disable redirect forwarding of credentials, verify hostnames, support rotation by replacing the connection pool, and close obsolete clients safely. Mutual TLS and bearer-token authentication are distinct protocols and must not be conflated.

### Vision

When an endpoint accepts images or other binary inputs, model their transport explicitly: URL references, byte content, content type, size policy, and ownership of streams. Never assert that every backend supports image analysis. Avoid decoding and re-encoding media unnecessarily; handle multipart uploads and base64 only when the target contract requires them.

```python
from pathlib import Path

image_bytes = Path("path/to/image.png").read_bytes()
content_type = "image/png"
# Pass bytes and content_type through an adapter that actually supports images.
```

Reject unsafe file paths, validate caller-declared media types when appropriate, and apply protocol-defined limits—not arbitrary global caps.

## Async usage

Expose an asynchronous interface only when the underlying transport and adapters genuinely support it. Sync and async APIs should share validation rules, request/response models, retry semantics, error types, and default policies, while using separate lifecycle management.

```python
import asyncio
import os
from your_sdk import AsyncClient  # illustrative interface

async def main():
    async with AsyncClient(base_url=os.environ["SERVICE_API_BASE_URL"]) as client:
        return await client.request("GET", "/resources")

# asyncio.run(main())  # enable after a real AsyncClient is implemented
```

Cancellation must close pending streams and release pool resources promptly; do not swallow cancellation or spawn untracked background work.

### With an optional async transport

Keep the HTTP adapter behind a small protocol so an implementation can choose a maintained asynchronous HTTP library without exposing that choice in the core public API. Verify connection limits, timeout layering, TLS policy, proxy routing, cancellation, decompression, and resource cleanup for each supported transport.

### HTTP transport migration

A transport swap is a potentially breaking change even when high-level methods stay stable. Compare redirects, exception mapping, timeout defaults, HTTP/2 behavior, proxy authentication, streaming interfaces, TLS configuration, and connection pooling. Supply a migration guide and tests instead of claiming interchangeable semantics.

## Streaming responses

Support streaming only where the endpoint explicitly defines a streaming protocol (such as SSE, JSON Lines, or framed binary messages). Parse incrementally across arbitrary chunk boundaries; do not assume a chunk equals one event. Provide sync and async iterators where implemented and close streams on exhaustion, errors, or cancellation.

```python
# Illustrative only; real event types must come from a documented endpoint.
with client.stream("GET", "/events") as events:
    for event in events:
        handle(event)
```

Apply bounded buffering, backpressure, heartbeat handling, and an explicit termination/error model. Avoid retrying after partial irreversible stream side effects without a documented resume mechanism.

## Realtime API

Realtime transports—WebSocket, SSE, or another duplex protocol—need their own connection, state, authentication, reconnection, and event-ordering policies. A provider-neutral core should not hardcode event names from a particular service. Document message acknowledgments, progress, idle timeouts, interruption, deduplication, and whether resumption is supported.

Never infer success from a closed socket alone; distinguish complete, cancelled, interrupted, and failed sessions. Authenticate new connections explicitly and restrict credential-bearing redirects.

### Realtime error handling

Define a typed error envelope with an error code, human-readable message, optional request/event identifier, retryability, and redacted diagnostics. Some protocols report errors as events without closing the connection; applications must handle both event-level failures and transport exceptions. Avoid unbounded automatic reconnect loops.

## Using types

Generate types from the **chosen project's** API schema when one exists, or maintain carefully reviewed handwritten models. Distinguish required, optional, nullable, unknown, and deprecated fields. Preserve unknown response fields where forward compatibility requires it; avoid accepting silently invalid request values.

Provide typed request parameters, response objects, discriminator unions, and round-trip serialization tests. Schema generation should be reproducible; generated files must not be hand-edited without ownership documentation.

## Pagination

Represent continuation as opaque provider data. Do not assume page numbers, offset keys, cursor directions, or stable ordering. Support manual pagination and a convenience iterator without making extra requests unexpectedly in performance-sensitive code.

```python
# Design sketch; the adapter defines the actual token field.
page = client.list_resources(limit=20)
for item in page.items:
    process(item)
while page.next_cursor is not None:
    page = client.list_resources(cursor=page.next_cursor, limit=20)
    for item in page.items:
        process(item)
```

## Nested params

Serialize nested dictionaries and typed request models according to the service contract. Preserve distinction between omitted and explicit `null`; do not flatten nested structures opportunistically. Encode files, arrays, and query parameters consistently, with tests for escaping and repeated values.

## File uploads

Accept file paths, bytes, or file-like objects **if supported by the actual endpoint**. Define who owns handles, whether uploads are rewindable for retries, how MIME types and filenames are inferred, and how multipart boundaries are generated. Never read an unbounded stream into memory merely to calculate its size unless the contract requires it.

```python
from pathlib import Path

file_path = Path("path/to/input.dat")
# The real adapter decides whether to stream, chunk, or use multipart encoding.
```

## Webhook Verification

Verify authenticity **before** parsing or acting on user-controlled payloads. Specify which bytes are signed, canonicalization rules, signed headers, allowed clock skew, and replay protection. Do not infer that all services use the same signing algorithm or header names.

### Parsing webhook payloads

A safe handler first enforces a protocol-defined size policy, captures the exact raw body, checks signatures/timestamps, then deserializes with schema validation. Authentication and JSON syntax are different checks; valid JSON is not evidence of a trusted sender.

### Verifying webhook payloads directly

An HMAC-SHA256 recipe for a protocol that explicitly specifies HMAC-SHA256 might use:

```python
import hashlib
import hmac

def verify_hmac_sha256(raw_body: bytes, signature_hex: str, secret: bytes) -> bool:
    expected = hmac.new(secret, raw_body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, signature_hex)
```

This is a **component example**, not a complete webhook verification protocol: it does not validate signature framing, timestamps, key IDs, or replay windows. Follow the sending service's documented scheme and test exact byte handling.

## Handling errors

Map failures to a stable, documented hierarchy without losing the original cause. Preserve HTTP status, provider error code if available, retry hint, request ID, and a **sanitized** diagnostic message. Avoid placing entire response bodies, authorization headers, or personal data into exceptions by default.

| Failure class | Expected handling |
| --- | --- |
| Connection / DNS / TLS | Classify and attach safe cause; retry only when appropriate |
| Timeout / cancellation | Differentiate caller cancellation from network timeout |
| Rate limit / unavailable | Respect bounded `Retry-After`, backoff, and idempotency |
| Authentication / permission | Fail without leaking secrets or blindly retrying |
| Validation / unexpected schema | Surface actionable details without silently coercing data |
| Streaming / partial response | Report incomplete output and preserve cleanup guarantees |

## Request IDs

Expose a sanitized request or correlation ID when supplied by the server or transport. Accept caller-provided trace context only under documented rules and avoid leaking it between independent sessions. Correlate requests and errors without logging raw credentials or user content.

## Retries

Use a bounded retry budget, exponential backoff with jitter, and server retry hints only when safe. Default retryability should depend on transport failure, HTTP status, body replayability, and idempotency—not merely on an exception name. Never duplicate non-idempotent actions unless the endpoint provides an idempotency key or an explicit safety guarantee.

Disclose default attempts, maximum delay, per-request overrides, and the relationship between retry delays and total deadlines.

## Timeouts

Define connect, read, write, pool-acquisition, and total-operation timeouts where supported. A total deadline must include retries and backoff. Offer per-client and per-request overrides with validation, and keep cancellation usable for sync and async transports alike.

## Advanced

Advanced configuration must remain opt-in. Avoid exposing internal transport objects as permanent compatibility promises unless they are deliberately part of the public interface.

### Logging

Provide structured, opt-in logs for endpoint identity, method, duration, status, retry count, and request ID. Redact headers, query parameters, bodies, cookies, and credentials by default. Keep logging behavior deterministic across sync and async clients, and document telemetry opt-in/out.

### How to tell whether `None` means `null` or missing

Use an explicit sentinel for an omitted field and reserve `None` for JSON `null` when the schema differentiates them. Keep the distinction through parse, validation, and serialization. Do not auto-fill absent data with fabricated defaults.

```python
UNSET = object()
value = UNSET  # omitted; not the same as JSON null (None)
```

### Accessing raw response data (e.g. headers)

Expose raw status, selected headers, final URL, timing, and response bytes through an explicit lower-level escape hatch. Make it clear when the body is buffered and which operations consume a stream. Preserve security filtering even in raw and debug modes.

#### `.with_streaming_response`

If a streaming-response convenience interface is implemented, prefer context management to guarantee closure. Do not require callers to rely on garbage collection; make body-reading methods explicit and detect use-after-close conditions.

### Making custom/undocumented requests

Permit optional custom endpoints only if doing so is compatible with the project's security and stability model. Generic request helpers should still enforce the configured origin, TLS, authentication boundaries, timeouts, and error filtering.

#### Undocumented endpoints

Treat unsupported paths as experimental: do not promise typing, response schemas, or compatibility without a contract. Never allow a caller-provided path to silently switch hosts or bypass security controls.

#### Undocumented request params

A narrowly scoped extension mechanism may pass through additional parameters; keep collisions, reserved names, and serialization behavior explicit. Validate values before logging or transmission.

#### Undocumented response properties

Preserve unknown fields when feasible; clearly separate raw data access from typed, supported properties. Schema drift must be observable and testable, not silently normalized into success.

### Configuring the HTTP client

Provide a replaceable transport abstraction for certificates, proxies, pooling, event hooks, and testing. Use origin-bound authentication; refuse unsafe credential forwarding on redirects; support trusted CA bundles and default TLS verification. Test proxy and no-proxy behavior and keep transport customization separate from API business logic.

#### Mutual TLS

Load certificate and key material into a dedicated, origin-restricted SSL context. Keep trust configuration independent of the client certificate and never disable hostname checks to make an example work. Rotation should build a fresh context/pool and allow outstanding requests to finish or cancel safely.

### Managing HTTP resources

Support deterministic `close()` / `aclose()` and context managers. Document ownership of external transports, connection pools, live streams, and sockets. Close resources on exceptions and cancellation; do not depend exclusively on garbage collection to release file descriptors.

## Provider integrations

Integration modules should configure the base URL, protocol dialect, authentication, signing, optional endpoints, versioning, data residency, and provider-specific errors—**without changing the core SDK**. A provider adapter may declare capabilities instead of pretending every vendor supports the same API.

Do not put a provider's trademark or service name in the public neutral package identity. If optional adapters are published, label them accurately and avoid implying endorsement or affiliation.

## Hosted gateway adapters

Gateways and managed inference endpoints may add request signing, regional routing, custom certificates, compatibility shims, and extra HTTP headers. Keep those differences isolated and test them with synthetic fixtures. Define where retries, pagination, stream framing, and error mappings are owned; do not invent compatibility with an untested platform.

### Legacy adapters

Retain compatibility adapters only with documented deprecation windows, migration instructions, security review, and tests. Never silently activate obsolete authentication flows or reintroduce removed dependencies merely because the inspiration source once supported them.

## Versioning

Choose a documented versioning policy (such as Semantic Versioning) and maintain release notes for **this independent project only**. Distinguish package versions from individual provider API versions. Describe breaking changes, support windows, schema regenerations, and deprecations without borrowing upstream version numbers.

### Determining the installed version

Once an actual distribution exists, Python packaging metadata can report its version:

```python
from importlib.metadata import version

# Replace with the project's real distribution name after publication.
# print(version("your-package-name"))
```

## Requirements

Declare the real minimum supported Python version in packaging metadata, not by inheriting a number from an unrelated project. Document required/optional dependencies, platform constraints, TLS requirements, transport feature flags, and the compatibility-test matrix. Until a package is implemented, **no installation or runtime compatibility is claimed**.

## Contributing

Contributions should include a scoped proposal, tests, security review where relevant, documentation, and a release note for user-visible changes. Keep core behavior provider-neutral; isolate service-specific exceptions in adapters. Document how to report security issues privately, how generated files are owned, and how to run **actual repository commands** once the codebase and CI exist.

**Source credit and independent ownership:** Inspired by [openai/openai-python](https://github.com/openai/openai-python). Credit does not replace an applicable license, grant permission to reuse trademarks, or establish technical compatibility. Do not copy upstream release claims or package-install instructions into a new project without implementing and validating them.
