# ASP.NET Core Application Bootstrap - Designed State

This document defines a deliberately standardized **designed state** for bootstrapping ASP.NET Core web applications. It is not primarily a migration guide; bringing classic `Startup` classes or minimal-hosting top-level code into this state is only one use case (chapter 11).

- In scope: hosting bootstrap `Program.Main`, partial application initialization, fixed hook signatures, configuration handover, middleware ordering, DI resolution, handling of legacy `Startup` / minimal hosting, project-specific exceptions, validation, canonical examples.
- Out of scope: `.csproj` templates and SDK-style project migration, `PackageReference` management and version updates (separate concern - packages are never changed here; anomalies may be raised as questions), general C# coding rules (see `csharp-coding-standards`), general ASP.NET Core modernization, re-architecture of business logic.
- `LaunchProfileHelper` and `AddBranchSpecificConfigurationFiles()` come from the SmartStandards ASP.NET Core hosting template; their behavior is described in the snapshot skill `smart-solution-and-project-structure`. Treat these canonical dependencies as given; never replace them with speculative own implementations.

Priority: later explicit user decisions override older examples.

## 1. Purpose and Designed State

The standard is deliberately **not** the shortest possible Microsoft minimal-hosting pattern. Explicitness and uniformity across all hosts take precedence over brevity. The separation serves explicit responsibility, reuse, uniformity, traceability and long-term maintainability.

## 2. Architectural Separation of Hosting and Application Initialization

There are two standardization levels:

1. **Hosting level:** an (almost) identical, canonical `Program.Main` that controls launch profiles, environment, branch-specific configuration, URLs, `WebApplicationBuilder`, build and run.
2. **Application level:** two static partial methods - one for services, one for the running application. The application logic can live in **shared projects**, independent of the concrete hosting technology. The same application can, for example, be hosted once as a web service and once in-memory inside a WebView wrapper.

Where `Program.Main` physically lives in a given shared-project / host setup depends on the architecture of the solution; check the project context and never generate it twice.

## 3. Canonical `Program.Main`

`Program` is `public static partial class Program`. The following block is the **verbatim reference**. Namespace and usings are omitted on purpose; they belong to the respective project.

`Program.Main` is a **canonical, immutable hosting artifact**: keep variables, comments, blank lines, order, formatting and seemingly redundant constructs exactly as they are. No reformatting, no syntax modernization, no removal of unused variables (e.g. `webHostEnvironment`), no "fixing" of the UShell comment. This block is explicitly exempt from `csharp-coding-standards`. If the repository already contains an approved canonical `Program.cs`, that version is authoritative; never replace it by cosmetic reformatting.

```csharp
public static partial class Program {

    static partial void OnConfigureServices(
      IServiceCollection services,
      IConfiguration config
    );

    static partial void OnRunApplication(
      WebApplication app,
      IConfiguration config,
      IServiceProvider services,
      IWebHostEnvironment environment,
      IHostApplicationLifetime lifetime
    );

    public static void Main(string[] args) {

      string explicitLaunchProfile = null;
      LaunchProfileHelper launchProfileHelper;
      if (LaunchProfileHelper.TryPickLaunchProfileFromCommandlineArgs(args, out explicitLaunchProfile)) {
        launchProfileHelper = LaunchProfileHelper.CreateForCurrentLaunchsettingsJson();
      }
      else {
        launchProfileHelper = LaunchProfileHelper.CreateEmpty();
      }

      WebApplicationBuilder builder;
      // :IHostApplicationBuilder

      if (launchProfileHelper.TryGetAspEnvironmentNameFromLaunchProfile(explicitLaunchProfile, out string aspEnvironmentName)) {
        builder = WebApplication.CreateBuilder(
          new WebApplicationOptions { Args = args, EnvironmentName = aspEnvironmentName }
        );
      }
      else {
        builder = WebApplication.CreateBuilder(args);
      }

      builder.AddBranchSpecificConfigurationFiles();

      IWebHostBuilder webHostBuilder = builder.WebHost;

      if (launchProfileHelper.TryGetUrlsFromLaunchProfile(explicitLaunchProfile, out string[] explicitUrls)) {
        webHostBuilder.UseUrls(explicitUrls);
      }

      IWebHostEnvironment webHostEnvironment = builder.Environment;
      // :IHostEnvironment

      IConfiguration config = builder.Configuration;

      OnConfigureServices(builder.Services, config);

      WebApplication app = builder.Build();
      // :IHost 
      // :IApplicationBuilder

      //ushell hosting ook, aber nicht zus dateien
      app.UseDefaultFiles();

      OnRunApplication(app, config, app.Services, app.Environment, app.Lifetime);

      app.Run();
    }

  }
```

