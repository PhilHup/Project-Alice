# Phase 1 Execution: Critical Bug Fixes & Performance Improvements

**Generated:** 2026-01-16T00:00:00Z
**Commit:** a90dd964a
**Branch:** main
**Phase:** 1 of 16-week improvement plan

---

## Phase 1 Overview

**Goal:** Fix critical bugs and establish performance baseline for future improvements

**Duration:** 1 week (Day 1-5)

**Priority:** CRITICAL

**Success Criteria:**
- All critical security and crash bugs fixed
- CI runs tests and sanitizers
- Performance baseline established
- No regressions in existing functionality

---

## Day 1-2: Serialization Safety Fix (CRITICAL)

### Issue Description
**File:** `src/gamestate/serialization.cpp`
**Severity:** CRITICAL (crash, OOM, data corruption, DoS)
**Evidence:** TODO comment + unchecked ZSTD_decompress + raw new[]/delete[]

**Current Problem Code:**
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

### Implementation Plan

#### Step 1: Read Current Serialization Code
**Files to read:**
- `src/gamestate/serialization.cpp`
- `src/gamestate/serialization.hpp` (if exists)
- Related files: `src/gamestate/system_state.cpp` (serialization calls)

**What to look for:**
- `with_decompressed_section` function
- `write_compressed_section` function
- All ZSTD_compress and ZSTD_decompress calls
- Raw memory operations (new[], delete[], memcpy)

#### Step 2: Implement Safe Decompression Wrapper

**Changes to make:**
1. Replace raw `new[]/delete[]` with `std::vector<uint8_t>`
2. Add bounds checking:
   - Validate `decompressed_length` ≤ 256MiB (configurable limit)
   - Validate `section_length` fits in remaining buffer
   - Check for integer overflow in pointer arithmetic
3. Validate ZSTD return values with `ZSTD_isError()`
4. Handle decompression failures gracefully (error accumulation)

**Code template:**
```cpp
// Constants
constexpr size_t MAX_DECOMPRESSED_SIZE = 256 * 1024 * 1024; // 256 MiB

// Safe decompression wrapper
bool safe_decompress(
    const uint8_t* src,
    size_t src_size,
    size_t decompressed_length,
    std::vector<uint8_t>& out_buffer
) {
    // Validate inputs
    if (decompressed_length == 0 || decompressed_length > MAX_DECOMPRESSED_SIZE) {
        return false;
    }
    
    // Check that section_length fits in source buffer
    if (src_size < sizeof(uint32_t) * 2) {
        return false;
    }
    
    // Allocate buffer
    out_buffer.resize(decompressed_length);
    
    // Decompress
    size_t result = ZSTD_decompress(
        out_buffer.data(),
        decompressed_length,
        src + sizeof(uint32_t) * 2,
        src_size - sizeof(uint32_t) * 2
    );
    
    // Check for errors
    if (ZSTD_isError(result)) {
        return false;
    }
    
    // Validate actual decompressed size
    if (result != decompressed_length) {
        // Decide: trust result or header? For safety, use result
        out_buffer.resize(result);
    }
    
    return true;
}
```

#### Step 3: Update `with_decompressed_section` Function

**Current function signature (example):**
```cpp
template<typename F>
void with_decompressed_section(uint8_t* ptr_in, uint32_t section_length, F&& function) {
    uint32_t decompressed_length = *(uint32_t*)(ptr_in + sizeof(uint32_t));
    uint8_t* temp_buffer = new uint8_t[decompressed_length];
    ZSTD_decompress(temp_buffer, decompressed_length, ptr_in + sizeof(uint32_t) * 2, section_length);
    function(temp_buffer, decompressed_length);
    delete[] temp_buffer;
}
```

