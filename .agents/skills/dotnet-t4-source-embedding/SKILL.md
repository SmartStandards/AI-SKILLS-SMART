---
name: dotnet-t4-source-embedding
description: Einbetten fremder C#-Quelldateien (einzelne Helper bis Teilkopien ganzer Libraries wie Logging) per T4-Template, das die Quellen aus dem Origin-Git-Repo lädt, statt NuGet/ProjectReference. Nutzen beim Erstkontakt mit einer einzubettenden Datei sowie bei Updates bestehender Embedded-Source-.tt-Dateien.
---

# T4 Source Embedding (.NET)

## Wozu

Manche Bibliotheken (Contracts, tiefe Basis-DLLs) dürfen keine zusätzlichen Referenzen bekommen: weder Helper-Pakete noch Cross-Cutting-Concern-Bibliotheken wie Logging. Statt einer Referenz wird der benötigte Code als **Quellcode-Snapshot** eingebettet.

Das Werkzeug dafür ist ein T4-Template (`.tt`) mit dem Custom Tool `TextTemplatingFileGenerator`. Das Template lädt die Quellen selbst aus dem Origin-Repo und schreibt sie in genau eine generierte `.cs`.

Damit gilt:
- Der Snapshot-Charakter ist offensichtlich, und die Herkunft steht im Template.
- Ein Update braucht weder KI noch Suche: Custom Tool erneut ausführen, fertig.

**Die generierte `.cs` wird nie von Hand editiert.** Änderungen gehören ins Origin-Repo oder in die Transformationslogik des `.tt`.

**Referenzen:** `references/` enthält zwei echte Templates mit ihrem Output: `clone-minimal/` (Variante K) und `composition-maximal/` (Variante C). Lies `references/README.md`, bevor du ein neues Template baust.

## Ablauf beim Erstkontakt

### 1. Existiert schon ein kanonisches Template?

Embeddable Libraries pflegen ihr `.tt` selbst im Origin-Repo, typischerweise in einem Testprojekt wie `dotnet/test/<Lib>.ThirdPartyTests/<Lib>.tt`. Dieses Testprojekt beweist, dass der Output kompiliert.

- **Template vorhanden:** Es wird unverändert übernommen, nur heruntergeladen. Nicht nachbauen und nicht „verbessern“. Weiter mit Schritt 2.
- **Schon im Zielprojekt eingebettet:** Kein zweites Template anlegen, nur aktualisieren (siehe „Update“).
- **Kein Template vorhanden:** Variante bestimmen (1a), dann aus dem Skelett unten bauen.

### 1a. Variante bestimmen: Klon oder Komposition

| | **Variante K – Klon** | **Variante C – Komposition** („Königsklasse“) |
|---|---|---|
| Quellen | genau **eine** Datei, vollständig, 1:1 | mehrere Dateien, aggregiert zu einer `.cs` |
| Auswahl | keine | Tree-Shaking: Teilmenge einer Library, `*.Minimum.cs`-Partials, `#if !UsedByT4`-Ausschnitte |
| Dateiliste | ändert sich nie | kann sich ändern, wenn das Origin wächst |
| Transformation | nur Standard (Namespace, `internal`, `partial`) | Standard plus ggf. Spezielles |
| `ttVersionCheck` | `false`, `ttLocation = ""` | **`true`, Pflicht** |
| Kanonisches `.tt` im Origin-Repo | optional, empfehlen | **Pflicht, durchsetzen** |
| Beispiel | ein Adapter- oder Helper-Klasse | Logging-Library als Teilkopie |

**Regel:** Variante K nur, wenn **alle** K-Kriterien erfüllt sind. Im Zweifel gilt Variante C.

Bei Variante C ist das Template selbst ein Implementierungsdetail mit eigener Versionierung. Kommt im Origin eine Klasse dazu, die zum Snapshot gehört, muss das kanonische `.tt` angepasst und `ttVersion` erhöht werden. Jede eingebettete Kopie erkennt dann beim nächsten Lauf selbst, dass sie veraltet ist.

**Vorgehen bei Variante K:**
- Das `.tt` lokal aus dem Skelett erzeugen und dabei strikt an die Norm halten.
- Dem User in einem Satz empfehlen, die Vorlage als Helfer auch im Origin-Repo abzulegen. Das ist kein Muss.

