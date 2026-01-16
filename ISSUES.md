# Project-Alice Fork: Comprehensive Issue Analysis & Fix Plan

**Generated:** 2026-01-15T23:00:00Z
**Commit:** a90dd964a
**Branch:** main
**Source:** DeepWiki (schombert/Project-Alice) + Repository Analysis + Oracle Guidance

---

## Executive Summary

This fork inherits a large, mature C++ game engine with strong architectural foundations (ECS via dcon, determinism focus, SPSC queues for UI). However, it contains several **high-severity** issues that can cause crashes, data loss, or multiplayer desyncs. The most critical issues are in **serialization** (unchecked ZSTD decompression, raw memory ops) and **CI security** (opencode workflow).

**Top 3 Immediate Fixes:**
1. **Serialization Safety** (CRITICAL) - Add bounds checks, ZSTD error handling, RAII buffers
2. **CI Security** (CRITICAL) - Secure opencode workflow, pin actions, add test execution
3. **Determinism Testing** (HIGH) - Add cross-platform CI for determinism tests

---

## Critical Issues (Fix Immediately)

### 1. Serialization Decompression Vulnerability (CRITICAL)

**File:** `src/gamestate/serialization.cpp`
**Severity:** CRITICAL (crash, OOM, data corruption, DoS)
**Evidence:** TODO comment + unchecked ZSTD_decompress + raw new[]/delete[]

**Problem:**
```cpp
// Current code (simplified):
uint8_t* temp_buffer = new uint8_t[decompressed_length];
// TODO: allocate memory for decompression and decompress into it
ZSTD_decompress(temp_buffer, decompressed_length, ptr_in + sizeof(uint32_t) * 2, section_length);
function(temp_buffer, decompressed_length);
delete[] temp_buffer;
```

**Risks:**
- No bounds checking on `decompressed_length` → OOM on malformed save
- No ZSTD error checking → silent corruption
- Raw allocation → memory leaks on exceptions
- No validation of `section_length` vs available data → buffer overruns

**Fix:**
1. Replace raw allocation with `std::vector<uint8_t>`
2. Add bounds checks:
   - Validate `decompressed_length` ≤ sane limit (e.g., 256MiB)
   - Validate `section_length` fits in remaining buffer
   - Check for integer overflow
3. Validate ZSTD return value with `ZSTD_isError()`
4. Handle decompression failure gracefully (error accumulation)

**Files to Modify:**
- `src/gamestate/serialization.cpp` (primary)
- `src/gamestate/serialization.hpp` (if needed for error types)

**Test Coverage Needed:**
- Round-trip serialization of small state
- Truncated compressed buffer (should fail gracefully)
- Corrupted compressed block (should fail gracefully)
- Oversized decompressed_length (should fail gracefully)

---

### 2. CI Security: opencode.yml Workflow (CRITICAL)

**File:** `.github/workflows/opencode.yml`
**Severity:** CRITICAL (secret exposure risk)
**Evidence:** Comment-triggered workflow with OPENCODE_API_KEY secret

**Problem:**
- Workflow triggers on `issue_comment` and `pull_request_review_comment` (public events)
- Anyone with comment access can trigger the workflow
- Third-party action `anomalyco/opencode/github@latest` receives the secret
- No maintainer verification before secret usage
- Floating tag `@latest` is a supply-chain risk

**Risks:**
- Secret exfiltration via compromised third-party action
- Unauthorized workflow runs
- Supply-chain attack via tag manipulation

**Fix:**
1. **Immediate:** Add maintainer-only gating or disable secret injection
2. Pin action to specific commit SHA
3. Remove unnecessary permissions (`id-token: write` unless required)
4. Add explicit minimal permissions block
5. Consider moving to `workflow_dispatch` for manual approval

**Files to Modify:**
- `.github/workflows/opencode.yml`

**Verification:**
- Test that non-maintainer comments don't trigger secret-using steps
- Verify action pinning works
- Rotate OPENCODE_API_KEY if exposure suspected

---

### 3. Missing Test Execution in CI (HIGH)

**File:** `.github/workflows/build.yml`
**Severity:** HIGH (no test coverage in CI)
**Evidence:** Build matrix exists but no `ctest` or test execution

**Problem:**
- CI builds but doesn't run tests
- No sanitizer coverage (ASAN/UBSAN)
- No artifact upload for debugging
- No ccache/build caching

**Risks:**
- Regressions slip through unnoticed
- Harder to debug CI failures
- Slower CI due to no caching

