# Phase 2 Execution: AI Improvements & Modernization

**Generated:** 2026-01-16T00:00:00Z
**Commit:** a90dd964a
**Branch:** main
**Phase:** 2 of 16-week improvement plan

---

## Phase 2 Overview

**Goal:** Modernize AI with utility-based decisions, personality systems, and better coordination

**Duration:** 3 weeks (Weeks 2-4)

**Priority:** HIGH

**Success Criteria:**
- AI decisions are more varied and less predictable
- AI performance improved (better win rates in automated tournaments)
- Code is more maintainable and testable
- Performance improvements measurable (25%+ reduction in AI decision time)

---

## Week 2: Performance Profiling & Optimization

### Objective
Identify and fix performance bottlenecks before AI refactor

### Step 1: Profile Late-Game Scenario

#### Build with Profiling Flags
```bash
mkdir build && cd build
cmake -S .. -B . -G "Ninja" \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DCMAKE_CXX_FLAGS="-g -O2" \
  -DCMAKE_C_FLAGS="-g -O2"
cmake --build . -- -j$(nproc)
```

#### Run Representative Scenario
```bash
# Run with a late-game save (create or use existing)
./build/src/AliceExecutable --scenario late_game_save.sav --headless --ticks 1000
```

#### Capture CPU Profiles

**Windows:**
- Use Visual Studio Profiler
- Or Windows Performance Recorder (WPR)
- Or Intel VTune

**Linux:**
```bash
# Record with perf
perf record -g -- ./build/src/AliceExecutable --scenario late_game_save.sav --headless --ticks 1000

# Generate flamegraph
perf script | flamegraph.pl > profile.svg
```

#### Analyze Hotspots
**Expected hotspots:**
1. `src/ai/ai.cpp` - `take_ai_decisions()`
2. `src/ai/ai_war.cpp` - `prepare_list_of_targets_for_cb()`
3. `src/economy/economy.cpp` - `update_ai_econ_construction()`
4. `src/gamestate/system_state.cpp` - `ui_cache::process_update()`
5. `src/map/map.cpp` - `update_borders_mesh()`

### Step 2: Quick Wins (Low-Risk Optimizations)

#### 1. Serialization Optimization
**File:** `src/gamestate/serialization.cpp`
**Change:** Replace per-decompression allocations with thread-local buffers

**Implementation:**
```cpp
// Add thread-local buffer
thread_local std::vector<uint8_t> decompression_buffer;

bool with_decompressed_section(uint8_t* ptr_in, uint32_t section_length, F&& function) {
    // ... validation ...
    
    // Reuse thread-local buffer
    decompression_buffer.resize(decompressed_length);
    
    size_t result = ZSTD_decompress(
        decompression_buffer.data(),
        decompressed_length,
        ptr_in + sizeof(uint32_t) * 2,
        section_length - sizeof(uint32_t) * 2
    );
    
    // ... error handling ...
    
    function(decompression_buffer.data(), decompression_buffer.size());
    
    return true;
}
```

#### 2. UI Cache Optimization
**File:** `src/gamestate/system_state.cpp`
**Change:** Replace polling sleep with condition variables

**Implementation:**
```cpp
// Add condition variable
std::condition_variable ui_cache_cv;
std::mutex ui_cache_mutex;
bool ui_cache_update_requested = false;

// In ui_cache::process_update:
void process_update() {
    while (!should_exit) {
        std::unique_lock<std::mutex> lock(ui_cache_mutex);
        ui_cache_cv.wait(lock, [this] { 
            return ui_cache_update_requested || should_exit; 
        });
        
        if (ui_cache_update_requested) {
            // Perform updates
            update_slots();
            ui_cache_update_requested = false;
        }
    }
}

// When update is needed:
void request_ui_update() {
    {
        std::lock_guard<std::mutex> lock(ui_cache_mutex);
        ui_cache_update_requested = true;
    }
    ui_cache_cv.notify_one();
}
```

#### 3. Map VBO Optimization
**File:** `src/map/map.cpp`
**Change:** Use persistent mapped buffers instead of glBufferData

**Implementation:**
```cpp
// Check for GL_ARB_buffer_storage support
if (has_buffer_storage) {
    // Use persistent mapping
    glBufferStorage(GL_ARRAY_BUFFER, size, nullptr, 
        GL_MAP_WRITE_BIT | GL_MAP_PERSISTENT_BIT | GL_MAP_COHERENT_BIT);
    
    // Map once
    void* mapped = glMapBufferRange(GL_ARRAY_BUFFER, 0, size,
        GL_MAP_WRITE_BIT | GL_MAP_PERSISTENT_BIT | GL_MAP_COHERENT_BIT);
    
    // Update only changed regions
    // ... update logic ...
} else {
    // Fallback to glBufferData
    // ... existing code ...
}
```

