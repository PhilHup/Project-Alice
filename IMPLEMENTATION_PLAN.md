# Project-Alice Fork: Implementation Plan

**Generated:** 2026-01-15T23:00:00Z
**Commit:** a90dd964a
**Branch:** main

---

## Overview

This plan prioritizes fixes based on severity, impact, and effort. We'll start with **critical** issues that can cause crashes or security problems, then move to **high** priority issues affecting determinism and CI, followed by **medium** priority improvements.

---

## Phase 1: Critical Fixes (Week 1)

### Day 1-2: Serialization Safety (CRITICAL)

**Goal:** Eliminate unchecked ZSTD decompression and raw memory operations in save/load.

**Files to Modify:**
- `src/gamestate/serialization.cpp`

**Changes:**
1. Replace raw `new[]/delete[]` with `std::vector<uint8_t>`
2. Add bounds checking:
   - Validate `decompressed_length` ≤ 256MiB (configurable limit)
   - Validate `section_length` fits in remaining buffer
   - Check for integer overflow in pointer arithmetic
3. Validate ZSTD return values with `ZSTD_isError()`
4. Handle decompression failures gracefully (error accumulation)

**Test Files to Create:**
- `tests/serialization_tests.cpp` (round-trip, error paths)
- `tests/serialization_fuzz_tests.cpp` (corrupted/truncated inputs)

**Success Criteria:**
- [ ] Code compiles without warnings
- [ ] All existing tests pass
- [ ] New tests cover error paths
- [ ] No crashes on malformed input
- [ ] CI builds successfully

**Evidence Required:**
- Build output (exit code 0)
- Test output (all tests pass)
- Manual verification: load a save file successfully
- Regression: existing save/load still works

---

### Day 3: CI Security (CRITICAL)

**Goal:** Secure opencode workflow to prevent secret exposure.

**Files to Modify:**
- `.github/workflows/opencode.yml`

**Changes:**
1. Add maintainer-only gating:
   - Check `github.actor` is in trusted list before secret usage
   - Use GitHub API to verify collaborator status
2. Pin action to specific commit SHA (not `@latest`)
3. Remove unnecessary permissions (`id-token: write` unless required)
4. Add explicit minimal permissions block
5. Consider moving to `workflow_dispatch` for manual approval

**Success Criteria:**
- [ ] Workflow compiles without errors
- [ ] Non-maintainer comments don't trigger secret-using steps
- [ ] Action is pinned to SHA
- [ ] Minimal permissions are set

**Evidence Required:**
- Workflow run logs showing maintainer check works
- Verification that secret is not exposed to untrusted triggers

---

### Day 4-5: CI Test Execution (HIGH)

**Goal:** Ensure tests run in CI and catch regressions.

**Files to Modify:**
- `.github/workflows/build.yml`

**Changes:**
1. Add `ctest` execution step
2. Add sanitizer job (ASAN + UBSAN on Linux)
3. Add artifact upload for logs/binaries
4. Add ccache/actions/cache for build speed
5. Add concurrency controls to prevent redundant runs

**Success Criteria:**
- [ ] CI runs tests on every PR/push
- [ ] Sanitizer job runs and catches issues
- [ ] Artifacts are uploaded on failure
- [ ] Build times are reduced via caching

**Evidence Required:**
- CI run logs showing test execution
- Sanitizer run logs (if any issues found)
- Artifact download verification

---

## Phase 2: High Priority (Week 2-3)

### Week 2: Performance & Complexity

**Goal:** Improve maintainability and performance of AI and economy code.

**Files to Modify:**
- `src/ai/ai.cpp`
- `src/economy/economy.cpp`

**Changes:**
1. Profile to identify actual hotspots (use `perf` or similar)
2. Extract helper functions (small, focused, testable)
3. Add unit tests for extracted logic
4. Optimize hot loops (data-oriented where possible)

**Success Criteria:**
- [ ] Code complexity reduced (functions < 100 lines)
- [ ] Performance not degraded (profile before/after)
- [ ] New tests pass
- [ ] Existing tests still pass

