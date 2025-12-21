# FsAutoComplete SourceLink Bug - .NET 10 Go-to-Definition Failure

## Problem Summary

**Go-to-definition fails for FSharp.Core symbols (like `List.map`) on .NET 10 with the error `MissingSourceFile`**, while the same code works correctly on .NET 9.

## How SourceLink Go-to-Definition Works

### The Goal

User Ctrl+clicks on `List.map` → FSAC needs to show them the source code.

**Problem:** `List.map` is in `FSharp.Core.dll` - an external library. The source code isn't on the user's machine.

**Solution:** SourceLink - a technology that embeds metadata in PDBs to fetch source from GitHub.

### The Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ 1. User clicks "Go to Definition" on List.map                               │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ 2. FSAC asks FCS: "Where is List.map defined?"                              │
│                                                                             │
│    FCS looks at FSharp.Core.dll metadata and returns:                       │
│    DeclFound { FileName = "/__w/1/s/.../list.fs", Line = 97, Col = 8 }      │
│                                                                             │
│    This is the path where the source was WHEN IT WAS COMPILED               │
│    (on Microsoft's CI machine, not on the user's machine)                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ 3. FSAC checks: Does this file exist locally?                               │
│                                                                             │
│    File.Exists("/__w/1/s/.../list.fs") → FALSE                              │
│                                                                             │
│    The file doesn't exist - need to fetch via SourceLink                    │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ 4. FSAC extracts a "targetFile" to search for in the PDB                    │
│                                                                             │
│    getFileName(range) → uses Path.GetFileName                               │
│    Result: "list.fs"  (or full path if Windows-style, as we discovered)     │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ 5. FSAC opens FSharp.Core.pdb and reads SourceLink metadata                 │
│                                                                             │
│    The PDB contains:                                                        │
│    - Document entries (source file paths from build machine)                │
│    - SourceLink JSON mapping paths → GitHub URLs                            │
│                                                                             │
│    Document entries in PDB metadata: [                                      │
│      "/__w/1/s/src/fsharp/src/FSharp.Core/list.fs",                         │
│      "/__w/1/s/src/fsharp/src/FSharp.Core/array.fs",                        │
│      ...                                                                    │
│    ]                                                                        │
│                                                                             │
│    SourceLink JSON:                                                         │
│    { "/__w/1/s/*": "https://raw.githubusercontent.com/dotnet/fsharp/..." }  │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ 6. FSAC searches: Which document entry matches our targetFile?              │
│                                                                             │
│    compareRepoPath(documentEntry, targetFile) for each entry                │
│                                                                             │
│    ❌ BUG IS HERE:                                                          │
│    targetFile     = "list.fs"                                               │
│    document entry = "/__w/1/s/src/fsharp/src/FSharp.Core/list.fs"           │
│    "list.fs" == "/__w/.../list.fs" → FALSE, no match found!                 │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ 7. If match found: Build GitHub URL and download                            │
│                                                                             │
│    document entry = "/__w/1/s/src/fsharp/src/FSharp.Core/list.fs"           │
│    SourceLink mapping: "/__w/1/s/*" → "https://raw.githubusercontent.com/..." │
│                                                                             │
│    → https://raw.githubusercontent.com/dotnet/fsharp/.../list.fs            │
│    → Download to local cache                                                │
│    → Return cached file path to editor                                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ 8. Editor opens the source file at the correct line/column                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Where the Bug Occurs

Step 6 fails because `compareRepoPath` uses exact equality:
- `targetFile` = `"list.fs"` (basename only)
- Document entry in PDB metadata = `"/__w/1/s/.../list.fs"` (full path)
- No match → Error `MissingSourceFile`

### The Fix

Modified `compareRepoPath` to handle basename-only targets:
```fsharp
if isTargetJustFilename then
  docStr.EndsWith("/" + targetStr)  // Match by suffix
else
  docNormalized = targetNormalized   // Exact match
```

## Environment

- **Failing environment**: Linux (Ubuntu-based devcontainer), .NET 10.0.100
- **Working environment**: Linux, .NET 9
- **FsAutoComplete**: Used via Ionide VS Code extension
- **FSharp.Core version**: 10.0.100 (from `/home/vscode/.nuget/packages/fsharp.core/10.0.100/lib/netstandard2.1/FSharp.Core.dll`)

## Symptoms

When attempting Ctrl+hover on `List.map` in VS Code with Ionide:

```
[Error] Request textDocument/definition failed.
  Message: MissingSourceFile
  Code: -32603
```

Log output showed:
```
[WRN] [FsAutoComplete.Sourcelink] No sourcelinked source file matched list.fs.
Available documents were: ["/__w/1/s/src/fsharp/src/FSharp.Core/list.fs", ...]
```

The irony: `list.fs` IS in the available documents, but as a full path!

## Root Cause Analysis

### The Bug Location

Two files work together to cause this issue:

#### 1. `src/FsAutoComplete.Core/ParseAndCheckResults.fs` (lines 45-49)

```fsharp
let getFileName (loc: range) =
  if Ionide.ProjInfo.ProjectSystem.Environment.isWindows then
    UMX.tag<NormalizedRepoPathSegment> loc.FileName
  else
    UMX.tag<NormalizedRepoPathSegment> (Path.GetFileName loc.FileName)
```

On non-Windows, this calls `Path.GetFileName` on the path from FCS. **Critically, the behavior depends on whether the path uses forward slashes or backslashes.**

#### 2. `src/FsAutoComplete.Core/Sourcelink.fs` - `compareRepoPath` function (original code)

The original code compared paths with exact equality:
```fsharp
let private compareRepoPath (d: Document) targetFile =
  // ...
  else
    let t = UMX.untag targetFile |> UMX.tag<RepoPathSegment>
    let t' = normalizeRepoPath t
    normalizeRepoPath d.Name = t'  // <-- EXACT EQUALITY!
```

This means:
- `targetFile` = `"list.fs"` (just basename from `getFileName`)
- `d.Name` = `"/__w/1/s/src/fsharp/src/FSharp.Core/list.fs"` (full path from SourceLink PDB)
- Comparison: `"/__w/1/s/.../list.fs" = "list.fs"` → **FALSE**

### Why .NET 10 Fails but .NET 9 Works

#### Initial Hypothesis (WRONG)

We initially hypothesized that FCS returns different result types (`ExternalDecl` vs `DeclFound`) between versions. **This was incorrect.**

#### Confirmed Root Cause: PDB Path Format Differences

Testing with debug logging in the .NET 9 environment revealed the true cause.

##### What PDB Files Contain

The PDB (Program Database) file (e.g., `FSharp.Core.pdb`) contains SourceLink metadata that includes the **paths to source files as they existed on the build machine**. When FSharp.Core is compiled, the compiler records these source file paths in the PDB, reflecting the build environment's filesystem.

These are the paths that FSAC iterates through in `compareRepoPath` when trying to match a source file. The error log showed:
```
Available documents were: ["/__w/1/s/src/fsharp/src/FSharp.Core/list.fs", ...]
```
Those paths come directly from reading the PDB's SourceLink metadata.

##### The Build Environment Changed

**Both .NET 9 and .NET 10 return `DeclFound`** - the difference is in the **path format embedded in the FSharp.Core PDB**:

| FSharp.Core Version | Built On | PDB Path Format | Example Path |
|---------------------|----------|-----------------|--------------|
| **9.0.303** | **Windows** (Azure DevOps) | Windows backslashes | `D:\a\_work\1\s\src\FSharp.Core\prim-types.fs` |
| **10.0.100** | **Linux** (GitHub Actions) | Linux forward slashes | `/__w/1/s/src/fsharp/src/FSharp.Core/list.fs` |

- **FSharp.Core 9.x** was built on Windows CI → PDB contains paths like `D:\a\_work\1\s\src\FSharp.Core\list.fs`
- **FSharp.Core 10.x** was built on Linux CI → PDB contains paths like `/__w/1/s/src/fsharp/src/FSharp.Core/list.fs`

#### How FCS Returns the Path

FCS returns a `FileName` that **prepends the workspace/project directory** to the PDB source path:

```
FCS FileName = <workspace prefix> + <PDB source path>
```

From the .NET 9 logs:
```
[DEBUG] DeclFound full FileName: /workspaces/devcont-fs-src-test/D:\a\_work\1\s\src\FSharp.Core\prim-types.fs
                                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                                 Linux workspace prefix            Windows PDB path (backslashes)
```

From the .NET 10 logs:
```
[DEBUG] DeclFound full FileName: /__w/1/s/src/fsharp/src/FSharp.Core/list.fs
                                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                                 Linux PDB path (forward slashes throughout)
```

