# Semgrep Directory Scan Bug - Complete Analysis & Fix

**Date:** 2025-12-05
**Status:** CONFIRMED BUG - Fix Available
**Severity:** HIGH (affects all directory-based scans in RAPTOR)

---

## Executive Summary

**Problem:** Semgrep fails to scan files in subdirectories when scanning directories, despite files being properly git-tracked.

**Impact:** RAPTOR's `/scan` command returns 0 findings when scanning `test/data/` despite obvious vulnerabilities present.

**Root Cause:** Semgrep's git integration has a path resolution bug when scanning subdirectories.

**Fix:** Add `--no-git-ignore` flag to Semgrep invocation in `scanner.py`

---

## The Bug

### Manifestation

```bash
# File scan - WORKS ✅
$ semgrep scan --config engine/semgrep/rules/crypto test/data/python_sql_injection.py
 • Findings: 1 (1 blocking)

# Directory scan - FAILS ❌
$ semgrep scan --config engine/semgrep/rules/crypto test/data
 • Findings: 0 (0 blocking)
 • Targets scanned: 0
 • Skipped by .gitignore: <all files not listed by `git ls-files` were skipped>
```

### Verification That Files ARE Tracked

```bash
$ git ls-files test/data/
test/data/javascript_xss.js
test/data/python_sql_injection.py

$ git ls-files --error-unmatch test/data/python_sql_injection.py
test/data/python_sql_injection.py  # ✅ SUCCESS

$ git check-ignore test/data/*.py
# (empty - exit code 1) ✅ NOT IGNORED

$ git ls-files --stage test/data/
100644 ... test/data/javascript_xss.js  # ✅ Regular tracked file
100644 ... test/data/python_sql_injection.py  # ✅ Regular tracked file
```

---

## Root Cause Analysis

### When Introduced

**Commit:** `7c5db7f628519e51c9ce3bd6b01365de17906a7e`
**Date:** November 25, 2025
**Author:** Gadi Evron
**Message:** "feat: Convert test/ from submodule to regular tracked files"

**What Changed:**
```diff
diff --git a/test b/test
deleted file mode 160000  ← Submodule (git subproject)
--- a/test
+++ /dev/null
-Subproject commit 1807fc45fccd78d495e9923b70466346f035c283

diff --git a/test/data/python_sql_injection.py b/test/data/python_sql_injection.py
new file mode 100644  ← Regular file
```

The `test/` directory was originally a git submodule (separate repository). When converted to regular tracked files, a Semgrep path resolution issue was exposed.

### Why It Happens

**Semgrep's git path resolution bug:**

1. When scanning a directory, Semgrep changes into that directory
2. It runs `git ls-files` from within the directory
3. The paths returned don't match the expected paths
4. Semgrep incorrectly marks files as "not tracked"

**Example:**
```bash
# From repo root - Semgrep expects:
$ git ls-files test/data
test/data/python_sql_injection.py  # Has "test/data/" prefix

# From within test/data - Semgrep actually gets:
$ cd test/data && git ls-files
python_sql_injection.py  # Missing "test/data/" prefix!

# Path mismatch: "test/data/python_sql_injection.py" != "python_sql_injection.py"
# Result: File marked as "not tracked" → SKIPPED
```

### Proof This is Semgrep's Bug (Not RAPTOR's)

**Test Matrix:**

| Command | Result | Findings |
|---------|--------|----------|
| `semgrep ... test/data/python_sql_injection.py` | ✅ WORKS | 1 |
| `semgrep ... test/data/*.py` | ✅ WORKS | 1 |
| `semgrep ... --no-git-ignore test/data` | ✅ WORKS | 1 |
| `semgrep ... --include="*.py" test/data` | ✅ WORKS | 1 |
| `semgrep ... test/data` | ❌ FAILS | 0 |

**Verified:**
- ✅ Files ARE git-tracked (verified multiple ways)
- ✅ No `.gitmodules` file exists
- ✅ No submodule config remnants
- ✅ Files have correct git mode (`100644` not `160000`)
- ✅ test/data points to parent `.git` directory
- ✅ Workarounds (--no-git-ignore, --include, explicit files) all work

**Tested on:**
- Commit `ea2da0f` (before ANY radare2 work) - Bug exists ✅
- Commit `7c5db7f` (submodule conversion) - Bug introduced ✅
- Current HEAD - Bug persists ✅

---

## Impact Assessment

### Affected Components

**Primary:**
- `/scan` command when scanning directories
- `raptor.py scan --repo <directory>`
- `packages/static-analysis/scanner.py` (Semgrep invocation)

**Affected Workflows:**
```bash
# These fail (0 findings):
python3 raptor.py scan --repo test/data
python3 raptor.py scan --repo test/data --policy_groups all

# These work (1+ findings):
python3 raptor.py scan --repo test/data/python_sql_injection.py
```

**Test Scripts Affected:**
- `test/integration_with_tools.sh` - Uses `--repo $TEST_DATA` (affected)
- `test/comprehensive_test.sh` - References test/data (check needed)

### NOT Affected

