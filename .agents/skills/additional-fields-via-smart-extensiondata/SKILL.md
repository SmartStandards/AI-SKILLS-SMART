---
name: additional-fields-via-smart-extensiondata
description: Use when a .NET contract class must accept, keep and round-trip unknown JSON fields (additionalProperties) with typed access, via the SmartExtensionBag pattern (STJ + Newtonsoft, zero config).
---

# Additional Fields via Smart Extension Data

A norm for making .NET data contracts **open and extensible**: well-known fields stay typed CLR
properties, everything else a payload delivers is kept losslessly in an `AdditionalFields` bag,
written back flat on the root level, and readable with a typed `GetField<T>`.
It works with **System.Text.Json and Newtonsoft.Json at default settings, without any consumer configuration**.

This skill tells you **when** to apply the norm, **how** to apply it to a contract class, **how to use it**
(consumers and adapters), **how to test it** and — most important for AI agents — **which "simplifications"
must not be made**, because they were evaluated and rejected.

Reference implementation (read these before changing anything):

| File | Role |
|---|---|
| `references/SmartExtensionBag.cs` | internal bag: owner-aware redirect, ordinal keys, eager normalization. Its class comment holds the full rationale (R1–R8, D1–D12, L1–L5). |
| `references/SmartValueConverter.cs` | internal, stateless conversion chain for typed reads (reusable for envelopes / side channels). |
| `references/WorkItem.AdditionalFields.cs` | the canonical usage pattern on a real contract. Copy this for new contracts. |

If the `references/` folder is not available, the sources live in the contract assembly next to the
contracts (`SmartExtensionBag.cs`, `SmartValueConverter.cs`). Never re-invent them from this description.

---

## 1. When to use / when not

**Use it when**
- a contract (DTO, work item, request/response object, envelope) comes from or goes to a web interface and
  the provider may deliver **more fields than the contract declares** (Trello custom fields, Azure DevOps
  schema extensions, GitHub/Outlook specifics, newer contract versions);
- those extra fields must **survive a roundtrip** (load → modify → save back) or be **usable by code that
  knows about them**;
- the contract must stay **precisely declared** in OpenAPI/Swagger/UJMW (typed properties, not a map).

**Do not use it when**
- the payload is entirely dynamic with no well-known part → use a plain `Dictionary<string, object>`
  (canonicalized via the same rules) or `JsonElement`/`JToken` locally;
- unknown fields must be **rejected** (strict contracts) → use the serializers' "disallow unmapped members"
  options instead;
- you only need a single opaque blob → declare a `string`/`object` property for it explicitly.

---

## 2. The norm (exactly this, nothing else)

For a contract class `X`:

1. The class is `partial`.
2. Add **one** file `X.AdditionalFields.cs`.
3. It declares **one** property named `AdditionalFields` of type `IDictionary<string, object>`, annotated with
   **both** `[Newtonsoft.Json.JsonExtensionData]` and `[System.Text.Json.Serialization.JsonExtensionData]`,
   backed by a lazily created `SmartExtensionBag`, with a setter that goes through `SmartExtensionBag.Adopt`.
4. Optionally (recommended for central contracts) the unified accessors `GetField<T>`, `TryGetField<T>`,
   `SetField<T>` and the string indexer.

Minimal template:

```csharp
using System.Collections.Generic;
using System.Diagnostics;

namespace My.Contracts {

  public partial class X {

    [DebuggerBrowsable(DebuggerBrowsableState.Never)]
    private SmartExtensionBag _AdditionalFields = null;

    /// <summary>
    /// All JSON members NOT covered by a well-known property ("additionalProperties").
    /// Filled by the serializers, written back flat on the root level.
    /// </summary>
    [Newtonsoft.Json.JsonExtensionData]
    [System.Text.Json.Serialization.JsonExtensionData]
    public IDictionary<string, object> AdditionalFields {
      get {
        return this.AdditionalFieldsBag;
      }
      set {
        _AdditionalFields = SmartExtensionBag.Adopt(this, value); // serializer- or code-assigned dictionaries
      }
    }

    [DebuggerBrowsable(DebuggerBrowsableState.Never)]
    private SmartExtensionBag AdditionalFieldsBag {
      get {
        if (_AdditionalFields == null) {
          _AdditionalFields = new SmartExtensionBag(this); // 'this' is not allowed in field initializers
        }
        return _AdditionalFields;
      }
    }

    public TValue GetField<TValue>(string fieldName, TValue defaultValue = default(TValue)) {
      object value;
      return this.AdditionalFieldsBag.TryGetField(fieldName, typeof(TValue), out value) ? (TValue)value : defaultValue;
    }

    public bool TryGetField<TValue>(string fieldName, out TValue value) {
      object rawValue;
      if (this.AdditionalFieldsBag.TryGetField(fieldName, typeof(TValue), out rawValue)) {
        value = (TValue)rawValue;
        return true;
      }
      value = default(TValue);
      return false;
    }

    public void SetField<TValue>(string fieldName, TValue value) {
      this.AdditionalFieldsBag[fieldName] = value;
    }

  }

}
```

Rules for the template:
- **Never** initialize the bag in a field initializer (`this` is not allowed there) and **never** let the getter return `null`
  (otherwise serializers create their own `Dictionary` and bypass redirect + normalization).
- The bag identifies its property **by the `[JsonExtensionData]` attributes**, not by the name. The name
  `AdditionalFields` is the human/tooling convention (mirrors OpenAPI `additionalProperties`); keep it.
- One bag property per class (both serializers allow only one). Subclasses inherit it; do not add a second one.
- `SmartExtensionBag` / `SmartValueConverter` are `internal`. The public surface is only `IDictionary<string, object>`
  and the accessors. Do not make them public and do not expose the bag type in signatures.
- System.Text.Json is in-box on .NET Core 3.0+/.NET 5+. For .NET Framework / netstandard2.0 either add the
  System.Text.Json package or wrap the STJ attribute in `#if NETCOREAPP3_0_OR_GREATER`. Old frameworks also need a
  reference to `System.Numerics` (BigInteger normalization).
- Code style of the reference: C# 7.3 compatible (no `??=`, no switch expressions, no `is not`), braces on the
  same line, 2-space indent, `_PascalCase` private fields, regions named `" Name "`.

---

## 3. Requirements the design satisfies (check every alternative against ALL of them)

