---
name: csharp-coding-standards
description: MANDATORY implementation constraint for writing, generating, reviewing or refactoring any C# source code - conservative, explicit C# coding standards ("syntactic salt over syntactic sugar"). Covers no var, explicit types, defensive nullability, _PascalCase fields and constants, this. qualification, expanded properties, XML summary documentation, brace and else/catch/finally layout, no ternary operator, parenthesized lambda parameters, arrays at API boundaries, purposeful exception handling with DevLogger.LogError, trace logging via DevLogger.LogTrace, no async/await by default, Newtonsoft.Json instead of System.Text.Json, no P/Invoke without approval, reserved placeholder syntaxes (_FOO_, #FOO#, {Foo}, [Foo]; prohibited (Foo), %FOO%), migration of existing code, documentation and test deliverables, and the leading .editorconfig (assets) mirrored into every repository. Load before producing C# code and use its agent checklist before returning code.
metadata:
  version: "2026-10-21"
---

# C# Coding Standards and AI Implementation Rules

**Version:** 1.0  
**Status:** Initial desired-state baseline  
**Scope:** C# source-code implementation standards  
**Primary audience:** Software developers and AI coding agents

------

## 1. Purpose and Design Philosophy

This document defines the desired state for C# source code produced or maintained within projects adopting these standards.

The rules are intentionally conservative. They prioritize:

- static type safety;
- compile-time validation;
- explicit and predictable behavior;
- readability and maintainability;
- long-term source-code stability;
- platform independence;
- uniformity across developers, projects, and AI-generated code.

The fundamental principle is:

> **Prefer syntactic salt over syntactic sugar.**

A language feature is not desirable merely because it reduces the number of characters or is considered modern or idiomatic. Explicit code is preferred when it makes types, control flow, state, ownership, or failure behavior easier to understand and verify.

C#'s strong static type system is considered a major engineering advantage. Language features that weaken, obscure, bypass, or unnecessarily defer type information should therefore be treated conservatively.

These standards deliberately favor established, conservative C# and enterprise software-engineering practices over trend-driven language constructs.

Uniformity itself is considered a quality attribute.

------

## 2. Normative Language and Rule Priority

The following terms are used throughout this document:

| Term           | Meaning                                                      |
| -------------- | ------------------------------------------------------------ |
| **MUST**       | Mandatory rule. Deviations require an explicit project-specific reason. |
| **MUST NOT**   | Prohibited by default.                                       |
| **SHOULD**     | Strong recommendation. Deviations should have a concrete technical reason. |
| **SHOULD NOT** | Avoid unless a concrete technical reason justifies its use.  |
| **MAY**        | Permitted where appropriate.                                 |

When rules appear to conflict, apply the following priorities:

1. correctness;
2. compile-time safety and static typing;
3. explicit project-specific requirements;
4. these coding standards;
5. readability and maintainability;
6. syntactic convenience.

Project-specific rules MAY strengthen these standards. They SHOULD NOT silently weaken them.

------

## 3. Language and Communication

### 3.1 Developer and Prompt Communication

Human-facing communication MAY use the language appropriate for the project or conversation.

For AI-assisted development in a German-speaking project context, prompt communication SHOULD be in German unless explicitly requested otherwise.

### 3.2 Source Code Language

All source-code identifiers, comments, XML documentation, exception messages intended for developers, and logging messages MUST be written in English.

```csharp
// Validate the configured endpoint before opening the connection.
if (endpoint == null) {
  throw new InvalidOperationException("The endpoint has not been configured.");
}
```

Mixed-language source code SHOULD NOT be introduced.

------

## 4. General Coding Principles

### 4.1 Explicitness

Code MUST favor explicit constructs over unnecessary inference or abbreviated syntax.

The reader SHOULD be able to determine important types, control flow, state transitions, and error behavior without mentally expanding syntactic shortcuts.

### 4.2 Static Typing

Static typing MUST be preserved wherever reasonably possible.

Features that unnecessarily move errors from compile time to runtime SHOULD NOT be used.

Dynamic typing or equivalent mechanisms MUST NOT be introduced merely for convenience.

### 4.3 Maintainability Over Brevity

Reducing line count is not a design objective.

Code SHOULD be optimized for:

- comprehensibility;
- debugging;
- refactoring safety;
- compiler assistance;
- predictable maintenance.

