# Implementation Plan: Installation Status and Control Improvements

**Goal:** Add installation status API and thread control based on architecture analysis recommendations

**Priority:** Low (optional enhancements, not blocking)
**Risk:** Low (additive changes, no breaking changes)
**Estimated Time:** 2-3 hours

---

## Current State Analysis

### Existing State Tracking
```python
# Line 75
self._install_in_progress = False  # Boolean only

# Lines 107, 279, 283
self._install_in_progress = True   # Set during install
self._install_in_progress = False  # Cleared after install
```

**Limitations:**
- Only tracks if installation is in progress
- No success/failure state
- No error information
- No timestamp
- No thread reference

### Existing APIs
```python
def is_radare2_ready(self) -> bool:
    """Check if radare2 is initialized."""
    return self.radare2 is not None

def reload_radare2(self) -> bool:
    """Attempt to initialize radare2 after installation."""
    # ... (Lines 305-334)
```

**Limitations:**
- No detailed status information
- No way to cancel installation
- No error details if installation failed
- No way to check when installation completed

---

## Improvement 1: Installation Status Tracking

### New State Variables

```python
# In __init__ (after Line 75):
self._install_in_progress = False
self._install_success = None      # None=not started, True=success, False=failed
self._install_error = None        # Exception or error message if failed
self._install_timestamp = None    # When installation started
self._install_thread = None       # Thread reference for control
```

### State Transitions

```
Initial State:
  _install_in_progress: False
  _install_success: None
  _install_error: None
  _install_timestamp: None

Installation Starts:
  _install_in_progress: True
  _install_success: None
  _install_error: None
  _install_timestamp: time.time()

Installation Succeeds:
  _install_in_progress: False
  _install_success: True
  _install_error: None

Installation Fails:
  _install_in_progress: False
  _install_success: False
  _install_error: "Error message"

Installation Cancelled:
  _install_in_progress: False
  _install_success: False
  _install_error: "Cancelled by user"
```

### New Method: get_install_status()

```python
def get_install_status(self) -> dict:
    """
    Get detailed installation status.

    Returns:
        dict with keys:
            - in_progress (bool): Installation currently running
            - success (bool|None): True if succeeded, False if failed, None if not attempted
            - error (str|None): Error message if failed
            - timestamp (float|None): When installation started (Unix timestamp)
            - duration (float|None): How long installation took/is taking (seconds)
    """
    status = {
        "in_progress": self._install_in_progress,
        "success": self._install_success,
        "error": self._install_error,
        "timestamp": self._install_timestamp,
        "duration": None
    }

    if self._install_timestamp:
        if self._install_in_progress:
            # Still running
            status["duration"] = time.time() - self._install_timestamp
        elif self._install_success is not None:
            # Completed (success or failure)
            # Duration was tracked when installation finished
            status["duration"] = self._install_duration
        else:
            # Started but status unknown
            status["duration"] = time.time() - self._install_timestamp

    return status
```

### Usage Example

```python
analyser = CrashAnalyser(binary_path, use_radare2=True)

# Check status
status = analyser.get_install_status()
if status["in_progress"]:
    print(f"Installing... ({status['duration']:.1f}s elapsed)")
elif status["success"]:
    print(f"Installed successfully in {status['duration']:.1f}s")
elif status["success"] is False:
    print(f"Installation failed: {status['error']}")
else:
    print("No installation attempted")
```

---

## Improvement 2: Thread Control API

### Cancel Flag

```python
# In __init__ (after new state variables):
self._install_cancelled = False  # Flag to signal cancellation
```

### New Method: cancel_install()

```python
def cancel_install(self) -> bool:
    """
    Cancel background installation if running.

    Returns:
        True if cancellation was initiated, False if nothing to cancel
    """
    if not self._install_in_progress:
        logger.debug("No installation in progress to cancel")
        return False

    if self._install_thread and self._install_thread.is_alive():
        logger.info("Cancelling radare2 installation")
        self._install_cancelled = True
        # Thread will check flag and exit gracefully
        return True

    return False
```

### Modified install() Function