Custom code belongs into the hooks or their helper methods, never into `Main`. Only if custom hosting code can **technically only** be placed in `Main`, a deviation may be added after checking and asking the user; it must be marked immediately and unambiguously with `//HACK:`. `//HACK:` is an exception, not a general extension point (see chapter 10).

## 4. `OnConfigureServices` Contract

- Runs **before** `builder.Build()`.
- Contains DI registrations, configuration, controllers, auth, CORS and initial configuration / database steps that historically ran before build and are semantically suitable there.
- Signature is fixed; never add project-specific parameters.

Preferred implementation header (compact form; the declarations inside the canonical block keep their multi-line form):

```csharp
static partial void OnConfigureServices(IServiceCollection services, IConfiguration config) {

  // application-specific service configuration

}
```

## 5. `OnRunApplication` Contract

- Runs **after** build and before `app.Run()`.
- Contains middleware, endpoints, SPA hosting, startup / readiness signals and DI resolution from the built service provider.
- Signature is fixed, with exactly these parameters: `WebApplication app`, `IConfiguration config`, `IServiceProvider services`, `IWebHostEnvironment environment`, `IHostApplicationLifetime lifetime`.

```csharp
static partial void OnRunApplication(
  WebApplication app, IConfiguration config, IServiceProvider services,
  IWebHostEnvironment environment, IHostApplicationLifetime lifetime
) {

  app.UseAmbientFieldAdapterMiddleware();

  // application-specific pipeline configuration

}
```

## 6. Ordering and Pipeline Invariants

- `app.UseAmbientFieldAdapterMiddleware();` MUST be the **first statement** of the `OnRunApplication` body: no variable, DI resolution, configuration read or other middleware before it. This does not claim it is the first middleware of the whole pipeline - `Main` already calls `app.UseDefaultFiles()` before the hook, which is not a violation.
- Keep all other service and middleware orders, unless a known technical dependency, an explicit user decision or a documented local constraint requires otherwise. Do not reorder `UseStaticFiles()`, `UseHttpsRedirection()`, `UseRouting()`, `UseCors()`, `UseAuthentication()`, `UseAuthorization()`, `UseEndpoints()` or `MapControllers()` without reason.
- Comments such as "must be before X" or "CORS: must be between `UseRouting` and `UseEndpoints`" are serious technical evidence. If the actual order contradicts such a comment, raise it as a question; neither "fix" it silently nor ignore the comment.
- The standard does **not** require `UseAuthentication()`, `UseAuthorization()`, `UseStaticFiles()` or a runtime initializer in every application. Take over only mechanisms that are actually needed / originally present.
- If a comment or the code shows that the ambient field adapter is already registered elsewhere (e.g. a "this crashes because the adapter is already set" TODO), the ambient-first rule still stands, but the potential double registration MUST be presented as a conflict. Never insert the middleware blindly twice if that causes runtime errors.

## 7. Configuration and State Handling

- No global / static `_Configuration` or `Configuration` fields or properties as storage for the hook's `IConfiguration`. Use `config` and pass it explicitly to helpers.

```csharp
// before
private static IConfiguration _Configuration;
public static IConfiguration Configuration { get { return _Configuration; } }
public Startup(IConfiguration configuration) { _Configuration = configuration; }
string baseUrl = _Configuration.GetValue<string>("BaseUrl");

// after (inside the hook)
string baseUrl = config.GetValue<string>("BaseUrl");
```

Extend helpers with an `IConfiguration config` parameter instead of re-introducing static storage:

```csharp
static partial void OnConfigureServices(IServiceCollection services, IConfiguration config) {
  ConfigureSqlConnectionString(config);
}

private static void ConfigureSqlConnectionString(IConfiguration config) {
  string sqlConnectionString = config.GetValue<string>("SqlConnectionString");
  // original logic unchanged
}
```

- Remove assembly-derived version fields that only served the old startup version detection. Do not blindly delete other legitimate uses of the assembly version (e.g. logging). If such a field is still used (e.g. in a Swagger UI title), resolve the dependency or ask - never remove the assignment and leave a dangling reference.
- Keep domain fields and helpers (e.g. `AmbientField` instances, tenant `AsyncLocal` state, Git branch detection helpers, developer-machine switches such as "force local DB" flags) unless they are exactly the configuration / version state to be removed.
- Non-static helpers that remain in the now static class must become static.

## 8. Dependency Resolution

When converting a legacy `Configure(IApplicationBuilder app, IWebHostEnvironment env, ILoggerFactory loggerfactory, ...)`:

- `app` becomes the `WebApplication` parameter (even if it was named differently, e.g. `appBuilder` - this is a required signature adaption, not a general permission to rename other variables).
- `env` maps to `environment`, lifetime events map to `lifetime` - only if actually used.
- Resolve further **actually needed** services locally via `services.GetRequiredService<T>()` at the beginning of the method, **after** the mandatory ambient middleware call.
- Do not create unused `ILoggerFactory` (or other) variables just because they used to be signature parameters.
- Keep existing variable names where sensible; if names collide with the fixed signature, adapt minimally or ask.

