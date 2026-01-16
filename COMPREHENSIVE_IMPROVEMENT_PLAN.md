# Project-Alice Fork: Comprehensive Improvement Plan

**Generated:** 2026-01-16T00:00:00Z
**Commit:** a90dd964a
**Branch:** main
**Sources:** DeepWiki (schombert/Project-Alice) + Repository Analysis + Oracle Guidance + Modern Strategy Game Research

---

## Executive Summary

This plan transforms Project-Alice from a faithful Victoria 2 retro clone into a **modern, high-performance strategy game engine** that preserves the depth and emergent gameplay that made Victoria 2 great, while incorporating cutting-edge advancements in AI, simulation, UI/UX, and performance.

**What Made Victoria 2 Great:**
- **Deep systemic gameplay:** POP-driven economy, industrialization, politics, diplomacy, and military all tightly coupled
- **Emergent narratives:** Complex systems create organic historical outcomes and player stories
- **Long-term strategic planning:** Decisions have cascading consequences over decades
- **Historical plausibility:** Systems feel rooted in 19th/early-20th century mechanics

**Modern Advancements to Incorporate:**
- **AI/ML:** Hybrid rule-based + learned models, adaptive difficulty, personality systems
- **Performance:** ECS architecture, data-oriented design, parallel processing, GPU acceleration
- **UI/UX:** Context-sensitive interfaces, dynamic tooltips, accessibility, visual feedback
- **Multiplayer:** Deterministic lockstep, snapshot/rollback, anti-cheat
- **Modding:** Data-driven design, safe scripting APIs, community content tools

**Top 3 Immediate Priorities:**
1. **Fix critical serialization vulnerability** (CRASH/SECURITY)
2. **Secure CI workflows** (SECURITY)
3. **Add test execution to CI** (QUALITY)

---

## Part 1: What Made Victoria 2 Great & How to Preserve It

### Core Mechanics That Defined Victoria 2

#### 1. Economy System (POP-Driven Industrialization)
**What it was:**
- Population (POPs) supplied labor AND created demand
- Factories consumed inputs, produced goods, balanced via global/local markets
- Capital formation and industrialization were slow, strategic processes
- Jobs were explicit (workers, craftsmen, clerks) with real consequences

**Why it mattered:**
- Industrialization felt organic and observable
- Consumer goods shortages created political/economic consequences
- Players could see long-term shifts (urbanization, regional specialization)

**How to preserve & modernize:**
- Keep POP-driven demand/supply loop
- Add **market overlays** with price history graphs and predicted consumption
- Add **supply chain visualization** (Sankey diagrams) showing goods flow
- Implement **economic advisors** (AI-run policies) with adjustable aggressiveness
- Add **data-driven factory recipes** (JSON/YAML) for moddability

#### 2. Military System (Logistics & Consequences)
**What it was:**
- Armies/navies assembled from brigades/ships with tech/modifiers
- Combat resolved with attrition, terrain, supply, and relative tech
- Wars were costly and consequential
- Logistics and theater management mattered more than micro

**Why it mattered:**
- Losses were persistent and manpower mattered
- Mobilization and casus belli forced long-term planning
- Geography shaped strategy

**How to preserve & modernize:**
- Keep logistics and supply as core mechanics
- Add **supply overlays** and projected attrition estimates
- Add **battle reports** showing why results happened (tech advantage, morale, terrain)
- Implement **utility-based war planning** for AI (strategic objectives, probabilistic path planners)
- Add **automated transport suggestions** for logistics

#### 3. Diplomacy System (Geopolitical Balancing)
**What it was:**
- Great power system, spheres of influence, infamy/relations
- Diplomatic actions (guarantees, demands), colonial "scramble"
- Soft-power manipulation and geopolitical balancing

**Why it mattered:**
- Created emergent rivalries and international tension
- Multiple viable strategies (trade-focused, industrialization, colonial expansion)

**How to preserve & modernize:**
- Keep spheres and influence mechanics
- Add **influence heatmaps** and cost visualization
- Add **diplomatic opportunity suggestions** (AI-driven recommendations)
- Implement **colonial negotiation minigames** for contested territories

#### 4. Politics & Population (POPs)
**What it was:**
- Internal politics driven by POP consciousness/militancy
- Party platforms, reforms, elections
- POPs had demographics, culture, religion, literacy, occupations

**Why it mattered:**
- Domestic choices constrained foreign/industrial policy
- Long-term societal effects from political decisions
- Human-scale feel to grand strategy

**How to preserve & modernize:**
- Keep POP-driven politics
- Add **POP timeline visualization** showing consciousness/militancy changes
- Add **reform simulator** showing immediate/long-term effects
- Implement **political preference vectors** (economic, cultural, authoritarian/liberal) for richer party policies

#### 5. Technology System
**What it was:**
- Long-running investments that unlocked factories, military improvements, institutions
- Timing and focus mattered

**Why it mattered:**
- Tech changed national capabilities gradually
- Strategic choices had long-term consequences

**How to preserve & modernize:**
- Keep meaningful tech progression
- Add **tech tooltips** showing exact gameplay effects and time-to-impact
- Add **tech path suggestions** based on player goals
- Implement **research queue** with auto-prioritization

#### 6. Map & Territory
**What it was:**
- Province-level resources, trade nodes, strategic chokepoints
- Terrain, infrastructure (railroads), colonial territories

**Why it mattered:**
- Geography shaped strategy
- Created asymmetric national advantages

**How to preserve & modernize:**
- Keep province-level detail
- Add **strategic overlays** (rail capacity, chokepoint risk, trade node throughput)
- Add **province aggregation** at large scales for performance while keeping frontlines detailed

### Common Criticisms & Pain Points to Address