```python
def install():
    try:
        system = platform.system().lower()
        success = False

        # Check cancellation flag before each major operation
        if self._install_cancelled:
            logger.info("Installation cancelled by user")
            self._install_success = False
            self._install_error = "Cancelled by user"
            self._install_in_progress = False
            return

        if system == "darwin":
            if shutil.which("brew"):
                # Check cancellation before installing
                if self._install_cancelled:
                    logger.info("Installation cancelled before execution")
                    self._install_success = False
                    self._install_error = "Cancelled by user"
                    self._install_in_progress = False
                    return

                success = self._install_package("brew", "radare2", requires_sudo=False)

        # ... similar for linux platforms

        # Track success/failure
        if not self._install_cancelled:
            if success:
                self._verify_radare2_installation()
                self._install_success = True
            else:
                self._install_success = False
                self._install_error = "Installation command failed"
        else:
            self._install_success = False
            self._install_error = "Cancelled by user"

        self._install_in_progress = False
        self._install_duration = time.time() - self._install_timestamp

    except Exception as e:
        logger.error(f"Installation process failed: {e}")
        self._install_success = False
        self._install_error = str(e)
        self._install_in_progress = False
        if self._install_timestamp:
            self._install_duration = time.time() - self._install_timestamp
```

### Usage Example

```python
analyser = CrashAnalyser(binary_path, use_radare2=True)

# Wait a bit
time.sleep(5)

# User decides to cancel
if analyser.cancel_install():
    print("Installation cancelled")

# Check status
status = analyser.get_install_status()
if status["error"] == "Cancelled by user":
    print("Confirmed: installation was cancelled")
```

---

## Implementation Order

### Step 1: Add State Tracking (No Breaking Changes)
**Files:** `packages/binary_analysis/crash_analyser.py`

**Changes:**
- Add new instance variables in `__init__()`
- Track `_install_success`, `_install_error`, `_install_timestamp`, `_install_duration`, `_install_thread`
- Update state in `install()` function
- No changes to existing behavior

**Risk:** VERY LOW - Only additions, no modifications

### Step 2: Implement get_install_status() (Read-Only API)
**Files:** `packages/binary_analysis/crash_analyser.py`

**Changes:**
- Add `get_install_status()` method
- Returns dictionary with all status information
- Pure read operation, no side effects

**Risk:** VERY LOW - Read-only method, cannot break anything

### Step 3: Implement cancel_install() (Cancellation Logic)
**Files:** `packages/binary_analysis/crash_analyser.py`

**Changes:**
- Add `_install_cancelled` flag
- Add `cancel_install()` method
- Modify `install()` to check cancellation flag
- Graceful exit when cancelled

**Risk:** LOW - Only affects installation process, main workflow unaffected

### Step 4: Write Comprehensive Tests
**Files:** `test/test_crash_analyser_install.py` (extend existing)

**New Tests:**
1. `test_get_install_status_not_started` - Status before installation
2. `test_get_install_status_in_progress` - Status during installation
3. `test_get_install_status_success` - Status after success
4. `test_get_install_status_failure` - Status after failure
5. `test_get_install_status_with_duration` - Duration calculation
6. `test_cancel_install_no_installation` - Cancel when nothing running
7. `test_cancel_install_during_installation` - Cancel during install
8. `test_cancel_install_already_completed` - Cancel after complete
9. `test_cancelled_installation_sets_error` - Verify error message
10. `test_cancelled_flag_checked_before_install` - Early cancellation

**Risk:** NONE - Only tests, cannot break production code

### Step 5: Update Documentation
**Files:**
- `packages/binary_analysis/crash_analyser.py` (docstrings)
- `AUTO_INSTALL.md` (API reference section)

**Changes:**
- Document `get_install_status()` with examples
- Document `cancel_install()` with examples
- Add troubleshooting section for cancellation

**Risk:** NONE - Only documentation

### Step 6: Run Full Test Suite
**Verification:**
- All existing tests still pass (136/136)
- All new tests pass (10 additional)
- Total: 146/146 tests

**Risk:** NONE - Just verification

---

## Test Plan

### Test Coverage

**Status API Tests (5 tests):**
1. Status before installation started
2. Status during installation (in progress)
3. Status after successful installation
4. Status after failed installation
5. Duration calculation accuracy

**Cancel API Tests (5 tests):**
1. Cancel when no installation running
2. Cancel during installation
3. Cancel after installation completed
4. Cancelled installation sets correct error
5. Early cancellation (before execution)