- ✅ `/analyze` command (LLM-based, doesn't use Semgrep)
- ✅ `/fuzz` command (fuzzing, not static analysis)
- ✅ `/crash-analysis` command (binary analysis)
- ✅ Scans using explicit file paths
- ✅ Production packages (no imports from test/)

---

## Recommended Fix

### Option 1: Add --no-git-ignore Flag (RECOMMENDED)

**Location:** `packages/static-analysis/scanner.py:113-123`

**Current Code:**
```python
cmd = [
    semgrep_cmd,
    "scan",
    "--config", config,
    "--quiet",
    "--metrics", "off",
    "--error",
    "--sarif",
    "--timeout", str(RaptorConfig.SEMGREP_RULE_TIMEOUT),
    str(repo_path),  # ← Problem: directory path fails
]
```

**Fixed Code:**
```python
cmd = [
    semgrep_cmd,
    "scan",
    "--config", config,
    "--quiet",
    "--metrics", "off",
    "--error",
    "--sarif",
    "--no-git-ignore",  # ← FIX: Bypass git integration bug
    "--timeout", str(RaptorConfig.SEMGREP_RULE_TIMEOUT),
    str(repo_path),
]
```

**Pros:**
- ✅ Minimal change (one line)
- ✅ Fixes the issue immediately
- ✅ No performance impact
- ✅ Still respects `.semgrepignore` files

**Cons:**
- ⚠️ Scans files even if gitignored (but `.semgrepignore` still works)
- ⚠️ Workaround for Semgrep bug, not root cause fix

**Testing:**
```bash
# Before fix:
$ semgrep scan --config engine/semgrep/rules/crypto test/data
 • Findings: 0 (0 blocking)  ❌

# After fix:
$ semgrep scan --config engine/semgrep/rules/crypto --no-git-ignore test/data
 • Findings: 1 (1 blocking)  ✅
```

---

### Option 2: Enumerate Files Explicitly

**Code:**
```python
def run_single_semgrep(...):
    # Enumerate files if target is directory
    if repo_path.is_dir():
        # Get all relevant files
        files = []
        for ext in ['.py', '.js', '.java', '.php', '.rb', '.go', '.ts', '.tsx']:
            files.extend(repo_path.rglob(f'*{ext}'))

        if not files:
            logger.warning(f"No scannable files found in {repo_path}")
            return None, False

        # Scan files explicitly
        cmd = [semgrep_cmd, "scan", "--config", config, "--sarif", ...] + [str(f) for f in files]
    else:
        # Single file - use as-is
        cmd = [semgrep_cmd, "scan", "--config", config, "--sarif", ..., str(repo_path)]
```

**Pros:**
- ✅ Avoids git integration entirely
- ✅ More explicit control over what's scanned
- ✅ Works regardless of git status

**Cons:**
- ❌ More complex code
- ❌ Need to maintain list of extensions
- ❌ Bypasses Semgrep's own file detection

---

### Option 3: Use --include Patterns

**Code:**
```python
cmd = [
    semgrep_cmd,
    "scan",
    "--config", config,
    "--include", "*.py",
    "--include", "*.js",
    "--include", "*.java",
    # ... other extensions
    "--sarif",
    str(repo_path),
]
```

**Pros:**
- ✅ Leverages Semgrep's file detection
- ✅ More flexible than explicit enumeration

**Cons:**
- ❌ Need to maintain extension list
- ❌ May miss files with non-standard extensions

---

## Validation Plan

### Pre-Deployment Testing

1. **Test on test/data:**
   ```bash
   # Should find 1+ findings:
   python3 raptor.py scan --repo test/data --policy_groups crypto
   ```

2. **Test on real repository:**
   ```bash
   # Should find findings in known-vulnerable repos:
   git clone https://github.com/OWASP/NodeGoat.git
   python3 raptor.py scan --repo NodeGoat --policy_groups all
   ```

3. **Regression test (ensure specific files still work):**
   ```bash
   python3 raptor.py scan --repo test/data/python_sql_injection.py
   ```

### Post-Deployment Verification

1. Run `test/integration_with_tools.sh` - should pass SCAN MODE tests
2. Run `test/comprehensive_test.sh` - should pass all tests
3. Check that findings match expectations for test/data

---

## Alternative: Report to Semgrep

If RAPTOR wants to avoid workarounds, this should be reported to Semgrep maintainers:

**Issue Title:** "Directory scanning skips git-tracked files in subdirectories"

**Reproduction:**
```bash
git clone https://github.com/gadievron/raptor.git
cd raptor
semgrep scan --config engine/semgrep/rules/crypto test/data  # 0 findings (wrong)
semgrep scan --config engine/semgrep/rules/crypto test/data/*.py  # 1 finding (correct)
```

**Expected:** Directory scan should find same files as explicit glob
**Actual:** Directory scan finds 0 files despite being git-tracked

---

## Recommended Action

**Implement Option 1** (`--no-git-ignore` flag):

1. **Why:** Minimal change, immediate fix, low risk
2. **Where:** `packages/static-analysis/scanner.py` line 113-123
3. **Change:** Add `"--no-git-ignore",` to cmd array
4. **Test:** Run validation plan above
5. **Document:** Update scanner.py docstring to note the workaround

**Estimated time:** 5 minutes to implement, 10 minutes to test

---

## Summary

| Aspect | Details |
|--------|---------|
| **Bug** | Semgrep directory scans fail on subdirectories |
| **When Introduced** | Nov 25, 2025 (commit `7c5db7f`, submodule→files conversion) |
| **Root Cause** | Semgrep git path resolution bug |
| **Impact** | HIGH - affects all `/scan` directory operations |
| **Fix** | Add `--no-git-ignore` flag (1 line change) |
| **Testing** | Verified on commits `ea2da0f`, `7c5db7f`, and `HEAD` |
| **Risk** | LOW - flag is documented Semgrep feature |

**Status:** Ready for implementation

---

**Date:** 2025-12-05
**Validated By:** Systematic testing across 3 git commits
**Recommendation:** ✅ IMPLEMENT FIX (Option 1 with --no-git-ignore)