### 4.4 Modern Language Features

New C# language features MUST NOT be adopted solely because they are newer or shorter.

They MAY be used when they provide a concrete improvement without reducing explicitness, static safety, readability, or maintainability.

Where an explicit established construct and a syntactically abbreviated construct provide equivalent behavior, the explicit construct SHOULD generally be preferred.

------

## 5. Types and Variable Declarations

### 5.1 Do Not Use `var`

`var` MUST NOT be used.

The declared type MUST be explicit.

```csharp
Customer customer = repository.GetCustomer(customerId);
Dictionary<string, Customer> customers = new Dictionary<string, Customer>();
```

Do not write:

```csharp
var customer = repository.GetCustomer(customerId);
var customers = new Dictionary<string, Customer>();
```

This applies even where the inferred type appears obvious from the right-hand side.

### 5.2 Nullable Types

Nullability MUST be treated defensively.

Nullable types SHOULD only be used where `null` is genuinely part of the semantic or technical value domain.

Do not introduce nullable types merely to suppress compiler concerns or avoid designing explicit state handling.

```csharp
Customer customer = repository.GetRequiredCustomer(customerId);
```

A nullable result MAY be appropriate when absence is an intentional result:

```csharp
Customer? customer = repository.FindCustomer(customerId);
```

Nullable annotations and nullable operators SHOULD NOT spread through an API without a concrete semantic reason.

### 5.3 Null-Coalescing Operators

The null-coalescing operator (`??`) and related abbreviated null-handling constructs SHOULD be avoided when they obscure why a value may be `null` or silently replace an invalid state.

Prefer explicit validation when absence represents an exceptional or invalid state.

```csharp
if (configuration == null) {
  throw new InvalidOperationException("The configuration is required.");
}

this.Configuration = configuration;
```

Use null-coalescing only where fallback semantics are intentional, obvious, and part of the actual design.

------

## 6. Naming and Member Conventions

### 6.1 Fields

Class fields MUST use PascalCase with an underscore prefix.

```csharp
private readonly ICustomerRepository _CustomerRepository;
private string _DisplayName;
```

### 6.2 Constants

Constants MUST use PascalCase with an underscore prefix.

```csharp
private const Int32 _MaximumRetryCount = 3;
```

### 6.3 Instance Member Qualification

Access to instance members other than fields SHOULD explicitly use `this.`.

```csharp
this.ValidateCustomer(customer);
this.CustomerName = customer.Name;
```

Fields already have the `_` prefix and therefore MUST NOT require `this.` for disambiguation.

```csharp
_CustomerRepository.Save(customer);
```

The purpose is to make instance access visually explicit and distinguish it from local variables, parameters, and static members.

------

## 7. Properties

Properties MUST be written in an explicit, expanded form.

Abbreviated or expression-bodied properties SHOULD NOT be used.

```csharp
private String _Name;

public String Name {
  get {
    return _Name;
  }
  set {
    _Name = value;
  }
}
```

Do not prefer syntactic compactness such as:

```csharp
public String Name { get; set; }
```

or:

```csharp
public String Name => _Name;
```

The expanded form provides a uniform location for future validation, tracing, debugging, or behavioral extensions.

------

## 8. Methods and Documentation

### 8.1 XML Documentation

Methods MUST have XML `<summary>` documentation.

Parameters and return values SHOULD additionally be documented where their semantics are not completely self-evident.

```csharp
/// <summary>
/// Loads the customer identified by the specified customer identifier.
/// </summary>
/// <param name="customerId">The unique customer identifier.</param>
/// <returns>The loaded customer.</returns>
public Customer LoadCustomer(Int64 customerId) {
  return _CustomerRepository.GetCustomer(customerId);
}
```

Public APIs SHOULD be documented comprehensively.

Other important artifacts such as classes, interfaces, properties, events, and delegates SHOULD receive XML documentation where it adds semantic information.

### 8.2 Comments

Comments SHOULD be used generously where they improve understanding.

Comments SHOULD explain:

- intent;
- constraints;
- non-obvious decisions;
- compatibility requirements;
- important invariants;
- reasons for unusual implementation choices.

Comments SHOULD NOT merely translate an obvious statement into prose.

```csharp
// Keep validation before repository access because invalid identifiers
// must never result in an external data-source call.
this.ValidateCustomerId(customerId);
```

