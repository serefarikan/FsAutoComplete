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

## Implemented Fix (Improved)

The improved fix addresses the root cause instead of working around it:

**1. Changed `getFileName` (ParseAndCheckResults.fs):**
```fsharp
let getFileName (loc: range) =
  // Keep the full FCS path, just normalize backslashes to forward slashes.
  // FCS may prepend a workspace prefix if the PDB path isn't absolute on this platform.
  // The comparison in Sourcelink.compareRepoPath will handle the prefix via suffix matching.
  let normalized = loc.FileName.Replace('\\', '/')
  UMX.tag<NormalizedRepoPathSegment> normalized
```

**2. Changed `compareRepoPath` (Sourcelink.fs):**
```fsharp
// PDB doc should be suffix of (or equal to) FCS path
let result =
  fcsStr = docStr
  || fcsStr.EndsWith("/" + docStr, System.StringComparison.Ordinal)
```

This correctly handles both scenarios:
- When FCS adds workspace prefix: suffix matching finds the PDB doc
- When FCS returns path as-is: exact matching works

**Advantages over the original workaround:**
- Uses full path information instead of just basename
- Correctly handles files with same name in different directories
- More robust and semantically correct

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

## Verified Test Results with Improved Fix

### .NET 10 Environment (FSharp.Core 10.0.100)

```
[DEBUG] FCS GetDeclarationLocation RAW result:
  DeclFound(FileName='/__w/1/s/src/fsharp/src/FSharp.Core/list.fs', ...)

[DEBUG] DeclFound FileName analysis:
  raw='/__w/1/s/src/fsharp/src/FSharp.Core/list.fs'
  extracted='/__w/1/s/src/fsharp/src/FSharp.Core/list.fs'  ← Full path preserved!
  startsWithSlash=True, containsBackslash=False

[DEBUG] compareRepoPath:
  fcsPath=/__w/1/s/src/fsharp/src/FSharp.Core/list.fs
  docNormalized=/__w/1/s/src/fsharp/src/FSharp.Core/list.fs
  result=True  ← EXACT MATCH

[DEBUG] Document search complete: found=True
```

**Match type:** Exact equality (`fcsPath == pdbDoc`)

### .NET 9 Environment (FSharp.Core 9.0.303)

```
[DEBUG] FCS GetDeclarationLocation RAW result:
  DeclFound(FileName='/workspaces/devcont-fs-src-test/D:\a\_work\1\s\src\FSharp.Core\prim-types.fs', ...)

[DEBUG] DeclFound FileName analysis:
  raw='/workspaces/devcont-fs-src-test/D:\a\_work\1\s\src\FSharp.Core\prim-types.fs'
  extracted='/workspaces/devcont-fs-src-test/D:/a/_work/1/s/src/FSharp.Core/prim-types.fs'
             ↑ backslashes normalized to forward slashes
  startsWithSlash=True, containsBackslash=True

[DEBUG] compareRepoPath:
  fcsPath=/workspaces/devcont-fs-src-test/D:/a/_work/1/s/src/FSharp.Core/prim-types.fs
  docNormalized=D:/a/_work/1/s/src/FSharp.Core/prim-types.fs
  result=True  ← SUFFIX MATCH

  fcsPath.EndsWith("/" + pdbDoc) =
  "/workspaces/.../D:/a/_work/.../prim-types.fs".EndsWith("/D:/a/_work/.../prim-types.fs") = TRUE

[DEBUG] Document search complete: found=True
```

**Match type:** Suffix matching (`fcsPath.EndsWith("/" + pdbDoc)`)

## Why This Approach is Better

### Comparison: Original Workaround vs Improved Fix

| Aspect | Original Workaround | Improved Fix |
|--------|---------------------|--------------|
| **Path info used** | Basename only (`list.fs`) | Full path |
| **Match strategy** | `pdbDoc.EndsWith(basename)` | `fcsPath.EndsWith(pdbDoc)` or exact |
| **Same-name files** | Could match wrong file | Correctly distinguished |
| **Semantic correctness** | Hack that happens to work | Matches FCS/PDB relationship |

### Why Suffix Matching on FCS Path is Correct

The key insight is understanding the relationship between FCS path and PDB document:

```
FCS path = [optional workspace prefix] + PDB document path
```

Therefore:
- **PDB doc is always a suffix of FCS path** (or equal to it)
- Checking `fcsPath.EndsWith("/" + pdbDoc)` correctly handles the prefix
- Checking `fcsPath == pdbDoc` handles the no-prefix case

### Why Basename Matching is Fragile

The original workaround used basename matching:
```fsharp
pdbDoc.EndsWith("/" + basename)  // e.g., pdbDoc.EndsWith("/list.fs")
```

This is fragile because:
1. **Multiple files with same name:** If PDB has both `src/FSharp.Core/list.fs` and `tests/list.fs`, basename matching could return the wrong one
2. **Discards useful information:** The full path is available, why throw it away?
3. **Relies on accident:** Only worked in .NET 9 because `Path.GetFileName` accidentally preserved Windows paths on Linux

### Summary

| Environment | FCS Path | PDB Doc | Match Type | Result |
|-------------|----------|---------|------------|--------|
| .NET 10 | `/__w/.../list.fs` | `/__w/.../list.fs` | Exact | ✓ |
| .NET 9 | `/workspaces/.../D:/a/_work/.../prim-types.fs` | `D:/a/_work/.../prim-types.fs` | Suffix | ✓ |

Both scenarios work correctly with the improved fix!

---

*Created: 2025-12-21*
*Updated: 2025-12-21 - Added verified test results from both environments*
*Related: fsac-sourcelink-bug.md*