| Issue | Modern Solution |
|-------|-----------------|
| **UI/UX: dense systems, poor discoverability** | Progressive disclosure, contextual help, search/filter, visual feedback |
| **Performance: late-game slowdown** | ECS architecture, lazy evaluation, spatial partitioning, GPU acceleration |
| **AI: predictable, weak governors** | Hybrid rule-based + learned models, utility functions, personality systems |
| **Multiplayer: fragility, desyncs** | Deterministic lockstep, snapshot/rollback, authoritative server model |
| **Learning curve: steep, limited tutorials** | Role-based tutorials, scenario-based learning, progressive complexity |
| **Modding: limited, hard to extend** | Data-driven design, safe scripting APIs, mod manager with conflict detection |

---

## Part 2: Modern Strategy Game Advancements (Industry Research)

### AI/ML in Strategy Games (2024-2025)

#### Current State (Paradox, Firaxis, CA)
- **Hybrid approach:** Rule-based core + ML experiments
- **Offline training:** ML models trained on replays, exported as deterministic policies
- **Data-driven tuning:** Designer-tunable weights, personality profiles
- **Replay analytics:** Telemetry to identify AI weaknesses

#### What Works in Production
- **Paradox (Stellaris/CK3/HOI4):** Scripted rule systems + data-driven decision weights + careful profiling
- **Firaxis (Civ):** Community-driven AI improvements via modding and weight tuning
- **Creative Assembly (Total War):** Decoupled workloads, background processing for AI pathfinding

#### Recommendations for Project-Alice
1. **Keep deterministic core:** No online ML in multiplayer/simulation
2. **Add telemetry & replay:** Record decisions for offline analysis
3. **Implement hybrid AI:** Rule-based decisions + learned policy approximations (offline trained)
4. **Data-driven personalities:** Move AI weights to JSON/YAML for moddability
5. **AI test harness:** Automated AI-vs-AI tournaments for evaluation

### Simulation & Performance Architecture

#### ECS (Entity Component System) Pattern
- **What:** Decouple data from behavior, use component arrays for cache efficiency
- **Why:** Massive performance gains for simulation-heavy games
- **Project-Alice:** Already uses dcon (data container) which is ECS-like; can be optimized further

#### Data-Oriented Design
- **What:** Structure data for CPU cache efficiency (SoA vs AoS)
- **Why:** 2-10x speedups for hot loops
- **Project-Alice:** Already uses `ve::fp_vector` for some operations; expand to more subsystems

#### Parallel Processing
- **What:** Multi-threaded simulation with deterministic merging
- **Why:** Utilize modern multi-core CPUs
- **Project-Alice:** Already uses `concurrency::parallel_for` in some places; expand with job system

#### GPU Acceleration
- **What:** Offload compute-heavy operations to GPU (pathfinding, map overlays, some simulation)
- **Why:** Free up CPU for other tasks
- **Project-Alice:** Map rendering already uses GPU; consider compute shaders for simulation

#### Predictive Simulation & Rollback
- **What:** Run simulation ahead, rollback on input changes (for multiplayer)
- **Why:** Smooth multiplayer experience despite latency
- **Project-Alice:** Deterministic core enables this; needs snapshot/rollback infrastructure

### UI/UX Modernization

#### Context-Sensitive Interfaces
- **What:** UI adapts to player context, shows relevant information
- **Why:** Reduces information overload, improves discoverability
- **Project-Alice:** Can add context-sensitive tooltips, dynamic UI panels

#### Visual Feedback Systems
- **What:** Clear visual indicators for system states and changes
- **Why:** Players understand cause-and-effect immediately
- **Project-Alice:** Add market flow visuals, political change indicators, combat outcome explanations

#### Accessibility Features
- **What:** Color-blind modes, font scaling, keyboard navigation, screen reader support
- **Why:** Broader player base, better UX for all
- **Project-Alice:** Add theme manager, focus management, scalable fonts

#### Mobile/Touch Support Concepts
- **What:** Touch-friendly UI patterns, controller support
- **Why:** Expand platform reach
- **Project-Alice:** Can add touch gestures, controller navigation as future enhancement

### Multiplayer & Networking

#### Deterministic Lockstep
- **What:** All clients run identical simulation, only inputs are transmitted
- **Why:** Minimal bandwidth, perfect sync
- **Project-Alice:** Already has deterministic core; needs input management and desync detection

#### Authoritative Server Model
- **What:** Server runs authoritative simulation, clients are thin UIs
- **Why:** Prevents cheating, handles desyncs gracefully
- **Project-Alice:** Can implement server-authoritative mode for competitive play

#### Snapshot & Rollback
- **What:** Periodic state snapshots, rollback on desync
- **Why:** Tolerates network issues, enables reconnection
- **Project-Alice:** Needs snapshot infrastructure and rollback logic

#### Anti-Cheat Considerations
- **What:** Validate inputs, detect anomalies, use server authority
- **Why:** Fair competitive play
- **Project-Alice:** Server-authoritative model + input validation

### Modding & Community Content

#### Data-Driven Design
- **What:** Rules, templates, and content in readable data files
- **Why:** Easy modding, no code changes needed
- **Project-Alice:** Already uses some data files; expand to factory recipes, tech trees, AI weights

#### Safe Scripting APIs
- **What:** Sandboxed scripting for complex behaviors
- **Why:** Enable advanced mods without security risks
- **Project-Alice:** Can add Lua or similar for event scripting

#### Mod Manager & Conflict Detection
- **What:** Integrated mod loader with dependency resolution
- **Why:** Better user experience, fewer broken mods
- **Project-Alice:** Needs mod manifest system and conflict detection

#### Version Compatibility
- **What:** Mod versioning and compatibility checks
- **Why:** Prevent broken mods after updates
- **Project-Alice:** Needs mod version metadata and compatibility layer

---

## Part 3: Current State Analysis (Project-Alice)

### Architecture Overview

#### Strengths
1. **Deterministic core:** Well-structured for multiplayer and replays
2. **ECS-like data model:** dcon provides structure-of-arrays storage
3. **SPSC queues:** Good for UI/game communication
4. **Vectorized operations:** `ve::fp_vector` used in hot paths
5. **Modular subsystems:** Clear separation (economy, military, AI, etc.)