#### The `Path.GetFileName` Platform Behavior

`Path.GetFileName` finds the last **path separator** and returns everything after it.

**The catch:** Path separators are platform-specific:
- **Linux/macOS:** Only `/` (forward slash) is a separator
- **Windows:** Both `/` and `\` are separators

On Linux, `Path.GetFileName` only recognizes **forward slashes** as path separators. Backslashes are treated as regular characters in the filename:

```fsharp
// .NET 9 scenario - FCS returns Linux prefix + Windows PDB path:
Path.GetFileName "/workspaces/devcont-fs-src-test/D:\a\_work\1\s\src\FSharp.Core\prim-types.fs"
// Linux scans for '/' → finds last one after "devcont-fs-src-test"
// → "D:\a\_work\1\s\src\FSharp.Core\prim-types.fs"  (Windows path SURVIVES!)

// .NET 10 scenario - FCS returns Linux PDB path:
Path.GetFileName "/__w/1/s/src/fsharp/src/FSharp.Core/list.fs"
// Linux scans for '/' → finds last one before "list.fs"
// → "list.fs"  (just the filename)
```

| FCS Returns | `Path.GetFileName` Result on Linux | Matches PDB Document? |
|-------------|-----------------------------------|----------------------|
| `/workspace/.../D:\a\...\prim-types.fs` | `D:\a\...\prim-types.fs` (Windows path portion) | ✓ **Yes** - matches PDB |
| `/__w/.../list.fs` | `list.fs` (just basename) | ✗ **No** - basename ≠ full path |

#### Why .NET 9 Works "By Accident"

1. FSharp.Core 9.0.303 PDB contains Windows-style path: `D:\a\_work\1\s\src\FSharp.Core\prim-types.fs`
2. FCS returns: `/workspaces/devcont-fs-src-test/D:\a\_work\1\s\src\FSharp.Core\prim-types.fs`
3. `getFileName` calls `Path.GetFileName` on Linux
4. Linux finds last `/` after workspace prefix → returns `D:\a\_work\1\s\src\FSharp.Core\prim-types.fs`
5. `compareRepoPath` compares this with PDB document `D:\a\_work\1\s\src\FSharp.Core\prim-types.fs`
6. **Exact match → SUCCESS** ✓

#### Why .NET 10 Fails

1. FSharp.Core 10.0.100 PDB contains Linux-style path: `/__w/1/s/src/fsharp/src/FSharp.Core/list.fs`
2. FCS returns: `/__w/1/s/src/fsharp/src/FSharp.Core/list.fs` (no separate prefix, or prefix merges)
3. `getFileName` calls `Path.GetFileName` on Linux
4. Linux finds last `/` before `list.fs` → returns **just `list.fs`**
5. `compareRepoPath` compares `list.fs` with PDB document `/__w/1/s/.../list.fs`
6. **No match → FAILURE** ✗

#### The Matching Flow Visualized

FSAC's requirement is simple:
```
(what getFileName returns) == (one of the PDB documents)
```

**.NET 9 - Both sides are the same Windows path:**
```
What's in PDB:
  Document: D:\a\_work\1\s\src\FSharp.Core\prim-types.fs

What getFileName produces:
  Input from FCS: /workspaces/devcont-fs-src-test/D:\a\_work\1\s\src\FSharp.Core\prim-types.fs
  Path.GetFileName → D:\a\_work\1\s\src\FSharp.Core\prim-types.fs

The comparison in compareRepoPath:
  targetFile (from getFileName):  D:\a\_work\1\s\src\FSharp.Core\prim-types.fs
  PDB document (d.Name):          D:\a\_work\1\s\src\FSharp.Core\prim-types.fs
                                  ↓ normalize (\ → /)
  targetNormalized:               D:/a/_work/1/s/src/FSharp.Core/prim-types.fs
  docNormalized:                  D:/a/_work/1/s/src/FSharp.Core/prim-types.fs
                                  ↓
                                  EQUAL ✓
```

**.NET 10 - Basename vs full path mismatch:**
```
What's in PDB:
  Document: /__w/1/s/src/fsharp/src/FSharp.Core/list.fs

What getFileName produces:
  Input from FCS: /__w/1/s/src/fsharp/src/FSharp.Core/list.fs
  Path.GetFileName → list.fs