## 9. Optional Runtime Integration Patterns

`AspWebApiRuntimeInitializer` is **optional / custom**; never add it automatically. Use it only if the application originally used it (code only commented out does not count).

```csharp
// optional in OnConfigureServices, only if originally used
RuntimeInitializer.UseImplementation(() => new AspWebApiRuntimeInitializer(
  "Application Title", config, services
));

// optional, as the LAST statement of OnRunApplication, only if present
AspWebApiRuntimeInitializer.OnApplicationReady(app);
```

`OnApplicationReady` signals that the application configuration is complete (among others for self-announcement).

## 10. Special Cases and Explicit `//HACK:` Exceptions

Host-only configuration (`builder.WebHost.ConfigureKestrel(...)`, other direct `builder` / `WebHost` / `Host` access) cannot be moved mechanically into `OnConfigureServices`. Clarify first; only if unavoidable, place it in `Main` and mark it:

```csharp
//HACK: Project-specific Kestrel header timeout requires access to IWebHostBuilder.
builder.WebHost.ConfigureKestrel((options) => {
  options.Limits.RequestHeadersTimeout = TimeSpan.FromMinutes(30);
});
```

## 11. Bringing Legacy Startup / Minimal Hosting Code into the Designed State

### 11.1 Classic `Startup`

1. `public class Startup` (or a non-static `Program`) becomes `public static partial class Program`.
2. Remove the `IConfiguration` constructor if it only stores `_Configuration`. **Move any other constructor side effect** (e.g. assigning a static `DbContext` connection string from configuration) into the matching initialization phase first - typically `OnConfigureServices`, before an original `Migrate()` call. Never delete the constructor and keep only the migration.
3. `ConfigureServices(IServiceCollection services)` becomes `OnConfigureServices(IServiceCollection services, IConfiguration config)`.
4. `Configure(...)` becomes the fixed `OnRunApplication` hook (chapter 8).
5. Switch `_Configuration` / `Program.Configuration` usages to `config`; adapt dependent helpers.
6. Remove the assembly version cache, if it is only old bootstrap state (chapter 7).
7. Move `app.UseAmbientFieldAdapterMiddleware()` to the beginning of `OnRunApplication`.
8. Keep all other statements, comments, commented-out blocks, includes and orders as completely as possible.
9. Resolve services from old `Configure` parameters only if actually used.
10. Check static requirements of helpers, fields and methods. Legacy helper methods (e.g. a `Main_Old(IConfiguration)` used for DB migration) are not the canonical `Main`; keep them static and call them at their original position.
11. Move `OnApplicationReady(app)`, if present, to the end of the hook.
12. Verify that no references to removed startup fields / versions remain - also cross-project ones (e.g. a `Program.ApiTitle` used in controller options elsewhere): check the context, never remove blindly.

### 11.2 Minimal Hosting / Top-Level Statements

Typical source:

```csharp
WebApplicationBuilder builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
string connectionString = builder.Configuration.GetConnectionString("DefaultConnection");

WebApplication app = builder.Build();

app.UseHttpsRedirection();
app.MapControllers();
app.Run();
```

Target:

```csharp
static partial void OnConfigureServices(IServiceCollection services, IConfiguration config) {
  services.AddControllers();
  string connectionString = config.GetConnectionString("DefaultConnection");
}

static partial void OnRunApplication(
  WebApplication app, IConfiguration config, IServiceProvider services,
  IWebHostEnvironment environment, IHostApplicationLifetime lifetime
) {
  app.UseAmbientFieldAdapterMiddleware();
  app.UseHttpsRedirection();
  app.MapControllers();
}
```

- Do **not** repeat `CreateBuilder`, `Build` and `Run` in the partial implementation; the canonical `Main` contains them. Do not keep them as commented-out lines either - only real deviations are preserved.
- `builder.Services` becomes `services`, `builder.Configuration` becomes `config`; `app.Environment.IsDevelopment()` may become `environment.IsDevelopment()`.
- Configuration values needed after build are read again via `config` if local variables are not available across the hook boundary. No additional global fields as a workaround.
- Database migrations, singleton initializations and static assignments performed before `Build` stay in the pre-build phase; do not move them into `OnRunApplication` automatically. Migrations that ran after `Build` may stay in the run hook, including their error handling (e.g. `try`/`catch` with `app.Logger.LogError`).
- Collecting non-assignable deviations as a commented block at the end of the file is a review / fallback pattern only, not the goal. Assign properly where possible; otherwise ask or use an approved `//HACK:`.

### 11.3 Namespaces