#### Weaknesses
1. **Serialization vulnerability:** Raw memory ops, no error handling
2. **CI gaps:** No tests, no sanitizers, insecure workflows
3. **Performance bottlenecks:** Late-game slowdown, heavy UI updates
4. **AI limitations:** Pure heuristics, no learning, predictable behavior
5. **UI/UX issues:** Information overload, poor discoverability, performance problems
6. **Technical debt:** Large functions, platform-specific code, missing tests

### Critical Issues (Fix Immediately)

#### 1. Serialization Vulnerability (CRITICAL)
**File:** `src/gamestate/serialization.cpp`
**Problem:** Raw `new[]/delete[]`, no bounds checks, no ZSTD error handling
**Risk:** Crashes, OOM, save corruption, data loss
**Fix:** RAII buffers, bounds checking, ZSTD error validation

#### 2. CI Security (CRITICAL)
**File:** `.github/workflows/opencode.yml`
**Problem:** Comment-triggered workflow exposes OPENCODE_API_KEY to untrusted actors
**Risk:** Secret exfiltration, supply-chain attack
**Fix:** Maintainer-only gating, pin actions, minimal permissions

#### 3. Missing Test Execution (HIGH)
**File:** `.github/workflows/build.yml`
**Problem:** Builds but doesn't run tests or sanitizers
**Risk:** Regressions slip through, hard to debug
**Fix:** Add ctest, ASAN/UBSAN jobs, artifact upload

### Performance Hotspots

#### CPU Bottlenecks
1. **Map rendering:** VBO reuploads, many GL state changes
2. **UI updates:** Tooltip generation, per-frame Lua calls
3. **AI decisions:** `take_ai_decisions`, province loops
4. **Economy simulation:** Per-province/nation loops
5. **Serialization:** Per-section allocations and decompression

#### Memory Issues
1. **Per-decompression allocations:** `new[]/delete[]` in hot path
2. **Temporary vectors:** Repeated allocations in loops
3. **VBO updates:** Frequent `glBufferData` calls

#### Threading Patterns
1. **UI cache thread:** Polling with sleep (can be event-driven)
2. **AI parallel_for:** Good, but can be optimized further
3. **Game loop:** Single-threaded simulation (deterministic)

### AI Implementation Analysis

#### Current Capabilities
- **Economic AI:** Factory selection, budget allocation, construction
- **Military AI:** War goals, unit production, logistics
- **Diplomatic AI:** Alliances, spheres, influence
- **Political AI:** National focuses, reforms
- **Research AI:** Tech selection by weights

#### Limitations
1. **Pure heuristics:** No learning or adaptation
2. **Predictable:** Same inputs → same outputs (can be exploited)
3. **Performance:** Heavy per-province loops
4. **No personality:** All nations behave similarly
5. **Limited coordination:** Subsystems don't communicate well

#### Improvement Opportunities
1. **Utility-based decisions:** Replace simple heuristics with utility functions
2. **Personality system:** Different AI profiles (aggressive, defensive, opportunistic)
3. **Learning from replays:** Offline ML training on human games
4. **Better coordination:** Shared blackboard between AI subsystems
5. **Predictive planning:** Multi-turn lookahead for decisions

### Gameplay System Analysis

#### Economy System (Strengths)
- **Detailed simulation:** POP-driven demand, factory inputs/outputs
- **Vectorized operations:** Good performance potential
- **Market history:** Price tracking and GDP calculation
- **Presimulation:** Proper scenario initialization

#### Economy System (Weaknesses)
- **Complexity:** Hard to understand for new players
- **Feedback:** Limited visual indicators of economic health
- **Automation:** Minimal player assistance in late-game
- **Balance:** Some factory types may be over/underpowered

#### Military System (Strengths)
- **Comprehensive helpers:** Unit selection, supply, logistics
- **War mechanics:** CB system, wargoals, peace treaties
- **Mobilization:** Real manpower constraints

#### Military System (Weaknesses)
- **Combat resolution:** Not fully inspected (may be in separate files)
- **AI war planning:** Limited strategic depth
- **Logistics UI:** Poor visibility of supply constraints
- **Battle feedback:** Minimal explanation of outcomes

#### Diplomacy System (Strengths)
- **AI alliance logic:** Heuristics for alliance formation
- **CB system:** Comprehensive casus belli mechanics
- **War intervention:** Logic for joining wars

#### Diplomacy System (Weaknesses)
- **Limited player tools:** Poor visibility of influence/spheres
- **No negotiation:** Static diplomatic actions
- **AI predictability:** Same patterns every game

#### Population & Politics (Strengths)
- **Detailed POPs:** Demographics, culture, religion
- **Political mechanics:** Consciousness, militancy, voting
- **Reform system:** Meaningful political changes

#### Population & Politics (Weaknesses)
- **Poor visibility:** Hard to see political trends
- **Limited feedback:** Players don't understand why POPs change
- **No timeline:** Can't see historical political shifts

#### Technology System (Strengths)
- **Meaningful progression:** Techs have real impact
- **Research management:** Weight-based selection

#### Technology System (Weaknesses)
- **Poor discoverability:** Hard to understand tech effects
- **No planning:** Can't queue or plan research paths
- **Limited feedback:** No visual indicators of tech impact

#### Map & Territory (Strengths)
- **Province-level detail:** Resources, terrain, infrastructure
- **Map loading:** Proper province data integration

#### Map & Territory (Weaknesses)
- **Performance:** May be heavy for large maps
- **Strategic overlays:** Limited visualization tools
- **No aggregation:** Can't simplify distant provinces

### UI/UX Analysis

#### Current UI Architecture
- **Central state:** `ui::state` manages roots, tooltips, windows
- **Element model:** Virtual functions for input, update, render
- **Tooltip system:** Primary information channel, built on hover
- **Map/minimap:** Click-to-center, map-mode toggles