#### 4. Province Distance Caching
**File:** `src/ai/ai.cpp`
**Change:** Cache province distances, invalidate only on map changes

**Implementation:**
```cpp
// Add distance cache to system_state
struct distance_cache {
    std::unordered_map<dcon::province_id, float> distances;
    uint64_t last_update_tick = 0;
};

// In ai.cpp, use cached distances:
float get_cached_distance(sys::state& state, dcon::province_id a, dcon::province_id b) {
    auto& cache = state.distance_cache;
    
    // Check if cache is valid for current tick
    if (cache.last_update_tick == state.current_date.value) {
        auto key = std::make_pair(a, b);
        auto it = cache.distances.find(key);
        if (it != cache.distances.end()) {
            return it->second;
        }
    }
    
    // Compute and cache
    float dist = province::sorting_distance(state, a, b);
    cache.distances[std::make_pair(a, b)] = dist;
    return dist;
}

// Invalidate cache when map changes
void invalidate_distance_cache(sys::state& state) {
    state.distance_cache.distances.clear();
    state.distance_cache.last_update_tick = 0;
}
```

### Step 3: Measure Improvements

#### Before/After Comparison
```bash
# Run benchmark before changes
./build/src/AliceExecutable --scenario late_game_save.sav --headless --ticks 1000 --profile

# Apply optimizations
# ... code changes ...

# Run benchmark after changes
./build/src/AliceExecutable --scenario late_game_save.sav --headless --ticks 1000 --profile
```

#### Success Criteria
- Frame time reduced by 20%+ in late-game scenarios
- Memory allocations reduced in hot paths
- Profile shows measurable improvement in hotspots

---

## Week 3: AI Architecture Refactor

### Objective
Modernize AI with utility-based decisions, personality systems, and better coordination

### Step 1: Extract Pure Decision Logic

#### Create Testable Functions
**File:** `src/ai/ai_types.hpp`
**Add:**
```cpp
namespace ai {

// Utility scoring framework
struct decision_context {
    dcon::nation_id nation;
    dcon::state_instance_id target;
    float strength_ratio;
    float economic_value;
    float political_risk;
    float personality_aggression;  // 0.0 = defensive, 1.0 = aggressive
    float personality_opportunism; // 0.0 = cautious, 1.0 = opportunistic
};

// Pure decision functions (no side effects)
float utility_of_war_goal(sys::state& state, const decision_context& ctx);
float utility_of_factory_build(sys::state& state, const decision_context& ctx);
float utility_of_research_tech(sys::state& state, const decision_context& ctx);

// Personality parameters
struct personality {
    float aggression;      // Willingness to declare war
    float opportunism;     // Likelihood to exploit weakness
    float caution;         // Risk tolerance
    float expansionism;    // Desire for territory
    float industrialism;   // Focus on economy vs military
};

// Decision outcome
struct decision_result {
    dcon::decision_id decision;
    float utility;
    bool execute;
};

} // namespace ai
```

#### Extract from `ai_war.cpp`
**Current code (example):**
```cpp
// In ai_war.cpp - utility_of_offensive_war
float utility_of_offensive_war(sys::state& state, dcon::nation_id nation, dcon::nation_id target) {
    // Complex logic mixed with side effects
    // ... 200 lines of code ...
}
```

**Refactored:**
```cpp
// In ai_war.cpp - pure utility calculation
float utility_of_war_goal(sys::state& state, const ai::decision_context& ctx) {
    // Extract pure logic into testable function
    float base_utility = 0.0f;
    
    // Economic factors
    base_utility += ctx.economic_value * 0.4f;
    
    // Military factors
    base_utility += (ctx.strength_ratio - 1.0f) * 0.3f;
    
    // Personality factors
    base_utility += ctx.personality_aggression * 0.2f;
    base_utility += ctx.personality_opportunism * 0.1f;
    
    // Risk factors (negative)
    base_utility -= ctx.political_risk * 0.1f;
    
    return std::max(0.0f, base_utility);
}

// In ai_war.cpp - decision execution (side effects)
void execute_war_decision(sys::state& state, dcon::nation_id nation, dcon::nation_id target) {
    // ... existing war declaration logic ...
}
```