**Vorgehen bei Variante C:**
- Das kanonische `.tt` gehört ins Origin-Repo: ein Testprojekt (`<Lib>.ThirdPartyTests`), das den Output kompiliert. Dazu kommen `ttVersion` und `ttVersionCheck = true`, und `ttLocation` zeigt darauf.
- Existiert es noch nicht, legst du es dort an (Änderung bzw. PR im Origin-Repo) **bevor** oder **gemeinsam mit** der Einbettung im Zielprojekt.
- Hast du keinen Zugriff auf das Origin-Repo: stoppen und den User entscheiden lassen. Keine Komposition ohne Update-Logik einbetten.

### 1b. Abhängigkeiten des Snapshots prüfen

Jedes `using` und jeder Fremdtyp im Snapshot muss im Zielprojekt auflösbar sein. Beispiel: Ein Adapter ruft `DevLogger` aus `Logging.SmartStandards` auf.

Pro Abhängigkeit gibt es drei Fälle:
- **Schon vorhanden** (Framework, erlaubte PackageReference): ok.
- **Selbst nur als Embedded Source erlaubt:** zuerst diese einbetten. Beachten: Deren Namespace ist dann `…CopyFor<X>`. Das `using` im Snapshot passt nicht mehr und muss im Template mit umgeschrieben werden.
- **Würde das Zielprojekt belasten:** stoppen und den User fragen. Das Zielprojekt ist dann evtl. falsch gewählt.

### 2. Zielprojekt bestimmen: so tief wie möglich, so hoch wie nötig

1. **Verwender ermitteln.** Das sind alle Projekte, die die eingebettete API aufrufen werden.
2. **Abhängigkeitsgraph aufbauen.** Quellen sind die `ProjectReference`s aus allen `.csproj`. Shared Projects (`.projitems`) zählen als Teil jedes Target-Projekts, das sie importiert.
3. **Gemeinsamen Punkt suchen.** Gesucht ist die gemeinsame Abhängigkeit aller Verwender, die den Verwendern am nächsten liegt. Ist ein Verwender selbst Abhängigkeit der anderen, ist er der Zielort.
4. **Nie tiefer als nötig.** Liegt darunter noch ein Projekt, das den Output nicht braucht (Contracts, Abstractions, Models), wird dort **nicht** eingebettet.
5. **Saubere Projekte bleiben sauber.** Ist der einzige gemeinsame Punkt ein Projekt, das sauber bleiben muss, oder gibt es gar keinen gemeinsamen Punkt, bekommt jeder Verwender-Ast seine eigene Kopie. Das ist unproblematisch, weil die Namespaces pro Projekt eindeutig sind (siehe Schritt 3).
6. **Im Zweifel fragen.** Bei mehrdeutigen Kandidaten oder unklarem Status „muss sauber bleiben“ fragst du den User, bevor du etwas anlegst.

### 3. Namespace und Sichtbarkeit (IL-Merge-Kompatibilität)

Mehrere DLLs mit derselben eingebetteten Implementierung können gemeinsam laufen oder per IL-Merge verschmelzen. Jede Kopie braucht deshalb einen **eigenen Typ**. Das Template sorgt dafür:

- **Namespace-Rewrite.** `namespaceHint` liefert der VS-Host: `CustomToolNamespace`, falls gesetzt, sonst der Root-Namespace des Projekts. Liegt das `.tt` in einem Unterordner, kann VS den Ordnerpfad anhängen.

  Es gibt zwei Strategien. Welche gilt, erkennst du am Origin:

  | Origin-Namespace | Strategie | Ergebnis |
  |---|---|---|
  | echter Namespace-Baum der Library (z. B. `Logging.SmartStandards.*`) | **CopyFor** | `<OriginRoot>.CopyFor<hint>[.Sub]`, `using`s mit umschreiben |
  | Platzhalter (Konvention: `TemplateNamespace`) | **Absorb** | `<hint>[.Sub]`: der Code wird Teil des Verwender-Namespace |

  Beide sind pro Projekt eindeutig. „Absorb“ ist für Vorlagen-Code gedacht, der beim Verwender „wohnen“ soll.

  **Mehrere Namespace-Roots.** Der Rewrite erfasst nur `originRootNamespace`. Enthält die Quelle weitere Roots, bleiben diese unverändert und sind damit nicht pro Projekt eindeutig. Typisch sind Extension-Klassen in `Microsoft.AspNetCore.Builder` oder `Microsoft.Extensions.DependencyInjection`. Als `originRootNamespace` wählst du den Root mit den Implementierungstypen. Den unveränderten Rest meldest du dem User und lässt ihn entscheiden: akzeptieren (z. B. in Exe- oder Testprojekten, die nie gemergt werden) oder upstream so umbauen, dass alles unter einem Root liegt.