#### UI/UX Pain Points
1. **Information overload:** Too much data, poor prioritization
2. **Poor discoverability:** Hover-only, no search/filter
3. **Performance:** Heavy tooltip generation, per-frame Lua calls
4. **Accessibility:** No color-blind modes, limited keyboard nav
5. **Visual feedback:** Minimal indicators for system changes

#### Modern UI Opportunities
1. **Context-sensitive interfaces:** Show relevant info based on context
2. **Dynamic tooltips:** Summary on hover, full details on click/pin
3. **Visual feedback:** Charts, graphs, flow diagrams
4. **Accessibility:** Theme manager, focus management, scalable fonts
5. **Performance:** Tooltip caching, async computation, incremental updates

---

## Part 4: Comprehensive Improvement Plan

### Phase 0: Foundation & Safety (Week 1)

#### Day 1-2: Critical Serialization Fix
**Goal:** Eliminate crashes and data corruption from unsafe decompression

**Files to Modify:**
- `src/gamestate/serialization.cpp`
- `src/gamestate/serialization.hpp` (if needed)

**Changes:**
1. Replace raw `new[]/delete[]` with `std::vector<uint8_t>`
2. Add bounds checking:
   - Validate `decompressed_length` ≤ 256MiB (configurable)
   - Validate `section_length` fits in remaining buffer
   - Check for integer overflow
3. Validate ZSTD return with `ZSTD_isError()`
4. Handle failures gracefully (error accumulation)

**Tests to Add:**
- `tests/serialization_tests.cpp` (round-trip, error paths)
- `tests/serialization_fuzz_tests.cpp` (corrupted/truncated inputs)

**Success Criteria:**
- Code compiles without warnings
- All existing tests pass
- New tests cover error paths
- No crashes on malformed input

**Evidence Required:**
- Build output (exit code 0)
- Test output (all tests pass)
- Manual verification: load save successfully
- Regression: existing save/load still works

#### Day 3: CI Security Fix
**Goal:** Prevent secret exposure and supply-chain attacks

**Files to Modify:**
- `.github/workflows/opencode.yml`

**Changes:**
1. Add maintainer-only gating:
   - Check `github.actor` is repo collaborator before secret usage
   - Use GitHub API to verify collaborator status
2. Pin action to specific commit SHA (not `@latest`)
3. Remove unnecessary permissions (`id-token: write` unless required)
4. Add explicit minimal permissions block
5. Consider moving to `workflow_dispatch` for manual approval

**Success Criteria:**
- Workflow compiles without errors
- Non-maintainer comments don't trigger secret-using steps
- Action is pinned to SHA
- Minimal permissions are set

**Evidence Required:**
- Workflow run logs showing maintainer check works
- Verification that secret is not exposed to untrusted triggers

#### Day 4-5: CI Test Execution
**Goal:** Ensure tests run in CI and catch regressions

**Files to Modify:**
- `.github/workflows/build.yml`

**Changes:**
1. Add `ctest` execution step
2. Add sanitizer job (ASAN + UBSAN on Linux)
3. Add artifact upload for logs/binaries
4. Add ccache/actions/cache for build speed
5. Add concurrency controls to prevent redundant runs

**Success Criteria:**
- CI runs tests on every PR/push
- Sanitizer job runs and catches issues
- Artifacts are uploaded on failure
- Build times are reduced via caching

**Evidence Required:**
- CI run logs showing test execution
- Sanitizer run logs (if any issues found)
- Artifact download verification

### Phase 1: Performance & AI Modernization (Weeks 2-4)

#### Week 2: Performance Profiling & Optimization
**Goal:** Identify and fix performance bottlenecks

**Files to Analyze:**
- `src/gamestate/system_state.cpp` (render, UI cache)
- `src/map/map.cpp` (VBO updates, rendering)
- `src/ai/ai.cpp` (decision evaluation)
- `src/economy/economy.cpp` (simulation loops)
- `src/gamestate/serialization.cpp` (save/load)

**Tools to Use:**
- Windows: Visual Studio Profiler, WPR/WPA, VTune
- Linux: perf + FlameGraph
- GPU: RenderDoc for draw call analysis

**Optimizations to Implement:**
1. **Serialization:** Replace per-decompression allocations with thread-local buffers
2. **UI:** Move Lua calls out of render loop, reduce frequency
3. **UI cache:** Replace polling with condition variables
4. **Map rendering:** Use persistent mapped buffers for VBOs, reduce GL state changes
5. **AI:** Cache province distances, precompute neighbor lists
6. **Economy:** Reuse temporary vectors, add incremental updates

**Success Criteria:**
- Profile shows measurable improvement in hotspots
- Frame time reduced by 20%+ in late-game scenarios
- Memory allocations reduced in hot paths

**Evidence Required:**
- Before/after profiles with flamegraphs
- Frame time measurements
- Memory allocation traces

#### Week 3: AI Architecture Refactor
**Goal:** Modernize AI with utility-based decisions and better coordination

**Files to Modify:**
- `src/ai/ai.cpp` (decision evaluation)
- `src/ai/ai_war.cpp` (war planning)
- `src/ai/ai_economy.cpp` (economic decisions)
- `src/ai/ai_campaign.cpp` (strategy layer)

**Changes:**
1. **Utility-based decisions:** Replace simple heuristics with utility functions
2. **Personality system:** Add AI profiles (aggressive, defensive, opportunistic)
3. **Better coordination:** Shared blackboard between AI subsystems
4. **Predictive planning:** Multi-turn lookahead for decisions
5. **Data-driven weights:** Move AI constants to JSON/YAML

**Implementation Steps:**
1. Extract pure decision logic into testable functions
2. Add utility scoring framework
3. Implement personality parameters
4. Add blackboard for inter-subsystem communication
5. Create data-driven weight system

