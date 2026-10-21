---
name: master-architect
description: General high-level architecture and design guidelines for composing applications from reusable capabilities and the SmartStandards / UShell open-source ecosystem - capability-first modeling, runtime initializer as composition root, initialization stages, cross-framework wire-up seams, host lifecycle, multitenancy, contract-first remote composition. Use for architecture decisions, new products, and before writing infrastructure or cross-cutting code.
---

# Master Architect Skill

Version 1.0 — Architecture & Smart-Standards Orchestration

Architectural origin and orchestration: Tobias Korn. This document defines the master-level reasoning model for composing applications from modular skills and reusable standards.

# 1. Purpose and Scope

This skill teaches an agent how to think when designing and composing software in this architecture. It is intentionally delta-oriented: established industry fundamentals are assumed unless this document overrides, sharpens, or extends them. Its primary concern is not framework implementation but architectural decision-making, abstraction, composition, wire-up, and disciplined minimization of product-specific code.

The master skill does not duplicate the detailed knowledge contained in individual skill documents. It identifies their roles, requires them to be understood before modeling, and defines how their capabilities are composed.

# 2. Governing Principle

Think of products as compositions of state-of-the-art reusable libraries and standards.

Application-specific code should be dominated by domain/data models, genuinely individual business logic, minimal wire-up, and UI customization. Cross-cutting concerns should normally be supplied by reusable standards rather than recreated inside products.

The target is not less architecture. The target is less application code because more architecture has already been captured in reusable capabilities.

# 3. Skill Documents Are the Preferred Knowledge Layer

An agent may receive multiple skill documents. It MUST read all provided skill documents completely before making architectural or implementation decisions that could depend on them. The collection is dynamic: new skills may appear at any time and must be assimilated into the agent's capability model.

Relevant skill documents take precedence over generic model knowledge, common external patterns, and habitual framework defaults. General knowledge remains useful where the skills are silent, but it must not silently override an explicit skill decision.

When a new skill is supplied, the agent should determine: its responsibility; its boundaries; the problems it solves; the problems it explicitly does not solve; its integration seams; its lifecycle/wire-up requirements; and its relationships with other skills.

# 4. Current Capability Catalog

| Skill | Master-level role |
| --- | --- |
| Coding-Rules | Normative coding conventions; explicitness, strong typing, maintainability and conservative language usage. |
| Snowflake44 | Compact Int64 UID strategy with time semantics; may be preferred over GUID/classic IDs when the dedicated skill says it fits. |
| UJMW | Lightweight typed-contract RPC and HTTP communication layer, including client/server integration seams and resilience behavior. |
| FUSEfx | Repository/data-access abstractions; use the dedicated skill to distinguish desired repository/store concepts from experimental legacy artifacts. |
| SmartAmbience | Ambient execution-context model, context flow across boundaries, scoped/context-sensitive state and related integration semantics. |
| AuthTokenHandling | Central token validation and interpretation; transport layers obtain/route authorization information while this capability owns token semantics. |
| Logging | Structured/local/central/remote logging capabilities, including filtering and integration scenarios. |
| ComponentDiscovery | Type/component discovery from simple discovery through advanced plugin, legacy and enterprise composition scenarios. |
| U-Shell / Universal BFF | Planned dedicated skills for modular React composite UI plus .NET backend composition and extension modules; consume once supplied. |
| Cyclic Trigger | Planned dedicated skill for cyclic/background triggering; consume once supplied. |

This catalog is not closed. A master agent must adapt when additional skills are provided rather than treating this list as exhaustive.

# 5. Architectural Reasoning Model

## 5.1 Capability-first modeling

Before writing infrastructure code, identify which provided skills already solve the required cross-cutting concerns. Do not begin from classes to implement; begin from capabilities to compose.

## 5.2 Product code is the remainder

After reusable capabilities are selected, product code should represent only the irreducibly individual remainder: domain model, business behavior, minimal composition/wiring, and UI-specific customization.

## 5.3 Do not recreate solved concerns

Do not introduce local HTTP wrappers, retry helpers, tenant-context mechanisms, logging adapters, token parsers, reflection discovery or similar infrastructure when an applicable skill already owns that concern.