### Step 2: Implement Personality System

#### Add Personality Data
**File:** `src/ai/ai_types.hpp`
**Add:**
```cpp
// Personality profiles (data-driven)
struct personality_profile {
    std::string name;
    personality params;
    
    // Factory preferences
    std::vector<dcon::factory_type_id> preferred_factories;
    
    // Tech priorities
    std::vector<dcon::technology_id> tech_priorities;
    
    // War behavior
    bool will_declare_wars;
    bool will_accept_peace;
    bool will_form_alliances;
};

// Load from data files
std::vector<personality_profile> load_personality_profiles(sys::state& state);
```

#### Create Personality Profiles (JSON)
**File:** `data/ai/personalities.json`
```json
{
  "personalities": [
    {
      "name": "aggressive_expansionist",
      "params": {
        "aggression": 0.8,
        "opportunism": 0.9,
        "caution": 0.2,
        "expansionism": 0.9,
        "industrialism": 0.4
      },
      "preferred_factories": ["arms_factory", "canned_food_factory"],
      "tech_priorities": ["military_tech_1", "military_tech_2"],
      "will_declare_wars": true,
      "will_accept_peace": false,
      "will_form_alliances": true
    },
    {
      "name": "defensive_industrialist",
      "params": {
        "aggression": 0.2,
        "opportunism": 0.3,
        "caution": 0.8,
        "expansionism": 0.3,
        "industrialism": 0.9
      },
      "preferred_factories": ["steel_factory", "machine_parts_factory"],
      "tech_priorities": ["industry_tech_1", "industry_tech_2"],
      "will_declare_wars": false,
      "will_accept_peace": true,
      "will_form_alliances": true
    }
  ]
}
```

#### Assign Personalities to Nations
**File:** `src/ai/ai_campaign.cpp`
**Add:**
```cpp
// Assign personality at game start
void initialize_ai_personalities(sys::state& state) {
    auto profiles = load_personality_profiles(state);
    
    for (auto nation : state.world.nation_range()) {
        if (nation.is_player_controlled()) {
            continue; // Skip human players
        }
        
        // Randomly assign personality (or based on nation type)
        uint32_t index = rng::get_random(state, nation.id.value) % profiles.size();
        state.ai_personalities[nation.id] = profiles[index];
    }
}
```

### Step 3: Add Blackboard for Coordination

#### Create Blackboard System
**File:** `src/ai/ai_blackboard.hpp`
**Add:**
```cpp
namespace ai {

// Shared knowledge between AI subsystems
struct blackboard {
    // Strategic level
    std::vector<dcon::nation_id> major_rivals;
    std::vector<dcon::nation_id> potential_allies;
    float global_threat_level;
    
    // Economic level
    std::vector<dcon::factory_type_id> needed_factory_types;
    std::vector<dcon::commodity_id> critical_shortages;
    
    // Military level
    std::vector<dcon::province_id> strategic_frontlines;
    std::vector<dcon::nation_id> active_enemies;
    
    // Decision history
    std::vector<decision_result> recent_decisions;
    
    // Update methods
    void update_strategic_assessment(sys::state& state);
    void update_economic_assessment(sys::state& state);
    void update_military_assessment(sys::state& state);
};

// Access pattern
blackboard& get_blackboard(sys::state& state);

} // namespace ai
```

#### Integrate with Existing AI
**File:** `src/ai/ai.cpp`
**Modify:**
```cpp
// In take_ai_decisions():
void take_ai_decisions(sys::state& state) {
    // Update blackboard first
    auto& board = ai::get_blackboard(state);
    board.update_strategic_assessment(state);
    board.update_economic_assessment(state);
    board.update_military_assessment(state);
    
    // Now make decisions using shared knowledge
    // ... existing decision logic ...
}
```

### Step 4: Implement Utility-Based Decisions