**Integration Tests (2 tests):**
1. Status API + reload workflow
2. Cancel + status check workflow

**Total New Tests:** 12

### Test Implementation Strategy

**Pattern:**
```python
class TestInstallationStatus:
    """Test get_install_status() API."""

    @pytest.fixture
    def test_binary(self, tmp_path):
        binary = tmp_path / "test_binary"
        binary.write_bytes(b"\x7fELF" + b"\x00" * 100)
        return binary

    @patch("packages.binary_analysis.crash_analyser.is_radare2_available")
    def test_get_install_status_not_started(self, mock_r2_available, test_binary):
        """Test status when installation not started."""
        mock_r2_available.return_value = True

        analyser = CrashAnalyser(test_binary, use_radare2=False)
        status = analyser.get_install_status()

        assert status["in_progress"] is False
        assert status["success"] is None
        assert status["error"] is None
        assert status["timestamp"] is None
        assert status["duration"] is None
```

---

## Risk Assessment

| Change | Risk Level | Mitigation |
|--------|-----------|------------|
| Add state variables | VERY LOW | Only additions, no modifications |
| get_install_status() | VERY LOW | Read-only, no side effects |
| cancel_install() | LOW | Only affects install process |
| Modify install() | LOW | Graceful cancellation, no breaking changes |
| Add tests | NONE | Tests cannot break production |
| Update docs | NONE | Documentation only |

**Overall Risk:** VERY LOW

**Rollback Plan:**
- Changes are additive only
- Can be reverted in single commit
- Existing functionality unchanged
- All existing tests still pass

---

## Success Criteria

1. All new state variables tracked correctly
2. `get_install_status()` returns accurate information
3. `cancel_install()` gracefully stops installation
4. All 12 new tests pass
5. All 136 existing tests still pass
6. Total: 148/148 tests passing (100%)
7. Documentation updated completely
8. No performance regressions
9. No breaking changes to existing API

---

## Expected Outcome

**Before:**
- Installation status: Boolean only (in progress or not)
- No error details
- No cancellation mechanism
- User has limited visibility

**After:**
- Installation status: Complete details (success, error, duration)
- Full error information when failures occur
- Cancellation API available
- User has complete visibility and control

**API Example:**
```python
# Complete workflow
analyser = CrashAnalyser(binary_path, use_radare2=True)

# Monitor status
while analyser.get_install_status()["in_progress"]:
    status = analyser.get_install_status()
    print(f"Installing... {status['duration']:.1f}s")
    time.sleep(1)

# Check result
status = analyser.get_install_status()
if status["success"]:
    analyser.reload_radare2()
    print("radare2 ready!")
elif status["error"]:
    print(f"Installation failed: {status['error']}")
```

**Score Improvement:**
- Current subordination score: 8/10
- After improvements: 9/10 (better observability and control)

---

## Timeline

**Total Estimated Time:** 2-3 hours

1. Add state tracking: 20 minutes
2. Implement get_install_status(): 15 minutes
3. Implement cancel_install(): 30 minutes
4. Write 12 tests: 60 minutes
5. Run full test suite: 10 minutes
6. Update documentation: 20 minutes
7. Commit and review: 10 minutes

**Total:** ~165 minutes (2.75 hours)

---

## Implementation Notes

**Careful Considerations:**

1. **Thread Safety:**
   - Use flag (`_install_cancelled`) for communication
   - Don't terminate thread forcefully (Python doesn't support it safely)
   - Thread checks flag and exits gracefully

2. **State Consistency:**
   - Always update all related state variables together
   - Use try/finally to ensure state is updated even on exception

3. **Backward Compatibility:**
   - All new methods are additions
   - Existing `is_radare2_ready()` and `reload_radare2()` unchanged
   - No breaking changes to constructor or existing methods

4. **Error Handling:**
   - Capture all exceptions and store in `_install_error`
   - Provide meaningful error messages
   - Never raise exceptions from background thread (just log)

5. **Testing:**
   - Mock time.time() for duration testing
   - Mock threading.Thread for cancellation testing
   - Test all state transitions
   - Test edge cases (cancel when not running, double cancel, etc.)

**Ready to implement:** YES - Plan is complete and well-structured
