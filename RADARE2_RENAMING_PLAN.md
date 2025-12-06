# Complete r2 → radare2 Renaming Plan

**Date:** December 4, 2025
**Scope:** Rename ALL instances of "r2" to "radare2" across codebase
**Estimated Time:** 30-45 minutes
**Test Coverage:** Maintain 95/95 passing tests

---

## Part 1: Scope Analysis

### Files to Rename (1 file)
- `packages/binary_analysis/r2_wrapper.py` → `radare2_wrapper.py`

### Classes to Rename (3 classes)
- `R2Wrapper` → `Radare2Wrapper`
- `R2Function` → `Radare2Function`
- `R2DisasmInstruction` → `Radare2DisasmInstruction`

### Functions to Rename (1 function)
- `is_r2_available()` → `is_radare2_available()`

### Variables to Rename (~30+ instances)
- `self.r2` → `self.radare2`
- `r2_path` → `radare2_path`
- `r2_wrapper` → `radare2_wrapper`
- `use_r2` → `use_radare2`

### Config Constants to Rename (5 constants)
- `R2_PATH` → `RADARE2_PATH`
- `R2_TIMEOUT` → `RADARE2_TIMEOUT`
- `R2_ANALYSIS_DEPTH` → `RADARE2_ANALYSIS_DEPTH`
- `R2_ANALYSIS_TIMEOUT` → `RADARE2_ANALYSIS_TIMEOUT`
- `R2_ENABLE` → `RADARE2_ENABLE`

### Import Statements (~15 files)
All files importing r2_wrapper need updates:
- `from packages.binary_analysis.r2_wrapper import ...`
  → `from packages.binary_analysis.radare2_wrapper import ...`

### Test Files (~20 files)
- `test/test_r2_wrapper.py` → `test_radare2_wrapper.py`
- `implementation-tests/test_step_*` → Update references

### Comments and Docstrings (~50+ instances)
- All "r2" in comments → "radare2"
- All "R2" in docstrings → "Radare2"

---

## Part 2: Reference Count

**Found references:**
- Lowercase "r2": 37 instances
- Uppercase "R2": 25 instances
- **Total: ~62 references to rename**

---

## Part 3: Renaming Strategy (18-Step Process)

### Step 0: Build Tests FIRST

**Test 1: Import compatibility**
```python
def test_radare2_wrapper_imports():
    """Verify new import names work."""
    from packages.binary_analysis.radare2_wrapper import (
        Radare2Wrapper,
        Radare2Function,
        Radare2DisasmInstruction,
        is_radare2_available
    )
    assert Radare2Wrapper is not None
    assert is_radare2_available is not None
```

**Test 2: Config naming**
```python
def test_radare2_config_names():
    """Verify config constants renamed."""
    from core.config import RaptorConfig
    assert hasattr(RaptorConfig, 'RADARE2_PATH')
    assert hasattr(RaptorConfig, 'RADARE2_ENABLE')
    assert not hasattr(RaptorConfig, 'R2_PATH')  # Old name gone
```

**Test 3: Variable naming**
```python
def test_crash_analyser_radare2_attribute():
    """Verify CrashAnalyser uses .radare2 not .r2"""
    from packages.binary_analysis.crash_analyser import CrashAnalyser
    analyser = CrashAnalyser(test_binary)
    assert hasattr(analyser, 'radare2')
    assert not hasattr(analyser, 'r2')  # Old name gone
```

**Test 4: No r2 references remain**
```python
def test_no_r2_references_in_code():
    """Verify all r2 references renamed to radare2."""
    import subprocess
    result = subprocess.run(
        ["grep", "-r", r"\br2\b", "packages/binary_analysis/", "--include=*.py"],
        capture_output=True, text=True
    )
    # Should only find "radare2", not standalone "r2"
    assert "radare2" in result.stdout or result.stdout == ""
```

### Step 1: Rename File
```bash
git mv packages/binary_analysis/r2_wrapper.py packages/binary_analysis/radare2_wrapper.py
```

### Step 2: Rename Classes (in radare2_wrapper.py)
1. R2Wrapper → Radare2Wrapper
2. R2Function → Radare2Function
3. R2DisasmInstruction → Radare2DisasmInstruction

### Step 3: Rename Functions (in radare2_wrapper.py)
1. is_r2_available → is_radare2_available

### Step 4: Rename Variables (in radare2_wrapper.py)
1. r2_path → radare2_path (all instances)
2. Update all internal references

### Step 5: Rename Config Constants (in core/config.py)
1. R2_PATH → RADARE2_PATH
2. R2_TIMEOUT → RADARE2_TIMEOUT
3. R2_ANALYSIS_DEPTH → RADARE2_ANALYSIS_DEPTH
4. R2_ANALYSIS_TIMEOUT → RADARE2_ANALYSIS_TIMEOUT
5. R2_ENABLE → RADARE2_ENABLE

### Step 6: Update crash_analyser.py
1. Import: r2_wrapper → radare2_wrapper
2. Variable: self.r2 → self.radare2
3. Parameter: use_r2 → use_radare2
4. Config: R2_* → RADARE2_*
5. Comments: "r2" → "radare2"

### Step 7: Update raptor_fuzzing.py
- No direct r2 references (uses CrashAnalyser)
- May have comments to update