#### Replace Heuristics with Utility Functions
**File:** `src/ai/ai_war.cpp`
**Modify:**
```cpp
// Current code (simplified):
void decide_on_war(sys::state& state, dcon::nation_id nation) {
    // Simple heuristic: if strong enough, declare war
    float strength = estimate_strength(state, nation);
    if (strength > 1.5f) {
        // Declare war
    }
}

// Refactored:
void decide_on_war(sys::state& state, dcon::nation_id nation) {
    auto& board = ai::get_blackboard(state);
    auto& personality = state.ai_personalities[nation];
    
    // Evaluate multiple war goals
    std::vector<ai::decision_result> candidates;
    
    for (auto target : board.major_rivals) {
        ai::decision_context ctx;
        ctx.nation = nation;
        ctx.target = target;
        ctx.strength_ratio = estimate_strength_ratio(state, nation, target);
        ctx.economic_value = estimate_economic_value(state, target);
        ctx.political_risk = estimate_political_risk(state, nation, target);
        ctx.personality_aggression = personality.params.aggression;
        ctx.personality_opportunism = personality.params.opportunism;
        
        float utility = ai::utility_of_war_goal(state, ctx);
        
        if (utility > 0.5f && personality.will_declare_wars) {
            candidates.push_back({dcon::decision_id{}, utility, true});
        }
    }
    
    // Sort by utility and execute top decision
    std::sort(candidates.begin(), candidates.end(), 
        [](const auto& a, const auto& b) { return a.utility > b.utility; });
    
    if (!candidates.empty() && candidates[0].utility > 0.6f) {
        execute_war_decision(state, nation, candidates[0].decision);
    }
}
```

### Step 5: Add Data-Driven Weights

#### Move Constants to JSON
**File:** `data/ai/weights.json`
```json
{
  "war_utility": {
    "economic_value_weight": 0.4,
    "strength_ratio_weight": 0.3,
    "aggression_weight": 0.2,
    "opportunism_weight": 0.1,
    "risk_penalty_weight": 0.1
  },
  "factory_selection": {
    "profitability_weight": 0.5,
    "shortage_weight": 0.3,
    "strategic_weight": 0.2
  },
  "research_selection": {
    "cost_weight": 0.3,
    "utility_weight": 0.5,
    "synergy_weight": 0.2
  }
}
```

#### Load Weights at Runtime
**File:** `src/ai/ai_campaign_values.cpp`
**Modify:**
```cpp
// Current code:
const float STRENGTH_RATIO_TOTAL_VICTORY = 3.0f;

// Refactored:
struct ai_weights {
    float war_economic_value_weight = 0.4f;
    float war_strength_ratio_weight = 0.3f;
    float war_aggression_weight = 0.2f;
    float war_opportunism_weight = 0.1f;
    float war_risk_penalty_weight = 0.1f;
    
    // ... other weights ...
};

ai_weights load_ai_weights(sys::state& state) {
    // Load from data/ai/weights.json
    // ... implementation ...
}
```

### Step 6: Create AI Test Harness

#### Automated AI Tournaments
**File:** `tests/ai_tournament.cpp`
**Add:**
```cpp
#include <catch2/catch.hpp>
#include "ai/ai.hpp"

TEST_CASE("AI tournament: aggressive vs defensive", "[ai][tournament]") {
    // Create multiple AI nations with different personalities
    // Run simulation for N ticks
    // Measure outcomes (win rates, economic performance, etc.)
    
    sys::state game_state;
    initialize_test_scenario(game_state);
    
    // Assign personalities
    game_state.ai_personalities[nation_a] = aggressive_profile;
    game_state.ai_personalities[nation_b] = defensive_profile;
    
    // Run simulation
    for (int tick = 0; tick < 10000; tick++) {
        game_state.tick();
    }
    
    // Measure results
    auto win_rate_a = calculate_win_rate(game_state, nation_a);
    auto win_rate_b = calculate_win_rate(game_state, nation_b);
    
    // Assert that personalities produce different outcomes
    REQUIRE(win_rate_a != win_rate_b);
}

TEST_CASE("AI decision variety", "[ai][variety]") {
    // Run same scenario multiple times with same seed
    // Measure decision diversity
    
    std::vector<decision_log> logs;
    
    for (int run = 0; run < 100; run++) {
        sys::state game_state;
        game_state.game_seed = run;
        initialize_test_scenario(game_state);
        
        // Run simulation
        for (int tick = 0; tick < 1000; tick++) {
            game_state.tick();
        }
        
        logs.push_back(collect_decision_log(game_state));
    }
    
    // Calculate decision variety
    float variety = calculate_decision_variety(logs);
    
    // Assert variety is above threshold
    REQUIRE(variety > 0.3f);
}
```

### Step 7: Performance Optimization

#### Profile AI Decision Functions
**Tools:**
- Visual Studio Profiler
- perf + FlameGraph
- Custom timing in code

