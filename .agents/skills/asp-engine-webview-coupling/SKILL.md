---
name: asp-engine-webview-coupling
description: Procedure for building small Windows desktop utilities as a WinForms/WebView2 shell with a real, in-memory ASP.NET Core engine behind it (WebViewAspNetCoreAdapter from the SmartStandards AspNetCoreTemplate, embedded via T4), so that the HTML/JS UI talks to genuine UJMW services instead of a proprietary host bridge. Covers serving UI files from embedded resources (no wwwroot, pattern from UniversalBFF), dual hosting of the same UI and services in the EXE and in a regular web host, and responsive UIs via UJMW long-running operations (CancellationToken, progress callbacks). Load when creating or reviewing a WebView2 desktop app with an ASP.NET Core backend, when choosing between postMessage bridge and in-process HTTP, or when a UI should later become a UShell module.
metadata:
	version: "2026-11-14"
---

# ASP Engine / WebView Coupling

## Purpose

A desktop utility with a web UI usually ends up with a proprietary JS-to-C# bridge (`chrome.webview.postMessage`, command dispatcher, hand-written DTO mapping). That bridge is a dead end: the UI cannot run anywhere else, and the backend cannot be reused by other clients.

This skill describes the alternative: **couple a real ASP.NET Core engine in-process to the WebView2 control**. The browser control sends ordinary HTTP requests. They are routed into an in-memory ASP.NET Core pipeline (`TestServer`, no TCP port, no firewall prompt). There, the backend is exposed as regular **UJMW** services.

The UI therefore speaks the production protocol from day one. The same UI and the same services can be hosted unchanged in a normal web host.

## Positioning: a lightweight predecessor of a UShell module

> This is **not** a UShell module. It is a deliberately lighter **predecessor**: one EXE, one embedded mini UI, no shell, no module federation, no BFF composition.
>
> It fits **small utilities extremely well**: tools, test clients, local admin consoles, proof-of-concept clients. These need a modern UI and a clean service contract, but not the UShell infrastructure.
>
> Because the UI already talks UJMW to a real ASP.NET Core pipeline, the step up to a UShell module (UniversalBFF hosting, UShell frontend) only replaces the UI shell. The contracts, the services and the wire protocol stay the same.

## Advantages

| Advantage | Compared to a postMessage host bridge |
| --- | --- |
| One protocol | The UI uses UJMW over HTTP everywhere. There is no second transport for the desktop case. |
| Reusable backend | The services are plain UJMW controllers. Other clients (dynamic client, test client, MCP facade) can call them too. |
| Dual hosting for free | The same UI and services also run in a web host (see "Dual hosting"). |
| Browser debugging | During development the UI can run in a normal browser against the web-hosted variant. |
| Standard middleware | Auth, logging, ambience and side channels come from the ASP.NET Core pipeline and do not need to be re-implemented in a bridge. |
| No network exposure | `TestServer` is purely in-memory: no port, no firewall, no CORS. |
| Clear upgrade path | Becoming a UShell module later changes only the shell, not the contract (see above). |

## Building blocks

| Block | Source | Rule |
| --- | --- | --- |
| `WebViewAspNetCoreAdapter` | `SmartStandards/AspNetCoreTemplate`, `dotnet/src/WebView2Template/` (branch `10.0`) | Embed via its **T4 template** `WebViewAspNetCoreAdapter.tt` (see skill `dotnet-t4-source-embedding`). Never copy or edit the generated `.cs` by hand. |
| Shell project | `WebView2Template` in the same repository | Reference layout for `Program.cs`, `FormMain.cs` and the `.csproj`. |
| Bootstrap | Skill `asp-net-core-development-guidelines` | Use the same two hooks `OnConfigureServices` / `OnRunApplication` as in web hosts. |
| Static UI files | Pattern of `EmbeddedBundleFileProvider` / `StaticFileConsolidatorIApp` in `UniversalBFF` | Serve the UI from **embedded resources**. Do not use `wwwroot` or loose files. |
| Service exposure | UJMW (`UJMW.DynamicController`) | Use explicit `AddControllerFor<TContract>(...)`, no accidental discovery. |
| Long-running calls | UJMW long-running operations | Use a `CancellationToken` plus `Action<...>` progress callbacks (see "Responsive UIs"). |