### Step 8: Update __init__.py
```python
# OLD:
from .r2_wrapper import R2Wrapper, is_r2_available

# NEW:
from .radare2_wrapper import Radare2Wrapper, is_radare2_available
```

### Step 9: Update test/test_r2_wrapper.py
1. Rename file: test_radare2_wrapper.py
2. Update all imports
3. Update all class references
4. Update all variable names

### Step 10: Update implementation-tests/* (8 files)
1. test_step_1_1_string_filtering.py
2. test_step_1_2_call_graph.py
3. test_step_1_3_backward_disasm.py
4. test_step_1_4_tool_name.py
5. test_step_2_1_default_analysis.py
6. test_step_2_3_timeout_scaling.py
7. test_step_2_4_security_helper.py
8. test_step_2_5_analysis_free.py
9. test_integration_crash_analysis.py
10. test_with_real_binary.py

### Step 11: Update Comments and Docstrings
Search and replace in all files:
- "r2 " → "radare2 "
- "r2." → "radare2."
- "r2)" → "radare2)"
- " R2 " → " Radare2 "
- Keep: "radare2" command name (already correct)

### Step 12: Update Documentation Files
- RADARE2_INTEGRATION.md
- IMPLEMENTATION_SUMMARY.md
- README.md
- Any other docs mentioning r2

### Step 13: Run Renamed Tests
```bash
pytest test/test_radare2_wrapper.py -v
```

### Step 14: Run Implementation Tests
```bash
pytest implementation-tests/ -v
```

### Step 15: Run ALL Regression Tests
```bash
pytest test/ implementation-tests/ -v
# Expected: 95/95 pass
```

### Step 16: Verify No "r2" Remains
```bash
grep -r "\br2\b" packages/binary_analysis/ core/config.py --include="*.py"
# Should find ZERO standalone "r2" (only "radare2")
```

### Step 17: Persona Review
- Developer: Code quality check
- Architect: Pattern consistency
- QA: Test coverage
- Security: No issues introduced

### Step 18: Get User Approval
- Show summary of changes
- Confirm all tests pass
- Get approval to proceed with Phase 3+4

---

## Part 4: Files Affected (Complete List)

### Production Code (4 files)
1. `packages/binary_analysis/r2_wrapper.py` → `radare2_wrapper.py` (RENAME)
2. `packages/binary_analysis/crash_analyser.py` (UPDATE imports, variables)
3. `packages/binary_analysis/__init__.py` (UPDATE imports)
4. `core/config.py` (RENAME constants)

### Test Code (11 files)
1. `test/test_r2_wrapper.py` → `test_radare2_wrapper.py` (RENAME)
2. `implementation-tests/test_step_1_1_string_filtering.py` (UPDATE)
3. `implementation-tests/test_step_1_2_call_graph.py` (UPDATE)
4. `implementation-tests/test_step_1_3_backward_disasm.py` (UPDATE)
5. `implementation-tests/test_step_1_4_tool_name.py` (UPDATE)
6. `implementation-tests/test_step_2_1_default_analysis.py` (UPDATE)
7. `implementation-tests/test_step_2_3_timeout_scaling.py` (UPDATE)
8. `implementation-tests/test_step_2_4_security_helper.py` (UPDATE)
9. `implementation-tests/test_step_2_5_analysis_free.py` (UPDATE)
10. `implementation-tests/test_integration_crash_analysis.py` (UPDATE)
11. `implementation-tests/test_with_real_binary.py` (UPDATE)

### Documentation (3+ files)
1. `RADARE2_INTEGRATION.md` (UPDATE)
2. `IMPLEMENTATION_SUMMARY.md` (UPDATE)
3. `README.md` (UPDATE)

### Total: 18 files affected

---

## Part 5: Safety Checklist

- [ ] All imports updated
- [ ] All class names updated
- [ ] All function names updated
- [ ] All variable names updated
- [ ] All config constants updated
- [ ] All test files updated
- [ ] All comments updated
- [ ] All docstrings updated
- [ ] No "r2" references remain (except "radare2")
- [ ] All 95 tests still pass
- [ ] No breaking changes introduced
- [ ] Git history preserved (use git mv for renames)

---

## Part 6: Rollback Plan

If anything breaks:
```bash
git reset --hard HEAD
# OR
git checkout packages/binary_analysis/r2_wrapper.py
git checkout core/config.py
git checkout test/test_r2_wrapper.py
# ... restore other files
```

---

## Part 7: Expected Timeline

1. Build tests: 5 minutes
2. Rename file: 1 minute
3. Update radare2_wrapper.py: 5 minutes
4. Update crash_analyser.py: 3 minutes
5. Update config.py: 2 minutes
6. Update __init__.py: 1 minute
7. Update test files: 10 minutes
8. Update comments/docs: 5 minutes
9. Run tests: 3 minutes
10. Verify and review: 5 minutes

**Total: ~40 minutes**

---

## Part 8: Success Criteria

✅ **All 95 tests pass**
✅ **Zero "r2" references remain (only "radare2")**
✅ **All imports work**
✅ **All configs accessible**
✅ **Git history clean**
✅ **No breaking changes**

---

**Ready to execute renaming following strict 18-step process.**