**Fix:**
1. Add `ctest` execution step
2. Add sanitizer job (ASAN + UBSAN)
3. Add artifact upload for logs/binaries
4. Add ccache/actions/cache for build speed

**Files to Modify:**
- `.github/workflows/build.yml`

---

## High Priority Issues (Fix Soon)

### 4. Large, Complex Functions (HIGH)

**Files:** `src/ai/ai.cpp`, `src/economy/economy.cpp`
**Severity:** HIGH (maintainability, testability, performance)
**Evidence:** Functions >100 lines, deep nesting, heavy allocations

**Problems:**
- `ai.cpp`: Many large functions with parallel_for usage
- `economy.cpp`: Per-province loops, heavy allocations
- Hard to test, debug, and optimize

**Fix Strategy:**
1. Profile to identify actual hotspots
2. Extract helper functions (small, focused)
3. Consider data-oriented refactoring for hot loops
4. Add unit tests for extracted logic

**Files to Modify:**
- `src/ai/ai.cpp`
- `src/economy/economy.cpp`

---

### 5. JSON Web API Performance (MEDIUM-HIGH)

**File:** `src/network/webapi/jsonlayer.cpp`
**Severity:** MEDIUM-HIGH (UI latency)
**Evidence:** Functions iterate entire world per call

**Problem:**
- `jsonlayer.cpp` formats JSON by traversing whole world
- Repeated allocations and iterations
- No caching or incremental updates

**Fix Strategy:**
1. Profile to measure impact
2. Add caching for expensive computations
3. Consider incremental updates (only changed data)
4. Batch operations where possible

**Files to Modify:**
- `src/network/webapi/jsonlayer.cpp`

---

### 6. UI Cached Vector Locking (MEDIUM)

**File:** `src/gamestate/system_state.hpp`
**Severity:** MEDIUM (contention, performance)
**Evidence:** Coarse-grained lock/unlock in `ui_cached_vector`

**Problem:**
- Manual lock()/unlock() without exception safety
- `try_lock()` returns optional (okay but contention-prone)
- No RW-lock for readers

**Fix Strategy:**
1. Replace with `std::scoped_lock` or `std::unique_lock`
2. Consider snapshot pattern: atomic shared_ptr to vector
3. Add benchmark to measure improvement

**Files to Modify:**
- `src/gamestate/system_state.hpp`

---

### 7. Network Platform-Specific Code (MEDIUM)

**File:** `src/network/network.cpp`
**Severity:** MEDIUM (platform divergence, bugs)
**Evidence:** Large #ifdef sections for Winsock vs BSD

**Problem:**
- Platform-specific behavior can cause multiplayer mismatches
- Hard to test cross-platform
- UPnP/NAT logic may have edge cases

**Fix Strategy:**
1. Abstract platform-specific code behind interface
2. Add unit tests with mocks
3. Add logging for platform differences
4. Ensure graceful fallbacks

**Files to Modify:**
- `src/network/network.cpp`

---

### 8. Serialization Ordering & Unordered Containers (MEDIUM)

**Files:** `src/gamestate/system_state.hpp`, various serialization paths
**Severity:** MEDIUM (determinism risk)
**Evidence:** `ankerl::unordered_dense::map/set` used in serialization

**Problem:**
- Unordered containers can produce non-deterministic iteration order
- This affects checksums and multiplayer determinism
- Already uses contiguous dcon storage (good), but some maps remain

**Fix Strategy:**
1. Audit all serialization paths for unordered containers
2. Add stable-iteration wrappers when serializing
3. Add tests that detect ordering changes
4. Consider sorting keys before serialization

**Files to Modify:**
- `src/gamestate/system_state.hpp`
- Serialization code paths

---

## Medium Priority Issues (Fix Later)

### 9. Magic Numbers & Repeated Allocations (MEDIUM)

**Evidence:** Inline magic numbers, repeated vector allocations in hot loops

**Fix Strategy:**
1. Extract constants to named variables
2. Pre-allocate buffers where possible
3. Use reserve() on vectors in loops

---

### 10. Missing Test Coverage (MEDIUM-HIGH)

**Evidence:** Good parser tests exist; missing serialization, AI, network tests

**Fix Strategy:**
1. Add serialization tests (round-trip, error paths)
2. Add AI determinism tests
3. Add network endpoint tests
4. Add fuzz tests for parsers

---

### 11. Vendored Code Management (MEDIUM)