**Code for timing:**
```cpp
// Add timing to hot functions
void take_ai_decisions(sys::state& state) {
    auto start = std::chrono::high_resolution_clock::now();
    
    // ... existing logic ...
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
    
    // Log to file or telemetry
    log_ai_performance(state, duration.count());
}
```

#### Optimize Hot Paths
**Expected optimizations:**
1. **Cache province distances** (already planned in Week 2)
2. **Precompute neighbor lists** (avoid repeated graph traversals)
3. **Batch decision evaluations** (vectorized operations)
4. **Lazy evaluation** (only compute when needed)

#### Measure Improvements
**Success criteria:**
- 25%+ reduction in AI decision time
- 2x+ improvement in decision variety
- Better win rates in automated tournaments

---

## Week 4: Deterministic Job System & Parallelism

### Objective
Improve performance with parallel processing while preserving determinism

### Step 1: Design Job System API

#### Job System Interface
**File:** `src/concurrency/job_system.hpp`
**Add:**
```cpp
namespace concurrency {

// Job priority
enum class job_priority {
    high,
    normal,
    low,
    background
};

// Job task
struct job_task {
    std::function<void()> work;
    job_priority priority;
    std::vector<size_t> dependencies; // Task IDs this depends on
};

// Job system
class job_system {
public:
    job_system(size_t num_threads);
    ~job_system();
    
    // Submit job and return task ID
    size_t submit(job_task task);
    
    // Wait for specific task
    void wait(size_t task_id);
    
    // Wait for all tasks
    void wait_all();
    
    // Get number of worker threads
    size_t num_workers() const { return workers.size(); }
    
private:
    std::vector<std::thread> workers;
    std::priority_queue<job_task> queue;
    std::mutex queue_mutex;
    std::condition_variable cv;
    std::atomic<bool> shutdown{false};
    
    // Task tracking
    std::vector<std::atomic<bool>> task_completed;
    std::vector<std::vector<size_t>> task_dependencies;
    
    void worker_thread();
};

// Deterministic job scheduler (ensures ordered execution)
class deterministic_scheduler {
public:
    deterministic_scheduler(size_t num_threads);
    
    // Submit deterministic task
    size_t submit_deterministic(job_task task);
    
    // Execute tasks in deterministic order
    void execute_deterministic();
    
private:
    job_system system;
    std::vector<size_t> execution_order;
};

} // namespace concurrency
```

### Step 2: Integrate with Existing Parallelism

#### Replace `concurrency::parallel_for`
**File:** `src/ai/ai.cpp`
**Modify:**
```cpp
// Current code:
void take_ai_decisions(sys::state& state) {
    // ... collect decisions ...
    
    concurrency::parallel_for(0, num_decisions, [&](uint32_t i) {
        // evaluate decision i
    });
    
    // ... merge results ...
}

// Refactored:
void take_ai_decisions(sys::state& state) {
    // ... collect decisions ...
    
    // Submit evaluation tasks
    std::vector<size_t> task_ids;
    for (uint32_t i = 0; i < num_decisions; i++) {
        job_task task;
        task.work = [&, i]() {
            // evaluate decision i
        };
        task.priority = job_priority::normal;
        task_ids.push_back(state.job_system.submit(task));
    }
    
    // Wait for all evaluations
    for (auto task_id : task_ids) {
        state.job_system.wait(task_id);
    }
    
    // ... merge results (deterministic) ...
}
```

### Step 3: Implement Deterministic Merging

#### Ordered Result Combination
**File:** `src/concurrency/job_system.cpp`
**Add:**
```cpp
// Deterministic merge function
template<typename T>
std::vector<T> deterministic_merge(
    const std::vector<std::vector<T>>& partial_results,
    const std::vector<size_t>& order
) {
    std::vector<T> merged;
    merged.reserve(order.size() * partial_results[0].size());
    
    for (size_t idx : order) {
        for (const auto& result : partial_results[idx]) {
            merged.push_back(result);
        }
    }
    
    // Sort if needed for determinism
    std::sort(merged.begin(), merged.end(), [](const auto& a, const auto& b) {
        return a.id < b.id; // Use stable ordering
    });
    
    return merged;
}
```

### Step 4: Background Processing