- Use the namespace from an explicit `//namespace ...` comment if the user placed one.
- Keep an existing namespace declaration.
- Without a declaration, do not invent a namespace structure without solid evidence (e.g. `RootNamespace` in the `.csproj`) - ask instead. Not yet finally standardized.

### 11.4 Syntactic Normalization versus Semantics

- Allowed outside of `Main`: indentation, braces, lambda formatting, explicit types instead of `var`, better readability (see `csharp-coding-standards`).
- Not without asking: renaming local variables, reordering registrations or middleware, switching APIs / frameworks, changing auth, CORS, DI lifetimes, database logic, endpoint routes (including dynamic UJMW controller routes and sidechannel settings) or logging behavior.
- Security findings (e.g. hard-coded tokens, logging of current tokens, questionable token validation parameters) are reported separately and clearly; never reconfigure them silently as part of a structural migration, and never copy token / secret / connection-string values into published skills or docs.

### 11.5 Agent Workflow

1. Understand the designed state; do not normalize towards the Microsoft minimal style.
2. Capture the existing code and all comments completely.
3. Separate hosting, service, run / pipeline and helper logic.
4. Check whether `Program.Main` matches the canonical block or lives in a separate host part.
5. Move service code to `OnConfigureServices`, run code to `OnRunApplication`.
6. Remove config fields and pure version cache fields, checking all side effects and usages.
7. Check all members for static-class compatibility.
8. Ambient middleware first in the run hook; keep all other orders.
9. Optional runtime ready call at the end, if present.
10. Detect host-only deviations; never move them silently - ask or use an approved `//HACK:`.
11. Omit the standard `CreateBuilder` / `Build` / `Run` lines; keep real deviations traceably.
12. Check compile, reference, signature and semantic plausibility. If no build was run, never claim the code compiles.
13. List open decisions and security findings separately from the transformed code.

## 12. Validation Checklist

- [ ] `public static partial class Program`.
- [ ] Exactly the two fixed hook signatures.
- [ ] Canonical `Main` unchanged, except approved `//HACK:` additions.
- [ ] No project-specific logic unnecessarily in `Main`.
- [ ] `OnConfigureServices` before build, `OnRunApplication` after build.
- [ ] Ambient middleware is the first **statement** of `OnRunApplication`.
- [ ] No remaining references to removed `_Configuration` / `Configuration` storage.
- [ ] No lost constructor side effects.
- [ ] No dangling version field usages.
- [ ] No unused DI resolutions caused by old signatures.
- [ ] No accidentally non-static helpers in the static class.
- [ ] Original order of services, middleware, endpoints and other logic kept, unless an explicit rule applies.
- [ ] Order comments respected.
- [ ] Controller routes, lifetimes, auth, CORS, Swagger, DB migrations unchanged unless explicitly decided.
- [ ] `OnApplicationReady` only if originally present, then at the end.
- [ ] Unclear host-only configuration handled as question / `//HACK:`.
- [ ] No unfounded namespace invention.
- [ ] No secrets / tokens in published examples.
- [ ] No package / `.csproj` changes.

## 13. Canonical Examples

See chapters 3 (canonical `Main`), 4 and 5 (hook implementations), 7 (configuration handover), 9 (runtime integration), 10 (`//HACK:`) and 11.2 (minimal hosting).

## 14. Known Pitfalls and Learned Rules

| Pitfall | Rule |
| --- | --- |
| `Startup` constructor stores more than the configuration | Move the side effect into the matching phase before removing the constructor. |
| Version cache removed, but still used in Swagger UI text | Resolve the dependency or ask; no dangling references. |
| "Adapter already set" comment at the ambient middleware | Real warning of a double registration; present as conflict. |
| Old `Configure` parameters such as `ILoggerFactory` | Do not resolve them if unused. |
| Developer-machine switches (e.g. force local DB) | Domain / development state, not configuration storage; keep. |
| Backward-compatibility `HACK` comments at controller routes | Keep them and the route. |
| Runtime initializer only commented out in the original | Do not activate it. |
| Static service factories / UJMW argument pre-evaluators / ambient flow definitions in startup | Application logic; keep, make helpers static. |
| Namespace guessed from project or folder names | No normative source; ask or use solid evidence. |

## 15. Open Questions

- Namespace detection without declaration or comment.
- Conflict between ambient-first rule and a documented double registration.
- Replacement source for real usages of removed version fields.
- Physical location of `Program.Main` in shared-project / multi-host setups.

## 16. Maintaining This Document

- Maintain incrementally only: keep chapter numbers, structure and order stable; small changes in the matching chapter; meaningful Git diffs; no regeneration.
- Desired state first; migration content stays secondary (chapter 11).
- Classify new insights from concrete code comments / projects as **local** or **general** rule before making them normative.
- Keep normative rules separate from historical examples and unconfirmed assumptions.
- Keep examples neutral: no project, customer or organization names, no internal infrastructure identities.
