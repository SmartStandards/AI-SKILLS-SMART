---
name: smart-persistence-entityframework-fusefx
description: Data access via FUSE-fx repositories over Entity Framework (EF Core on modern .NET, EF6 on .NET Framework). Keywords - FUSE-fx.RepositoryContract, EfSupport, EfDataStore, EfRepository, IDbContextInstanceProvider, ShortLivingDbContextInstanceProvider, ModelVsEntityDataStore, FUSE-fx.ModelDesigner, fuse schema JSON, SchemaRoot, PrimaryIdentity, PropertyGroup, HasLookup, HasPrincipal. Use when adding database persistence to an application or service, designing the persistence entity model, wiring a typed DataStore into DI, generating entities/DbContext/DataStores from a FUSE schema, or troubleshooting EF-backed FUSE repositories. Snapshot of the upstream skills and sources; escalate to the upstream repositories for library development or deep troubleshooting.
metadata:
	snapshot-date: "2026-11-27"
---

# Persistence with Entity Framework and FUSE-fx (Snapshot)

<!-- Layer 1 - Domain Overview -->

Applications never talk to `DbContext`/`DbSet` directly from business or service code. They consume a **typed DataStore** whose properties are FUSE repositories (`IRepository<TEntity, TKey>`). Underneath, `EfDataStore<TDbContext>` creates `EfRepository<TEntity, TKey>` instances over an EF model made of clean, FUSE-attributed persistence POCOs. The model is ideally maintained graphically in the FUSE ModelDesigner and persisted as a fuse schema JSON file, from which entities, DbContext and DataStores are generated.

- Libraries / repositories in scope: `FUSE-fx.RepositoryContract` (contract + `EfSupport`), `FUSE-fx.ModelDesigner`, `FUSE-fx.ModelDescription` (`SchemaRoot`), `FUSE-fx.EntityAnnotations` (attributes), `FUSE-fx.ModelReader`.
- Use for: new database-backed persistence, entity model design, DataStore wiring, code generation from the schema, EF-backed repository troubleshooting.
- Out of scope: repository contract semantics, query/filter rules and non-EF providers - see the upstream `fuse-fx` skill; remote access to repositories - see `smart-http-remoting-standard`; legacy model-first EDMX models (EF6 designer) are explicitly **not** the designed state and must not be used as a template for new code.

<!-- Layer 2 - Cached Essentials -->

## Snapshot Status

- Snapshot date: 2026-11-27
- Synchronized against: FUSE-fx.RepositoryContract @ 1e9f388, FUSE-fx.ModelDesigner @ 7e0bd13
- Staleness thresholds: fresh < 90 days, aging 90-180 days, stale > 180 days
- Volatility: medium (ModelDesigner code generation is still evolving)

Apply the alarm logic from `smart-skill-maintaining-guidelines` before relying on this snapshot.

## Core Concepts

- **Layering**: service/application code -> typed DataStore interface -> `EfDataStore<TDbContext>` -> `EfRepository<TEntity, TKey>` -> `IDbContextInstanceProvider` -> `DbContext` -> database. Only the DataStore interface is visible to consumers.
- **Persistence entities**: plain POCOs (suffix `Entity`, e.g. `BoardEntity`) with public get/set properties and FUSE ModelDescription attributes on the class: `[PrimaryIdentity("<index>")]`, `[PropertyGroup("<index>", "<field>", ...)]`, `[UniquePropertyGroup(...)]`, `[HasLookup(...)]`, `[HasPrincipal(...)]`, `[HasDependent(...)]`. The attributes are the source for the FUSE schema (`SchemaCache`/ModelReader) - keys, indices and relations must be declared there, not only in EF fluent configuration.
- **Typed DataStore**: an interface `I<Schema>DataStore` with one `IRepository<TEntity, TKey>` property per entity, implemented by `<Schema>EfDataStore : EfDataStore<TDbContext>, I<Schema>DataStore`, each property returning `this.GetRepository<TEntity, TKey>()`.
- **DbContext lifetime** is owned by an `IDbContextInstanceProvider`:
	- `ShortLivingDbContextInstanceProvider<TDbContext>` - new context per repository operation, disposed afterwards. **Default** for services and web hosts; the DataStore can then be a singleton.
	- `LongLivingDbContextInstanceProvider<TDbContext>` - one context, access serialized via `lock`; only for single-user tools or tests where change tracking across calls is wanted.
	- `DelegateBasedDbContextInstanceProvider` - custom acquisition (e.g. scoped DI context).
- **Domain objects vs. entities**: if the public contract must differ from the persistence model, use `ModelVsEntityDataStore` (wraps an `EfDataStore`, maps `(DomainType, EntityType)` pairs) or the generator's optional selectors/copy methods (`<Entity>Selector`, `<Domain>Selector`, `CopyContentFrom/To`). Persistence entities never leave the persistence layer through public service contracts unless they are deliberately identical to the domain objects.
- **Fuse schema JSON** (`SchemaRoot`: `entities`, `fields`, `indices`, `relations`, `designerData`) is the graphically maintained source of truth when the ModelDesigner is used.

## Typical Usage

### Model-first with the FUSE ModelDesigner (preferred for new models)