#### Offload Expensive Computations
**File:** `src/concurrency/job_system.cpp`
**Add:**
```cpp
// Background task queue for non-critical work
class background_scheduler {
public:
    void schedule_background(std::function<void()> work) {
        // Submit with low priority
        job_task task;
        task.work = work;
        task.priority = job_priority::background;
        system.submit(task);
    }
    
    // Example: Pathfinding in background
    void schedule_pathfinding(sys::state& state, dcon::province_id start, dcon::province_id end) {
        schedule_background([&, start, end]() {
            auto path = find_path(state, start, end);
            // Store result for later use
            state.path_cache.store(start, end, path);
        });
    }
};
```

### Step 5: Preserve Determinism

#### Deterministic Execution Order
**Key principles:**
1. **Same seed → same execution order**
2. **Ordered merging of results**
3. **No race conditions on shared state**
4. **Reproducible outcomes**

**Implementation:**
```cpp
// In game loop:
void game_loop(sys::state& state) {
    // Use deterministic RNG for task ordering
    uint32_t seed = state.game_seed + state.current_date.value;
    
    // Generate deterministic task order
    std::vector<size_t> task_order = generate_deterministic_order(seed, num_tasks);
    
    // Execute in order
    for (size_t idx : task_order) {
        execute_task(state, idx);
    }
}
```

### Step 6: Measure Parallel Performance

#### Speedup Measurement
**Code:**
```cpp
// Benchmark parallel vs sequential
void benchmark_parallelism(sys::state& state) {
    auto start = std::chrono::high_resolution_clock::now();
    
    // Run AI decisions sequentially
    take_ai_decisions_sequential(state);
    
    auto seq_end = std::chrono::high_resolution_clock::now();
    
    // Run AI decisions in parallel
    take_ai_decisions_parallel(state);
    
    auto par_end = std::chrono::high_resolution_clock::now();
    
    auto seq_time = std::chrono::duration_cast<std::chrono::microseconds>(seq_end - start);
    auto par_time = std::chrono::duration_cast<std::chrono::microseconds>(par_end - seq_end);
    
    float speedup = static_cast<float>(seq_time.count()) / par_time.count();
    
    // Log results
    log_performance(state, "parallel_speedup", speedup);
}
```

#### Success Criteria
- 2-4x speedup on multi-core systems
- Deterministic outcomes preserved
- No race conditions or deadlocks

---

## Phase 2 Success Criteria

### AI Quality Metrics
- [ ] Decision variety increased by 30%+
- [ ] AI win rates improved by 15%+ in automated tournaments
- [ ] Personality system produces distinct behaviors
- [ ] Blackboard improves coordination between subsystems

### Performance Metrics
- [ ] AI decision time reduced by 25%+
- [ ] Parallel speedup of 2-4x on multi-core systems
- [ ] Late-game frame time improved by 20%+
- [ ] Memory allocations reduced in hot paths

### Code Quality Metrics
- [ ] 70%+ test coverage for AI subsystems
- [ ] All AI functions are pure and testable
- [ ] Data-driven weights in JSON
- [ ] No compiler warnings
- [ ] All existing tests pass

### Documentation
- [ ] AI architecture documentation
- [ ] Personality system guide
- [ ] Modding guide for AI weights
- [ ] Performance optimization guide

---

## Quick Reference Commands

### Build & Profile:
```bash
mkdir build && cd build
cmake -S .. -B . -G "Ninja" \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DCMAKE_CXX_FLAGS="-g -O2"
cmake --build . -- -j$(nproc)

# Profile with perf (Linux)
perf record -g -- ./src/AliceExecutable --scenario late_game.sav --headless --ticks 1000
perf script | flamegraph.pl > ai_profile.svg
```

### Run AI Tournament:
```bash
./build/tests/ai_tournament --runs 100 --ticks 10000
```

### Measure Performance:
```bash
# Before changes
./build/src/AliceExecutable --benchmark --scenario late_game.sav --ticks 1000

# After changes
./build/src/AliceExecutable --benchmark --scenario late_game.sav --ticks 1000
```

---

## Phase 2 Summary

**Week 2:** Performance profiling & optimization
- Profile late-game scenario
- Implement quick wins (serialization, UI cache, VBO, distance caching)
- Measure improvements

**Week 3:** AI architecture refactor
- Extract pure decision logic
- Implement personality system
- Add blackboard for coordination
- Create utility-based decisions
- Add data-driven weights
- Create AI test harness

**Week 4:** Deterministic job system
- Design job system API
- Integrate with existing parallelism
- Implement deterministic merging
- Add background processing
- Preserve determinism
- Measure parallel performance

**Success:** AI is more strategic, varied, and performant

---

**Phase 2 starts now. Let's modernize the AI.**