**Files:** `src/zstd/`, `dependencies/*`
**Severity:** MEDIUM (security, maintenance)
**Evidence:** Vendored zstd, prebuilt libs in `/libs`

**Fix Strategy:**
1. Document vendor upgrade procedure
2. Track upstream CVEs for zstd
3. Consider using system libs where possible

---

### 12. Build System Complexity (MEDIUM)

**Files:** `CMakeLists.txt`, `CMakePresets.json`
**Severity:** MEDIUM (developer experience)
**Evidence:** Complex flag handling, generator dependencies

**Fix Strategy:**
1. Simplify CMake logic where possible
2. Improve preset documentation
3. Add better error messages for generator failures

---

## Implementation Plan

### Phase 1: Critical Fixes (Week 1)

**Day 1-2: Serialization Safety**
- [ ] Modify `src/gamestate/serialization.cpp`
- [ ] Add bounds checks and ZSTD error handling
- [ ] Create `tests/serialization_tests.cpp`
- [ ] Run tests and verify no regressions

**Day 3: CI Security**
- [ ] Modify `.github/workflows/opencode.yml`
- [ ] Add maintainer gating or disable secret usage
- [ ] Pin actions to specific SHAs
- [ ] Test workflow changes

**Day 4-5: CI Test Execution**
- [ ] Modify `.github/workflows/build.yml`
- [ ] Add ctest execution
- [ ] Add sanitizer job (ASAN/UBSAN)
- [ ] Add artifact upload and caching

### Phase 2: High Priority (Week 2-3)

**Week 2: Performance & Complexity**
- [ ] Profile AI and economy code
- [ ] Extract helper functions from `ai.cpp` and `economy.cpp`
- [ ] Add unit tests for extracted logic
- [ ] Optimize JSON web API

**Week 3: Determinism & Testing**
- [ ] Add cross-platform determinism CI
- [ ] Add serialization fuzz tests
- [ ] Add AI determinism tests
- [ ] Add network endpoint tests

### Phase 3: Medium Priority (Week 4+)

**Week 4: Code Quality**
- [ ] Fix UI cached vector locking
- [ ] Abstract network platform code
- [ ] Add stable-iteration wrappers for unordered containers
- [ ] Extract magic numbers and optimize allocations

**Week 5: Build & Dependencies**
- [ ] Simplify CMake logic
- [ ] Document vendor upgrade procedure
- [ ] Add better error messages for generators
- [ ] Update CMakePresets documentation

---

## Success Criteria

### For Each Fix:
1. **Functional:** Code compiles and runs without crashes
2. **Observable:** Tests pass, CI green, no performance regressions
3. **Pass/Fail:** Binary criteria (tests pass/fail, CI succeeds/fails)

### For Critical Fixes:
1. Serialization: All error paths tested, no crashes on malformed input
2. CI Security: Secret not exposed to untrusted triggers, actions pinned
3. Determinism: Cross-platform tests pass, OOS logs generated on failure

---

## Evidence Requirements

### For Each Fix:
1. **Build:** `cmake --build` exit code 0, no errors
2. **Test:** `ctest` output showing all tests pass
3. **Manual Verify:** Demonstrate fix works (describe what observed)
4. **Regression:** Existing tests still pass

### For CI Changes:
1. Workflow run logs showing test execution
2. Artifact upload verification
3. Sanitizer run logs (if applicable)

---

## Files to Create/Modify

### New Files:
- `tests/serialization_tests.cpp`
- `tests/serialization_fuzz_tests.cpp`
- `tests/ai_determinism_tests.cpp`
- `tests/network_endpoint_tests.cpp`

### Modified Files:
- `src/gamestate/serialization.cpp`
- `src/ai/ai.cpp`
- `src/economy/economy.cpp`
- `src/network/webapi/jsonlayer.cpp`
- `src/gamestate/system_state.hpp`
- `.github/workflows/opencode.yml`
- `.github/workflows/build.yml`
- `CMakeLists.txt` (for test additions)
- `CMakePresets.json` (for sanitizer flags)

---

## Next Steps

1. **Immediate:** Start with serialization safety fix (Day 1)
2. **Parallel:** Begin CI security fix while serialization tests are written
3. **Follow-up:** Add test execution to CI and verify all tests pass
4. **Long-term:** Follow phased implementation plan

---

## References

- DeepWiki: https://deepwiki.com/schombert/Project-Alice
- Repository: C:\Users\Phili\Project-Alice
- Commit: a90dd964a
- Branch: main

---

**End of Analysis**
