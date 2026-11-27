---
name: asp-engine-webview-coupling
description: Default pattern for every locally executable Windows tool with a UI and for web UIs in general - a TypeScript UI layer in WebView2 inside a .NET WinForms EXE with a real, in-memory ASP.NET Core engine behind it (WebViewAspNetCoreAdapter from the SmartStandards AspNetCoreTemplate, embedded via T4), so the UI talks to genuine UJMW services instead of a proprietary host bridge. Covers where the logic lives (portable / statically hosted apps - all logic in TypeScript, host = convenience; Windows tools or apps with a server backend - protectable business logic in .NET), serving one embedded classic-script bundle (no wwwroot), the native message channel for push and frequent traffic, host capabilities, static deliverable and app icon, dual hosting in the EXE and a web host, and responsive UIs via UJMW long-running operations. Load when creating or reviewing a WebView2 desktop app, when choosing between postMessage bridge and in-process HTTP, or when a UI should later become a UShell module.
metadata:
  version: "2026-11-24"
---

# ASP Engine / WebView Coupling

## Purpose

A desktop utility with a web UI usually ends up with a proprietary JS-to-C# bridge (`chrome.webview.postMessage`, command dispatcher, hand-written DTO mapping). That bridge is a dead end: the UI cannot run anywhere else, and the backend cannot be reused by other clients.

This skill describes the alternative: **couple a real ASP.NET Core engine in-process to the WebView2 control**. The browser control sends ordinary HTTP requests. They are routed into an in-memory ASP.NET Core pipeline (`TestServer`, no TCP port, no firewall prompt). There, the backend is exposed as regular **UJMW** services.

The UI therefore speaks the production protocol from day one. The same UI and the same services can be hosted unchanged in a normal web host.

## Default stance and where the logic lives

Use this combination **by default** for every locally executable Windows tool with a UI, and generally whenever a web UI is built: a **TypeScript UI layer** + WebView2 + .NET WinForms EXE + in-memory ASP.NET Core engine. Decide per application where the logic lives:

| Kind of application | Logic | Role of the .NET host |
| --- | --- | --- |
| Portable app or game, or one that must also be **statically hostable** | **All** logic in TypeScript; the UI runs without any host. | Convenience only, offered as optional capabilities (window functions, LAN transport, ...). Without the host the app still works. |
| Windows tool, or application with a **server backend** | The business logic worth protecting may live in .NET (UJMW services). | Real backend: services, persistence, integrations. |
| Either kind, host as convenience layer | Unchanged | Shortcuts a browser cannot offer, e.g. reading / writing the file system directly instead of an upload / download dialog. |

Rules: the UI detects the host and its capabilities at runtime and degrades gracefully (feature hidden, browser fallback). Never move portable logic into the host "because it is easier there".

## Positioning: a lightweight predecessor of a UShell module

> This is **not** a UShell module. It is a deliberately lighter **predecessor**: one EXE, one embedded mini UI, no shell, no module federation, no BFF composition.
>
> It fits **small utilities extremely well**: tools, test clients, local admin consoles, proof-of-concept clients. These need a modern UI and a clean service contract, but not the UShell infrastructure.
>
> Because the UI already talks UJMW to a real ASP.NET Core pipeline, the step up to a UShell module (UniversalBFF hosting, UShell frontend) only replaces the UI shell. The contracts, the services and the wire protocol stay the same.


### Host communication and optional endpoint pre-evaluation