## Procedure

### 1. Create the shell project

- `net10.0-windows`, `UseWindowsForms`, `FrameworkReference Microsoft.AspNetCore.App`.
- Typical packages, following the template: `Microsoft.Web.WebView2`, `Microsoft.AspNetCore.TestHost`, `Microsoft.AspNetCore.Mvc.NewtonsoftJson`, `UJMW.DynamicController`, `SmartStandards.Logging`. Take the versions from skill `package-version-preferences-dot-net`.
- Add `WebViewAspNetCoreAdapter.tt` with the custom tool `TextTemplatingFileGenerator`. Adjust only the target namespace transformation, then run the custom tool.

### 2. Bootstrap the engine (`Program.Main`)

Follow the template:

- Create `new WebViewAspNetCoreAdapter(Assembly.GetExecutingAssembly(), onConfigureServices, onRunApplication)` **before** `Application.Run(...)`.
- Inside `onConfigureServices`, call `services.AddControllers().AddNewtonsoftJson()`. Keep this `//HACK:` from the template: `System.Text.Json` throws `PipeWriterDoesNotImplementUnflushedBytes` inside the TestServer pipeline.
- Dispose the main form and the adapter in `finally`.
- In `FormMain_Load`, call `Program._AspApplication.Attach(webView2Control)`. `Attach` initializes WebView2 without blocking the message loop and navigates to `/`.

### 3. Keep the base address on a custom scheme

The default base address is `app://local`. Do **not** switch to fake `http(s)://` hosts: WebView2 runs a domain check of about 3 seconds before the request handler is called (WebView2Feedback issue 2381).

The adapter registers the custom scheme with `TreatAsSecure = true`. The page is therefore a secure context, so APIs such as `crypto.subtle` are available to the UI.

### 4. Serve the UI from embedded resources

- Put the UI (`index.html`, `app.js`, `app.css`, ...) into a folder of the assembly that owns the UI and mark it `<EmbeddedResource Include="ui\**" />`.
- Expose it with `EmbeddedFileProvider` (`Microsoft.Extensions.FileProviders.Embedded`) plus `UseDefaultFiles` / `UseStaticFiles` for a mount path. That is the pattern of UniversalBFF's `StaticFileConsolidatorIApp`.
- Pitfalls of the embedded base namespace (copied from the UniversalBFF warning). Errors here are silent: the file simply returns 404.
	- The base namespace is `{DefaultNamespaceOfTheProject}.{Folder}` with `.` as the folder separator.
	- It is case-sensitive.
	- `-` and spaces become `_`.
- Log a missing embedded file explicitly, as `EmbeddedBundleFileProvider` does. Otherwise you will spend hours searching.

### 5. Expose the backend via UJMW

- Contracts stay UJMW-transportable: synchronous methods, simple DTOs, arrays.
- Register them explicitly in `OnRunApplication` / `OnConfigureServices`.
- The JS UI calls `POST <mount>/<Contract>/<Method>` with the UJMW request DTO. It uses **relative** URLs only, so it works under `app://local/`, under any mount point, and behind a reverse proxy.

### 6. UI technology

For mini UIs, plain vanilla HTML/CSS/JS without a build step is acceptable and preferred: it is fast to change and easy to embed. Move to TypeScript/Tailwind (the UShell stack) once the UI is meant to become a UShell module.

## Dual hosting (EXE and web host)

Put everything that is not shell-specific into **one library** (for example `<Product>.Ui`). Both hosts then share exactly two calls:

```csharp
public static class ChatUiHostingExtensions {

	/// <summary>
	/// Registers the backend services and the UJMW controllers of the UI.
	/// </summary>
	public static void AddChatUi(this IServiceCollection services, IConfiguration config) { ... }

	/// <summary>
	/// Mounts the embedded UI files under the given path (default document + static files).
	/// </summary>
	public static void UseChatUi(this WebApplication app, string mountPath) { ... }

}
```

- **EXE:** call both inside the hooks passed to `WebViewAspNetCoreAdapter`, typically with mount path `/`.
- **Web host:** call the same two lines in its `OnConfigureServices` / `OnRunApplication`, for example with mount path `/chat/`.
- The UI is mount-agnostic because it only uses relative URLs. Contracts, services, embedded files and protocol are identical in both hosts. Only the shell and the authentication setup differ.
- This is also the natural seam for later UniversalBFF / UShell integration: register the same embedded bundle there as a module.