- **Sichtbarkeit.** Alle `public`-Typen werden `internal`.
- **`partial`.** Klassen, Structs, Interfaces und Records werden `partial`, damit Verwender sie lokal erweitern können.

Haben zwei Projekte denselben Root-Namespace, gibt es eine Kollision. Dann setzt du auf dem `.tt` `CustomToolNamespace` explizit.

### 4. Zugriff von oben: `InternalsVisibleTo`

Höher liegende Verwender werden **im Projekt, das das `.tt` enthält**, freigeschaltet, und zwar in der Projektdatei:

```xml
<ItemGroup>
  <InternalsVisibleTo Include="Company.Product.Higher" />
  <!-- strong-named: <InternalsVisibleTo Include="Company.Product.Higher" Key="0024000004800000..." /> -->
</ItemGroup>
```

Alte Nicht-SDK-`.csproj` (.NET Framework) kennen dieses Item nicht. Dort gehört `[assembly: InternalsVisibleTo("...")]` in `Properties/AssemblyInfo.cs`.

Ein Verwender, der die Kopie per IVT erreicht, bekommt **keine eigene zweite Kopie**. Sonst entstehen mehrdeutige Typnamen über zwei `CopyFor*`-Usings.

### 5. Multi-Targeting: Shared Project + ein Projekt pro Framework

**Hintergrund.** Ihr nutzt bewusst kein SDK-Multitargeting (`<TargetFrameworks>`). Stattdessen gibt es ein Shared Project plus ein Projekt pro Framework, z. B. `.Net4`, `.Net10` und `.Net12`. Kompatibilität wird per Compiler-Direktiven gesteuert.

**Targets sind eine Wand.** Pro AppDomain läuft immer genau ein Target. Die Targets sind maximal identisch und unterscheiden sich nur im Framework, gegen das kompiliert wird.

**Platzierung.** Shared Projects haben keinen brauchbaren IDE-Support für T4 und den Forms Designer. Deshalb:
- Das `.tt` kommt ins Projekt mit dem **neuesten** Target.
- Die Nachbar-Targets binden **nur die generierte `.cs`** als Link ein, nie das `.tt`.
- Der identische Namespace über alle Targets ist gewollt, denn sie laufen nie gemeinsam.

**Verboten:** Auf `<TargetFrameworks>` umbauen. Das `.tt` in jedes Target kopieren. Das `.tt` ins Shared Project legen.

**Sprachstand beachten.** Der eingebettete Code muss mit der `LangVersion` des ältesten Targets kompilieren, z. B. C# 7.3 bei .NET Framework.

### 6. Projektdateien

**SDK-Style, Projekt mit dem `.tt`:**

```xml
<ItemGroup>
  <None Update="External\Foo.tt">
    <Generator>TextTemplatingFileGenerator</Generator>
    <LastGenOutput>Foo.cs</LastGenOutput>
    <!-- optional: <CustomToolNamespace>Company.Product.Core</CustomToolNamespace> -->
  </None>
  <Compile Update="External\Foo.cs">
    <AutoGen>True</AutoGen>
    <DesignTime>True</DesignTime>
    <DependentUpon>Foo.tt</DependentUpon>
  </Compile>
</ItemGroup>
<ItemGroup>
  <Service Include="{508349b6-6b84-4df5-91f0-309beebad82d}" />
</ItemGroup>
```

Bei `EnableDefaultItems=false` oder in Legacy-`.csproj` verwendest du `Include` statt `Update`.

**Nachbar-Target, nur Link auf das Ergebnis:**

```xml
<!-- SDK-Style -->
<Compile Include="..\Company.Product.Core.Net12\External\Foo.cs" Link="External\Foo.cs" />

<!-- Legacy -->
<Compile Include="..\Company.Product.Core.Net12\External\Foo.cs">
  <Link>External\Foo.cs</Link>
</Compile>
```

