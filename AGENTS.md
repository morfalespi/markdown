# Repository Guidance

This document defines implementation and review guardrails for an **independent, provider-neutral Python API client project**. It adapts useful documentation patterns from the [OpenAI Python SDK repository](https://github.com/openai/openai-python), with attribution to that source; it does not claim that this project is an OpenAI product, a fork with working feature parity, or an implementation of any particular service. Respect applicable upstream licenses and retain copyright notices if source code is reused.

These rules describe **desired engineering behavior**, not features proven to exist. Confirm the actual repository layout, supported APIs, package name, tests, and release process before applying examples. Do not invent files, credentials, workflows, or implemented capabilities.

<a id="generated-file-ownership"></a>

## Generated SDK

Determine ownership **before editing**: schema-generated code, generator templates, and handwritten code may have different authoritative sources. If the repository generates clients from a schema, change that schema or its generator first, regenerate outputs deterministically, and avoid maintaining hand-edited generated artifacts. If the project is entirely handwritten, document that decision instead.

Maintain clear boundaries between public types, transport, authentication, provider adapters, retries, response parsing, and tests. Keep vendor-specific request paths, credentials, endpoints, and payload conventions in optional adapters or configuration, not the core client. Avoid binding the public interface to a particular model, cloud, or API unless explicitly required and reviewed.

Preserve intended APIs and backwards compatibility; document breaking changes. Do not assume that any SDK example from the inspiration source works in this independent project without corresponding implementation and tests.

## Security requirements for coding agents

Treat credentials, tokens, private keys, session cookies, user data, log payloads, and diagnostics as sensitive. Follow the repository's own contribution and security policies **when those files exist**; otherwise, establish private reporting and a review process before accepting sensitive changes. Never disclose vulnerabilities in public issues, pull requests, or CI logs.

- Use environment variables or a proper secret manager; never commit live secrets or copy confidential requests into fixtures. Generate clearly synthetic and redacted test data.
- Redact authorization, cookie, proxy credentials, query-string tokens, webhook secrets, certificate keys, and nested sensitive fields in exceptions, telemetry, and debug logging. Limit diagnostic-body exposure.
- Validate URL schemes and redirect destinations; guard against accidental credential forwarding across hosts. Default to TLS verification and restrict certificate-bearing transports to approved origins.
- Review direct and transitive dependencies, lock files, optional extras, build backends, hooks, and provenance. Pin privileged CI actions and run untrusted contributions without publishing credentials.
- Separate build/test jobs from publishing. Use short-lived, scoped credentials and trusted publishing where supported; never expose publishing tokens to untrusted jobs.
- Require owner review and focused synchronous/asynchronous regression tests for changes to authentication, proxies, redirects, TLS/mTLS, request signing, logging, serialization, streaming, and release automation.

## Python version policy

Use the project's packaging metadata (for example, `pyproject.toml` → `requires-python`) as the source of truth **if present**. Align classifiers, dependency markers, documentation, lock files, static analysis, and CI with that declared range. Do not import a Python minimum from the upstream repository without checking actual compatibility.

Distinguish a tested minimum from the newest tested Python release and from optional prerelease support. Record an explicit support and deprecation policy before changing the floor.

## Changing the minimum Python version

1. Confirm the project's compatibility promise and explain the impact on users; check supported Python lifecycles and dependency requirements.
2. Update packaging metadata, interpreter pins, dependency resolutions, classifiers, compatibility documentation, and CI matrices **where those files actually exist**.
3. Build release artifacts; verify their `Requires-Python` metadata and that unsupported interpreters cannot install incompatible releases. Exercise the oldest and newest supported interpreters.
4. Publish a release note identifying the new minimum, the final compatible release when known, and any migration steps; obtain maintainer approval.

A consistency script can detect mismatched declarations, but it cannot decide whether dropping an interpreter is appropriate.

## Automation map

Inventory the workflows in the repository rather than assuming the upstream filenames are present. Where applicable, provide:

- **Pull request / push CI:** formatting, lint, typing, dependency audit, unit tests, sync/async behavior, package builds, and artifact metadata checks with minimum privileges.
- **Scheduled compatibility review:** supported-runtime matrix, dependency updates, breaking-change review, and a documented decision trail. Automated agents may propose changes but must not silently raise the runtime floor or merge changes.
- **Release pipeline:** reproducible build verification, tag/version checks, changelog review, isolated publishing, and provenance. A privileged publish job must not run untrusted contributed code.

Document real workflow paths once implemented. Keep CI and release configuration independent of any single commercial model or hosted assistant.

## Validation

Before merging changes, run **the commands implemented by this repository**. A baseline checklist is:

```sh
# Illustrative checks; use only after the corresponding tools are configured.
python -m compileall .
python -m pytest
python -m build
```

If the repository uses another package manager or custom scripts, substitute its **actual, locked** commands. In addition, verify public imports, wheel/sdist metadata, minimum/current Python behavior, sync/async parity, error redaction, retries and cancellation, streaming termination, redirect protection, and adapter boundaries. Validate any generated files against their authoritative source. Do not claim tests passed unless they were executed.

Run the full compatibility matrix before a release whenever that matrix is part of the project's supported contract.

## Large-payload compatibility

Large JSON requests and streaming events can be legitimate traffic. Do not introduce arbitrary body, line, or event limits solely to ease parsing. Prefer incremental parsing, bounded intermediate allocations, amortized-linear buffering, backpressure, cleanup, caller cancellation, and explicit documented size limits when the protocol actually requires them. Any new hard rejection limit needs an approved API contract and review across transports and data types.

Use deterministic, synthetic, in-memory fixtures—not production captures or paid/live generation calls. Exercise large structured responses, binary uploads, chunked transfer, server-sent events, early disconnects, and cancellation; run resource-heavy scenarios sequentially. Fixture size measures regressions, **not** the provider's or project's maximum payload size.