------

## 9. Control Flow and Formatting

### 9.1 Braces

All control-flow blocks MUST use braces, including single-line blocks.

```csharp
if (customer == null) {
  return;
}
```

Do not write:

```csharp
if (customer == null)
  return;
```

### 9.2 Opening Braces

Opening brackets - above all curly braces - MUST always be placed at the end of a line: on the same line as the associated declaration (namespace, type, method, constructor, property, accessor) or control-flow statement, never on a line of their own. This applies to every example in this document.

When a parenthesized list (parameters, arguments) is wrapped over several lines, the opening parenthesis likewise stays at the end of the first line.

```csharp
public class CustomerService {

  public void ProcessCustomer(Customer customer) {
    if (customer != null) {
      this.ProcessCustomer(customer);
    }
  }

}
```

Do not write:

```csharp
if (customer != null)
{
  this.ProcessCustomer(customer);
}
```

### 9.3 `else`, `else if`, `catch`, and `finally`

`else`, `else if`, `catch`, and `finally` MUST directly follow the preceding closing brace without an intervening line break, and their opening brace ends the same line (`} else {`).

```csharp
if (customer != null) {
  this.ProcessCustomer(customer);
} else {
  this.ProcessMissingCustomer();
}
```

Exception handling follows the same style:

```csharp
try {
  this.ProcessCustomer(customer);
} catch (ExpectedCustomerException ex) {
  DevLogger.LogError(ex);
  throw;
} finally {
  this.ReleaseResources();
}
```

### 9.4 Inline Conditional Expressions

Inline conditional expressions, including the ternary `?:` operator, MUST NOT be used for normal control flow.

Prefer explicit branches.

```csharp
String displayName;

if (customer.HasDisplayName) {
  displayName = customer.DisplayName;
} else {
  displayName = customer.Name;
}
```

### 9.5 Lambdas

Lambda parameters MUST be enclosed in parentheses.

```csharp
customers.Where((customer) => customer.IsActive);
```

Do not write:

```csharp
customers.Where(customer => customer.IsActive);
```

Lambda expressions SHOULD remain simple. Complex behavior SHOULD be extracted into named methods.

### 9.6 Indentation, Line Endings and Encoding

- Indentation MUST use **two spaces** per level; no tabs. This applies to every example in this document.
- Line endings are CRLF (the landscape is developed almost exclusively on Windows).
- File encoding: UTF-8; **with BOM is preferred** on Windows, but not enforced - do not convert existing files only to add or remove a BOM. Exceptions where tools require no BOM (e.g. npm `package.json`) take precedence.

------

## 10. Collections and Method Signatures

When collections are transported through method or API signatures, arrays SHOULD be preferred over `List<T>`.

```csharp
public Customer[] LoadCustomers() {
  // ...
}
```

Prefer:

```csharp
public void ProcessCustomers(Customer[] customers) {
  // ...
}
```

over:

```csharp
public void ProcessCustomers(List<Customer> customers) {
  // ...
}
```

`List<T>` MAY be used internally where dynamic collection construction or mutation is appropriate.

```csharp
List<Customer> customers = new List<Customer>();

// Build the collection dynamically.
customers.Add(customer);

return customers.ToArray();
```

A mutable implementation type SHOULD NOT unnecessarily become part of an API contract.

------

## 11. Properties, State, and Side Effects

State changes SHOULD be explicit and easy to locate.

Hidden side effects SHOULD be avoided.

Methods SHOULD have clear responsibilities and SHOULD NOT rely on implicit mutation that is difficult to discover from their names and signatures.

Where state transitions are important, explicit methods and explicit assignments SHOULD be preferred over compressed expressions.

------

## 12. Exception Handling

### 12.1 General Principle

Exception handling MUST be purposeful.

Do not introduce `try`/`catch` blocks merely as defensive boilerplate.

An exception SHOULD be caught only when the current layer can intentionally:

- handle it;
- translate it;
- enrich it with meaningful context;
- perform required recovery or cleanup;
- log an expected failure at the appropriate boundary.

Unexpected exceptions SHOULD normally propagate to the established global error-handling boundary.

### 12.2 Logging Expected Exceptions

Exceptions that are deliberately caught and are relevant for diagnostics MUST be logged using:

```csharp
DevLogger.LogError(ex);
```

Example:

```csharp
try {
  this.OpenConfigurationFile(fileName);
} catch (FileNotFoundException ex) {
  DevLogger.LogError(ex);
  this.CreateDefaultConfiguration();
}
```

Do not add catch blocks solely to call `DevLogger.LogError(ex)` and immediately rethrow unless the current architectural boundary explicitly requires that logging.

### 12.3 Catch-All Blocks

Broad catch-all blocks SHOULD NOT be used.

```csharp
catch (Exception ex)
```

is only appropriate when the layer has a genuine responsibility for handling arbitrary failures, such as a defined application boundary.

### 12.4 Error States

Expected business or state conditions SHOULD be modeled explicitly rather than routinely using exceptions as control flow.

------

## 13. Trace Logging

Trace output MUST use the project logging abstraction rather than ad-hoc console output.

The standard trace form is:

```csharp
DevLogger.LogTrace(0, 99999, "Customer processing started.");
```

The numeric values `0` and `99999` MAY remain constant unless a more specific project rule defines different identifiers.

Messages MUST be written in English.

Trace logging SHOULD provide diagnostic value and SHOULD NOT flood the log with trivial implementation details.

------

## 14. Asynchronous Programming

The `async`/`await` pattern MUST NOT be introduced by default.

Synchronous and explicit execution flow is the standard.

Asynchronous programming MAY only be introduced when a concrete technical requirement makes it necessary or provides a clearly demonstrated architectural benefit.

It MUST NOT be introduced merely because an underlying framework offers asynchronous variants or because asynchronous APIs are currently fashionable.

Where asynchronous behavior is genuinely required, its use SHOULD be treated as an explicit architectural decision rather than a local stylistic choice.

------

## 15. Serialization

Newtonsoft.Json MUST be used as the standard JSON implementation.

```csharp
using Newtonsoft.Json;

String json = JsonConvert.SerializeObject(customer);
Customer deserializedCustomer = JsonConvert.DeserializeObject<Customer>(json);
```

`System.Text.Json` SHOULD NOT be introduced unless a project-specific technical requirement explicitly mandates it.

A project SHOULD avoid mixing JSON frameworks without a strong reason.

------

## 16. Platform Independence and Interoperability

Managed and platform-independent implementations MUST be preferred.

Windows-specific interoperability, including P/Invoke and `DllImport`, SHOULD NOT be introduced when a portable managed alternative exists.

Before introducing an implementation that necessarily depends on Windows-specific interoperability, explicit approval MUST be obtained from the responsible project owner.

Platform-specific code MUST NOT be introduced merely as a shortcut.

When unavoidable, platform dependencies SHOULD be isolated behind a clearly defined abstraction.

------

## 17. Dependency and API Design

Dependencies SHOULD be explicit.

Public and cross-component contracts SHOULD expose stable, intentional types rather than implementation details.

APIs SHOULD:

- make required inputs explicit;
- avoid unnecessary nullable states;
- avoid exposing mutable implementation collections;
- provide predictable failure behavior;
- preserve static type information;
- minimize hidden conventions.

Convenience SHOULD NOT outweigh contract clarity.

------

### 17.1 Placeholder and Magic-Value Conventions

Technical placeholders and runtime-resolved magic values MUST use a clearly defined syntax. The syntax itself is part of the contract and MUST NOT be changed for readability or personal preference.

| Syntax  | Purpose                                                   | Rule                                                         |
| ------- | --------------------------------------------------------- | ------------------------------------------------------------ |
| `_FOO_` | Deployment pipeline / tokenization                        | Reserved for deployment-time replacement.                    |
| `#FOO#` | Deployment pipeline / tokenization                        | Reserved for deployment-time replacement.                    |
| `(Foo)` | Human-readable placeholder notation                       | **PROHIBITED.** It is too easily mistaken for ordinary prose and does not make the technical placeholder semantics sufficiently explicit. |
| `%FOO%` | Environment-variable-like placeholder notation            | **PROHIBITED.** It is ambiguous and dangerous because `%...%` is conventionally associated with Windows environment-variable expansion. |
| `{Foo}` | Runtime substring replacement by business logic           | Reserved exclusively for substring replacement performed during later business-logic processing at runtime. Any replacement before runtime violates the reserved meaning of this syntax. |
| `[Foo]` | Fixed magic value selecting a runtime resolution strategy | This is **not** a substring placeholder. It represents a complete, enum-like strategy selector such as `[Auto]`, `[Default]`, `[Legacy]`, `[Discover]`, or `[AskUser]`. It MUST represent the complete value and MUST NOT be embedded as only part of a larger string. |

