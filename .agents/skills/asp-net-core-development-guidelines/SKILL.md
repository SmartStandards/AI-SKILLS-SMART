---
name: asp-net-core-development-guidelines
description: SmartStandards ASP.NET Core development guidelines - designed state of the application bootstrap (canonical immutable Program.Main, public static partial class Program with the two fixed hooks OnConfigureServices / OnRunApplication, UseAmbientFieldAdapterMiddleware first, config handover instead of static Configuration fields, optional AspWebApiRuntimeInitializer, //HACK: exceptions, bringing legacy Startup classes or minimal-hosting top-level code into that state) as well as explicit controller exposure, controlled endpoint registration and avoiding accidental public endpoint discovery. Load when writing, reviewing or migrating Program.cs / Startup.cs of an ASP.NET Core host, or when adding controllers or endpoints to reusable libraries.
metadata:
  version: "2026-10-21"
---

# ASP.NET Core Development Guidelines for SmartStandards

## Purpose

This skill defines SmartStandards-wide ASP.NET Core integration rules for libraries and reusable components that expose controllers, endpoints or middleware into host applications.

The goal is controlled host integration: endpoints must become visible because the host application explicitly opted in, not because ASP.NET Core discovered implementation types by accident.

## Application bootstrap (designed state)

ASP.NET Core hosts follow a standardized, deliberately explicit bootstrap. The full designed state - canonical `Program.Main` verbatim, hook contracts, ordering invariants, legacy migration, validation checklist and learned pitfalls - is in [references/application-bootstrap.md](references/application-bootstrap.md). Load it before writing, reviewing or migrating any `Program.cs` / `Startup.cs`.

Core invariants:

- `Program` is `public static partial class Program`; `Program.Main` is a canonical, immutable artifact - never reformat or modernize it (also exempt from `csharp-coding-standards`). Custom hosting code that technically can only live in `Main` needs approval and a `//HACK:` marker.
- Exactly two hooks with fixed signatures: `OnConfigureServices(IServiceCollection services, IConfiguration config)` before build, `OnRunApplication(WebApplication app, IConfiguration config, IServiceProvider services, IWebHostEnvironment environment, IHostApplicationLifetime lifetime)` after build.
- `app.UseAmbientFieldAdapterMiddleware();` is the first statement of `OnRunApplication`; all other orders are kept.
- No static `Configuration` storage; pass `config` explicitly.
- `AspWebApiRuntimeInitializer` only if originally used; `OnApplicationReady(app)` is then the last statement.
- Migration never changes semantics, order, variable names or security-relevant configuration on its own; ask instead of guessing.

`LaunchProfileHelper` and `AddBranchSpecificConfigurationFiles()` are described in the snapshot skill `smart-solution-and-project-structure`, together with the designed state of `launchSettings.json` and the appsettings files (`Launch Profile and Configuration Design`).

When the bootstrap is reviewed as part of bringing a whole solution up to date, follow the procedure in `refactoring-processes` (analysis and plan first, migration only after confirmation).

## Controller exposure rule

ASP.NET Core controllers provided by SmartStandards libraries SHOULD be hidden from ordinary automatic controller discovery.

Reusable library controllers SHOULD be:

- `internal` where practical,
- added to MVC explicitly through a host-facing convenience extension method,
- activated only by intentional one-line service registration such as `services.Add...(...)`,
- registered through controlled MVC application-part or feature-provider mechanisms when a controller is the appropriate host integration artifact.

Agents and maintainers MUST NOT make reusable SmartStandards controllers public merely so ASP.NET Core can discover them automatically.

## Convenience extension methods

A reusable SmartStandards ASP.NET Core feature SHOULD expose a concise service-registration entry point.

Preferred shape:

```csharp
services.AddSomeSmartStandardsFeature((cfg) => {
	cfg.SomeSetting = "value";
});
```

The extension method is responsible for:

- validating required settings early,
- registering the hidden controller or endpoint integration artifact,
- configuring any shared library-level hooks needed by the feature,
- keeping the host application's opt-in visible and reviewable.

## Configuration transport rule

Do not introduce DI configuration bags whose only purpose is to move values from a convenience registration method into a controller constructor.

If configuration is owned by the explicit feature registration and is not a normal host service, prefer a controlled internal registration seam over artificial constructor-injected option bags.

Acceptable alternatives include:

- an internal feature registry owned by the extension method,
- a static/internal configuration seam when the surrounding library already uses static global setup semantics,
- an actual DI service only when it represents reusable host behavior, lifecycle-managed infrastructure or test-replaceable functionality rather than mere value transport.

## Endpoint discovery and safety

Automatic endpoint discovery is hard to audit in reusable libraries. Therefore, SmartStandards library endpoints should fail closed unless explicitly registered.

A hidden controller that is reachable only through an explicit feature provider SHOULD still behave safely if misconfigured:

- do not emit misleading default metadata,
- validate required configuration before registration where possible,
- return a clear server-side configuration error if a route is somehow reached without successful registration,
- keep public protocol metadata free of secrets and request-specific credentials.

## OAuth and well-known endpoints

OAuth and `.well-known` endpoints are public protocol surfaces and must be especially explicit.

For OAuth-related SmartStandards endpoints:

- OAuth server controllers should be activated through explicit service extensions such as `AddOAuthServerController()`.
- OAuth protected-resource metadata endpoints should be activated through explicit service extensions such as `AddWellKnownOAuthProtectedResourceEndpoint(...)`.
- Publishing metadata must not imply token validation, token issuing or hosting an authorization server unless those capabilities are explicitly registered as separate features.
- Keep metadata publication, authentication challenges, token validation and token issuing as separate concerns.

## Agent guidance

When adding or modifying ASP.NET Core integration in a SmartStandards repository:

1. Check whether a controller would become visible through automatic discovery.
2. Prefer hiding reusable-library controllers and exposing them only through explicit service extensions.
3. Avoid constructor-injected configuration bags that only mirror extension-method parameters.
4. Keep endpoint activation deterministic, reviewable and opt-in.
5. Preserve existing public signatures and host startup patterns unless a breaking change is explicitly approved.