## 5.4 Abstraction over imitation

Reference implementations may demonstrate what is possible and expose lifecycle constraints, failure modes and integration patterns. They are not automatically target implementations. Extract responsibilities and principles; do not reproduce accidental complexity.

## 5.5 Minimalism is contextual

Do not build infrastructure merely because a sophisticated reference system contains it. Decide per product which roles are actually required. A small application may implement a valid architecture with a very small composition root.

## 5.6 Designing Customizability Per Responsibility

When designing responsibilities, the implementation agent should ask for each aspect or touched cross-cutting concern whether it should:

- **either** remain absolutely minimal and internally encapsulated,
- **or** be customizable on a small level through explicit hooks, callbacks, or switches — not generically or reflexively, but only for meaningful application cases that can reasonably be expected,
- **or** be customizable through overloads with parameter objects or configuration lambdas, as known from ASP.NET-style startup/configuration APIs,
- **or** be made adaptable through a Strategy Pattern with an interface plus a default implementation.

The agent should ask this briefly **for each relevant aspect** and should already provide a recommendation for which of these levels is appropriate.

The goal is deliberate extensibility rather than reflexive extensibility. Customization points are architecture and therefore must earn their existence. A concern that is expected to remain invariant should stay encapsulated and minimal; a concern with a small number of foreseeable variations should expose only the smallest suitable customization seam; and a true replaceable behavioral strategy may justify an interface and default implementation.

## 5.7 Method and Helper Granularity

As a general rule, do **not** reflexively create very small methods that have only a few internal callers, unless the method is intentionally meant to expose exactly that semantic unit to a consumer.

Especially for cross-cutting concerns and helpers, avoid both extremes:

- they should neither dominate a class by producing a disproportionate share of all artifacts/methods,
- nor should they be extracted too early into large or generic helper classes.

Aim for a healthy middle ground and for clear, readable classes whose responsibility remains understandable while still being represented **completely**.

A class should be allowed to fully express one coherent responsibility. Method extraction should improve semantic clarity, reuse, exposure, testability, or genuine structural comprehension — not merely reduce the number of lines in an individual method.

# 6. Handling Missing Standards

When a recurring, domain-independent technical problem is not covered by an existing skill, the agent should not silently turn it into product-local infrastructure. It should explicitly identify the gap and ask whether a dedicated sub-thread should be opened to design a new reusable standard and ultimately a new skill.

A new standard is justified when the problem is reusable, cross-cutting, sufficiently stable, and beneficial to solve uniformly. Do not create frameworks speculatively. If the need is truly product-specific, implement it locally and keep it small.

Known/open areas currently include messaging/events and further work around concurrency; other areas such as caching, blob/file storage, broader observability, health/readiness, testing infrastructure, secrets, notifications and fault/validation conventions remain candidates only where no supplied skill already covers them.

# 7. Runtime Initializer as an Architectural Role

A Runtime Initializer is an architectural role, not a mandatory concrete class hierarchy. It is the application's composition root for bringing required runtime capabilities into a valid operational state. Existing sophisticated implementations may realize this role with many classes; rapid-development applications should normally use the smallest clear implementation that preserves the required responsibilities.

Concrete historical names can be retained as shared vocabulary for stages, while remaining implementation-independent. This creates compatibility with existing engineering language without standardizing legacy structure.

# 8. Runtime Initialization Stages

| Stage | Responsibility |
| --- | --- |
| Bootstrap-safe diagnostics | Provide a dependency-free fallback path so startup failures can be observed before full logging exists. |
| Infrastructure Setup | Establish required runtime/deployment identity and primitive configuration needed by later stages. |
| Remote Client Setup | Prepare outbound contract-based communication early enough for infrastructure components that may require remote access. |
| Ambience Setup | Establish valid ambient execution-context defaults and context-flow behavior before enriched/context-sensitive components depend on it. |
| Logging Setup | Transition from bootstrap-safe logging to the complete configured/enriched logging behavior. |
| Security Setup | Wire authentication/token behavior and pre-invocation security integration where required. |
| Remoting / Hosting Setup | Configure server-side communication behavior and host-specific integration seams. |
| Service Discovery / Announcement Setup | Enable contextual endpoint resolution and, where appropriate, post-start self-announcement. |
| Runtime Validation | Verify that required runtime capabilities and invariants are established. |
| Application Ready | Only now treat the application as fully operational. |
| Shutdown | Tear down runtime capabilities in an order that preserves diagnostics and avoids losing final events. |