#### 17.1.1 Deployment Placeholders

`_FOO_` and `#FOO#` are reserved for deployment pipeline or tokenization scenarios.

They MUST NOT be reused for runtime business-logic substitution.

Example:

```csharp
private const String _ServiceEndpointTemplate = "https://#HOST#/api/_VERSION_";
```

The actual replacement mechanism is deployment-specific and outside the scope of this coding-standard document.

#### 17.1.2 Runtime Substring Placeholders

Curly-brace placeholders use the following form:

```text
{Foo}
```

They are reserved for substring replacement performed by business logic during runtime.

Example:

```csharp
String template = "Hello {UserName}";
```

A preprocessing step, deployment pipeline, build target, configuration transform, or tokenization tool MUST NOT consume `{UserName}` before the application reaches the runtime processing stage for which the placeholder was designed.

This distinction is intentional. The syntax communicates lifecycle ownership.

#### 17.1.3 Runtime Strategy Magic Values

Square-bracket notation represents a complete strategy-selection value rather than a replaceable substring.

Valid examples include:

```text
[Auto]
[Default]
[Legacy]
[Discover]
[AskUser]
```

These values MAY be transported as strings where a textual configuration boundary requires it, but their semantics are enum-like: they select a defined strategy for dynamically obtaining or interpreting the actual value.

A square-bracket magic value MUST occupy the complete semantic value.

Valid:

```csharp
String sourceMode = "[Auto]";
```

Invalid:

```csharp
String fileName = "customer-[Auto].json";
```

If a type-safe enum or equivalent strongly typed contract can be used without crossing a textual configuration boundary, the strongly typed representation SHOULD be preferred. Square-bracket magic values exist for cases where a textual protocol or configuration representation is intentionally required.

#### 17.1.4 Reserved Meanings

The placeholder syntaxes defined in this section are reserved conventions.

An AI agent MUST NOT:

- invent additional meanings for these syntaxes;
- substitute one syntax for another because it appears more readable;
- pre-resolve `{Foo}` placeholders during deployment or build processing;
- use `[Foo]` as a partial-string replacement marker;
- use `(Foo)` or `%FOO%` as technical placeholders.

When an existing codebase uses one of the prohibited forms, migration SHOULD preserve behavior first and then replace the notation with the correct reserved syntax in a separate, reviewable change where practical.

## 18. Reference Example

The following example demonstrates several rules together.

```csharp
public class CustomerService {
  private readonly ICustomerRepository _CustomerRepository;

  /// <summary>
  /// Initializes a new customer service.
  /// </summary>
  /// <param name="customerRepository">The repository used for customer persistence.</param>
  public CustomerService(ICustomerRepository customerRepository) {
    if (customerRepository == null) {
      throw new ArgumentNullException(nameof(customerRepository));
    }

    _CustomerRepository = customerRepository;
  }

  /// <summary>
  /// Loads all active customers.
  /// </summary>
  /// <returns>An array containing all active customers.</returns>
  public Customer[] LoadActiveCustomers() {
    DevLogger.LogTrace(0, 99999, "Loading active customers.");

    Customer[] customers = _CustomerRepository.LoadCustomers();
    List<Customer> activeCustomers = new List<Customer>();

    foreach (Customer customer in customers) {
      if (customer.IsActive) {
        activeCustomers.Add(customer);
      }
    }

    return activeCustomers.ToArray();
  }

  /// <summary>
  /// Processes the specified customers.
  /// </summary>
  /// <param name="customers">The customers to process.</param>
  public void ProcessCustomers(Customer[] customers) {
    if (customers == null) {
      throw new ArgumentNullException(nameof(customers));
    }

    foreach (Customer customer in customers) {
      this.ProcessCustomer(customer);
    }
  }

  /// <summary>
  /// Processes a single customer.
  /// </summary>
  /// <param name="customer">The customer to process.</param>
  private void ProcessCustomer(Customer customer) {
    if (customer == null) {
      throw new ArgumentNullException(nameof(customer));
    }

    customer.MarkAsProcessed();
    _CustomerRepository.Save(customer);
  }
}
```