### 7. Generieren

Bevorzugt: Der Mensch führt in Visual Studio auf dem `.tt` „Run Custom Tool“ aus. Kannst du das nicht selbst, sag es explizit an.

CLI-Alternative, nur wenn verifiziert:
- `TextTransform.exe` (liegt bei VS unter `Common7\IDE`) mit `-a !!namespaceHint!<Wert>`.
- Oder `dotnet-t4` mit dem äquivalenten Parameter.

Bei der CLI-Variante muss der Wert **exakt** dem entsprechen, was VS übergeben würde. Sonst ändert der nächste VS-Lauf den Namespace. Deshalb ist `CustomToolNamespace` bei CLI-Generierung Pflicht.

Die generierte `.cs` wird **eingecheckt**. Sie ist der Snapshot.

### 8. Verifizieren

1. Alle Targets bauen, auch die verlinkten alten.
2. In der generierten `.cs` darf es keinen `public`-Typ auf Namespace-Ebene geben.
   - Bei der Strategie CopyFor enthalten alle Namespaces `.CopyFor<...>`.
   - Bei der Strategie Absorb taucht kein `TemplateNamespace` mehr auf.
   - Jedes `using` ist im Zielprojekt auflösbar, und zwar über eine eigene, gewollte Referenz. Ein `using`, das nur zufällig über eine transitive Abhängigkeit auflöst (z. B. `Microsoft.CodeAnalysis` via EF Core Design), meldest du dem User als Origin-Befund.
3. Keine `#error`-Zeile im Output.
4. Im `.tt` steht kein `<…>`-Platzhalter, kein fremdes Beispiel und kein Kommentar aus dem Skelett oder einem anderen Template, der nicht auf diesen Snapshot zutrifft.
5. `git diff` der `.cs` lesen. Unerwartete Änderungen meldest du dem User, bevor du committest.

## Update

1. Custom Tool erneut ausführen.
2. Meldet `ttVersionCheck` ein veraltetes Template: das `.tt` neu von `ttLocation` laden, ersetzen und erneut generieren.
3. Diff reviewen und bauen.
4. Bricht die Generierung ab (offline, Fehler): Die `.cs` enthält dann `#error` bzw. ist leer. Mit `git checkout -- <Foo>.cs` wiederherstellen und nie halb generiert committen.

## Anforderungen an die Origin-Quellen (Hinweise für Library-Autoren)

- **Block-scoped Namespaces** (`namespace X {`). File-scoped Namespaces (`namespace X;`) gehen nicht, weil nach der Aggregation nur einer pro Datei erlaubt ist.
- **Vollständige `using`s pro Datei**, keine Abhängigkeit von `ImplicitUsings` oder `global using`. Ungenutzte `using`s entfernen: Jedes `using` wird zur Abhängigkeit jedes Verwenders.
- **Kopfkommentare passend zur Datei.** Stammt die Datei selbst aus einem anderen Snapshot, werden dessen Header und Typnamen beim Übernehmen angepasst. Sonst trägt jeder Output widersprüchliche Herkunftsangaben.
- **Keine voll qualifizierten Selbstreferenzen** (`Origin.Root.Foo.Bar()`). Der Namespace-Rewrite erfasst nur `namespace`- und `using`-Zeilen.
- **Nicht einzubettender Code** steht in `#if !UsedByT4 … #else … #endif`. Der `#if`-Teil fällt weg, der `#else`-Teil bleibt. Keine Verschachtelung.
- **`[assembly: …]`-Zeilen** werden verworfen.
- **Versionskommentar pro Typ** (`// v 1.0.0`) ist Konvention.
- **Kanonisches `.tt` plus `ttVersion`** im Origin-Repo, in einem Testprojekt, das den Output kompiliert.

## Template-Skelett

Nur verwenden, wenn das Origin-Repo kein eigenes `.tt` liefert. Der Konfigurationsblock wird angepasst, der Rest bleibt unverändert.

Beim Anpassen des Konfigurationsblocks gilt:
- Jeden `<…>`-Platzhalter ersetzen. Kommentierte Beispiele, die nicht zutreffen, löschen. Bleibt z. B. `extraNamespaceRewrites` leer, bleibt es ohne Beispielzeile.
- Den Zweck-Kommentar neu schreiben, passend zu genau diesem Snapshot. Nie Kopfkommentare aus anderen Templates übernehmen, etwa aus dem Logging-Template unter `references/`.
- Mehrzeilige Kommentare mit echten Zeilenumbrüchen schreiben, jede Zeile mit eigenem `//`. Keine Escape-Sequenzen wie `\r\n` im Kommentar.