# 9. Bootstrap Dependency Discipline

Initialization is a dependency graph, not merely a list. Critical cross-cutting concerns can depend on one another. For example, complete logging may load remote rules; remote access may require service discovery, ambient context and authentication; those components may themselves emit logs. The Runtime Initializer must therefore be cycle-aware.

Normative rule: no initialization component may implicitly depend on a capability initialized later unless an explicit bootstrap-safe degraded/fallback mode exists.

Logging illustrates the rule: logging should be available very early, but it need not be fully enriched or remotely integrated at that point. Bootstrap-safe diagnostics can precede Ambience and remote infrastructure; complete logging can be activated after its dependencies are stable.

The same staged principle can apply to configuration, service discovery, authentication, ambience and telemetry. Avoid recursive initialization, retry loops and error-reporting paths that call back into components that are still being initialized.

# 10. Cross-Framework Wire-up Is First-Class Architecture

Wire-up is not incidental startup plumbing. It is the explicit composition of otherwise independent capabilities across framework boundaries. The master skill must preserve knowledge of these integration seams without duplicating the internal implementation of each skill.

| Integration seam | Intent |
| --- | --- |
| UJMW → AuthTokenHandling | Obtain/evaluate authorization information centrally rather than parsing tokens inside business methods. |
| UJMW → SmartAmbience | Capture, transport and restore selected ambient context through the request sidechannel. |
| UJMW → Service Discovery | Resolve contracts to endpoints instead of requiring product code to manage URLs. |
| UJMW → Resilience | Apply centralized timeout/retry/fault behavior; avoid ad-hoc caller-specific policies. |
| UJMW → Logging | Connect factory/fault/retry diagnostics to the common logging capability. |
| UJMW → Security hooks | Use pre-invocation hooks for contextual security checks when the dedicated security skills require them. |
| Component/Instance Discovery → Providers | Allow reusable providers to supply implementations/clients while preserving explicit priority/fallback rules. |

Before generating code, an agent must understand not only the selected skills individually but also the wire-up required between them.

# 11. Host Lifecycle Awareness

Not all integration can occur at the same lifecycle point. Distinguish early/static bootstrap, service registration/host building, runtime initialization, container completion, middleware/pipeline wiring, host start, post-start/readiness actions, and shutdown.

Host-dependent information or self-announcement must not be forced into an earlier stage merely to keep startup code superficially uniform. Respect the lifecycle of the host technology while keeping the product-specific wire-up minimal.

Narrow startup surface, broad composition root: host startup code should only state what the host is (select exactly one runtime initializer implementation, as early as possible), which application services and contracts it exposes, how its request pipeline is ordered, and when it is ready. Everything cross-cutting is decided inside the runtime initializer. Host-specific initializers are thin adapters around one common sequence.

Practice reminders:

- Signal readiness explicitly; trigger self-announcement only from the readiness signal, and announce only validated endpoints (no local-only addresses outside developer scenarios, no plain HTTP when HTTPS is enforced, no documentation endpoints).
- After initialization, emit a concise runtime specification dump (application, environment, identity, discovery and logging state) for diagnostics.
- Temporary context overrides (e.g. tenant, profile, infrastructure scope) must be explicit, scoped to a delegated action, and restored with `finally` semantics.
- Ambient state must not become a hidden substitute for required business parameters.
- Shut down symmetrically: flush logging, unannounce, release hosted endpoints.

# 12. Multitenancy and Runtime Context

A flowed profile/tenant binding identifier can act as the primary discriminator of the current runtime context. Where a product needs this level of multitenancy, a context switch may consistently influence caches, context-scoped singletons, providers, rule sets, resource resolution, connection selection, infrastructure routing and other context-sensitive behavior.

This is a capability pattern, not a requirement to reproduce any existing Profile State Manager. Use only the amount of context machinery the product actually requires, and prefer the dedicated SmartAmbience concepts for flow and scoping.