This example intentionally favors explicit types, explicit control flow, arrays at method boundaries, expanded methods, and compiler-visible contracts over syntactic compactness.

------

## 19. Requirements Summary

| Area                         | Desired State                                 | Strength             |
| ---------------------------- | --------------------------------------------- | -------------------- |
| Source-code language         | English                                       | MUST                 |
| Prompt communication         | German by default in German project context   | SHOULD               |
| Type declarations            | Explicit type; no `var`                       | MUST                 |
| Static typing                | Preserve compile-time type information        | MUST                 |
| Nullable types               | Only where semantically justified             | SHOULD               |
| Null-coalescing              | Avoid unnecessary implicit fallback           | SHOULD               |
| Fields                       | `_` + PascalCase                              | MUST                 |
| Constants                    | `_` + PascalCase                              | MUST                 |
| Instance members             | Explicit `this.` except fields                | SHOULD               |
| Properties                   | Expanded implementation                       | MUST                 |
| Method documentation         | XML `<summary>`                               | MUST                 |
| Comments                     | Explain intent and non-obvious decisions      | SHOULD               |
| Single-line blocks           | Always use braces                             | MUST                 |
| Opening brackets             | Always at the end of the line                 | MUST                 |
| `else` / `catch` / `finally` | Same line after `}`                           | MUST                 |
| Ternary expressions          | Avoid for normal control flow                 | MUST NOT             |
| Lambda parameters            | Always parenthesized                          | MUST                 |
| Indentation                  | Two spaces, no tabs; CRLF                     | MUST                 |
| Encoding                     | UTF-8, BOM preferred, never mass-converted    | SHOULD               |
| Collection signatures        | Prefer arrays                                 | SHOULD               |
| Exception handling           | Catch only intentionally handled failures     | MUST                 |
| Exception logging            | `DevLogger.LogError(ex)`                      | MUST when applicable |
| Trace logging                | `DevLogger.LogTrace(0, 99999, "...")`         | MUST                 |
| Async/await                  | Not part of default implementation style      | MUST NOT by default  |
| JSON                         | Newtonsoft.Json                               | MUST                 |
| Windows interop              | Avoid; obtain approval when unavoidable       | MUST                 |
| Modern syntax                | Adopt only with concrete engineering benefit  | SHOULD               |
| Code optimization goal       | Maintainability and explicitness, not brevity | MUST                 |

------

## 20. AI Agent Implementation Guidance

An AI agent using this document MUST treat it as an implementation constraint, not merely as stylistic advice.

Before returning generated C# code, the agent SHOULD verify at least the following:

- no `var` declarations were introduced;
- source-code text and comments are English;
- relevant methods have `<summary>` documentation;
- fields and constants follow the required naming convention;
- instance member access uses `this.` where required;
- properties are expanded;
- blocks contain braces;
- opening brackets are at the end of the line (never on a line of their own) and continuation keywords follow the required layout;
- ternary expressions were not introduced;
- lambda parameters are parenthesized;
- indentation uses two spaces;
- collection boundaries use arrays where appropriate;
- nullable states are intentional;
- exception handling is purposeful;
- expected caught exceptions use `DevLogger.LogError(ex)`;
- trace output uses `DevLogger.LogTrace(0, 99999, "...")`;
- no unnecessary `async`/`await` was introduced;
- JSON code uses Newtonsoft.Json;
- no Windows-specific interop was introduced without explicit approval;
- no syntactic shortcut reduced type visibility or maintainability.

When uncertain between a concise modern construct and a conventional explicit construct, the agent SHOULD select the conventional explicit construct.

The agent MUST NOT silently reinterpret these rules according to generic contemporary C# style recommendations.

------

## 21. Migration and Existing Code

This section is secondary to the desired-state rules above.

Existing code does not need to be rewritten merely to create stylistic churn.

When code is actively modified, the affected area SHOULD be moved toward the desired state where this can be done safely and without unrelated behavioral changes.

Migration work SHOULD:

1. preserve behavior first;
2. make small, reviewable changes;
3. avoid unrelated formatting churn;
4. separate semantic changes from stylistic normalization where practical;
5. use compiler feedback and tests to validate changes.