1. Maintain the model graphically; it is saved as `<SchemaName>.json` in `SchemaJsonDirectory`.
2. Configure `GenerationSettings.json` (next to the designer executable): `SchemaName`, `SchemaJsonDirectory`, `EntitiesDirectory`/`EntitiesNamespace`, `DbContextDirectory`/`DbContextNamespace`/`DbContextName`, `InterfaceDataStoreDirectory`/`InterfaceDataStoreNamespace`, `EfDatastoreDirectory`/`EfDatastoreNamespace`, `DomainObjectsNamespace`, `GenerateDomainObjectSelectors`, `GenerateDomainObjectCopyMethods`.
3. Generate code from JSON. Output: `GeneratedEntities.cs`, the DbContext (keys, indices, required/max length, identity columns, relations with `DeleteBehavior`), `I<SchemaName>DataStore.cs`, `<SchemaName>EfDataStore.cs`. All files are `// <auto-generated />` - never edit them by hand; put custom code into separate partial files.
4. Reverse direction exists: generating JSON from an annotated `GeneratedEntities.cs` re-reads the attributes and keeps designer layout data where entity/relation names still match. Keep both artifacts committed and in sync.
5. Create EF migrations as usual (`dotnet ef migrations add ...`) against the generated DbContext; provide an `IDesignTimeDbContextFactory` if the context has no usable default configuration.

### Wiring (designed state)

```csharp
// Persistence layer: DbContext with an options constructor (as in the upstream TechDemo)
public class BoardDbContext : DbContext {
	public BoardDbContext(DbContextOptions<BoardDbContext> options) : base(options) { }
	public DbSet<BoardEntity> Boards { get; set; }
}

// Host bootstrap
DbContextOptions<BoardDbContext> options = new DbContextOptionsBuilder<BoardDbContext>()
	.UseSqlServer(connectionString).Options;

services.AddSingleton<IBoardDataStore>((serviceProvider) => new BoardEfDataStore(
	new ShortLivingDbContextInstanceProvider<BoardDbContext>(() => new BoardDbContext(options))
));
```

- Read the connection string from configuration and pass it in; prefer the options constructor over the static `ConnectionString` setter and LocalDB default that the current generator emits.
- Apply migrations once at host startup (`Database.Migrate()`), not in service constructors.
- Cache repository instances in the DataStore if they are requested often: `EfDataStore.GetRepository` validates the type against the schema and returns a **new** `EfRepository` on every call.
- Multi-tenancy/user scoping: model `TenantId`/`UserId` as explicit key or indexed fields in the entities and enforce them in a decorator repository or the service layer - EF global query filters are invisible to the FUSE schema.

## Rules

- Business code uses only the typed DataStore interface; no `new DbContext()` and no `DbSet` access outside the persistence layer.
- Keep FUSE attributes and EF configuration consistent (same keys, indices and relations); prefer generation from the schema over hand-written duplication.
- Navigation properties must be `virtual` when lazy-loading proxies are enabled (the generator enables `UseLazyLoadingProxies`, which requires `Microsoft.EntityFrameworkCore.Proxies`).
- Do not rely on `EfDataStore.BeginTransaction/CommitTransaction/RollbackTransaction` - they currently throw `NotImplementedException`.
- `EfSupport` targets EF6 on .NET Framework and EF Core on modern .NET; metadata and query translation differ - test on every target framework you ship.

## Quick Troubleshooting

| Symptom | Likely cause | Quick check |
| --- | --- | --- |
| `Entity type ... is not part of the model for this context` | Entity missing as `DbSet` or schema cache built from another context | `DbSet<TEntity>` exists on the `TDbContext` of the DataStore |
| Wrong key / `GetRepository` key type mismatch | `TKey` differs from the `[PrimaryIdentity]` index field type, or composite key used with a scalar `TKey` | Compare `PrimaryIdentity` index members with `TKey` |
| `ObjectDisposedException` on navigation access | Lazy navigation touched after the short-living context was disposed | Load needed data inside the repository call or use field projections |
| Concurrency/threading exceptions in a web host | Long-living provider or shared context across requests | Switch to `ShortLivingDbContextInstanceProvider` |
| Migration tool cannot create the context | No parameterless/design-time configuration | Add an `IDesignTimeDbContextFactory<TDbContext>` |
| Regenerated code lost custom logic | Manual edits in `// <auto-generated />` files | Move custom code into partial classes |

<!-- Layer 3 - Upstream Escalation -->

## Upstream Sources

| Repository | URL | Skills | Docs | Packages |
| --- | --- | --- | --- | --- |
| FUSE-fx.RepositoryContract | https://github.com/SmartStandards/FUSE-fx.RepositoryContract | `.agents/skills/fuse-fx`, `.agents/skills/most-important-details-for-fusefx-repository-contract` | `doc/`; reference app `dotnet/src/TechDemo.WebApi` (EF, Sql and ModelVsEntity DataStores) | `FUSE-fx.RepositoryContract`, `FUSE-fx.EfSupport` |
| FUSE-fx.ModelDesigner | https://github.com/SmartStandards/FUSE-fx.ModelDesigner | `.agents/skills/most-important-details-for-fusefx-model-designer` | generator: `FUSE-fx.ModelDesigner.App/SchemaEditor.cs` | - |
| FUSE-fx.ModelDescription | https://github.com/SmartStandards/FUSE-fx.ModelDescription | `.agents/skills/` | `SchemaRoot`, `EntitySchema`, `RelationSchema` | - |
| FUSE-fx.EntityAnnotations | https://github.com/SmartStandards/FUSE-fx.EntityAnnotations | `.agents/skills/` | attribute definitions | `EntityAnnotations` |
| FUSE-fx.ModelReader | https://github.com/SmartStandards/FUSE-fx.ModelReader | `.agents/skills/` | attribute-to-schema reflection | - |

### Escalate to upstream when

- developing, extending or fixing FUSE-fx EfSupport or the ModelDesigner generator;
- deep troubleshooting needs implementation details not covered above (query translation, schema cache, relation handling);
- this snapshot is stale or its alarm fires and the task depends on the affected details;
- observed behavior or the consumed package version contradicts this snapshot.

Then load the upstream skills (local clone preferred, otherwise online) and treat them as authoritative.