**Evidence Required:**
- Profile output showing no regression
- Test output (all tests pass)
- Code review showing improved readability

---

### Week 3: Determinism & Testing

**Goal:** Add cross-platform determinism testing and fuzz tests.

**Files to Modify:**
- `.github/workflows/build.yml` (add determinism job)
- `tests/` (add new test files)

**Changes:**
1. Add determinism CI job (Linux + Windows matrix)
2. Add serialization fuzz tests
3. Add AI determinism tests
4. Add network endpoint tests

**Success Criteria:**
- [ ] Determinism tests pass on both platforms
- [ ] Fuzz tests catch corrupted input
- [ ] Network tests cover key endpoints
- [ ] OOS logs generated on failure

**Evidence Required:**
- CI run logs showing determinism tests pass
- Fuzz test corpus and crash reports (if any)
- Network test coverage report

---

## Phase 3: Medium Priority (Week 4+)

### Week 4: Code Quality

**Goal:** Improve locking, platform abstraction, and serialization stability.

**Files to Modify:**
- `src/gamestate/system_state.hpp`
- `src/network/network.cpp`
- Serialization code paths

**Changes:**
1. Fix UI cached vector locking (use `std::scoped_lock`)
2. Abstract network platform code behind interface
3. Add stable-iteration wrappers for unordered containers
4. Extract magic numbers and optimize allocations

**Success Criteria:**
- [ ] Locking is exception-safe
- [ ] Network code compiles on both platforms
- [ ] Serialization order is stable
- [ ] Magic numbers replaced with named constants

**Evidence Required:**
- Build output (no warnings)
- Test output (all tests pass)
- Code review showing improvements

---

### Week 5: Build & Dependencies

**Goal:** Simplify build system and document vendor management.

**Files to Modify:**
- `CMakeLists.txt`
- `CMakePresets.json`
- `docs/` (add vendor upgrade guide)

**Changes:**
1. Simplify CMake logic where possible
2. Document vendor upgrade procedure
3. Add better error messages for generator failures
4. Update CMakePresets documentation

**Success Criteria:**
- [ ] CMake configuration is simpler
- [ ] Vendor upgrade guide exists
- [ ] Generator failures have clear error messages
- [ ] Presets are well-documented

**Evidence Required:**
- Build output (no errors)
- Documentation review
- Generator failure test (simulate and verify error message)

---

## Implementation Order

### Critical (Start Immediately)
1. Serialization safety (Day 1-2)
2. CI security (Day 3)
3. CI test execution (Day 4-5)

### High Priority (Week 2-3)
4. AI/economy complexity reduction (Week 2)
5. Determinism testing (Week 3)

### Medium Priority (Week 4+)
6. Code quality improvements (Week 4)
7. Build system simplification (Week 5)

---

## Success Criteria by Phase

### Phase 1 (Critical)
- No crashes on malformed input
- CI secrets are secure
- Tests run in CI
- All existing functionality preserved

### Phase 2 (High)
- Code complexity reduced by 30%
- Determinism tests pass on both platforms
- Performance not degraded
- New tests cover critical paths

### Phase 3 (Medium)
- Code quality improved (readability, maintainability)
- Build system is simpler and more reliable
- Vendor management is documented

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

## Files to Create

### New Test Files:
- `tests/serialization_tests.cpp`
- `tests/serialization_fuzz_tests.cpp`
- `tests/ai_determinism_tests.cpp`
- `tests/network_endpoint_tests.cpp`

### New Documentation:
- `docs/VENDOR_UPGRADE_GUIDE.md`
- `docs/DETERMINISM_GUIDE.md`

---

## Risk Mitigation

### Serialization Changes:
- Keep on-disk format unchanged
- Add version checks if needed
- Test with existing save files

### CI Changes:
- Test workflow changes in a branch
- Verify secrets are not exposed
- Ensure backward compatibility

### Code Refactoring:
- Profile before and after
- Add tests for extracted functions
- Keep existing behavior

---

## Next Steps

1. **Immediate:** Start serialization safety fix (Day 1)
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

**End of Implementation Plan**
