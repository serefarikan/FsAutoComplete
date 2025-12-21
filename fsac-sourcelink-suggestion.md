# FSAC SourceLink Logic Improvement Suggestion

## Summary

The current FSAC SourceLink implementation has a logic flaw: it unnecessarily discards path information from the FCS result, then tries to recover by matching against PDB document entries. A simpler fix would be to use the FCS result directly.

## Current Flow (Flawed)

```
1. FCS returns: DeclFound { FileName = "/__w/1/s/src/fsharp/src/FSharp.Core/list.fs" }
                                        ↑ This IS the PDB path

2. FSAC calls getFileName():
   - On Windows: uses full path
   - On Linux: Path.GetFileName → "list.fs"  ← DISCARDS PATH INFO

3. FSAC searches PDB for document matching "list.fs"

4. PDB contains: "/__w/1/s/src/fsharp/src/FSharp.Core/list.fs"

5. Comparison: "list.fs" == "/__w/1/s/.../list.fs" → NO MATCH → ERROR
```

## Evidence from .NET 10 Testing

```
[DEBUG] FCS GetDeclarationLocation RAW result:
  DeclFound(FileName='/__w/1/s/src/fsharp/src/FSharp.Core/list.fs', Start=97:8, End=97:11)

[DEBUG] DeclFound FileName analysis:
  raw='/__w/1/s/src/fsharp/src/FSharp.Core/list.fs'
  extracted='list.fs'
  startsWithSlash=True
  containsBackslash=False

[DEBUG] compareRepoPath BASENAME MATCH:
  docName=/__w/1/s/src/fsharp/src/FSharp.Core/list.fs
  targetFile=list.fs
```

**Key observation:** `raw` and `docName` are IDENTICAL. If FSAC used `raw` directly for matching, it would work without any workaround.

## The Flawed Code

```fsharp
// src/FsAutoComplete.Core/ParseAndCheckResults.fs, lines 45-49
let getFileName (loc: range) =
  if Ionide.ProjInfo.ProjectSystem.Environment.isWindows then
    UMX.tag<NormalizedRepoPathSegment> loc.FileName      // Full path on Windows
  else
    UMX.tag<NormalizedRepoPathSegment> (Path.GetFileName loc.FileName)  // BASENAME ONLY on Linux!
```

**Why is this different for Windows vs Linux?**

The original intent seems to have been:
- On Windows, the path is already correct
- On Linux, strip some prefix to get the "real" path

But this assumption is wrong. The FCS `DeclFound.FileName` already contains the PDB source path, which should match PDB document entries directly.

## Why .NET 9 Worked "By Accident"

In .NET 9 testing, we observed:
```
raw='/workspaces/devcont-fs-src-test/D:\a\_work\1\s\src\FSharp.Core\prim-types.fs'
```

This appears to have a workspace prefix concatenated with a Windows-style PDB path.

`Path.GetFileName` on Linux:
- Scans for last `/` (forward slash)
- Finds it after `devcont-fs-src-test/`
- Returns everything after: `D:\a\_work\1\s\src\FSharp.Core\prim-types.fs`