**Success Criteria:**
- AI decisions are more varied and less predictable
- AI performance improved (better win rates in automated tournaments)
- Code is more maintainable and testable

**Evidence Required:**
- AI tournament results (before/after)
- Code coverage metrics
- Performance profiling of AI decisions

#### Week 4: Deterministic Job System & Parallelism
**Goal:** Improve performance with parallel processing while preserving determinism

**Files to Modify:**
- `src/gamestate/system_state.cpp` (game loop)
- `src/ai/ai.cpp` (decision evaluation)
- `src/economy/economy.cpp` (simulation updates)
- New: `src/concurrency/job_system.cpp`

**Changes:**
1. **Job scheduler:** Deterministic task ordering with worker threads
2. **Dependency tracking:** Read/write dependencies between tasks
3. **Result merging:** Deterministic combination of parallel results
4. **Background processing:** Offload expensive computations (pathfinding, planning)

**Implementation Steps:**
1. Design job system API (tasks, dependencies, priorities)
2. Implement deterministic scheduler (ordered execution)
3. Integrate with existing `concurrency::parallel_for`
4. Add background task queue for non-critical work
5. Preserve determinism with ordered merging

**Success Criteria:**
- Parallel speedup measurable (2-4x on multi-core)
- Determinism preserved (replays match, no desyncs)
- No race conditions or deadlocks

**Evidence Required:**
- Parallel speedup measurements
- Deterministic replay tests
- Stress test results (many threads, heavy load)

### Phase 2: Gameplay Refinements & Modern Features (Weeks 5-8)

#### Week 5: Economy Modernization
**Goal:** Improve economic gameplay with better feedback and automation

**Files to Modify:**
- `src/economy/economy.cpp` (core simulation)
- `src/economy/economy_production.cpp` (factory logic)
- `src/economy/economy_pops.cpp` (POP consumption)
- New: `src/economy/economy_ui.cpp` (UI helpers)

**Changes:**
1. **Market overlays:** Price history graphs, consumption predictions
2. **Supply chain visualization:** Sankey diagrams for goods flow
3. **Economic advisors:** AI-run policies with adjustable aggressiveness
4. **Data-driven factory recipes:** JSON/YAML for moddability
5. **Automation tools:** Late-game factory management assistance

**Implementation Steps:**
1. Add market data collection and history tracking
2. Create UI components for market visualization
3. Implement economic advisor framework
4. Add data-driven factory definition system
5. Build automation UI (factory build queues, upgrade suggestions)

**Success Criteria:**
- Players can understand economic health at a glance
- Late-game economic management is less tedious
- Modders can add new factory types easily

**Evidence Required:**
- UI mockups and wireframes
- Player feedback on new economic tools
- Modding documentation and examples

#### Week 6: Military Modernization
**Goal:** Improve military gameplay with better logistics and feedback

**Files to Modify:**
- `src/military/military.cpp` (core combat)
- `src/ai/ai_war.cpp` (AI war planning)
- New: `src/military/military_ui.cpp` (UI helpers)

**Changes:**
1. **Supply overlays:** Visual indicators of supply constraints
2. **Battle reports:** Detailed explanations of combat outcomes
3. **Automated logistics:** Suggestions for transport and supply
4. **War planning tools:** Strategic objective setting for players
5. **AI war improvements:** Utility-based war planning, better coordination

**Implementation Steps:**
1. Add supply visualization to map overlay system
2. Create battle report generator (tech, morale, terrain analysis)
3. Implement logistics suggestion system
4. Add war planning UI (objectives, troop allocation)
5. Refactor AI war planning with utility functions

**Success Criteria:**
- Players understand supply constraints visually
- Combat outcomes are explainable
- AI war planning is more strategic and less predictable

**Evidence Required:**
- Supply overlay screenshots
- Battle report examples
- AI war performance metrics

#### Week 7: Diplomacy & Politics Modernization
**Goal:** Improve diplomatic gameplay with better visibility and interaction

**Files to Modify:**
- `src/ai/ai_influence.cpp` (sphere mechanics)
- `src/ai/ai_alliances.cpp` (alliance logic)
- `src/military/military.cpp` (diplomatic messages)
- New: `src/diplomacy/diplomacy_ui.cpp` (UI helpers)

**Changes:**
1. **Influence heatmaps:** Visual representation of spheres and influence
2. **Diplomatic opportunities:** AI-suggested actions based on situation
3. **Negotiation tools:** Interactive diplomacy (trade, alliances, treaties)
4. **Political timeline:** Visual history of political changes
5. **Reform simulator:** Preview effects of political reforms

**Implementation Steps:**
1. Add influence visualization to map overlay system
2. Create diplomatic opportunity detector
3. Implement negotiation UI (trade screens, treaty editors)
4. Build political timeline component
5. Add reform preview tool

**Success Criteria:**
- Players can see diplomatic relationships clearly
- Diplomatic decisions are more interactive
- Political changes are understandable and predictable

**Evidence Required:**
- Influence heatmap examples
- Diplomatic negotiation UI mockups
- Political timeline screenshots

#### Week 8: Technology & Research Modernization
**Goal:** Improve research gameplay with better planning and feedback

**Files to Modify:**
- `src/ai/ai_campaign.cpp` (research decisions)
- `src/economy/economy.cpp` (tech effects)
- New: `src/technology/technology_ui.cpp` (UI helpers)

**Changes:**
1. **Tech tooltips:** Show exact effects and time-to-impact
2. **Research planning:** Queue system with auto-prioritization
3. **Tech path suggestions:** AI recommendations based on goals
4. **Invention tracking:** Visual progress and unlock notifications
5. **Research analytics:** Show historical research performance

**Implementation Steps:**
1. Enhance tech tooltips with detailed effect information
2. Implement research queue system
3. Add tech path suggestion algorithm
4. Create invention tracking UI
5. Build research analytics dashboard

**Success Criteria:**
- Players understand tech effects before research
- Research planning is strategic rather than reactive
- Tech progression feels meaningful and impactful