```csharp
<#@ template debug="false" hostspecific="true" language="C#" #>
<#@ assembly name="System.Core" #>
<#@ import namespace="System" #>
<#@ import namespace="System.Linq" #>
<#@ import namespace="System.Text" #>
<#@ import namespace="System.Text.RegularExpressions" #>
<#@ import namespace="System.IO" #>
<#@ import namespace="System.Collections.Generic" #>
<#@ output extension=".cs" #>
<#

// ===================== CONFIG =====================

// <One sentence: what is embedded and why no NuGet/ProjectReference.>

// Variant K (clone, 1 file):        ttVersionCheck = false, ttLocation = ""
// Variant C (composition, n files): ttVersionCheck = true,  ttLocation = canonical .tt in origin repo (mandatory)

string ttVersion = "1.0.0";   // bump on every change of file list or transform logic (origin copy is authoritative)
bool ttVersionCheck = true;

string projectUrl        = "https://github.com/<Org>/<Repo>";
string repositoryRootUrl = "https://raw.githubusercontent.com/<Org>/<Repo>/refs/heads/master/"; // branch (e.g. per major: 10.0), tag or SHA
string sourceDirectory   = "dotnet/src/<Project>/";
string ttLocation        = "dotnet/test/<Project>.ThirdPartyTests/<Name>.tt";

// Namespace strategy:
//   CopyFor: originRootNamespace = real root of the library      -> <root>.CopyFor<hint>
//   Absorb:  originRootNamespace = placeholder ("TemplateNamespace") -> <hint>
string originRootNamespace = "<Origin.Root.Namespace>";
bool absorbIntoConsumerNamespace = false;

// Optional: usings pointing to OTHER embedded snapshots (their namespace is CopyFor-rewritten as well).
// Key = namespace as written in the origin source, value = namespace in the consumer project.
var extraNamespaceRewrites = new Dictionary<string, string> {
  // { "<Other.Origin.Root>", "<Other.Origin.Root>.CopyFor<Consumer>" },
};

string headerNote = "Partial copy of <Library>. <Reduced features / limitations>.";

string[] files = new string[] {
  "Path/To/File1.cs",
  "Path/To/File2.cs",
};

// ==================================================

string currentFileUrl = "";
string content;

try {

  System.Net.ServicePointManager.SecurityProtocol |= System.Net.SecurityProtocolType.Tls12;

  var body = new StringBuilder(10000);
  var usings = new List<string>();

  using (var wc = new System.Net.WebClient()) {

    wc.Encoding = Encoding.UTF8; // default would be ANSI -> broken umlauts

    if (ttVersionCheck) {
      currentFileUrl = repositoryRootUrl + ttLocation;
      string origin = wc.DownloadString(currentFileUrl);
      var m = Regex.Match(origin, @"string\s+ttVersion\s*=\s*""([\d\.]+)""");
      if (!m.Success) throw new Exception("Could not determine ttVersion at origin.");
      if (new Version(ttVersion) < new Version(m.Groups[1].Value)) {
        string msg = "This T4 template is outdated (" + ttVersion + " < " + m.Groups[1].Value + "). Download again from " + currentFileUrl + " or set ttVersionCheck = false.";
        this.Error(msg);
        return "#error " + msg;
      }
    }

    foreach (string file in files) {

      currentFileUrl = repositoryRootUrl + sourceDirectory + file;
      string fileContent = wc.DownloadString(currentFileUrl).TrimStart('\uFEFF');

      body.AppendLine();
      body.AppendLine("// Taken from file: " + currentFileUrl);
      body.AppendLine();

      bool inIf = false, inElse = false, beforeNamespace = true;

      using (var reader = new StringReader(fileContent)) {
        for (string line = reader.ReadLine(); line != null; line = reader.ReadLine()) {
          string t = line.TrimStart();
          if (inIf && t.StartsWith("#else"))                    { inElse = true; }
          else if (inIf && t.StartsWith("#endif"))              { inIf = false; inElse = false; }
          else if (t.StartsWith("#if !UsedByT4"))               { inIf = true; }
          else if (inIf && !inElse)                             { /* skip: not for embedding */ }
          else if (t.StartsWith("[assembly:"))                  { /* skip */ }
          else if (beforeNamespace && t.StartsWith("using "))   { usings.Add(t); }
          else if (t.StartsWith("namespace "))                  { beforeNamespace = false; body.AppendLine(line); }
          else if (beforeNamespace && string.IsNullOrWhiteSpace(line)) { /* skip */ }
          else                                                  { body.AppendLine(line); }
        }
      }
    }
  }

  // ---- unique namespace per embedding project (IL-Merge) ----

  string hint = this.Host.ResolveParameterValue("directiveId", "namespaceDirectiveProcessor", "namespaceHint");
  // If CustomToolNamespace is empty, VS passes the project's root namespace (plus folder path, if any).
  if (string.IsNullOrWhiteSpace(hint)) throw new Exception("namespaceHint is empty (set CustomToolNamespace).");

  string targetNamespace = absorbIntoConsumerNamespace ? hint : originRootNamespace + ".CopyFor" + hint;
  string ns = Regex.Escape(originRootNamespace);

  string text = string.Join("\r\n", usings.Distinct().OrderBy(u => u, StringComparer.Ordinal)) + "\r\n";
  text += "\r\n// " + headerNote;
  text += "\r\n// Generated by " + Path.GetFileName(this.Host.TemplateFile) + " v " + ttVersion + " - DO NOT EDIT. Re-run the custom tool instead.";
  text += "\r\n// Sources downloaded from: " + projectUrl + "\r\n";
  text += body.ToString();

  text = Regex.Replace(text, @"(?m)^(\s*(?:using|namespace)\s+)" + ns + @"(?=[\s\.;{])", "${1}" + targetNamespace);

  foreach (var r in extraNamespaceRewrites) {
    text = Regex.Replace(text, @"(?m)^(\s*using\s+)" + Regex.Escape(r.Key) + @"(?=[\s\.;])", "${1}" + r.Value);
  }

  // ---- public -> internal (top-level and nested type declarations) ----

  text = Regex.Replace(text,
    @"(?m)^(?<i>[ \t]*)public(?<m>(?:[ \t]+(?:static|sealed|abstract|readonly|unsafe|ref|partial))*)[ \t]+(?<k>class|struct|interface|enum|record|delegate)\b",
    "${i}internal${m} ${k}");

  // ---- make class/struct/interface/record partial (not enum/delegate) ----

  text = Regex.Replace(text,
    @"(?m)^(?<p>[ \t]*(?:(?:internal|public|private|protected|static|sealed|abstract|readonly|unsafe|ref)[ \t]+)*)(?<k>class|struct|interface|record)\b",
    "${p}partial ${k}");

  content = text;

} catch (Exception ex) {
  string msg = "T4 embedding failed at '" + currentFileUrl + "': " + ex.Message.Replace("\r", " ").Replace("\n", " ");
  this.Error(msg);
  content = "#error " + msg;   // fail loud: build must break, never silently ship a broken snapshot
}
#><#= content #>
```

## Anti-Patterns

- Die generierte `.cs` von Hand ändern, oder Dateien manuell kopieren ohne `.tt`.
- NuGet- oder ProjectReference auf die Lib im sauber zu haltenden Projekt („ist doch einfacher“).
- Zu tief einbetten (Contracts belasten) oder zu hoch (mehrere Kopien, wo eine per IVT reicht).
- `public`-Typen im Output.
- Das `.tt` in mehrere Targets oder ins Shared Project legen. Auf `<TargetFrameworks>` umbauen.
- Ein halb generiertes oder fehlerhaftes Ergebnis committen.
- Ein vorhandenes kanonisches Origin-`.tt` lokal „optimieren“, statt die Änderung upstream zu machen.
- Eine Komposition (Variante C) ohne kanonisches Origin-`.tt` und ohne `ttVersionCheck` einbetten.
- Eine Komposition als „Klon“ einstufen, um sich die Origin-Pflege zu sparen.
- Skelett-Platzhalter, kommentierte Beispiele oder Kommentare aus einem anderen Template (z. B. Logging) stehen lassen, obwohl sie für diesen Snapshot nicht gelten.