The comparison in compareRepoPath:
  targetFile (from getFileName):  list.fs
  PDB document (d.Name):          /__w/1/s/src/fsharp/src/FSharp.Core/list.fs
                                  ↓
                                  NOT EQUAL ✗
```

### Verified with Debug Logging

**.NET 9 environment logs (FSharp.Core 9.0.303):**
```
[DEBUG] FCS GetDeclarationLocation returned: DeclFound
[DEBUG] DeclFound full FileName: /workspaces/.../D:\a\_work\1\s\src\FSharp.Core\prim-types.fs
         extracted sourceFile will be: D:\a\_work\1\s\src\FSharp.Core\prim-types.fs
[DEBUG] compareRepoPath BASENAME MATCH:
         docNormalized=D:/a/_work/1/s/src/FSharp.Core/prim-types.fs
         targetNormalized=D:/a/_work/1/s/src/FSharp.Core/prim-types.fs
         isTargetJustFilename=False
         result=True  ← SUCCESS because full paths match!
```

**.NET 10 environment logs (FSharp.Core 10.0.100):**
```
[DEBUG] FCS GetDeclarationLocation returned: DeclFound
[DEBUG] DeclFound full FileName: /__w/1/s/src/fsharp/src/FSharp.Core/list.fs
         extracted sourceFile will be: list.fs
[DEBUG] compareRepoPath: targetFile=list.fs vs docName=/__w/1/s/.../list.fs
         isTargetJustFilename=True
         result=False  ← FAILURE because basename ≠ full path!
```

## The Fix

### Modified File: `src/FsAutoComplete.Core/Sourcelink.fs`

The `compareRepoPath` function was modified to:
1. Detect when `targetFile` is just a filename (no `/` separator)
2. Use `EndsWith("/" + filename)` matching for filename-only targets
3. Use exact equality when both are full paths (preserving original behavior)

```fsharp
let private compareRepoPath (d: Document) targetFile =
  // ... normalization code ...

  // Check if targetFile appears to be just a filename (no path separators)
  let targetStr = UMX.untag targetNormalized
  let isTargetJustFilename = not (targetStr.Contains("/"))

  let result =
    if isTargetJustFilename then
      // Target is just a filename, so check if the document path ends with it
      let docStr = UMX.untag docNormalized
      docStr.EndsWith("/" + targetStr, System.StringComparison.Ordinal)
      || docStr = targetStr  // Handle edge case where doc is also just a filename
    else
      // Both are full paths, compare directly
      docNormalized = targetNormalized

  result
```

## Files Modified (with debug logging)

Current state includes debug logging that should be removed for production:

### 1. `src/FsAutoComplete.Core/ParseAndCheckResults.fs`

Debug logs added at:
- Line ~97: Logs FCS `GetDeclarationLocation` result type
- Line ~201: Logs full FileName from FCS and extracted sourceFile
- Line ~209: Logs assembly and source file after recovery
- Line ~226: Logs when FCS returns `ExternalDecl`

### 2. `src/FsAutoComplete.Core/Sourcelink.fs`

Debug logs added at:
- Line ~88-99: Logs comparison details when basename matches
- Line ~305: Logs entry to `tryFetchSourcelinkFile`
- Line ~325: Logs document search result

## Testing the Fix

### Build Commands

```bash
cd /workspaces/fsac-ghub/FsAutoComplete

# Restore tools
dotnet tool restore