**Evidence Required:**
- Tech tooltip examples
- Research queue UI mockups
- Tech path suggestion examples

### Phase 3: UI/UX Modernization (Weeks 9-12)

#### Week 9: Tooltip & Information System
**Goal:** Reduce information overload with smart, cached tooltips

**Files to Modify:**
- `src/gui/ui_state.cpp` (tooltip management)
- `src/gui/gui_tooltips.cpp` (tooltip generators)
- `src/gui/gui_element_base.cpp` (element lifecycle)

**Changes:**
1. **Tooltip caching:** Cache layout and content by (type, object, version)
2. **Pinned info panel:** Allow users to pin heavy tooltips for detailed view
3. **Summary vs full:** Show lightweight summary on hover, full details on click
4. **Async computation:** Offload expensive tooltip generation to background
5. **Context sensitivity:** Show relevant information based on player context

**Implementation Steps:**
1. Add tooltip cache structure (map keyed by signature)
2. Implement pinned panel UI element
3. Modify heavy tooltip generators to use cache
4. Add async worker for expensive computations
5. Implement context detection logic

**Success Criteria:**
- Tooltip rebuilds reduced by 50%+ on repeated hovers
- Heavy tooltips don't block UI thread
- Players can access detailed information on demand

**Evidence Required:**
- Performance measurements (before/after tooltip CPU time)
- User feedback on tooltip usability
- Screenshots of pinned panel

#### Week 10: Focus Management & Accessibility
**Goal:** Add keyboard navigation and accessibility features

**Files to Modify:**
- `src/gui/ui_state.cpp` (focus management)
- `src/gui/gui_element_base.cpp` (element focus)
- `src/gui/gui_minimap.cpp` (minimap navigation)

**Changes:**
1. **Focus manager:** Central focus API for keyboard navigation
2. **Tab cycle:** Navigate between UI elements with Tab/Shift+Tab
3. **Visual focus indicators:** Clear focus rectangles
4. **Accessibility features:** Color-blind modes, font scaling
5. **Controller support:** Basic controller navigation patterns

**Implementation Steps:**
1. Implement FocusManager class in ui_state
2. Add focus hooks to important UI elements
3. Add visual focus indicators
4. Create theme manager for accessibility
5. Add controller input handling

**Success Criteria:**
- Keyboard navigation works across major UI screens
- Accessibility features are functional
- Controller support is usable (if implemented)

**Evidence Required:**
- Keyboard navigation test results
- Accessibility feature screenshots
- Controller input logs

#### Week 11: Map & Visualization Modernization
**Goal:** Improve map interface with better overlays and interactions

**Files to Modify:**
- `src/map/map.cpp` (map rendering)
- `src/gui/gui_minimap.cpp` (minimap)
- `src/gui/gui_province_window.cpp` (province UI)

**Changes:**
1. **Dynamic overlays:** Toggleable layers (supply, trade, politics, etc.)
2. **Zoom & pan improvements:** Smoother navigation, zoom to fit
3. **Strategic view:** Simplified view for large-scale planning
4. **Visual feedback:** Animations and highlights for important events
5. **Map search:** Find provinces by name, resource, or criteria

**Implementation Steps:**
1. Add overlay system with multiple layers
2. Improve zoom/pan controls and smoothness
3. Implement strategic view (province aggregation)
4. Add visual feedback system (animations, highlights)
5. Build map search UI

**Success Criteria:**
- Map navigation is smooth and intuitive
- Overlays provide useful strategic information
- Players can find locations quickly

**Evidence Required:**
- Map overlay screenshots
- Navigation performance metrics
- Search functionality examples

#### Week 12: Event & Decision System
**Goal:** Modernize event presentation and decision-making

**Files to Modify:**
- `src/gui/gui_events.cpp` (event UI)
- `src/gui/gui_decisions.cpp` (decision UI)
- `src/gamestate/system_state.cpp` (event triggers)

**Changes:**
1. **Event cards:** Modern, visually appealing event presentation
2. **Decision previews:** Show consequences before choosing
3. **Event history:** Timeline of past events and decisions
4. **Event filtering:** Filter by type, importance, or category
5. **Event suggestions:** AI recommendations for decisions

**Implementation Steps:**
1. Redesign event card UI
2. Add decision preview system
3. Implement event history timeline
4. Add event filtering UI
5. Build decision suggestion system

**Success Criteria:**
- Events are visually appealing and informative
- Decision consequences are clear
- Event history is useful for learning

**Evidence Required:**
- Event card mockups
- Decision preview examples
- Event history timeline screenshots

### Phase 4: Multiplayer & Modding (Weeks 13-16)

#### Week 13: Multiplayer Infrastructure
**Goal:** Build robust multiplayer with deterministic lockstep

**Files to Modify:**
- `src/network/network.cpp` (network layer)
- `src/gamestate/system_state.cpp` (game loop)
- New: `src/network/multiplayer.cpp` (multiplayer manager)

**Changes:**
1. **Deterministic lockstep:** All clients run identical simulation
2. **Input management:** Collect and order player inputs
3. **Desync detection:** Periodic checksum comparison
4. **Snapshot system:** Periodic state snapshots for rollback
5. **Reconnection:** Allow players to rejoin with state sync

**Implementation Steps:**
1. Implement input collection and ordering
2. Add deterministic simulation verification
3. Build desync detection and reporting
4. Create snapshot system (compress, store, restore)
5. Implement reconnection protocol

**Success Criteria:**
- Multiplayer games run without desyncs
- Players can reconnect after disconnection
- Desyncs are detected and reported

**Evidence Required:**
- Multiplayer test results
- Desync detection logs
- Reconnection success rates

#### Week 14: Modding System
**Goal:** Create comprehensive modding support

**Files to Modify:**
- `src/gamestate/system_state.cpp` (mod loading)
- New: `src/modding/mod_manager.cpp` (mod management)
- New: `src/modding/mod_loader.cpp` (mod loading)