## Responsive UIs: UJMW long-running operations

UIs must stay responsive. Never block the UI on a plain request that runs for seconds. UJMW solves this without breaking the synchronous contract style:

- Mark a method as potentially long-running by adding **one** `CancellationToken` parameter (any position).
- Add `Action<...>` parameters as **progress callbacks**, for example `Action<string> onPartialAnswer` for streamed text or `Action<string, int> onProgress`. `Func<...>` is not supported.
- The host needs no extra setup: `AddControllerFor<T>` detects the convention.

Wire protocol for a JS client:

| Step | Request | Notes |
| --- | --- | --- |
| Initiate | `POST <url>/<Method>?wait=2` | `wait` of about 2 seconds is the recommendation for UI layers. |
| Poll | `GET` with `asyncId`, `pollingSalt`, `pollingHash` | Respect `pollNbf`. `pollingHash = Base64Url(SHA-256(pollingSecret + pollingSalt))` without padding. |
| Cancel | `DELETE` with the same query plus `wait` | Map an `AbortController` / Cancel button to it. |

- The `async.execState` values are: `0` queued, `1` running, `-1` cancel requested, `2` completed, `3` failed, `-2` cancelled.
- Callback invocations arrive as `<callbackName>Invokes: [{ "#": n, "arg1": ... }]`. Replay them strictly in `#` order.
- Servers that do not know the protocol ignore `wait` and answer synchronously. Treat a missing `async` object as the final result.
- **Reference implementation:** the UJMW Test Client (`dotnet/tools/UJMW.TestClient`, `Invocation/UjmwInvoker.LongRunning.cs`) does exactly this loop: initiate, poll respecting `pollNbf`, a Cancel button that sends `DELETE`, ordered callback replay, and a transcript.
	- It is itself a WebView2 utility, but it uses the older postMessage `HostBridge` style. Take its long-running loop as the reference, not its bridge.
- .NET callers (for example a dynamic client in another process) get the same behavior from `DynamicClientFactory`: a blocking call, callbacks replayed, and the token cancels via `DELETE`.

## Troubleshooting

| Symptom | Likely cause | Quick check |
| --- | --- | --- |
| First navigation hangs for about 3 s | `http(s)` fake host instead of a custom scheme | Base address must be `app://local` (or another custom scheme). |
| 404 for UI files | Wrong embedded base namespace or missing `EmbeddedResource` | List `GetManifestResourceNames()`. Check case, `_` instead of `-`, folder dots. |
| `PipeWriterDoesNotImplementUnflushedBytes` | `System.Text.Json` in the TestServer pipeline | `AddControllers().AddNewtonsoftJson()` is missing. |
| UI freezes during LLM or other long calls | Plain synchronous UJMW call | Add a `CancellationToken` and callbacks to the contract and use the poll loop. |
| `crypto.subtle` undefined | The page is not a secure context | The custom scheme must be registered with `TreatAsSecure = true` (the adapter does this). |
| UI works in the EXE but not in the web host | Absolute URLs in the JS | Use relative URLs only. |

## Upstream sources

| Repository | URL | Relevant paths |
| --- | --- | --- |
| AspNetCoreTemplate | https://github.com/SmartStandards/AspNetCoreTemplate | `dotnet/src/WebView2Template/` (adapter, `.tt`, sample shell); skill `.agents/skills/most-important-details-for-smartstandards-aspnetcore-template` |
| UnifiedJsonMessageWrapper | https://github.com/SmartStandards/UnifiedJsonMessageWrapper | `doc/features/long-running-operations.md`, `doc/features/testclient.md`, `dotnet/tools/UJMW.TestClient/`; skills `ujmw`, `ujmw-testclient-app-development` |
| UniversalBFF | https://github.com/KornSW/UniversalBFF | `dotnet/src/UniversalBFF.AspNetCore/AspSupport/[EmbeddedFileHandling]/`; skill `most-important-details-for-universalbff` |

`asp-net-core-development-guidelines`, `smart-solution-and-project-structure`