| ID | Requirement |
|---|---|
| R1 | **Contract-first**: well-known fields are typed properties; OpenAPI/UJMW shows the real contract plus `additionalProperties` — never an untyped map. |
| R2 | **Open contract**: unknown members survive deserialize → serialize and are usable by code that knows them. |
| R3 | **Zero configuration**: works with default STJ and Newtonsoft settings. No converters, resolvers, modifiers, options. |
| R4 | **Serializer neutrality**: deserialize with one serializer, serialize with the other (mixed pipelines). No JsonElement/JToken/JsonNode is ever stored. |
| R5 | **Transport never mutates data**: key casing is never changed, no naming policy on extension keys, no reformatting by guessing. |
| R6 | **The consumer decides the type**: JSON cannot carry CLR types; typing happens lazily on read via the generic argument (or an adapter's field schema). |
| R7 | **Minimal dependencies**: BCL + Newtonsoft.Json; STJ only where in-box. |
| R8 | **Fail loud**: no silent rounding, dropping or guessing; exceptions name field and types. |

---

## 4. Decisions and REJECTED alternatives — do not regress

Before proposing a "simpler" or "more modern" solution, read this table. Each of these was evaluated.

| Tempting alternative | Why it is rejected |
|---|---|
| Let the contract class implement `IDictionary<string, object>` ("dictionary hijack") | Serializers treat the whole class as a map → OpenAPI shows an untyped dictionary (R1); attributes/converters/options on well-known properties are ignored; the class needs its own mini-serializer for nested well-known properties (e.g. `LinkedItem[]` arriving as JObject/JsonElement). This was the predecessor of this norm. |
| `ExpandoObject` / `DynamicObject` | No typed well-known properties (R1); same untyped-object problem. |
| Custom `JsonConverter`, `ContractResolver`, STJ `TypeInfoResolver` modifier | Every consumer must register it (R3). |
| A naming convention ("serializers recognize a property called X") | No serializer follows it → still configuration (R3). |
| Store raw `JsonElement` / `JToken` and convert lazily | Newtonsoft cannot serialize JsonElement, STJ cannot serialize JObject meaningfully (R4). Converting on every read returns copies → mutations of nested values are silently lost (R8); fixing that needs a cache = eager normalization. |
| `Dictionary<string, JsonElement>` / `JsonObject` as bag type | STJ-only; breaks Newtonsoft and mixed pipelines (R4). |
| Type hints in the payload (`Field`T`, `$type`) | Mutates the wire format, duplicates keys on re-serialization, and `Type.GetType` from payload data is a deserialization vulnerability (TypeNameHandling class). Never reintroduce. |
| Guessing rich types on insert ("looks like a Guid/date → store Guid/DateTime") | Reformatting changes data (casing, braces, offsets) (R5) and is pointless: the next serializer turns it back into a string. |
| CamelCase/naming policies to "normalize" casing | Destroys dictionary keys coming from other layers (R5). Note: Newtonsoft's `CamelCasePropertyNamesContractResolver` sets `ProcessDictionaryKeys = true` — avoid it for these contracts. |
| Case-insensitive bag (`OrdinalIgnoreCase`) | "foo" and "Foo" are different data; merging them loses one (R5). |
| Exposing well-known properties through the bag too | `Keys`/`Count`/enumerator would disagree with the indexer and serializers would write keys twice. |
| Returning `default(T)` on conversion problems | Hides contract violations (R8). |

Core decisions (details in the class comment of `SmartExtensionBag`):
- **D1** One bag property with double `[JsonExtensionData]` — the only zero-config mechanism both serializers share.
- **D2** Owner-aware redirect: a key matching a well-known property case-insensitively (e.g. `"title"` from STJ, which is
  case-sensitive by default) is converted and written into the property, never stored. Exact match wins; ambiguous match throws.
- **D3** Bag keys are ordinal (case-sensitive).
- **D4** Eager normalization on insert into canonical JSON values.
- **D5** Only the JSON type system is stored (string, number, bool, null, object, array).
- **D6** Value semantics on insert (deep copy of containers; POCOs by reference).
- **D7** Honest `IDictionary`: the bag shows extension entries only; unified access goes via the owner.
- **D8** Well-known member detection mirrors the serializers: `[JsonIgnore]` (STJ only `Condition = Always`) → not
  well-known; renames (`[JsonProperty]`, `[JsonPropertyName]`) matched by both wire names; read-only properties → incoming values discarded.
- **D9** Last write wins if a payload has both `"Title"` and `"title"`.
- **D10** Redirect conversion failure throws (`InvalidCastException`).
- **D11** No type information in the payload, ever.
- **D12** Bag and converter are `internal`.

---

## 5. Behavior reference

### 5.1 Canonical storage (what is inside `AdditionalFields`)

| Incoming | Stored as |
|---|---|
| JSON object / `JObject` / `JsonElement` Object / any `IDictionary` | `Dictionary<string, object>` (ordinal, recursively canonical) |
| JSON array / `JArray` / any `IEnumerable` (not string) | `List<object>` |
| integer fitting Int64; `int`, `short`, `byte`, `uint`, … | `long` |
| other numbers | `decimal` (STJ input keeps scale: `1.50` stays `1.50`), else `double`; out of double range → raw text |
| `float` | `double` (shortest roundtrip form) |
| string | `string` — **no** guessing |
| bool / null | `bool` / `null` |
| `Guid` | `"D"` string |
| `DateTime` / `DateTimeOffset` | ISO 8601 `"o"` string |
| `TimeSpan` | `"c"` string |
| `DateOnly` / `TimeOnly` | `yyyy-MM-dd` / `HH:mm:ss.FFFFFFF` |
| enum | member name (flags `"A, B"`) |
| `byte[]` | base64 string |
| `Uri` / `Version` | original string / version text |
| custom POCO set by code | kept by reference (serializable by every serializer) |

### 5.2 Typed read resolution (`GetField<T>` / `TryGetField<T>`)

Field lookup: **well-known property** (exact wire name, then unique case-insensitive) → **exact bag key** →
**unique case-insensitive bag key** → otherwise not found (`defaultValue` / `false`). Ambiguity throws `AmbiguousMatchException`.

Conversion chain (`SmartValueConverter`, first match wins):
1. `T == object` → the stored (live) instance.
2. `null` → null for reference/nullable types; **throws** for non-nullable value types.
3. already assignable → returned as is.
4. canonicalize (serializer DOM types resolved; POCO → JSON shape).
5. object → `string` (JSON text) / dictionary types (recursive) / POCO via hardened Newtonsoft fallback.
6. array → `string` (JSON text) / `T[]`, `List<T>`, collection interfaces, `ISet<T>`, concrete `ICollection<T>` (recursive).
7. scalar → `string` (invariant; `true`/`false` lowercase).
8. scalar → **enum** (before Convert): `[EnumMember]`/`[JsonStringEnumMemberName]` aliases, exact name, case-insensitive name, flags, numeric text, integral numbers.
9. scalar → `DateTime` (RoundtripKind) / `DateTimeOffset` from strings only (numbers are never timestamps).
10. scalar → numeric primitives via `Convert` (invariant); numeric text parsed as decimal first; **fractional → integer throws** (no banker's rounding); overflow throws; bool never becomes a number.
11. `bool` from `"true"/"false"` text only; `char` from one-character strings.
12. string → static `TryParse(string, IFormatProvider, out T)` then `TryParse(string, out T)` (Guid in all formats, TimeSpan, Version, IPAddress, custom types).
13. `TypeConverter` (target `CanConvertFrom`, then source `CanConvertTo`).
14. hardened Newtonsoft fallback (`TypeNameHandling.None`, metadata ignored, no date/float guessing, private settings).
15. failure → `InvalidCastException` with field name, source and target type (value itself not included → no data in logs).

### 5.3 Known and accepted limitations

- **L1** Numbers keep value, not notation (`1e3` → `1000`). Newtonsoft input arrives as double already (`1.50` → `1.5`).
- **L2** Newtonsoft date guessing happens before the bag; such values are written back as ISO 8601 `"o"`. Use `DateParseHandling.None` where exact text matters.
- **L3** Numbers outside double range (STJ) stay raw text.
- **L4** Not thread-safe (like `Dictionary`).
- **L5** Case-insensitive fallback lookup is linear (bags are small).

---

## 6. Using it

### Consumers

```csharp
WorkItem item = JsonSerializer.Deserialize<WorkItem>(json);      // or Newtonsoft — no options needed

int storyPoints   = item.GetField<int>("StoryPoints");            // long in the bag → int
Guid sprintId     = item.GetField<Guid>("Trello.SprintId");        // string in the bag → Guid
Priority prio     = item.GetField<Priority>("Priority");           // name or number → enum
string[] labels   = item.GetField<string[]>("Labels", new string[0]);
SprintInfo sprint = item.GetField<SprintInfo>("Sprint");           // object → POCO

if (item.TryGetField<DateTime>("DueDate", out DateTime due)) { /* ... */ }

item.SetField("Trello.SprintId", Guid.NewGuid());                  // stored as "D" string
item.SetField("title", "New title");                               // redirected → item.Title

object raw = item["Meta"];                                         // live canonical value (Dictionary/List/…)
```

Guidelines:
- Always read **typed** (`GetField<T>`) when the type matters. `object`/indexer access yields canonical values
  (`long`, `decimal`, `string`, …) by design — never compare them against `int` or `Guid` instances.
- `defaultValue` applies only when the field is **absent**. A present field with a wrong shape throws.
- Values read as `object` or as their canonical container type are **live** (mutations stay); values converted to
  another shape (e.g. `IDictionary<string, int>`, `List<string>`) are **snapshots**. Write back with `SetField`.
- Containers passed to `SetField` are deep-copied; later changes to your instance do not affect the item.

### Adapters (Trello, Azure DevOps, GitHub, Outlook, …)

- Map what the universal contract knows to well-known properties; put everything provider-specific into
  `AdditionalFields` with **stable, provider-prefixed keys** where collisions are possible (e.g. `"Trello.SprintId"`).
- Build items from raw provider dictionaries with `FromDictionary<TConcrete>(dict)` / assigning `AdditionalFields`:
  well-known keys (any casing) fill properties, the rest becomes additional fields — one code path.
- The **type knowledge belongs to the adapter's field schema** (Trello custom field types, Azure DevOps field
  definitions). Use the contract's `Class` (e.g. `"Trello.Card"`, `"AzureDevOps.Bug"`) as the key to that schema and
  read with the matching `GetField<T>`. Never encode types into the payload.
- Do not keep provider DOM objects (`JObject`, `JsonElement`) around — write them into the bag; it canonicalizes.

### Reusing the converter elsewhere

`SmartValueConverter.ConvertTo<T>(value, fieldName)` is stateless and serializer-neutral. Use it for envelopes,
side channels or any `Dictionary<string, object>` with dynamic keys instead of writing ad-hoc conversions.

---

## 7. Test checklist (minimum for a new contract or a change to bag/converter)

Run every serialization test with **both** serializers at **default** settings.

Roundtrip and wire format
- [ ] Unknown members survive STJ → STJ, Newtonsoft → Newtonsoft, **STJ → Newtonsoft**, **Newtonsoft → STJ**; output is valid JSON with members at root level.
- [ ] No key is written twice; no `AdditionalFields` wrapper object appears in the output.
- [ ] Key casing of extension members is preserved exactly; `"foo"` and `"Foo"` stay two entries.
- [ ] After STJ deserialization the bag contains **no** `JsonElement`; after Newtonsoft **no** `JToken`.
- [ ] The serializers reuse the bag instance (getter never null) — assert the instance type after deserialization.

Redirect
- [ ] STJ default (case-sensitive) with `"title"` fills `Title` and does not appear in `AdditionalFields`.
- [ ] Nested well-known properties (e.g. `"linkedItems"` → `LinkedItem[]`) are redirected correctly.
- [ ] Redirect of an unconvertible value throws `InvalidCastException` naming the field.
- [ ] `[JsonIgnore]` property names stay in the bag; read-only property values are discarded; renamed properties match their wire names.

Typed reads
- [ ] `long` → `int`, overflow throws; `1.5` → `int` throws; `"42"` → `int`; `1e3` → `int` = 1000.
- [ ] Guid from `"D"`, uppercase, braces, `"N"`; DateTime keeps `Kind`; DateTimeOffset keeps offset.
- [ ] Enum from name, other casing, flags, number, `[EnumMember]` alias; bool never from numbers.
- [ ] `string[]`, `List<T>`, `IReadOnlyList<T>`, `Dictionary<string, T>`, POCO from nested objects.
- [ ] JSON null → `int` throws, → `int?` returns null; absent field → `defaultValue`.
- [ ] Ambiguous case-insensitive key throws `AmbiguousMatchException`.

Insert
- [ ] `SetField` with Guid/DateTime/enum/byte[] stores canonical strings; containers are deep-copied.
- [ ] Cyclic container graph → exception (max depth 64), no stack overflow.

---

## 8. Agent rules (short form)

1. **Apply the norm exactly** (section 2). Do not invent variants, do not rename the property, do not add a second bag.
2. **Never** store or return `JsonElement`, `JToken`, `JsonNode` from the bag; never put type hints into payloads;
   never call `Type.GetType` with payload data.
3. **Never** add naming policies, case-insensitive bag comparers or "smart" type guessing on insert.
4. **Never** swallow conversion errors or replace them with defaults.
5. Read typed with `GetField<T>`; let adapters own the type knowledge via their field schema.
6. Before replacing or "simplifying" the mechanism, prove the alternative against **all** of R1–R8 and section 4.
   If it fails one, keep the norm.
7. When modifying `SmartExtensionBag` or `SmartValueConverter`, keep their class comments (requirements,
   decisions, limitations) in sync and extend the test checklist.