Large-scale modernization SHOULD NOT introduce syntactic shortcuts that conflict with these standards.

Existing platform-specific interoperability SHOULD be evaluated before replacement. New platform-specific interoperability requires the approval described in Section 16.

------

## 22. Documentation and Test Deliverables

For substantial implementations, the developer or AI agent SHOULD actively offer to generate:

- unit tests;
- Markdown documentation.

When Markdown implementation documentation is requested, it SHOULD be maintained as one continuous English document rather than a collection of disconnected fragments.

The documentation SHOULD contain, in this order:

1. an introduction with abstract and motivation, including positive differentiation from competing implementation approaches;
2. top-down introductory examples;
3. advanced examples showing full complexity and explaining the use case;
4. a tabular specification of the underlying requirements;
5. bottom-up documentation of the most important implementation artifacts.

Documentation SHOULD describe the desired state first. Migration or transition guidance SHOULD appear later and remain secondary.

------

## 23. Versioning and Maintenance of This Standard

This document is intended to be maintained under version control.

Future versions MUST be evolved incrementally.

The established chapter structure SHOULD remain stable unless a structural change is itself necessary.

Updates SHOULD:

- modify only sections affected by the new or changed rule;
- avoid unrelated editorial rewrites;
- avoid reformatting unaffected sections;
- preserve examples that remain valid;
- produce small and meaningful source-control diffs.

New knowledge SHOULD normally extend or refine the existing document rather than trigger regeneration of the complete document.

If a rule changes, the normative rule, affected examples, requirements summary, and AI-agent guidance SHOULD be updated consistently.

### 23.1 Leading `.editorconfig`

The agreed tool configuration (IDE / analyzer settings, formatting, naming rules) is maintained as the leading [assets/.editorconfig](assets/.editorconfig) (agreed state 2026-10-21; change it only by explicit decision).

- Every repository carries an **identical mirror** of it in the repository root (see `smart-solution-and-project-structure`); the header marks it as a mirror with its version. Never edit a mirror; change the leading version here, raise its `Version` line and refresh the mirrors.
- Repository-specific deviations go into a nested `.editorconfig` (without `root = true`) next to the affected code, with a comment stating the reason.
- The `.editorconfig` does not encode every rule of this document, and some of its preferences are more permissive (mostly `silent`). For writing code, this document remains binding; IDE suggestions or code fixes that contradict it (e.g. auto-properties, conditional expressions, expression-bodied members, primary constructors) are not applied. Known differences are listed in section 24.
- When a rule of this document changes and is tool-enforceable, propose the matching `.editorconfig` change in the same step.

------

## 24. Open Refinement Areas

The following areas are intentionally left for iterative refinement:

- exact documentation requirements for every member type;
- detailed rules for records, primary constructors, pattern matching, collection expressions, and other newer C# features;
- precise exceptions to the synchronous programming standard;
- detailed test-code conventions;
- dependency-injection conventions;
- class and interface design conventions;
- LINQ usage and complexity limits;
- extension-method conventions;
- event and delegate conventions;
- reflection and runtime code-generation rules;
- project-specific logging identifiers beyond the generic trace defaults;
- additional rules for public API compatibility;
- alignment between this document and the agreed `.editorconfig` (section 23.1). Differences found 2026-10-21:
  - constants: the `.editorconfig` requires PascalCase **without** `_` (`error`), section 6.2 requires `_` + PascalCase; fields with `_` apply to all accessibilities there, not only private ones;
  - type names: the `.editorconfig` prefers language keywords (`string`, `int`), the examples here use BCL names (`String`, `Int32`) without a stated rule;
  - `var` is not covered by the `.editorconfig` (section 5.1 forbids it);
  - auto-properties, conditional expressions, `??` / `?.`, expression-bodied properties / accessors, top-level statements, primary constructors and collection expressions are preferred there (mostly `silent`, some `suggestion`) but restricted by sections 5.3, 7, 9.4 and 4.4;
  - additional rules of the `.editorconfig` not described here yet: namespace must match the folder (IDE0130, `error`), parentheses for clarity in binary operators, explicit accessibility modifiers, `this.` for methods and properties as `error`, camelCase parameters and locals, line guideline at 120 characters.

Until explicitly refined, these areas MUST be approached conservatively according to the principles in Sections 1 through 4.