**Changes:**
1. **Mod manifest system:** Metadata for mods (name, version, dependencies)
2. **Mod loader:** Load mods from directory with conflict detection
3. **Data-driven design:** Expand data files (factory recipes, tech trees, AI weights)
4. **Scripting API:** Safe Lua scripting for events and decisions
5. **Mod manager UI:** In-game mod browser and manager

**Implementation Steps:**
1. Design mod manifest format (JSON/YAML)
2. Implement mod loader with dependency resolution
3. Expand data-driven systems
4. Add Lua scripting sandbox
5. Build mod manager UI

**Success Criteria:**
- Mods can be installed and managed in-game
- Mod conflicts are detected and reported
- Modding documentation is comprehensive

**Evidence Required:**
- Mod examples and documentation
- Conflict detection test results
- Mod manager UI screenshots

#### Week 15: Performance & Polish
**Goal:** Final performance optimization and polish

**Files to Modify:**
- All performance-critical subsystems
- UI polish and bug fixes

**Changes:**
1. **Final profiling:** Identify remaining bottlenecks
2. **Optimization passes:** Apply targeted fixes
3. **Bug fixes:** Address remaining issues
4. **Polish:** UI refinements, visual improvements
5. **Documentation:** Update user guides and API docs

**Implementation Steps:**
1. Run comprehensive profiling
2. Apply targeted optimizations
3. Fix remaining bugs
4. Polish UI and visuals
5. Update documentation

**Evidence Required:**
- Final performance metrics
- Bug fix list
- Documentation updates

#### Week 16: Testing & Release Preparation
**Goal:** Comprehensive testing and release readiness

**Files to Modify:**
- Test infrastructure
- Release documentation

**Changes:**
1. **Comprehensive testing:** Unit, integration, performance, stress tests
2. **Beta testing:** Community beta program
3. **Release notes:** Document all changes and improvements
4. **Migration guide:** Help players transition from original
5. **Launch preparation:** Marketing, community engagement

**Implementation Steps:**
1. Run comprehensive test suite
2. Launch beta program
3. Prepare release notes
4. Create migration guide
5. Plan launch strategy

**Evidence Required:**
- Test coverage reports
- Beta feedback summary
- Release documentation

---

## Part 5: Success Metrics & Verification

### Performance Metrics

#### CPU Performance
- **Target:** 20%+ reduction in frame time for late-game scenarios
- **Measurement:** Profiling before/after each optimization phase
- **Tools:** Visual Studio Profiler, perf, FlameGraph

#### Memory Usage
- **Target:** 30%+ reduction in peak allocations during hot paths
- **Measurement:** Allocation tracking before/after
- **Tools:** Memory profilers, custom allocation tracking

#### Load/Save Performance
- **Target:** 50%+ reduction in save/load time
- **Measurement:** Timing save/load operations
- **Tools:** Custom timing, system clock

### AI Metrics

#### Decision Quality
- **Target:** 15%+ improvement in AI win rates vs baseline
- **Measurement:** Automated AI tournaments
- **Tools:** Custom tournament harness, replay analysis

#### Decision Variety
- **Target:** 30%+ increase in decision diversity
- **Measurement:** Decision tree analysis
- **Tools:** Decision logging and analysis

#### Performance
- **Target:** 25%+ reduction in AI decision time
- **Measurement:** Profiling AI decision functions
- **Tools:** CPU profiler

### Gameplay Metrics

#### Player Understanding
- **Target:** 50%+ reduction in time to understand economic health
- **Measurement:** User testing with new players
- **Tools:** A/B testing, user feedback surveys

#### Late-Game Performance
- **Target:** Smooth 60 FPS in late-game scenarios (target hardware)
- **Measurement:** Frame time measurements
- **Tools:** In-game profiler, external profilers

#### Modding Adoption
- **Target:** 10+ community mods within 3 months of release
- **Measurement:** Mod repository tracking
- **Tools:** Mod download statistics, community feedback

### Quality Metrics

#### Test Coverage
- **Target:** 70%+ code coverage for critical subsystems
- **Measurement:** Code coverage reports
- **Tools:** Coverage analysis tools

#### Bug Count
- **Target:** 90%+ reduction in critical bugs
- **Measurement:** Bug tracking system
- **Tools:** Issue tracker, automated testing

#### Build Stability
- **Target:** 95%+ CI success rate
- **Measurement:** CI build logs
- **Tools:** CI analytics

---

## Part 6: Implementation Roadmap Summary

### Phase 0: Foundation & Safety (Week 1)
- [ ] Serialization safety fix (Day 1-2)
- [ ] CI security fix (Day 3)
- [ ] CI test execution (Day 4-5)

### Phase 1: Performance & AI Modernization (Weeks 2-4)
- [ ] Performance profiling & optimization (Week 2)
- [ ] AI architecture refactor (Week 3)
- [ ] Deterministic job system (Week 4)

### Phase 2: Gameplay Refinements & Modern Features (Weeks 5-8)
- [ ] Economy modernization (Week 5)
- [ ] Military modernization (Week 6)
- [ ] Diplomacy & politics modernization (Week 7)
- [ ] Technology & research modernization (Week 8)

### Phase 3: UI/UX Modernization (Weeks 9-12)
- [ ] Tooltip & information system (Week 9)
- [ ] Focus management & accessibility (Week 10)
- [ ] Map & visualization modernization (Week 11)
- [ ] Event & decision system (Week 12)

### Phase 4: Multiplayer & Modding (Weeks 13-16)
- [ ] Multiplayer infrastructure (Week 13)
- [ ] Modding system (Week 14)
- [ ] Performance & polish (Week 15)
- [ ] Testing & release preparation (Week 16)

---

## Part 7: Immediate Next Steps

