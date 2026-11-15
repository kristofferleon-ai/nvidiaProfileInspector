---
name: nvidiaprofileinspector
description: Working guide for the NVIDIA Profile Inspector codebase (C# WPF, .NET Framework 4.8, NVAPI driver settings). Use when building the app, adding or editing driver setting definitions in CustomSettingNames.xml or Reference.xml, changing profile import/export (.nip), touching NVAPI interop, changing command-line options, or preparing a release.
---

# NVIDIA Profile Inspector

A Windows-only WPF desktop app (`net48`, C# 9) that edits NVIDIA driver profiles
through the NVAPI Driver Settings (DRS) API. Everything lives under
`nvidiaProfileInspector/`; the solution is
`nvidiaProfileInspector/nvidiaProfileInspector.sln`.

## Build and run

The app targets .NET Framework 4.8 with WPF and WinForms, so it only builds and
runs on Windows. On Linux or macOS (including cloud sessions) there is no way to
compile or launch it: verify changes by reading the code and validating XML, and
say plainly that the build was not run.

On Windows:

```powershell
dotnet build .\nvidiaProfileInspector\nvidiaProfileInspector.sln -c Release
```

CI uses `msbuild /m /p:Configuration=Release` on the same solution. Output goes
to `nvidiaProfileInspector\bin\Release\net48\`.

There is no test project. To exercise the UI without an NVIDIA GPU or driver,
start the executable with `-mock`: `NvapiDrsWrapper.Mock.cs` then serves
in-memory sessions, profiles and settings instead of calling NVAPI. Writing real
profiles needs an elevated process and an installed NVIDIA driver.

## Layout

| Path | Contents |
| --- | --- |
| `App.xaml.cs` | Startup, single-instance handling, command-line parsing (`ParseStartupOptions`) |
| `AppBootstrapper.cs` | TinyIoC container setup; loads `CustomSettingNames.xml` and `Reference.xml` |
| `Common/Drs*.cs` | DRS services: read/write settings, scan profiles, import, decrypt |
| `Common/Meta/` | Setting metadata sources (driver, custom, reference, scanned, constant) merged by `DrsSettingsMetaService` |
| `Common/CustomSettings/` | XML model for `CustomSettingNames.xml` / `Reference.xml` |
| `Common/Import/` | `.nip` profile model and import/export |
| `Common/Helper/` | DLSS version lookup, Steam/Epic app resolvers, user settings, admin check |
| `Common/Updates/` | GitHub release check and in-place updater |
| `Native/NVAPI/` | NVAPI P/Invoke wrapper, mock, and T4-generated `NvApiDriverSettings.cs` |
| `Native/WINAPI/` | Win32 interop (backdrop, taskbar, shell links) |
| `Services/` | Themes, dialogs, bit editor, splash screen |
| `UI/` | MVVM: `Views`, `ViewModels`, `Controls`, `Converters`, `Behaviors`, `Styles`, `Themes` |

`Native/NVAPI/NvApiDriverSettings.cs` is generated from `NvApiDriverSettings.tt`.
Change the template, not the generated file.

## Setting definitions

Most contributions are edits to the two setting XML files. Both use the same
schema (`CustomSettingNames` > `Settings` > `CustomSetting`).

- `CustomSettingNames.xml` is an embedded resource holding the curated,
  user-friendly settings shown by default. A `CustomSettingNames.xml` placed
  next to the executable overrides the embedded copy, and the app warns the user
  when that happens. `-createCSN` writes the embedded copy to the working
  directory.
- `Reference.xml` is copied to the output folder and supplies the larger
  reference list of known driver settings.

A setting entry:

```xml
<CustomSetting>
	<UserfriendlyName>DLSS - Enable DLL Override</UserfriendlyName>
	<HexSettingID>0x10E41E01</HexSettingID>
	<GroupName>05 - Upscaling and Frame Generation</GroupName>
	<MinRequiredDriverVersion>0</MinRequiredDriverVersion>
	<Description>Text shown in the UI.\r\nSecond line.</Description>
	<SettingValues>
		<CustomSettingValue>
			<UserfriendlyName>Off</UserfriendlyName>
			<HexValue>0x00000000</HexValue>
		</CustomSettingValue>
	</SettingValues>
	<SettingMasks />
</CustomSetting>
```

Rules to follow when editing:

- Indent with tabs and keep the file UTF-8.
- Write IDs and values as `0x` plus eight uppercase hex digits.
- `HexSettingID` must be unique within a file. Search for the ID before adding
  an entry; update the existing one instead of duplicating it.
- Reuse an existing `GroupName` exactly. The groups in `CustomSettingNames.xml`
  are `01 - Compatibility`, `02 - Sync and Refresh`,
  `03 - Antialiasing and Sharpening`, `04 - Texture Filtering`,
  `05 - Upscaling and Frame Generation`, `06 - Rendering Features` and
  `07 - System, Memory and Compute`. The numeric prefix sets display order.
- Line breaks in `Description` are the literal characters `\r\n`, not real
  newlines.
- `${DlssVersion}` in names and descriptions is replaced at runtime by
  `DlssHelper.ReplaceDlssVersions`. Check that helper for the other placeholders
  it supports before inventing one.
- `OverrideDefault` (see `Reference.xml`) replaces the driver-reported default.
- Escape XML special characters (`&amp;`, `&lt;`, `&gt;`).

After editing, confirm the file still parses, for example:

```bash
python3 -c "import xml.dom.minidom,sys; xml.dom.minidom.parse(sys.argv[1])" nvidiaProfileInspector/CustomSettingNames.xml
```

## Command line

Options are matched case-insensitively in `App.xaml.cs`:

| Option | Effect |
| --- | --- |
| `<file>.nip` | Import profiles; forwarded to a running instance if there is one |
| `-silentImport`, `-silent` | Import without the success dialog |
| `-exportCustomized` | Export all customized profiles to a timestamped `.nip` next to the exe, then exit |
| `-createCSN` | Write the embedded `CustomSettingNames.xml` to the working directory, then exit |
| `-showOnlyCSN` | Show only customized settings |
| `-disableScan` | Skip the initial profile scan |
| `-mock` | Use the in-memory mock driver instead of NVAPI |

When adding or changing an option, update `ParseStartupOptions`, the
`StartupOptions` class and the Command Line table in `README.md` together.

## Versioning and releases

- The base version is `AssemblyVersion` in `Properties/AssemblyInfo.cs`
  (`GenerateAssemblyInfo` is off, so this file is the source of truth).
- `.github/workflows/release.yml` runs on pushes and pull requests to the
  `release` branch. On push it computes the next version from the assembly
  version and existing tags, patches `AssemblyInfo.cs`, builds, tags, and
  publishes a pre-release zip.
- Do not bump the fourth version component by hand and do not create tags
  manually; the workflow owns both.

## Conventions

- Dependency injection goes through the bundled TinyIoC container; register new
  services in `AppBootstrapper.cs` rather than constructing them in view models.
- Keep NVAPI calls inside `Native/NVAPI` and the `Common/Drs*` services. View
  models should not P/Invoke.
- Any new NVAPI wrapper call needs a matching branch in
  `NvapiDrsWrapper.Mock.cs`, otherwise `-mock` mode breaks.
- Wrap DRS work in a session via `DrsSessionScope` and surface failures as
  `NvapiException`.
- User-visible behaviour changes belong in `README.md` as well.