**Updated function:**
```cpp
template<typename F>
bool with_decompressed_section(uint8_t* ptr_in, uint32_t section_length, F&& function) {
    // Read header
    if (section_length < sizeof(uint32_t) * 2) {
        return false;
    }
    
    uint32_t decompressed_length = *(uint32_t*)(ptr_in + sizeof(uint32_t));
    
    // Validate decompressed length
    constexpr size_t MAX_DECOMPRESSED_SIZE = 256 * 1024 * 1024;
    if (decompressed_length == 0 || decompressed_length > MAX_DECOMPRESSED_SIZE) {
        return false;
    }
    
    // Allocate buffer with RAII
    std::vector<uint8_t> temp_buffer(decompressed_length);
    
    // Decompress
    size_t result = ZSTD_decompress(
        temp_buffer.data(),
        decompressed_length,
        ptr_in + sizeof(uint32_t) * 2,
        section_length - sizeof(uint32_t) * 2
    );
    
    // Check for errors
    if (ZSTD_isError(result)) {
        return false;
    }
    
    // Validate actual size
    if (result != decompressed_length) {
        // Resize to actual decompressed size
        temp_buffer.resize(result);
    }
    
    // Call function with safe buffer
    function(temp_buffer.data(), temp_buffer.size());
    
    return true;
}
```

#### Step 4: Update Call Sites