### Today (First 24 Hours)
1. **Start serialization safety fix** (Day 1-2)
   - Open `src/gamestate/serialization.cpp`
   - Replace raw allocations with `std::vector<uint8_t>`
   - Add bounds checking and ZSTD error handling
   - Create test files

2. **Start CI security fix** (Day 3)
   - Open `.github/workflows/opencode.yml`
   - Add maintainer gating
   - Pin actions to SHA
   - Test workflow changes

3. **Start CI test execution** (Day 4-5)
   - Open `.github/workflows/build.yml`
   - Add ctest execution
   - Add sanitizer job
   - Add artifact upload

### Tomorrow (24-48 Hours)
1. **Run profiling** on late-game scenario
   - Build with profiling flags
   - Run representative save
   - Capture CPU profiles
   - Identify top 5 hotspots

2. **Implement first optimization**
   - Choose highest-impact hotspot
   - Apply targeted fix
   - Measure improvement
   - Document results

### This Week (First 7 Days)
1. **Complete Phase 0** (Foundation & Safety)
   - All critical fixes implemented
   - Tests passing
   - CI green

2. **Begin Phase 1** (Performance & AI)
   - Profiling complete
   - First optimization implemented
   - AI refactor started

### This Month (First 30 Days)
1. **Complete Phase 1** (Performance & AI)
   - Performance improvements measurable
   - AI modernization complete
   - Job system implemented

2. **Begin Phase 2** (Gameplay Refinements)
   - Economy modernization started
   - Military improvements planned
   - UI modernization design complete

---

## Part 8: Resources & References

### Internal Documentation
- `AGENTS.md` - Project knowledge base
- `ISSUES.md` - Comprehensive issue analysis
- `IMPLEMENTATION_PLAN.md` - Detailed implementation plan
- `FIXES_SUMMARY.md` - Fix summary and priorities
- `COMPREHENSIVE_IMPROVEMENT_PLAN.md` - This document

### External References
- **Victoria 2 Research:** DeepWiki analysis of mechanics and what made it great
- **Modern Strategy Games:** Industry research on AI, performance, UI/UX
- **Paradox Dev Diaries:** Stellaris/CK3/HOI4 development insights
- **Total War Technical:** Intel collaboration on performance and parallelism
- **Civ AI Mods:** Community-driven AI improvements

### Tools & Libraries
- **CMake:** Build system
- **Catch2:** Testing framework (already in repo)
- **ONNX Runtime:** ML inference (optional)
- **RenderDoc:** GPU profiling
- **Visual Studio Profiler / perf:** CPU profiling

### Community Resources
- **Discord:** Community engagement and feedback
- **GitHub Issues:** Bug tracking and feature requests
- **Modding Community:** Early adopters and testers

---

## Part 9: Risk Assessment & Mitigation

### Technical Risks

#### Serialization Changes Breaking Saves
**Risk:** Medium
**Mitigation:** Keep on-disk format unchanged, add version checks, extensive testing

#### Performance Optimizations Breaking Determinism
**Risk:** High
**Mitigation:** Test with deterministic replays, add checksums, preserve ordered execution

#### AI Changes Reducing Quality
**Risk:** Medium
**Mitigation:** A/B testing, automated tournaments, gradual rollout

#### UI Changes Reducing Usability
**Risk:** Low
**Mitigation:** User testing, A/B testing, feature flags

### Project Risks

#### Scope Creep
**Risk:** High
**Mitigation:** Strict phase-based approach, regular scope reviews, feature prioritization

#### Timeline Overruns
**Risk:** Medium
**Mitigation:** Buffer time in schedule, regular progress reviews, scope adjustment

#### Community Backlash
**Risk:** Low
**Mitigation:** Early community involvement, transparent communication, migration guides

#### Technical Debt Accumulation
**Risk:** Medium
**Mitigation:** Regular refactoring, code reviews, automated testing

### Mitigation Strategies

#### Early Testing
- Implement comprehensive test suite from day 1
- Add CI with multiple platforms and configurations
- Regular performance regression testing

#### Community Involvement
- Early beta program
- Regular community updates
- Feedback incorporation

#### Documentation
- Comprehensive user documentation
- Developer documentation for modders
- Migration guides from original

---

## Part 10: Call to Action

### For You (The User)
1. **Review this plan** and provide feedback on priorities
2. **Choose starting point** (recommend Phase 0: Foundation & Safety)
3. **Set timeline expectations** (16 weeks for full transformation)
4. **Engage community** for beta testing and feedback

### For Me (The AI)
1. **Execute Phase 0 immediately** (critical fixes)
2. **Begin profiling** and optimization work
3. **Coordinate with community** for beta program
4. **Track progress** and adjust plan as needed

### For the Community
1. **Beta testing** when ready (Week 4-5)
2. **Modding development** during Phase 4
3. **Feedback and suggestions** throughout
4. **Documentation contributions**

---

## Conclusion

This comprehensive plan transforms Project-Alice from a faithful Victoria 2 retro clone into a **modern, high-performance strategy game engine** that preserves the depth and emergent gameplay that made Victoria 2 great, while incorporating cutting-edge advancements across AI, simulation, UI/UX, and performance.

**Key Success Factors:**
1. **Foundation first:** Fix critical bugs and security issues immediately
2. **Measure everything:** Profile before and after every change
3. **Community involvement:** Early beta testing and feedback
4. **Incremental delivery:** Regular releases with visible improvements
5. **Quality focus:** Comprehensive testing and documentation

**Expected Outcomes:**
- **Performance:** 2-5x improvement in late-game performance
- **AI:** More strategic, varied, and human-like behavior
- **UI/UX:** Dramatically improved discoverability and usability
- **Multiplayer:** Robust, desync-free gameplay
- **Modding:** Rich ecosystem of community content

**Timeline:** 16 weeks for full transformation, with visible improvements every week.

**Next Action:** Start Phase 0 immediately (serialization safety, CI security, test execution).

---

**Let's build something amazing together.**

<promise>DONE</promise>
