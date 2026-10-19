# Referenz-Templates (Originale aus dem Produktiveinsatz)

Echte, unveränderte Templates samt generiertem Output. Sie zeigen den Hausstil. Sie sind **keine Kopiervorlage**: Neue Templates baust du aus dem Skelett in `SKILL.md`.

| Ordner | Variante | Namespace-Strategie | Was es zeigt |
|---|---|---|---|
| `clone-minimal/` | **K – Klon** | **Absorb** (`TemplateNamespace` → Verwender-Namespace `UniversalBFF`) | eine Datei, `ttVersionCheck = false`, `ttLocation = ""`, Branch pro Major (`refs/heads/10.0`) |
| `composition-maximal/` | **C – Komposition** | **CopyFor** (`Logging.SmartStandards` → `Logging.SmartStandards.CopyForAuthTokenHandling`) | 23 Dateien, Tree-Shaking (`*.Minimum.cs`, `#if !UsedByT4`), Selbst-Versionscheck gegen das kanonische `.tt` im Origin-Repo |

## Was du aus den Outputs (`.cs`) ablesen kannst

- **Aufbau:** sortierte, deduplizierte Usings, Header mit Herkunft und Template-Version, pro Quelldatei ein Kommentar `// Taken from File: <url>`.
- **Sichtbarkeit:** Alle Typen sind `internal` (bzw. `internal static`) und `partial`.
- **Abhängigkeit auf einen anderen Snapshot:** Der Klon-Output zieht `using Logging.SmartStandards;` und ruft `DevLogger` auf. Das Zielprojekt muss Logging bereitstellen (siehe SKILL.md, Schritt 1b).

## Bekannte Schwächen der Originale

Das Skelett in SKILL.md behebt diese Punkte. Übernimm sie nicht in neue Templates:

- **Sichtbarkeit über String-`Replace`.**
  - `public sealed/abstract class`, `struct`, `record` und `delegate` bleiben `public`.
  - `Replace("class ", "partial class ")` trifft auch Kommentare, Strings und `where T : class `.
- **Encoding.** `WebClient` ohne `Encoding = UTF8` lädt ANSI, dadurch kaputte Umlaute. Ein BOM wird nicht entfernt.
- **Fehlerfall.** Freitext bzw. Kommentar im Output statt `#error`.
- **Kopierter Header-Kommentar.** Im Klon-Template ist der Kopfkommentar aus dem Logging-Template übernommen und beschreibt den falschen Zweck.
- **Typo.** `fileDownloadSourceDirectroy` ist harmlos, nicht nachahmen.

Liegt im Origin-Repo eines dieser Templates als kanonische Version, gilt trotzdem: **unverändert übernehmen**. Verbesserungen gehören upstream.