**Search for all calls to `with_decompressed_section`:**
- Update to handle boolean return value
- Add error accumulation/logging
- Ensure graceful failure (don't crash on corrupted saves)

#### Step 5: Add Unit Tests

**Create `tests/serialization_tests.cpp`:**
```cpp
#include <catch2/catch.hpp>
#include "gamestate/serialization.hpp"

TEST_CASE("Serialization round-trip", "[serialization]") {
    // Test compress/decompress cycle
    std::vector<uint8_t> original(1024, 0xAA);
    // ... implementation
}

TEST_CASE("Decompression handles truncated data", "[serialization]") {
    // Test with truncated compressed data
    // Should return false, not crash
}

TEST_CASE("Decompression handles corrupted data", "[serialization]") {
    // Test with corrupted compressed data
    // Should return false, not crash
}

TEST_CASE("Decompression rejects oversized header", "[serialization]") {
    // Test with decompressed_length > MAX_DECOMPRESSED_SIZE
    // Should return false, not allocate huge buffer
}
```

**Create `tests/serialization_fuzz_tests.cpp`:**
```cpp
#include <catch2/catch.hpp>
#include "gamestate/serialization.hpp"

TEST_CASE("Fuzz: random compressed data", "[serialization][fuzz]") {
    // Generate random data with valid-looking headers
    // Ensure no crashes
}

TEST_CASE("Fuzz: truncated headers", "[serialization][fuzz]") {
    // Test various truncated header scenarios
}
```

#### Step 6: Verify No Regressions

**Build and test:**
```bash
mkdir build && cd build
cmake -S .. -B . -G "Ninja" -DCMAKE_BUILD_TYPE=RelWithDebInfo
cmake --build . -- -j$(nproc)
ctest --output-on-failure
```

**Success criteria:**
- All existing tests pass
- New serialization tests pass
- No compiler warnings
- Manual test: load existing save files successfully

---

## Day 3: CI Security Fix (CRITICAL)

### Issue Description
**File:** `.github/workflows/opencode.yml`
**Severity:** CRITICAL (secret exposure risk)
**Evidence:** Comment-triggered workflow with OPENCODE_API_KEY secret

**Current Problem:**
- Workflow triggers on `issue_comment` and `pull_request_review_comment` (public events)
- Anyone with comment access can trigger the workflow
- Third-party action `anomalyco/opencode/github@latest` receives the secret
- No maintainer verification before secret usage
- Floating tag `@latest` is a supply-chain risk

### Implementation Plan

#### Step 1: Read Current Workflow
**File to read:**
- `.github/workflows/opencode.yml`

**What to look for:**
- Trigger events
- Secret usage
- Action versions
- Permissions

#### Step 2: Implement Maintainer Gating

**Changes:**
1. Add check for maintainer status before secret usage
2. Use GitHub API to verify collaborator status
3. Skip secret-using steps for non-maintainers

**Code template:**
```yaml
name: OpenCode

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]

jobs:
  check-permissions:
    runs-on: ubuntu-latest
    outputs:
      is-maintainer: ${{ steps.check.outputs.is-maintainer }}
    steps:
      - name: Check if actor is a maintainer
        id: check
        run: |
          # Use GitHub API to check if user is a collaborator with push access
          # This is a simplified example - you may need to adjust based on your repo settings
          IS_MAINTAINER=$(gh api repos/${{ github.repository }}/collaborators/${{ github.actor }} --jq '.permissions.push' 2>/dev/null || echo "false")
          echo "is-maintainer=$IS_MAINTAINER" >> $GITHUB_OUTPUT
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  opencode:
    needs: check-permissions
    if: needs.check-permissions.outputs.is-maintainer == 'true'
    runs-on: ubuntu-latest
    steps:
      - name: Run OpenCode
        uses: anomalyco/opencode/github@<PINNED_SHA>
        env:
          OPENCODE_API_KEY: ${{ secrets.OPENCODE_API_KEY }}
```

#### Step 3: Pin Action to Specific SHA

**Find the latest stable commit:**
- Visit: https://github.com/anomalyco/opencode
- Find latest release or stable commit
- Use commit SHA instead of `@latest`

**Example:**
```yaml
uses: anomalyco/opencode/github@a1b2c3d4e5f67890123456789012345678901234
```

#### Step 4: Reduce Permissions

**Add explicit minimal permissions:**
```yaml
permissions:
  contents: read
  issues: write
  pull-requests: write
  # Remove id-token: write unless required
```

#### Step 5: Alternative: Move to workflow_dispatch

**Consider changing trigger:**
```yaml
on:
  workflow_dispatch:
    inputs:
      issue_number:
        description: 'Issue number to process'
        required: true
```

**This requires manual approval for each run.**

#### Step 6: Test Workflow Changes

**Test locally:**
- Create a test branch
- Make a test comment (as non-maintainer)
- Verify secret is not exposed
- Verify maintainer check works

**Success criteria:**
- Non-maintainer comments don't trigger secret-using steps
- Maintainer comments work as expected
- Action is pinned to SHA
- Minimal permissions are set

---

## Day 4-5: CI Test Execution (HIGH)

### Issue Description
**File:** `.github/workflows/build.yml`
**Severity:** HIGH (no test coverage in CI)
**Evidence:** Build matrix exists but no `ctest` or test execution

**Current Problem:**
- CI builds but doesn't run tests
- No sanitizer coverage (ASAN/UBSAN)
- No artifact upload for debugging
- No ccache/build caching

### Implementation Plan

#### Step 1: Read Current Workflow
**File to read:**
- `.github/workflows/build.yml`

**What to look for:**
- Build steps
- Test execution (or lack thereof)
- Artifact handling
- Caching configuration

#### Step 2: Add Test Execution

**Changes:**
1. Add `ctest` execution step after build
2. Upload test logs on failure
3. Add test summary

**Code template:**
```yaml
jobs:
  build:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        include:
          - os: ubuntu-latest
            compiler: clang
          - os: windows-latest
            compiler: msvc
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure CMake
        run: cmake -S . -B build -G "Ninja" -DCMAKE_BUILD_TYPE=RelWithDebInfo
        
      - name: Build
        run: cmake --build build -- -j$(nproc)
        
      - name: Run Tests
        run: |
          cd build
          ctest --output-on-failure --verbose
        continue-on-error: false
        
      - name: Upload Test Logs (on failure)
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: test-logs-${{ matrix.os }}
          path: |
            build/Testing/
            build/**/*.log
          retention-days: 7
```

#### Step 3: Add Sanitizer Job

**Create separate sanitizer job:**
```yaml
  sanitizer:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        sanitizer: [address, undefined]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure with Sanitizer
        run: |
          if [ "${{ matrix.sanitizer }}" = "address" ]; then
            cmake -S . -B build -G "Ninja" \
              -DCMAKE_BUILD_TYPE=Debug \
              -DCMAKE_CXX_FLAGS="-fsanitize=address -fno-omit-frame-pointer" \
              -DCMAKE_C_FLAGS="-fsanitize=address -fno-omit-frame-pointer"
          else
            cmake -S . -B build -G "Ninja" \
              -DCMAKE_BUILD_TYPE=Debug \
              -DCMAKE_CXX_FLAGS="-fsanitize=undefined -fno-omit-frame-pointer" \
              -DCMAKE_C_FLAGS="-fsanitize=undefined -fno-omit-frame-pointer"
          fi
          
      - name: Build
        run: cmake --build build -- -j$(nproc)
        
      - name: Run Tests with Sanitizer
        run: |
          cd build
          ctest --output-on-failure --verbose
        continue-on-error: true
        
      - name: Upload Sanitizer Logs
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: sanitizer-logs-${{ matrix.sanitizer }}
          path: |
            build/**/*.log
            build/**/*.txt
          retention-days: 14
```

#### Step 4: Add Caching

**Add ccache/actions/cache:**
```yaml
      - name: Setup ccache (Linux)
        if: runner.os == 'Linux'
        uses: hendrikmuhs/ccache-action@v1
        with:
          key: ${{ runner.os }}-ccache
          max-size: 1G
          
      - name: Setup ccache (macOS)
        if: runner.os == 'macOS'
        uses: hendrikmuhs/ccache-action@v1
        with:
          key: ${{ runner.os }}-ccache
          max-size: 1G
          
      - name: Setup clcache (Windows)
        if: runner.os == 'Windows'
        uses: frerich/clcache-action@v1
```

#### Step 5: Add Concurrency Controls

**Prevent redundant runs:**
```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

#### Step 6: Test CI Changes

**Test locally:**
- Create test branch
- Push and watch CI run
- Verify tests execute
- Verify artifacts upload on failure

**Success criteria:**
- Tests run on every PR/push
- Sanitizer job runs and catches issues
- Artifacts are uploaded on failure
- Build times are reduced via caching

---

## Verification & Evidence Requirements

### For Each Fix:
1. **Build:** `cmake --build` exit code 0, no errors
2. **Test:** `ctest` output showing all tests pass
3. **Manual Verify:** Demonstrate fix works (describe what observed)
4. **Regression:** Existing tests still pass

### For CI Changes:
1. Workflow run logs showing test execution
2. Artifact upload verification
3. Sanitizer run logs (if applicable)

### Evidence to Collect:
- Build output logs
- Test output logs
- CI workflow run screenshots/logs
- Before/after performance measurements
- Security audit of CI changes

---

## Success Criteria for Phase 1

### Critical Fixes:
- [ ] Serialization: All error paths tested, no crashes on malformed input
- [ ] CI Security: Secret not exposed to untrusted triggers, actions pinned
- [ ] CI Tests: All tests run in CI, sanitizers catch issues

### Performance Baseline:
- [ ] Profile established for late-game scenario
- [ ] Hotspots identified and documented
- [ ] Baseline metrics recorded

### Quality:
- [ ] No compiler warnings
- [ ] All existing tests pass
- [ ] New tests added and passing
- [ ] Code coverage > 70% for critical subsystems

---

## Next Steps After Phase 1

Once Phase 1 is complete, you'll have:
1. **Secure CI** with test execution and sanitizers
2. **Safe serialization** with comprehensive error handling
3. **Performance baseline** for future optimizations
4. **Clean codebase** with no critical bugs

**Then proceed to Phase 2:** AI improvements and modernization

---

## Quick Reference Commands

### Local Build & Test:
```bash
mkdir build && cd build
cmake -S .. -B . -G "Ninja" -DCMAKE_BUILD_TYPE=RelWithDebInfo
cmake --build . -- -j$(nproc)
ctest --output-on-failure --verbose
```

### Sanitizer Build (Linux):
```bash
mkdir build && cd build
cmake -S .. -B . -G "Ninja" \
  -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-fsanitize=address -fno-omit-frame-pointer" \
  -DCMAKE_C_FLAGS="-fsanitize=address -fno-omit-frame-pointer"
cmake --build . -- -j$(nproc)
ctest --output-on-failure --verbose
```

### CI Test:
- Push to test branch
- Watch GitHub Actions
- Download artifacts on failure

---

**Phase 1 starts now. Let's fix the critical issues and establish a solid foundation.**