This "accidentally" preserves the full Windows PDB path (because backslashes aren't path separators on Linux), which then matches the PDB document entry.

## Suggested Better Fix

### Why FCS Output Can't Be Used Directly

FCS performs platform-dependent path resolution. If the PDB path doesn't look absolute on the current platform, FCS prepends the workspace directory:

| Scenario | PDB Path | FCS Returns | Direct Match with PDB Doc? |
|----------|----------|-------------|---------------------------|
| Windows PDB on Linux | `D:\a\_work\...\file.fs` | `/workspaces/.../D:\a\_work\...\file.fs` | **No** (FCS added prefix) |
| Linux PDB on Linux | `/__w/.../file.fs` | `/__w/.../file.fs` | **Yes** (already absolute) |

So FSAC cannot simply use the FCS path directly for matching - it must account for the possible workspace prefix.

### The Correct Approach: Suffix Matching

The key insight is that the **PDB document entry is always a suffix of (or equal to) the FCS path**:

- .NET 9: PDB doc `D:\a\_work\...\file.fs` is a suffix of FCS path `/workspaces/.../D:\a\_work\...\file.fs` ✓
- .NET 10: PDB doc `/__w/.../file.fs` equals FCS path `/__w/.../file.fs` ✓

### Recommended Fix

**Option 1: Fix in `getFileName` - Keep full FCS path**

```fsharp
let getFileName (loc: range) =
  // Keep the full FCS path (don't use Path.GetFileName)
  // Normalize backslashes to forward slashes for consistent comparison
  let normalized = loc.FileName.Replace('\\', '/')
  UMX.tag<NormalizedRepoPathSegment> normalized
```

**Option 2: Fix in `compareRepoPath` - Use suffix matching**

```fsharp
let private compareRepoPath (d: Document) fcsPath =
  let docNormalized = normalize d.Name      // PDB document entry
  let fcsNormalized = normalize fcsPath     // FCS path (may have workspace prefix)

  // PDB doc should be suffix of (or equal to) FCS path
  fcsNormalized = docNormalized
  || fcsNormalized.EndsWith("/" + docNormalized)
```

**Why this works:**

```
FCS path:  /workspaces/devcont-fs-src-test/D:/a/_work/1/s/src/FSharp.Core/prim-types.fs
                                          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
PDB doc:                                  D:/a/_work/1/s/src/FSharp.Core/prim-types.fs

fcsPath.EndsWith("/" + pdbDoc) → TRUE ✓
```

### Why the Current `Path.GetFileName` Approach is Wrong

The current code tries to avoid the prefix problem by extracting just the filename:

```fsharp
Path.GetFileName loc.FileName  // → "list.fs"
```

This is **too aggressive** - it throws away all path information, then hopes the basename matches a PDB document. This fails when:
- Multiple files have the same name (e.g., `list.fs` in different directories)
- The PDB document is a full path (which it always is)

The `.NET 9 "accidental success"` only worked because `Path.GetFileName` on Linux doesn't recognize backslashes as separators, so it accidentally preserved the full Windows-style PDB path.

## Current Workaround (Implemented)

The current fix modifies `compareRepoPath` to handle basename-only targets:

```fsharp
if isTargetJustFilename then
  docStr.EndsWith("/" + targetStr, StringComparison.Ordinal)
  || docStr = targetStr
else
  docNormalized = targetNormalized
```

This works but is a workaround for the upstream logic flaw in `getFileName`.

## Confirmed: FCS Path Resolution Behavior

Testing confirmed that FCS performs **path resolution** on the PDB source path:

| PDB Path Type | On Linux | FCS Behavior |
|---------------|----------|--------------|
| Windows-style (`D:\a\_work\...`) | Not recognized as absolute | FCS resolves relative to workspace → prepends `/workspaces/.../` |
| Linux-style (`/__w/1/s/...`) | Recognized as absolute (starts with `/`) | FCS uses directly, no modification |

### .NET 9 Environment (FSharp.Core 9.0.303)

```
FCS RAW:     /workspaces/devcont-fs-src-test/D:\a\_work\1\s\src\FSharp.Core\prim-types.fs
             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ workspace prefix (FCS added)
                                             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ PDB path

startsWithSlash=True, containsBackslash=True

Path.GetFileName finds last '/' after "devcont-fs-src-test/"
extracted:   D:\a\_work\1\s\src\FSharp.Core\prim-types.fs  ← MATCHES PDB doc!
```

### .NET 10 Environment (FSharp.Core 10.0.100)

```
FCS RAW:     /__w/1/s/src/fsharp/src/FSharp.Core/list.fs
             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ PDB path (no prefix)

startsWithSlash=True, containsBackslash=False

Path.GetFileName finds last '/' before "list.fs"
extracted:   list.fs  ← does NOT match PDB doc!
```

## Root Cause Summary

The bug is a combination of factors:

1. **FSharp.Core build environment changed**: 9.x built on Windows, 10.x built on Linux
2. **FCS does path resolution**: Prepends workspace for non-absolute paths
3. **FSAC uses Path.GetFileName**: Which behaves differently for Windows vs Linux paths
4. **.NET 9 works by accident**: The "wrong" behavior accidentally produces the right result

## Remaining Questions

1. **Why does `getFileName` use `Path.GetFileName` on non-Windows?**
   - Was this intentional to handle some edge case?
   - Or was it a bug from the beginning that happened to work due to Windows PDB paths?

2. **What's the correct fix location?**
   - Fix in `getFileName` (don't discard path info)?
   - Fix in `compareRepoPath` (smarter matching)?
   - Both?

## Test Results Summary

| Environment | FSharp.Core | FCS Raw FileName | Path.GetFileName Result | PDB Doc | Match? |
|-------------|-------------|------------------|------------------------|---------|--------|
| .NET 10 | 10.0.100 | `/__w/1/s/.../list.fs` | `list.fs` | `/__w/1/s/.../list.fs` | No* |
| .NET 9 | 9.0.303 | `/workspaces/.../D:\a\_work\...\prim-types.fs` | `D:\a\_work\...\prim-types.fs` | `D:\a\_work\...\prim-types.fs` | Yes |

*Fixed with EndsWith matching in compareRepoPath

---

*Created: 2025-12-21*
*Related: fsac-sourcelink-bug.md*