# Build for .NET 10
dotnet build -p:BuildNet10=true src/FsAutoComplete/FsAutoComplete.fsproj -c Release
```

Output location: `src/FsAutoComplete/bin/Release/net10.0/fsautocomplete.dll`

### VS Code Configuration

Add to VS Code settings to use custom FSAC build:
```json
{
  "FSharp.fsac.netCoreDllPath": "/path/to/FsAutoComplete/src/FsAutoComplete/bin/Release/net10.0/fsautocomplete.dll"
}
```

### Test Procedure

1. Reload VS Code window
2. Open an F# file
3. Ctrl+hover over `List.map` or similar FSharp.Core function
4. Should show clickable link that navigates to source

### Expected Log Output (with fix)

```
[DEBUG] FCS GetDeclarationLocation returned: DeclFound
Got a declResult of (97,8--97,11) that doesn't exist
[DEBUG] DeclFound full FileName: /__w/1/s/src/fsharp/src/FSharp.Core/list.fs, extracted sourceFile will be: list.fs
[DEBUG] tryRecoverExternalSymbolForNonexistentDecl succeeded: assemblyFile=.../FSharp.Core.dll, sourceFile=list.fs
[DEBUG] tryFetchSourcelinkFile called: dllPath=.../FSharp.Core.dll, targetFile=list.fs, isWindows=False
[DEBUG] compareRepoPath BASENAME MATCH: ..., isTargetJustFilename=True, result=True
[DEBUG] Document search complete: found=True, targetFile=list.fs
```

## Next Steps

### 1. Clean Up Debug Logs

Remove all `[DEBUG]` warn logs added during investigation. The fix itself should remain.

Files to clean:
- `src/FsAutoComplete.Core/ParseAndCheckResults.fs` - Remove 4 debug log blocks
- `src/FsAutoComplete.Core/Sourcelink.fs` - Remove 3 debug log blocks

### 2. Prepare PR for FsAutoComplete Upstream

The fix in `compareRepoPath` should be submitted as a PR to:
https://github.com/fsharp/FsAutoComplete

PR should include:
- Description of the .NET 10 regression
- Explanation of why the fix is needed
- Note that this is a defensive fix (handles both full paths and basenames)

### 3. Consider Reporting FSharp.Core Build Environment Change

~~The underlying issue is that FCS changed behavior between .NET 9 and .NET 10.~~ **CORRECTED:** FCS behavior is the same. The issue is that FSharp.Core's **build environment changed**:
- FSharp.Core 9.x: Built on **Windows** (Azure DevOps) → backslash paths in PDB
- FSharp.Core 10.x: Built on **Linux** (GitHub Actions) → forward slash paths in PDB

This could be reported to the F# team, but it's arguably not a bug in FSharp.Core itself - the PDB paths are correct for the build environment. The bug is in FSAC's assumptions about path formats.

### 4. Consider Adding Decompilation Fallback

The `DeclFound` code path (lines 188-207 in ParseAndCheckResults.fs) lacks a decompilation fallback. Adding one would make this more robust:

```fsharp
| FindDeclResult.DeclFound rangeInNonexistentFile ->
    // ... existing code ...
    match! Sourcelink.tryFetchSourcelinkFile assemblyFile sourceFile with
    | Ok localFilePath -> return Ok(...)
    | Error reason ->
        // TODO: Add decompilation fallback here, similar to ExternalDecl path
        return ResultOrString.Error(sprintf "%A" reason)
```

## Relevant Code Paths

### Go-to-Definition Flow

1. `AdaptiveFSharpLspServer.fs` receives `textDocument/definition` request
2. Calls `ParseAndCheckResults.TryFindDeclaration`
3. `TryFindIdentifierDeclaration` calls FCS `GetDeclarationLocation`
4. FCS returns one of:
   - `DeclFound range` - file exists locally
   - `DeclFound range` - file doesn't exist (needs SourceLink/decompile)
   - `ExternalDecl(assembly, symbol)` - external symbol
   - `DeclNotFound reason` - couldn't find
5. For non-existent files: `tryRecoverExternalSymbolForNonexistentDecl` → `Sourcelink.tryFetchSourcelinkFile`
6. For external decls: tries SourceLink first, falls back to `Decompiler.tryFindExternalDeclaration`

### Key Files

| File | Purpose |
|------|---------|
| `ParseAndCheckResults.fs` | Contains `TryFindDeclaration`, `TryFindIdentifierDeclaration` |
| `Sourcelink.fs` | SourceLink integration, downloads source from GitHub |
| `Decompiler.fs` | ILSpy-based decompilation fallback |

## Debugging Tips

### Enable Verbose Logging

The "F#" output channel in VS Code shows FSAC logs. WRN level and above are visible by default.

### Check FSharp.Core PDB

The SourceLink metadata is embedded in the FSharp.Core PDB. Check it with:
```bash
# Location
ls ~/.nuget/packages/fsharp.core/10.0.100/lib/netstandard2.1/

# Should have FSharp.Core.dll and FSharp.Core.pdb
```

### Trace SourceLink URLs

The SourceLink JSON maps paths like `/__w/1/s/src/fsharp/*` to GitHub raw URLs.

---

*Document created: 2024-12-21*
*Issue discovered while investigating .NET 10 go-to-definition failures*

*Updated: 2025-12-21*
*Root cause confirmed via .NET 9 environment testing - PDB path format difference, not FCS behavior change*
