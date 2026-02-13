# PdfiumViewer Upgrade Plan

**Status: Phases 1-3 COMPLETED (2026-02-13)**

## Goal

Upgrade the forked PdfiumViewer library from legacy .NET Framework 4.8 project format to a modern SDK-style .NET 8 project. This eliminates cross-framework compatibility quirks, removes build warning noise, and makes the library a first-class dependency of CurveExtractor2 while keeping the project-reference workflow.

## Previous State

| Aspect | Value |
|--------|-------|
| Project format | Legacy (verbose XML, explicit file listing) |
| Target framework | .NET Framework 4.8 (`TargetFrameworkVersion v4.8`) |
| Build warnings | MSB3884: missing `MinimumRecommendedRules.ruleset` (x2 per build) |
| Reference from CE2 | `<ProjectReference Include="..\PdfiumViewer\PdfiumViewer\PdfiumViewer.csproj" />` |
| Native binary | `PdfiumViewer.Native.x86_64.v8-xfa` NuGet package (stays in CE2) |
| Source files | ~60 `.cs` files, all listed explicitly in csproj |
| Dependencies | System.Drawing, System.Windows.Forms, System.Resources.Extensions 8.0.0 |
| Custom mods | Partial-page `Render()` overload, `GetTextElementsInArea()`, `TextElement` class |
| Dutch localization | Already commented out (SDK 9.0 compat) |

## Problems Solved

1. **MSB3884 warning** — `MinimumRecommendedRules.ruleset` referenced in legacy csproj does not exist in modern SDK; removing it silences the warning.
2. **Cross-framework reference** — .NET 4.8 library referenced from .NET 8 project works but is fragile; same TFM eliminates implicit compatibility shims.
3. **Verbose csproj maintenance** — Legacy format requires every file listed explicitly; SDK-style auto-includes `**/*.cs`.
4. **473 build warnings** — Clean build of upgraded project initially produced 473 warnings across 5 categories, all resolved (see Phase 2).
5. **Stale configuration** — `CodeAnalysisRuleSet`, `Prefer32Bit`, `Platform=x86` defaults are all irrelevant now.

## Completed Work

### Phase 1: SDK-style csproj migration (DONE)

Replaced the legacy `PdfiumViewer.csproj` with SDK-style project file.

**Final csproj:**

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0-windows10.0.19041.0</TargetFramework>
    <UseWindowsForms>true</UseWindowsForms>
    <RootNamespace>PdfiumViewer</RootNamespace>
    <AssemblyName>PdfiumViewer</AssemblyName>
    <Nullable>annotations</Nullable>
    <ImplicitUsings>disable</ImplicitUsings>
    <GenerateResourceUsePreserializedResources>true</GenerateResourceUsePreserializedResources>
    <GenerateAssemblyInfo>false</GenerateAssemblyInfo>
    <NoWarn>SYSLIB0003;SYSLIB0004;SYSLIB0051</NoWarn>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="System.Resources.Extensions" Version="8.0.0" />
  </ItemGroup>

  <ItemGroup>
    <EmbeddedResource Include="pan.cur" />
  </ItemGroup>

  <!-- Exclude Dutch localization files (not needed) -->
  <ItemGroup>
    <EmbeddedResource Remove="PdfViewer.nl.resx" />
    <EmbeddedResource Remove="Properties\Resources.nl.resx" />
    <None Remove="PdfViewer.nl.resx" />
    <None Remove="Properties\Resources.nl.resx" />
    <Compile Remove="docs\**" />
    <EmbeddedResource Remove="docs\**" />
    <None Remove="docs\**" />
  </ItemGroup>
</Project>
```

**Key decisions:**
- `net8.0-windows10.0.19041.0` — matches CE2 TFM exactly
- `UseWindowsForms` — provides System.Drawing and System.Windows.Forms
- `Nullable=annotations` — allows adding `?` annotations without generating warnings for missing annotations everywhere (upgrade to `enable` later)
- `ImplicitUsings=disable` — existing code has explicit `using` statements; don't conflict
- `GenerateAssemblyInfo=false` — keeps existing `Properties/AssemblyInfo.cs`
- SDK-style auto-includes all `.cs` files — no need to list 60+ files
- Dutch `.nl.resx` localization files excluded
- `docs/` folder excluded from compilation

### Phase 2: Fix 473 build warnings (DONE)

Initial clean build after TFM change produced 473 warnings:

| Code | Count | Description | Fix Applied |
|------|-------|-------------|-------------|
| CA1416 | 864 | Platform compat ("only supported on windows") | Added `[assembly: SupportedOSPlatform("windows")]` to AssemblyInfo.cs |
| CS3001/3002/3003/3009 | 60 | CLS compliance | Removed `[assembly: CLSCompliant(true)]` from AssemblyInfo.cs |
| SYSLIB0003 | 14 | Code Access Security obsolete | `<NoWarn>` in csproj |
| SYSLIB0004 | 6 | Constrained Execution Region obsolete | `<NoWarn>` in csproj |
| SYSLIB0051 | 2 | Serialization API obsolete | `<NoWarn>` in csproj |

**Result: 473 warnings → 0 warnings, 0 errors**

No code changes required beyond AssemblyInfo.cs — all P/Invoke, rendering, and resource code compiled cleanly.

### Phase 3: Verify integration (DONE)

Full solution build (`CurveExtractor2.sln`):
- PdfiumViewer: 0 warnings, 0 errors
- CurveExtractor2: 0 warnings, 0 errors
- CurveExtractor2.Tests: 0 warnings, 0 errors

**Manual testing required:** User should verify PDF loading, all 3 Render overloads, and text extraction work at runtime.

## Phase 4: Future Cleanup (optional, low priority)

- Delete Dutch localization files (`PdfViewer.nl.resx`, `Properties/Resources.nl.resx`) if confirmed not needed
- Add nullable annotations to the API surface types used by CE2 (`PdfDocument`, `TextElement`, `IPdfDocument`)
- Add the custom `Render` overload and `GetTextElementsInArea` to `IPdfDocument` interface
- Remove unused UI components (PdfViewer, PdfRenderer, etc.) if CE2 only uses the document/rendering API — **evaluate first**, this reduces maintenance surface but is not urgent

## API Surface Used by CurveExtractor2

For reference — these are the only PdfiumViewer APIs that must remain working:

```
PdfDocument.Load(string path) -> PdfDocument
PdfDocument.Render(int page, int w, int h, int dpiX, int dpiY, bool hq) -> Image
PdfDocument.Render(int page, Graphics g, float dpiX, float dpiY, Rectangle bounds, PdfRenderFlags flags) -> void
PdfDocument.Render(int page, int w, int h, int offX, int offY, int pgW, int pgH, PdfRotation rot, PdfRenderFlags flags) -> Image  [CUSTOM]
PdfDocument.GetTextElementsInArea(int page, RectangleF area) -> List<TextElement>  [CUSTOM]
PdfDocument.PageSizes -> IList<SizeF>
PdfDocument.PageCount -> int
PdfDocument.Dispose()

PdfRenderFlags enum (None, Annotations)
PdfRotation enum (Rotate0)
TextElement class (Text, Bounds properties)  [CUSTOM]
```
