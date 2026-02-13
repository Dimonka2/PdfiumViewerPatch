# PdfiumViewer Upgrade Plan

## Goal

Upgrade the forked PdfiumViewer library from legacy .NET Framework 4.8 project format to a modern SDK-style .NET 8 project. This eliminates cross-framework compatibility quirks, removes build warning noise, and makes the library a first-class dependency of CurveExtractor2 while keeping the project-reference workflow.

## Current State

| Aspect | Value |
|--------|-------|
| Project format | Legacy (verbose XML, explicit file listing) |
| Target framework | .NET Framework 4.8 (`TargetFrameworkVersion v4.8`) |
| Build warnings | MSB3884: missing `MinimumRecommendedRules.ruleset` (×2 per build) |
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
4. **IDE noise** — Missing nullable annotations on PdfiumViewer types cause IDE warnings when used from CE2 (which has nullable enabled). Enabling `<Nullable>enable</Nullable>` lets us annotate the API surface incrementally.
5. **Stale configuration** — `CodeAnalysisRuleSet`, `Prefer32Bit`, `Platform=x86` defaults are all irrelevant now.

## Plan

### Phase 1: SDK-style csproj migration

Replace the legacy `PdfiumViewer.csproj` with an SDK-style project file.

**New csproj (target content):**

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
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="System.Resources.Extensions" Version="8.0.0" />
  </ItemGroup>

  <ItemGroup>
    <EmbeddedResource Include="pan.cur" />
  </ItemGroup>

</Project>
```

**Key decisions:**
- `net8.0-windows10.0.19041.0` — matches CE2 TFM exactly
- `UseWindowsForms` — provides System.Drawing and System.Windows.Forms
- `Nullable=annotations` — allows adding `?` annotations without generating warnings for missing annotations everywhere (upgrade to `enable` later)
- `ImplicitUsings=disable` — existing code has explicit `using` statements; don't conflict
- SDK-style auto-includes all `.cs` files — no need to list 60+ files
- Embedded resources (`.resx`, `.cur`) are handled by convention or explicit include
- `MinimumRecommendedRules.ruleset` references gone — warning eliminated
- Localization `.resx` files already commented out — remove the comments entirely or delete the files

**Steps:**
1. Back up existing `PdfiumViewer.csproj`
2. Replace with SDK-style content above
3. Delete `Properties/AssemblyInfo.cs` (auto-generated in SDK-style) or add `<GenerateAssemblyInfo>false</GenerateAssemblyInfo>` if we want to keep it
4. Verify `.resx` files are picked up by convention (Designer files may need explicit wiring)
5. Build and fix any compilation errors

### Phase 2: Fix compilation issues

Expected issues after TFM change from net48 to net8.0:

1. **`System.Drawing.Common` breaking changes** — In .NET 6+, `System.Drawing.Common` is Windows-only. Since we target `net8.0-windows`, this should work, but verify no cross-platform guards are needed.

2. **Obsolete APIs** — Some WinForms/GDI+ APIs may have new warnings in .NET 8. Fix or suppress as appropriate.

3. **P/Invoke compatibility** — `DllImport` for `pdfium.dll`, `kernel32.dll`, `gdi32.dll` should work unchanged on .NET 8 Windows. Verify `LoadLibrary`/`GetProcAddress` path resolution still works.

4. **Resource file generation** — `Resources.Designer.cs` may need regeneration. The `ResXFileCodeGenerator` custom tool should still work in SDK-style projects.

5. **`AssemblyInfo.cs` conflicts** — SDK projects auto-generate assembly attributes. Either delete `Properties/AssemblyInfo.cs` or disable auto-generation.

### Phase 3: Verify integration

1. Build CurveExtractor2.sln — confirm 0 warnings from PdfiumViewer
2. Run the application — verify PDF loading, rendering, partial page rendering, text extraction all work
3. Test the 3 Render overloads used by CE2:
   - Simple DPI render: `Render(page, w, h, dpiX, dpiY, hq)` — used in PdfPageControl, PdfPagePreviewControl
   - Graphics render: `Render(page, graphics, dpiX, dpiY, bounds, flags)` — used in PDFExtractionSource
   - Custom partial render: `Render(page, w, h, offX, offY, pgW, pgH, rot, flags)` — used in PdfZoomRenderer, ExtractionViewBase, PDFDocument.RenderArea
4. Test `GetTextElementsInArea()` — used by at-risk detection
5. Test `PageSizes`, `PageCount` — used in multiple locations

### Phase 4: Cleanup (optional, low priority)

- Remove commented-out Dutch localization files if no longer needed
- Add nullable annotations to the API surface types used by CE2 (`PdfDocument`, `TextElement`, `IPdfDocument`)
- Add the custom `Render` overload and `GetTextElementsInArea` to `IPdfDocument` interface
- Remove unused UI components (PdfViewer, PdfRenderer, etc.) if CE2 only uses the document/rendering API — **evaluate first**, this reduces maintenance surface but is not urgent

## API Surface Used by CurveExtractor2

For reference — these are the only PdfiumViewer APIs that must remain working:

```
PdfDocument.Load(string path) → PdfDocument
PdfDocument.Render(int page, int w, int h, int dpiX, int dpiY, bool hq) → Image
PdfDocument.Render(int page, Graphics g, float dpiX, float dpiY, Rectangle bounds, PdfRenderFlags flags) → void
PdfDocument.Render(int page, int w, int h, int offX, int offY, int pgW, int pgH, PdfRotation rot, PdfRenderFlags flags) → Image  [CUSTOM]
PdfDocument.GetTextElementsInArea(int page, RectangleF area) → List<TextElement>  [CUSTOM]
PdfDocument.PageSizes → IList<SizeF>
PdfDocument.PageCount → int
PdfDocument.Dispose()

PdfRenderFlags enum (None, Annotations)
PdfRotation enum (Rotate0)
TextElement class (Text, Bounds properties)  [CUSTOM]
```

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| P/Invoke breakage on .NET 8 | Low | High | P/Invoke is stable across .NET versions; test thoroughly |
| Resource file issues | Medium | Low | Regenerate Designer files if needed |
| Subtle rendering differences | Low | Medium | Compare rendered output before/after |
| pdfium.dll loading path change | Low | High | Verify `LoadLibrary` resolution; the Native NuGet package copies to `x64/pdfium.dll` in output |

## Estimated Effort

- Phase 1 (csproj migration): ~30 minutes
- Phase 2 (fix compilation): ~30-60 minutes depending on issues
- Phase 3 (integration testing): ~30 minutes manual testing
- Phase 4 (cleanup): optional, ~1-2 hours if pursued

Total: **1.5-2 hours** for phases 1-3.