Use the [general host contract](https://github.com/ProjectUShell/UShell.Docs/blob/master/.agents/skills/ushell-host-contract/SKILL.md) for standardized SPA/host communication (UJMW even in memory). Before wiring a backend URL, evaluate whether the host should resolve logical service names first via [Endpoint Discovery](https://github.com/SmartStandards/EndpointDiscovery/blob/master/.agents/skills/most-important-details-for-endpoint-discovery/SKILL.md). Direct/static routing remains valid when intentionally chosen. Ask the user when the choice matters and document it as a decision. This does not require U-Shell runtime.

Portability note: a suitably host-independent web UI may also be hosted in WordPress; see the [WordPress skill](../wordpress-plugin-development-guidelines/SKILL.md).

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

- **TypeScript is the default** (Vite; a framework only if the UI needs it). Plain HTML/JS without a build step remains acceptable for throw-away mini UIs only. Move to the UShell stack once the UI is meant to become a UShell module.
- Build **one classic IIFE script** (`app.js` + `app.css`, other assets in a subfolder) and let the index page load it with `defer`: module scripts under a custom scheme bring module / CORS subtleties. Vite: `base: "./"`, `build.rolldownOptions.output.format = "iife"`, a small `transformIndexHtml` plugin replacing `type="module" crossorigin` by `defer`.
- Write the build directly into the embedded folder of the UI assembly and **commit the bundle**: F5 then works without Node.js, and every host serves identical files. The price: a stale bundle if someone forgets `npm run build` - add a CI check (rebuild, fail on git diff).
- Keep the UI free of absolute URLs. For **backend-less** apps the bundle folder is a deliverable of its own: always ship a static `index.html` + script folder next to the EXE (e.g. `/static` at the repository root, embedded by the UI assembly from there). A classic script also runs under `file://`, so double-clicking `index.html` works.

### 7. App and window icon (mandatory from v1)

From version 1 on, every app has a designed icon: one SVG source, derived into a multi-size `.ico` (16 - 256 px) for `<ApplicationIcon>` and the window (`new Icon(embeddedStream)`; `Icon.ExtractAssociatedIcon` only yields 32 px), plus favicon / touch icon in `index.html`. Keep the derivation scripted and check 16 / 32 px.

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

## Native message channel: push and frequent traffic

Every request through the in-memory bridge is dispatched via the `WebResourceRequested` event on the UI thread. That is fine for the page load and occasional service calls, but not for polling, high-frequency calls or server push. For those, use the WebView2 message channel **in addition**, never instead of the contracts:

- Envelope `{ type, payload }` in both directions (`window.chrome.webview.postMessage` / `PostWebMessageAsJson`). The page sends `ready` after start; the host answers with `host.info` (version, **capability ids**). The UI enables features only for announced capabilities.
- Service calls over the channel reuse the same contracts: an `rpc` envelope (`id`, route, method, args) executed off the UI thread, result envelope `{ id, return | fault }` - transport-agnostic UI code.
- Host events (background threads, sockets, timers) are marshalled with `BeginInvoke` to the UI thread before `PostWebMessageAsJson`.
- A plain browser has no `chrome.webview`: the UI must keep working (capability absent).

## Host details that bite

- **Background throttling:** a hidden or minimized WebView2 gets no animation frames and Chromium throttles timers down to once per minute. If the page must keep working in the background (clocks, simulations, network roles), set `WEBVIEW2_ADDITIONAL_BROWSER_ARGUMENTS` to `--disable-background-timer-throttling --disable-renderer-backgrounding --disable-backgrounding-occluded-windows` before the environment is created, and drive logic from a timer, not only from `requestAnimationFrame`.
- **Profiles:** local storage belongs to the WebView2 user data folder. Offer a command line switch (e.g. `--profile NAME` -> separate folder) to run several instances on one PC (tests of network features). Changing the base address changes the origin and loses local storage.
- **Request bridge:** `OnWebResourceRequested` must buffer request and response completely and await the TestServer asynchronously on the UI thread's synchronization context; dispatching with `Invoke` plus `Application.DoEvents()` nests requests (hanging responses up to `StackOverflowException`). If the generated adapter still uses that pattern, fix it upstream in AspNetCoreTemplate before regenerating; a local hand-edit is lost on regeneration.
- **Runtime check:** call `CoreWebView2Environment.GetAvailableBrowserVersionString()` before creating the form and show a friendly message if the runtime is missing (the adapter throws inside its initialization handler otherwise).

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
| Blank page under `app://local`, script errors about modules / CORS | Module script bundle | Build one classic IIFE script loaded with `defer`. |
| EXE shows an old UI | Committed bundle not rebuilt | Run the UI build, rebuild the EXE. |
| Hangs or `StackOverflowException` under load | Request bridge pumps messages (`DoEvents`) | See "Host details that bite"; move frequent traffic to the native channel. |
| Logic stops while the window is minimized | Chromium background throttling | Browser arguments above plus a timer-driven loop. |

## Upstream sources

| Repository | URL | Relevant paths |
| --- | --- | --- |
| AspNetCoreTemplate | https://github.com/SmartStandards/AspNetCoreTemplate | `dotnet/src/WebView2Template/` (adapter, `.tt`, sample shell); skill `.agents/skills/most-important-details-for-smartstandards-aspnetcore-template` |
| UnifiedJsonMessageWrapper | https://github.com/SmartStandards/UnifiedJsonMessageWrapper | `doc/features/long-running-operations.md`, `doc/features/testclient.md`, `dotnet/tools/UJMW.TestClient/`; skills `ujmw`, `ujmw-testclient-app-development` |
| UniversalBFF | https://github.com/KornSW/UniversalBFF | `dotnet/src/UniversalBFF.AspNetCore/AspSupport/[EmbeddedFileHandling]/`; skill `most-important-details-for-universalbff` |

`asp-net-core-development-guidelines`, `smart-solution-and-project-structure`