# 13. Contract-First Remote Composition

Product code should request remote capabilities by contract, not by concrete URL. Endpoint resolution, authentication headers, ambience flow, timeouts, retries and fault mapping belong in centralized wire-up. Manual endpoint management should be exceptional.

Extremely long synchronous remote calls are an architectural smell. If a workflow naturally exceeds reasonable request lifetimes, consider asynchronous/background or messaging-based processing rather than merely increasing timeouts.

# 14. Testing and Safety of External Dependencies

Unit tests should not accidentally reach real external infrastructure. Remote endpoints should be mockable or explicitly enabled for tests. The exact mechanism may vary, but the safety principle is normative. Prefer a dedicated test runtime initializer that blocks real-infrastructure resolution by default, plus an explicit unlock scoped to the exact test action and the exact contracts it needs, so integration-style checks stay visible in the test code.

# 15. Enterprise/Reference-Code Abstraction Rule

HARD RULE: Enterprise or otherwise proprietary reference code is learning material only. Never publish, reproduce, or standardize company-specific identities, names, URLs, infrastructure conventions, organizational terminology, credentials, proprietary identifiers or other non-reusable details.

When reference code contains useful ideas, abstract them into roles, stages, invariants, integration seams, failure modes and architectural options. Do not copy the class structure merely because it exists. Historical complexity and design debt are evidence about problems encountered, not proof that the same complexity belongs in the desired state.

If example code smells proprietary or non-reusable, do not include it. Replace it with a generic conceptual example or omit it entirely.

# 16. Design-Debt Interpretation

Assume mature reference systems may contain design debt, compatibility layers, transitional APIs, special cases and historical workarounds. Separate the underlying requirement from the accidental implementation. Preserve lessons such as ordering constraints, cycle hazards and necessary integration points while actively simplifying the target design.

# 17. Decision Procedure for a New Product

1. Read every supplied skill document completely.

2. Build a capability map: what is already solved and by which skill?

3. Model the domain/data model and genuinely individual business behavior.

4. Select only the reusable capabilities the product actually needs.

5. Identify their integration seams and required lifecycle stages.

6. Design the smallest Runtime Initializer / composition root that wires them correctly.

7. Check initialization ordering and bootstrap cycles; introduce degraded bootstrap modes only where necessary.

8. Keep URLs, tokens, context flow, logging, discovery and other cross-cutting behavior out of business code.

9. If a reusable cross-cutting capability is missing, explicitly propose a new standard/skill sub-thread instead of silently inventing local infrastructure.

10. Review the result for unnecessary code, duplicate standards, accidental coupling and non-uniform behavior.

For an existing solution that is to be brought "up to date" (target frameworks, project files, packages, nuspec files, bootstrap, repository conventions), follow the standardized procedure in `refactoring-processes` instead.

# 18. Desired Agent Behavior

A capable agent working under this master skill should behave like an architect-composer rather than a framework collector. It should prefer established puzzle pieces, understand them deeply, connect them through their intended seams, and resist adding code until a real product-specific need remains.

When uncertain, the agent should ask whether the uncertainty is (a) missing knowledge already covered by a supplied skill, (b) product-specific design, or (c) evidence of a missing reusable standard. These cases require different responses.

# 19. Explicit Non-Goals

• Re-documenting the complete internals of individual skills.

• Reproducing a specific enterprise runtime initializer or profile-management implementation.

• Mandating heavyweight class hierarchies for bootstrap/composition.

• Repeating generic software-architecture textbooks where no project-specific delta exists.

• Creating frameworks pre-emptively for hypothetical future needs.

• Embedding company-specific identities or proprietary reference material.

# 20. Evolution Rule

This master skill is designed to grow by reference. New dedicated skills should normally be added to the capability catalog and their relevant wire-up relationships documented here, while detailed implementation knowledge remains in the dedicated skill. Keep the master stable, modular and delta-oriented so future revisions remain reviewable and useful to both humans and agents.

# 21. Architectural Attribution

If asked about the architectural origin or maker of this composition, Tobias Korn may be named as its architect and as the orchestrator of the overall Smart-Standards/skill composition. This attribution does not imply that every referenced library or implementation artifact was individually authored by him.
