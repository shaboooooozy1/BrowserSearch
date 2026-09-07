# CLAUDE.md

Guidance for AI assistants working in this repository.

## What this is

BrowserSearch is a **PowerToys Run plugin** (C# / .NET). It reads the user's
**default browser's history** from the browser's local SQLite databases and
exposes those entries as searchable results in PowerToys Run. Selecting a
result opens its URL in the default browser.

The plugin is invoked in PowerToys Run with the action keyword `b?` (see
`plugin.json`), and is also global (`IsGlobal: true`), so entries can surface
without the keyword.

- **Author / upstream**: TBM13 — <https://github.com/TBM13/BrowserSearch>
- **Current version**: `1.11.0` (source of truth is `BrowserSearch/plugin.json`)
- **Platform**: Windows only (WPF, PowerToys, Windows-specific APIs)

## Repository layout

```
BrowserSearch.sln              Visual Studio solution (single project)
publish.ps1                    Release build + install to local PowerToys plugins dir
README.md                      User-facing install/build/supported-browser docs
Screenshots/                   README images
BrowserSearch/                 The plugin project
├── BrowserSearch.csproj       Project file; reads Version from plugin.json
├── plugin.json                PowerToys plugin manifest (ID, keyword, version, entry DLL)
├── Main.cs                    Plugin entry point: IPlugin/ISettingProvider/IReloadable
├── HistoryResult.cs           Internal history-entry record → maps to PT Run Result
├── Images/                    Light/dark plugin icons
└── Browsers/                  Browser-specific history readers
    ├── IBrowser.cs            Interface all browser readers implement
    ├── Chromium.cs            Chromium-family reader (Chrome, Edge, Brave, Vivaldi, ...)
    ├── Firefox.cs             Firefox-family reader (Firefox, LibreWolf, Zen, Waterfox)
    └── OperaGX.cs             Opera GX (subclass of Chromium; single profile)
```

Note: `BrowserSearch/libs/` is **not** committed. It holds PowerToys DLLs that
must be supplied locally to build (see "Building" below).

## Architecture

The flow, all driven by PowerToys Run through the `IPlugin` contract:

1. **`Main.Init`** → `InitDefaultBrowser()` detects the OS default browser via
   `Wox.Plugin.Common.DefaultBrowserInfo` (aliased `BrowserInfo`). It polls up
   to 500 ms for the name to populate.
2. A `switch` on `BrowserInfo.Name` maps the browser to a concrete `IBrowser`
   implementation, passing the candidate **user-data directory paths** (built
   from `%LOCALAPPDATA%` / `%APPDATA%`) and the optionally configured profile
   name. Unrecognized browsers log an error and show a `MessageBox`.
3. **`IBrowser.Init()`** discovers profiles, then copies each profile's history
   DB to a temp file (the live DB is locked while the browser runs) and reads
   it via `Microsoft.Data.Sqlite`, building a `List<HistoryResult>`.
4. **`Main.Query`** scores each history entry against the search text with
   `CalculateScore` (a fast substring match, deliberately **not** PT Run's
   `FuzzySearch`, which is too slow for large histories), adds a browser-specific
   "extra score", sorts, and returns the top `_maxResults`.
5. `HistoryResult.ToResult()` converts an entry into a PT Run `Result` whose
   `Action` opens the URL via `Wox.Infrastructure.Helper.OpenInShell`.

### The `IBrowser` abstraction (`Browsers/IBrowser.cs`)

```csharp
void Init();                                              // discover profiles + load history
List<HistoryResult> GetHistory();                         // the loaded entries
int CalculateExtraScore(string query, string title, string url);  // ranking boost
```

- **Chromium** reads `urls` (url, title, visit_count) from the `History` SQLite
  DB, and reads the `Network Action Predictor` DB for autocomplete predictions;
  `CalculateExtraScore` boosts entries by the prediction's hit count. Profiles
  are discovered from `User Data\Local State` JSON (`profile.info_cache`), with
  profiles keyed by directory name **and** by human-readable name properties
  (`gaia_given_name`, `gaia_name`, `name`, `shortcut_name`).
- **Firefox** reads `moz_places` (url, title, frecency) from `places.sqlite`;
  `CalculateExtraScore` boosts by Firefox's **frecency** value. Profiles live in
  `<random>.<profile_name>` directories; the reader strips the random prefix.
- **OperaGX** subclasses `Chromium` but overrides `CreateProfiles` because Opera
  GX has no `Local State` file; it uses a single hardcoded `default` profile.
  Multiple profiles are unsupported.

### Settings (`Main.AdditionalOptions`)

- **Maximum number of results** (`MaxResults`, number, default 15). `-1` shows
  all entries — slower.
- **Browser profile** (`SingleProfile`, text). Empty = load history from **all**
  profiles; otherwise only the named profile (case-insensitive).

## Adding support for a new browser

Most browsers are Chromium- or Firefox-based, so no new class is usually needed:

1. Add a `case "<BrowserInfo.Name>":` in the `switch` in `Main.InitDefaultBrowser`.
2. Construct `new Chromium([...user data dir candidates...], _selectedProfileName)`
   or `new Firefox([...profiles dir candidates...], _selectedProfileName)` with
   the correct paths under `%LOCALAPPDATA%` / `%APPDATA%`.
3. Add the browser to the "Supported browsers" list in `README.md`.

Only write a new `IBrowser` implementation if the browser's history format or
profile discovery genuinely differs from both existing readers (as OperaGX did).

The browser name string must match exactly what `DefaultBrowserInfo.Name`
returns for that browser when it is set as the OS default.

## Conventions

- **Language / style**: modern C# (.NET 9), nullable reference types **enabled**
  (`<Nullable>enable</Nullable>`) — respect nullability. Uses collection
  expressions (`[]`, `[.. ...]`), target-typed `new()`, and file-scoped or
  block namespaces as already present. Match the surrounding file's style.
- **Namespaces**: the plugin entry lives in
  `Community.Powertoys.Run.Plugin.BrowserSearch`; helpers live in `BrowserSearch`
  and `BrowserSearch.Browsers`. Browser readers are `internal`.
- **Logging**: use `Wox.Plugin.Logger.Log` (`Log.Info/Warn/Error`), passing the
  originating `typeof(...)` as the second argument, as existing code does.
- **DB access**: never open the live browser DB directly — copy it to the temp
  path first (`FileShare.ReadWrite`) because the browser locks it while running.
- **Version bumps**: change the version in `plugin.json` only. `BrowserSearch.csproj`
  parses the version out of `plugin.json` at build time (do not add a separate
  `<Version>` literal).

## Building

Requires the .NET 9 Windows SDK and the PowerToys DLLs, which are **not**
committed. Before building:

1. Create `BrowserSearch/libs/`.
2. Copy these from `%ProgramFiles%\PowerToys\` into it:
   `Wox.Plugin.dll`, `Wox.Infrastructure.dll`, `Microsoft.Data.Sqlite.dll`,
   `PowerToys.Settings.UI.Lib.dll`.

Then:

- **Debug** (in Visual Studio): build and copy `BrowserSearch\bin\Debug` into
  `%LOCALAPPDATA%\Microsoft\PowerToys\PowerToys Run\Plugins\`.
- **Release / install**: run `publish.ps1`. It runs
  `dotnet publish BrowserSearch -c Release -o .\PublishOutput`, removes
  `Microsoft.Windows.SDK.NET.dll` and `WinRT.Runtime.dll` from the output
  (shipped by PowerToys itself), and copies the result into the local
  PowerToys plugins directory.

> This project builds and runs on **Windows only** — it depends on WPF, the
> Windows SDK target (`net9.0-windows10.0.22621.0`), PowerToys, and
> Windows-specific file paths. It cannot be built or exercised on Linux/macOS,
> and there is no automated test suite. Verify changes by building the plugin
> and loading it in PowerToys Run on Windows.

## Testing / CI

There are no unit tests and no CI workflow in this repo. Manual verification is
the norm: build, install into PowerToys Run, restart PowerToys, and confirm
history search works for the target browser.

## Git workflow

- Releases are tagged with commits titled `Release X.Y.Z` that bump
  `plugin.json`.
- Keep commit messages short and imperative, matching existing history
  (e.g. "support cent browser", "Fix direct activation command ...").
