# Phase 4 Execution: UI/UX Modernization

**Generated:** 2026-01-16T00:00:00Z
**Commit:** a90dd964a
**Branch:** main
**Phase:** 4 of 16-week improvement plan

---

## Phase 4 Overview

**Goal:** Modernize UI/UX with context-sensitive interfaces, dynamic tooltips, accessibility features, and performance optimization

**Duration:** 4 weeks (Weeks 9-12)

**Priority:** HIGH

**Success Criteria:**
- UI is intuitive and discoverable for new players
- Tooltips provide relevant information without overwhelming
- Accessibility features support diverse player needs
- UI performance is smooth even in late-game
- Visual feedback is clear and immediate
- Players can customize UI to their preferences

---

## Week 9: Tooltip & Information System

### Objective
Reduce information overload with smart, cached tooltips and pinned information panels

### Step 1: Tooltip Caching System

#### Cache Structure
**File:** `src/gui/ui_state.cpp`
**Add:**
```cpp
// Tooltip cache key
struct tooltip_cache_key {
    tooltip_type type;
    dcon::object_id object;  // Province, nation, factory, etc.
    uint32_t data_version;   // Changes when underlying data changes
    uint32_t player_context; // Player nation, current date, etc.
    
    bool operator==(const tooltip_cache_key& other) const {
        return type == other.type &&
               object == other.object &&
               data_version == other.data_version &&
               player_context == other.player_context;
    }
};

// Tooltip cache entry
struct tooltip_cache_entry {
    std::vector<ui_element_base*> cached_elements;  // Pre-built UI elements
    std::chrono::steady_clock::time_point timestamp;
    size_t access_count;
    
    bool is_stale(sys::state& state) const {
        // Cache invalidates after 5 seconds or 100 accesses
        auto now = std::chrono::steady_clock::now();
        return std::chrono::duration_cast<std::chrono::seconds>(now - timestamp).count() > 5 ||
               access_count > 100;
    }
};

// Tooltip cache
class tooltip_cache {
public:
    // Get cached tooltip or build new one
    std::vector<ui_element_base*> get_or_build(
        sys::state& state,
        const tooltip_cache_key& key,
        std::function<std::vector<ui_element_base*>()> builder
    ) {
        auto it = cache.find(key);
        
        if (it != cache.end() && !it->second.is_stale(state)) {
            // Cache hit
            it->second.access_count++;
            it->second.timestamp = std::chrono::steady_clock::now();
            return it->second.cached_elements;
        }
        
        // Cache miss - build new tooltip
        auto elements = builder();
        
        // Store in cache
        tooltip_cache_entry entry;
        entry.cached_elements = elements;
        entry.timestamp = std::chrono::steady_clock::now();
        entry.access_count = 1;
        
        cache[key] = entry;
        
        return elements;
    }
    
    // Invalidate cache for specific object
    void invalidate_object(dcon::object_id object) {
        for (auto it = cache.begin(); it != cache.end();) {
            if (it->first.object == object) {
                it = cache.erase(it);
            } else {
                ++it;
            }
        }
    }
    
    // Clear entire cache
    void clear() {
        cache.clear();
    }
    
private:
    std::unordered_map<tooltip_cache_key, tooltip_cache_entry, 
        std::hash<tooltip_cache_key>> cache;
};

// Hash function for tooltip_cache_key
namespace std {
    template<>
    struct hash<tooltip_cache_key> {
        size_t operator()(const tooltip_cache_key& key) const {
            size_t h1 = std::hash<tooltip_type>{}(key.type);
            size_t h2 = std::hash<dcon::object_id>{}(key.object);
            size_t h3 = std::hash<uint32_t>{}(key.data_version);
            size_t h4 = std::hash<uint32_t>{}(key.player_context);
            
            return h1 ^ (h2 << 1) ^ (h3 << 2) ^ (h4 << 3);
        }
    };
}
```

#### Cache-Aware Tooltip Generation
**File:** `src/gui/gui_tooltips.cpp`
**Modify:**
```cpp
// Current code (simplified):
void show_tooltip(sys::state& state, dcon::province_id province) {
    auto tooltip = create_province_tooltip(state, province);
    state.ui_state.tooltip = tooltip;
}

// Refactored:
void show_tooltip(sys::state& state, dcon::province_id province) {
    // Build cache key
    tooltip_cache_key key;
    key.type = tooltip_type::province;
    key.object = province;
    key.data_version = state.province_data_version[province];
    key.player_context = state.local_player_nation.value;
    
    // Get or build from cache
    auto elements = state.ui_state.tooltip_cache.get_or_build(
        state,
        key,
        [&]() {
            return create_province_tooltip(state, province);
        }
    );
    
    // Set tooltip
    state.ui_state.tooltip_elements = elements;
}

// Create province tooltip (cacheable)
std::vector<ui_element_base*> create_province_tooltip(sys::state& state, dcon::province_id province) {
    std::vector<ui_element_base*> elements;
    
    // Create text elements
    auto name_text = std::make_unique<text_element>();
    name_text->set_text(get_province_name(province));
    name_text->set_font_size(14);
    elements.push_back(name_text.release());
    
    // Create owner info
    auto owner = province.get_owner();
    if (owner) {
        auto owner_text = std::make_unique<text_element>();
        owner_text->set_text("Owner: " + get_nation_name(owner));
        owner_text->set_position({0, 20});
        elements.push_back(owner_text.release());
    }
    
    // Create resource info
    auto resources = get_province_resources(province);
    if (!resources.empty()) {
        auto resource_text = std::make_unique<text_element>();
        resource_text->set_text("Resources: " + format_resources(resources));
        resource_text->set_position({0, 40});
        elements.push_back(resource_text.release());
    }
    
    // Create infrastructure info
    auto infra = get_province_infrastructure(province);
    auto infra_text = std::make_unique<text_element>();
    infra_text->set_text("Infrastructure: " + format_infrastructure(infra));
    infra_text->set_position({0, 60});
    elements.push_back(infra_text.release());
    
    return elements;
}
```

### Step 2: Pinned Info Panel

#### Pinned Panel UI
**File:** `src/gui/gui_pinned_panel.cpp`
**Add:**
```cpp
class pinned_panel : public window_element_base {
public:
    void on_create(sys::state& state) override {
        window_element_base::on_create(state);
        
        // Header
        auto header = std::make_unique<text_element>();
        header->set_text("Pinned Information");
        header->set_font_size(16);
        header->set_position({10, 10});
        add_child(std::move(header));
        
        // Content area
        auto content = std::make_unique<pinned_content>();
        content->set_position({10, 30});
        content->set_size({380, 200});
        add_child(std::move(content));
        
        // Close button
        auto close_button = std::make_unique<button_element>();
        close_button->set_position({300, 10});
        close_button->set_size({80, 20});
        close_button->set_text("Close");
        close_button->on_click = [&]() {
            close();
        };
        add_child(std::move(close_button));
        
        // Pin button (for tooltip)
        auto pin_button = std::make_unique<button_element>();
        pin_button->set_position({10, 240});
        pin_button->set_size({100, 30});
        pin_button->set_text("Pin Current");
        pin_button->on_click = [&]() {
            pin_current_tooltip(state);
        };
        add_child(std::move(pin_button));
    }
    
    void pin_current_tooltip(sys::state& state) {
        if (state.ui_state.tooltip_elements.empty()) return;
        
        // Clone tooltip elements for pinned panel
        auto pinned_elements = clone_tooltip_elements(state.ui_state.tooltip_elements);
        
        // Add to pinned content
        auto content = get_child_by_index(1); // Content area
        content->clear_children();
        
        for (auto element : pinned_elements) {
            content->add_child(std::unique_ptr<ui_element_base>(element));
        }
        
        // Show panel
        show();
    }
    
    std::vector<ui_element_base*> clone_tooltip_elements(
        const std::vector<ui_element_base*>& elements
    ) {
        std::vector<ui_element_base*> clones;
        
        for (auto element : elements) {
            // Clone element (simplified - in reality would need proper cloning)
            auto clone = element->clone();
            clones.push_back(clone);
        }
        
        return clones;
    }
};

class pinned_content : public panel_base {
public:
    void clear_children() {
        children.clear();
    }
};
```

#### Pinned Panel Trigger
**File:** `src/gui/gui_element_base.cpp`
**Modify:**
```cpp
// Add to element_base
class element_base {
public:
    // ... existing code ...
    
    // Add pin button on tooltip
    virtual void on_tooltip_hover(sys::state& state) override {
        // Show tooltip
        show_tooltip(state);
        
        // Add pin button to tooltip
        if (state.ui_state.tooltip_elements.empty()) return;
        
        auto pin_button = std::make_unique<button_element>();
        pin_button->set_text("📌 Pin");
        pin_button->set_position({0, -25});
        pin_button->set_size({40, 20});
        pin_button->on_click = [&]() {
            state.ui_state.pinned_panel->pin_current_tooltip(state);
        };
        
        // Add to tooltip container
        state.ui_state.tooltip_container->add_child(std::move(pin_button));
    }
    
    // Long-press to pin (for touch/mobile)
    virtual void on_long_press(sys::state& state, int x, int y) override {
        show_tooltip(state);
        state.ui_state.pinned_panel->pin_current_tooltip(state);
    }
};
```

### Step 3: Summary vs Full Tooltips

#### Two-Level Tooltip System
**File:** `src/gui/gui_tooltips.cpp`
**Add:**
```cpp
// Tooltip levels
enum class tooltip_level {
    summary,    // Quick overview (shown on hover)
    detailed,   // Full information (shown on click/long-press)
    pinned      // Pinned in panel (user action)
};

// Summary tooltip generator
std::vector<ui_element_base*> create_summary_tooltip(
    sys::state& state, 
    dcon::province_id province
) {
    std::vector<ui_element_base*> elements;
    
    // Only essential information
    auto name_text = std::make_unique<text_element>();
    name_text->set_text(get_province_name(province));
    name_text->set_font_size(12);
    elements.push_back(name_text.release());
    
    auto owner = province.get_owner();
    if (owner) {
        auto owner_text = std::make_unique<text_element>();
        owner_text->set_text("Owner: " + get_nation_name(owner));
        owner_text->set_position({0, 15});
        owner_text->set_font_size(10);
        elements.push_back(owner_text.release());
    }
    
    return elements;
}

// Detailed tooltip generator
std::vector<ui_element_base*> create_detailed_tooltip(
    sys::state& state, 
    dcon::province_id province
) {
    std::vector<ui_element_base*> elements;
    
    // Full information
    auto name_text = std::make_unique<text_element>();
    name_text->set_text(get_province_name(province));
    name_text->set_font_size(14);
    elements.push_back(name_text.release());
    
    // Owner
    auto owner = province.get_owner();
    if (owner) {
        auto owner_text = std::make_unique<text_element>();
        owner_text->set_text("Owner: " + get_nation_name(owner));
        owner_text->set_position({0, 20});
        elements.push_back(owner_text.release());
    }
    
    // Resources
    auto resources = get_province_resources(province);
    if (!resources.empty()) {
        auto resource_text = std::make_unique<text_element>();
        resource_text->set_text("Resources: " + format_resources(resources));
        resource_text->set_position({0, 40});
        elements.push_back(resource_text.release());
    }
    
    // Infrastructure
    auto infra = get_province_infrastructure(province);
    auto infra_text = std::make_unique<text_element>();
    infra_text->set_text("Infrastructure: " + format_infrastructure(infra));
    infra_text->set_position({0, 60});
    elements.push_back(infra_text.release());
    
    // Population
    auto population = get_province_population(province);
    auto pop_text = std::make_unique<text_element>();
    pop_text->set_text("Population: " + format_population(population));
    pop_text->set_position({0, 80});
    elements.push_back(pop_text.release());
    
    // Economy
    auto economy = get_province_economy(province);
    auto econ_text = std::make_unique<text_element>();
    econ_text->set_text("GDP: " + format_gdp(economy.gdp));
    econ_text->set_position({0, 100});
    elements.push_back(econ_text.release());
    
    return elements;
}

// Tooltip level selector
std::vector<ui_element_base*> create_tooltip(
    sys::state& state,
    dcon::province_id province,
    tooltip_level level
) {
    switch (level) {
        case tooltip_level::summary:
            return create_summary_tooltip(state, province);
        case tooltip_level::detailed:
            return create_detailed_tooltip(state, province);
        default:
            return create_summary_tooltip(state, province);
    }
}
```

#### Click/Long-Press Behavior
**File:** `src/gui/gui_element_base.cpp`
**Modify:**
```cpp
// On click - show detailed tooltip
void element_base::on_click(sys::state& state, int x, int y) override {
    if (has_tooltip()) {
        // Show detailed tooltip
        auto elements = create_tooltip(state, get_tooltip_target(), tooltip_level::detailed);
        state.ui_state.tooltip_elements = elements;
        
        // Position tooltip near cursor
        state.ui_state.tooltip_position = {x + 10, y + 10};
    }
}

// On hover - show summary tooltip
void element_base::on_hover(sys::state& state, int x, int y) override {
    if (has_tooltip() && !state.ui_state.tooltip_visible) {
        // Show summary tooltip
        auto elements = create_tooltip(state, get_tooltip_target(), tooltip_level::summary);
        state.ui_state.tooltip_elements = elements;
        
        // Position tooltip near cursor
        state.ui_state.tooltip_position = {x + 10, y + 10};
        state.ui_state.tooltip_visible = true;
    }
}

// On mouse leave - hide tooltip
void element_base::on_mouse_leave(sys::state& state) override {
    if (state.ui_state.tooltip_visible) {
        state.ui_state.tooltip_visible = false;
        state.ui_state.tooltip_elements.clear();
    }
}
```

### Step 4: Async Tooltip Computation

#### Background Worker for Heavy Tooltips
**File:** `src/gui/gui_tooltips_async.cpp`
**Add:**
```cpp
class tooltip_async_worker {
public:
    tooltip_async_worker(size_t num_threads = 2) {
        for (size_t i = 0; i < num_threads; i++) {
            workers.emplace_back([this]() {
                worker_thread();
            });
        }
    }
    
    ~tooltip_async_worker() {
        {
            std::lock_guard<std::mutex> lock(queue_mutex);
            shutdown = true;
        }
        cv.notify_all();
        
        for (auto& worker : workers) {
            if (worker.joinable()) {
                worker.join();
            }
        }
    }
    
    // Request async tooltip computation
    void request_tooltip(
        tooltip_cache_key key,
        std::function<std::vector<ui_element_base*>()> builder,
        std::function<void(std::vector<ui_element_base*>)> on_complete
    ) {
        {
            std::lock_guard<std::mutex> lock(queue_mutex);
            queue.push({key, builder, on_complete});
        }
        cv.notify_one();
    }
    
private:
    struct tooltip_request {
        tooltip_cache_key key;
        std::function<std::vector<ui_element_base*>()> builder;
        std::function<void(std::vector<ui_element_base*>)> on_complete;
    };
    
    std::vector<std::thread> workers;
    std::queue<tooltip_request> queue;
    std::mutex queue_mutex;
    std::condition_variable cv;
    bool shutdown = false;
    
    void worker_thread() {
        while (true) {
            tooltip_request request;
            
            {
                std::unique_lock<std::mutex> lock(queue_mutex);
                cv.wait(lock, [this]() { return !queue.empty() || shutdown; });
                
                if (shutdown && queue.empty()) {
                    return;
                }
                
                request = queue.front();
                queue.pop();
            }
            
            // Compute tooltip
            auto elements = request.builder();
            
            // Call completion callback
            request.on_complete(elements);
        }
    }
};

// Usage in UI
class heavy_tooltip_element : public element_base {
public:
    void on_hover(sys::state& state, int x, int y) override {
        // Show loading indicator
        show_loading_indicator();
        
        // Request async computation
        state.tooltip_worker.request_tooltip(
            get_cache_key(),
            [&]() {
                // Heavy computation here
                return create_heavy_tooltip(state, get_target());
            },
            [&](std::vector<ui_element_base*> elements) {
                // Update UI with computed tooltip
                state.ui_state.tooltip_elements = elements;
                hide_loading_indicator();
            }
        );
    }
};
```

### Step 5: Context-Sensitive Tooltips

#### Smart Tooltip Content
**File:** `src/gui/gui_context_tooltips.cpp`
**Add:**
```cpp
class context_aware_tooltip {
public:
    std::vector<ui_element_base*> create_tooltip(
        sys::state& state,
        dcon::province_id province
    ) {
        std::vector<ui_element_base*> elements;
        
        // Determine context
        auto context = determine_context(state);
        
        // Add context-relevant information
        switch (context) {
            case tooltip_context::economic:
                add_economic_info(state, province, elements);
                break;
            case tooltip_context::military:
                add_military_info(state, province, elements);
                break;
            case tooltip_context::diplomatic:
                add_diplomatic_info(state, province, elements);
                break;
            case tooltip_context::general:
                add_general_info(state, province, elements);
                break;
        }
        
        return elements;
    }
    
    enum class tooltip_context {
        economic,
        military,
        diplomatic,
        general
    };
    
    tooltip_context determine_context(sys::state& state) {
        // Check what UI panel is open
        if (state.ui_state.active_panel == panel_type::economy) {
            return tooltip_context::economic;
        } else if (state.ui_state.active_panel == panel_type::military) {
            return tooltip_context::military;
        } else if (state.ui_state.active_panel == panel_type::diplomacy) {
            return tooltip_context::diplomatic;
        } else {
            return tooltip_context::general;
        }
    }
    
    void add_economic_info(sys::state& state, dcon::province_id province, 
                           std::vector<ui_element_base*>& elements) {
        // Add economic-focused information
        auto gdp = get_province_gdp(province);
        auto gdp_text = std::make_unique<text_element>();
        gdp_text->set_text("GDP: " + format_gdp(gdp));
        gdp_text->set_position({0, 20});
        elements.push_back(gdp_text.release());
        
        auto tax = get_province_tax_revenue(province);
        auto tax_text = std::make_unique<text_element>();
        tax_text->set_text("Tax Revenue: " + format_money(tax));
        tax_text->set_position({0, 40});
        elements.push_back(tax_text.release());
    }
    
    void add_military_info(sys::state& state, dcon::province_id province,
                           std::vector<ui_element_base*>& elements) {
        // Add military-focused information
        auto garrison = get_province_garrison(province);
        auto garrison_text = std::make_unique<text_element>();
        garrison_text->set_text("Garrison: " + format_units(garrison));
        garrison_text->set_position({0, 20});
        elements.push_back(garrison_text.release());
        
        auto supply = get_province_supply_limit(province);
        auto supply_text = std::make_unique<text_element>();
        supply_text->set_text("Supply Limit: " + format_quantity(supply));
        supply_text->set_position({0, 40});
        elements.push_back(supply_text.release());
    }
    
    void add_diplomatic_info(sys::state& state, dcon::province_id province,
                            std::vector<ui_element_base*>& elements) {
        // Add diplomatic-focused information
        auto owner = province.get_owner();
        if (owner) {
            auto relations = get_relations(state.local_player_nation, owner);
            auto rel_text = std::make_unique<text_element>();
            rel_text->set_text("Relations: " + format_relations(relations));
            rel_text->set_position({0, 20});
            elements.push_back(rel_text.release());
        }
    }
    
    void add_general_info(sys::state& state, dcon::province_id province,
                         std::vector<ui_element_base*>& elements) {
        // Add general information
        auto name_text = std::make_unique<text_element>();
        name_text->set_text(get_province_name(province));
        name_text->set_position({0, 20});
        elements.push_back(name_text.release());
    }
};
```

### Step 6: Success Criteria for Tooltip System

- [ ] Tooltip caching reduces rebuilds by 50%+
- [ ] Pinned panel allows users to keep important information visible
- [ ] Summary vs full tooltips reduce information overload
- [ ] Async computation prevents UI blocking on heavy tooltips
- [ ] Context-sensitive tooltips show relevant information
- [ ] Players can access detailed information on demand
- [ ] Tooltip performance is smooth even with many hovers

---

## Week 10: Focus Management & Accessibility

### Objective
Add keyboard navigation, accessibility features, and controller support

### Step 1: Focus Manager

#### Central Focus API
**File:** `src/gui/ui_focus.cpp`
**Add:**
```cpp
class focus_manager {
public:
    focus_manager() : current_focus(nullptr) {}
    
    // Set focus to element
    void set_focus(element_base* element) {
        if (current_focus) {
            current_focus->on_lose_focus();
        }
        
        current_focus = element;
        
        if (current_focus) {
            current_focus->on_get_focus();
        }
    }
    
    // Clear focus
    void clear_focus() {
        if (current_focus) {
            current_focus->on_lose_focus();
            current_focus = nullptr;
        }
    }
    
    // Get current focus
    element_base* get_current_focus() const {
        return current_focus;
    }
    
    // Tab navigation
    void tab_forward(sys::state& state) {
        if (!current_focus) {
            // Find first focusable element
            auto first = find_first_focusable(state.ui_state.root);
            if (first) {
                set_focus(first);
            }
            return;
        }
        
        // Find next focusable element
        auto next = find_next_focusable(state.ui_state.root, current_focus);
        if (next) {
            set_focus(next);
        }
    }
    
    void tab_backward(sys::state& state) {
        if (!current_focus) {
            // Find last focusable element
            auto last = find_last_focusable(state.ui_state.root);
            if (last) {
                set_focus(last);
            }
            return;
        }
        
        // Find previous focusable element
        auto prev = find_prev_focusable(state.ui_state.root, current_focus);
        if (prev) {
            set_focus(prev);
        }
    }
    
    // Arrow key navigation (for grids/lists)
    void navigate_up() {
        if (current_focus && current_focus->parent) {
            auto sibling = current_focus->parent->get_child_above(current_focus);
            if (sibling) {
                set_focus(sibling);
            }
        }
    }
    
    void navigate_down() {
        if (current_focus && current_focus->parent) {
            auto sibling = current_focus->parent->get_child_below(current_focus);
            if (sibling) {
                set_focus(sibling);
            }
        }
    }
    
    void navigate_left() {
        if (current_focus && current_focus->parent) {
            auto sibling = current_focus->parent->get_child_left(current_focus);
            if (sibling) {
                set_focus(sibling);
            }
        }
    }
    
    void navigate_right() {
        if (current_focus && current_focus->parent) {
            auto sibling = current_focus->parent->get_child_right(current_focus);
            if (sibling) {
                set_focus(sibling);
            }
        }
    }
    
    // Activate focused element
    void activate_focus() {
        if (current_focus) {
            current_focus->on_activate();
        }
    }
    
private:
    element_base* current_focus;
    
    element_base* find_first_focusable(element_base* root) {
        if (!root) return nullptr;
        
        if (root->is_focusable()) {
            return root;
        }
        
        for (auto& child : root->children) {
            auto found = find_first_focusable(child.get());
            if (found) return found;
        }
        
        return nullptr;
    }
    
    element_base* find_last_focusable(element_base* root) {
        if (!root) return nullptr;
        
        element_base* last = nullptr;
        
        for (auto& child : root->children) {
            auto found = find_last_focusable(child.get());
            if (found) last = found;
        }
        
        if (last) return last;
        if (root->is_focusable()) return root;
        
        return nullptr;
    }
    
    element_base* find_next_focusable(element_base* root, element_base* current) {
        // Simplified - in reality would need proper traversal
        if (!root) return nullptr;
        
        bool found_current = false;
        
        for (auto& child : root->children) {
            if (child.get() == current) {
                found_current = true;
                continue;
            }
            
            if (found_current && child->is_focusable()) {
                return child.get();
            }
            
            auto found = find_next_focusable(child.get(), current);
            if (found) return found;
        }
        
        return nullptr;
    }
    
    element_base* find_prev_focusable(element_base* root, element_base* current) {
        // Simplified - in reality would need proper traversal
        if (!root) return nullptr;
        
        element_base* prev = nullptr;
        
        for (auto& child : root->children) {
            if (child.get() == current) {
                return prev;
            }
            
            if (child->is_focusable()) {
                prev = child.get();
            }
            
            auto found = find_prev_focusable(child.get(), current);
            if (found) prev = found;
        }
        
        return prev;
    }
};
```

#### Focusable Elements
**File:** `src/gui/gui_element_base.cpp`
**Modify:**
```cpp
class element_base {
public:
    // ... existing code ...
    
    // Focus-related methods
    virtual bool is_focusable() const {
        return false; // Override in focusable elements
    }
    
    virtual void on_get_focus() {
        // Visual feedback for focus
        set_focused(true);
        invalidate();
    }
    
    virtual void on_lose_focus() {
        // Remove visual feedback
        set_focused(false);
        invalidate();
    }
    
    virtual void on_activate() {
        // Default activation behavior (click)
        on_click(0, 0);
    }
    
    void set_focused(bool focused) {
        is_focused = focused;
    }
    
    bool get_focused() const {
        return is_focused;
    }
    
private:
    bool is_focused = false;
};

// Focusable button
class focusable_button : public button_element {
public:
    bool is_focusable() const override {
        return true;
    }
    
    void on_get_focus() override {
        button_element::on_get_focus();
        // Show focus rectangle
        show_focus_rectangle();
    }
    
    void on_lose_focus() override {
        button_element::on_lose_focus();
        hide_focus_rectangle();
    }
    
    void on_activate() override {
        // Trigger button click
        on_click(0, 0);
    }
};
```

### Step 2: Visual Focus Indicators

#### Focus Rectangle
**File:** `src/gui/gui_focus.cpp`
**Add:**
```cpp
class focus_rectangle : public element_base {
public:
    void render(sys::state& state) override {
        if (!parent || !parent->get_focused()) return;
        
        // Draw focus rectangle around parent
        auto rect = parent->get_screen_rect();
        
        // Draw border
        draw_rectangle(rect.x - 2, rect.y - 2, 
                      rect.width + 4, rect.height + 4,
                      {1.0f, 1.0f, 0.0f}, // Yellow
                      2.0f); // Border width
        
        // Draw dashed pattern
        draw_dashed_rectangle(rect.x - 4, rect.y - 4,
                             rect.width + 8, rect.height + 8,
                             {1.0f, 1.0f, 0.0f},
                             1.0f,
                             4.0f, 4.0f); // Dash pattern
    }
};

// Add to element_base
void element_base::show_focus_rectangle() {
    if (!focus_rect) {
        focus_rect = std::make_unique<focus_rectangle>();
        focus_rect->set_parent(this);
        add_child(std::move(focus_rect));
    }
}

void element_base::hide_focus_rectangle() {
    // Remove focus rectangle
    for (auto it = children.begin(); it != children.end(); ++it) {
        if (dynamic_cast<focus_rectangle*>(it->get())) {
            children.erase(it);
            break;
        }
    }
}
```

### Step 3: Accessibility Features

#### Theme Manager
**File:** `src/gui/theme_manager.cpp`
**Add:**
```cpp
class theme_manager {
public:
    struct theme {
        std::string name;
        
        // Colors
        color background;
        color foreground;
        color accent;
        color error;
        color warning;
        color success;
        
        // Color-blind modes
        enum class color_blind_mode {
            none,
            protanopia,   // Red-blind
            deuteranopia, // Green-blind
            tritanopia,   // Blue-blind
            monochromatic
        };
        
        color_blind_mode cb_mode;
        
        // Font settings
        float font_size_multiplier; // 1.0 = normal
        bool bold_fonts;
        
        // High contrast
        bool high_contrast;
        
        // Apply color-blind correction
        color apply_correction(color c) const {
            if (cb_mode == color_blind_mode::none) return c;
            
            // Simplified color-blind simulation
            switch (cb_mode) {
                case color_blind_mode::protanopia:
                    // Reduce red component
                    return {c.g * 0.5f + c.b * 0.5f, c.g, c.b};
                case color_blind_mode::deuteranopia:
                    // Reduce green component
                    return {c.r, c.r * 0.5f + c.b * 0.5f, c.b};
                case color_blind_mode::tritanopia:
                    // Reduce blue component
                    return {c.r, c.g, c.r * 0.5f + c.g * 0.5f};
                case color_blind_mode::monochromatic:
                    // Grayscale
                    float gray = c.r * 0.299f + c.g * 0.587f + c.b * 0.114f;
                    return {gray, gray, gray};
            }
            return c;
        }
    };
    
    std::vector<theme> themes;
    size_t current_theme_index = 0;
    
    theme get_current_theme() const {
        return themes[current_theme_index];
    }
    
    void load_themes() {
        // Load from JSON
        auto json = load_json_file("data/ui/themes.json");
        
        for (auto& theme_json : json["themes"]) {
            theme t;
            t.name = theme_json["name"];
            
            // Parse colors
            t.background = parse_color(theme_json["background"]);
            t.foreground = parse_color(theme_json["foreground"]);
            t.accent = parse_color(theme_json["accent"]);
            t.error = parse_color(theme_json["error"]);
            t.warning = parse_color(theme_json["warning"]);
            t.success = parse_color(theme_json["success"]);
            
            // Parse color-blind mode
            std::string cb_mode = theme_json["color_blind_mode"];
            if (cb_mode == "protanopia") t.cb_mode = theme::color_blind_mode::protanopia;
            else if (cb_mode == "deuteranopia") t.cb_mode = theme::color_blind_mode::deuteranopia;
            else if (cb_mode == "tritanopia") t.cb_mode = theme::color_blind_mode::tritanopia;
            else if (cb_mode == "monochromatic") t.cb_mode = theme::color_blind_mode::monochromatic;
            else t.cb_mode = theme::color_blind_mode::none;
            
            // Parse font settings
            t.font_size_multiplier = theme_json["font_size_multiplier"];
            t.bold_fonts = theme_json["bold_fonts"];
            
            // Parse high contrast
            t.high_contrast = theme_json["high_contrast"];
            
            themes.push_back(t);
        }
    }
    
    void apply_theme(sys::state& state, const theme& t) {
        // Apply to all UI elements
        apply_theme_recursive(state.ui_state.root, t);
    }
    
    void apply_theme_recursive(element_base* element, const theme& t) {
        if (!element) return;
        
        // Apply colors
        element->set_background_color(t.apply_correction(t.background));
        element->set_foreground_color(t.apply_correction(t.foreground));
        
        // Apply font settings
        element->set_font_size(element->get_font_size() * t.font_size_multiplier);
        if (t.bold_fonts) {
            element->set_font_weight(font_weight::bold);
        }
        
        // Apply high contrast
        if (t.high_contrast) {
            element->set_contrast(2.0f); // Double contrast
        }
        
        // Recursively apply to children
        for (auto& child : element->children) {
            apply_theme_recursive(child.get(), t);
        }
    }
};
```

#### Theme Selector UI
**File:** `src/gui/gui_theme_selector.cpp`
**Add:**
```cpp
class theme_selector_window : public window_element_base {
public:
    void on_create(sys::state& state) override {
        window_element_base::on_create(state);
        
        // Theme list
        auto theme_label = std::make_unique<text_element>();
        theme_label->set_text("Select Theme:");
        theme_label->set_position({10, 30});
        add_child(std::move(theme_label));
        
        auto theme_list = std::make_unique<theme_listbox>();
        theme_list->set_position({10, 50});
        theme_list->set_size({300, 200});
        add_child(std::move(theme_list));
        
        // Color-blind mode selector
        auto cb_label = std::make_unique<text_element>();
        cb_label->set_text("Color-Blind Mode:");
        cb_label->set_position({320, 30});
        add_child(std::move(cb_label));
        
        auto cb_selector = std::make_unique<color_blind_selector>();
        cb_selector->set_position({320, 50});
        cb_selector->set_size({200, 30});
        add_child(std::move(cb_selector));
        
        // Font size slider
        auto font_label = std::make_unique<text_element>();
        font_label->set_text("Font Size:");
        font_label->set_position({10, 260});
        add_child(std::move(font_label));
        
        auto font_slider = std::make_unique<slider_element>();
        font_slider->set_position({10, 280});
        font_slider->set_range(0.5f, 2.0f);
        font_slider->set_value(state.ui_state.font_size_multiplier);
        font_slider->on_change = [&](float value) {
            state.ui_state.font_size_multiplier = value;
            state.ui_state.theme_manager.apply_theme(state, 
                state.ui_state.theme_manager.get_current_theme());
        };
        add_child(std::move(font_slider));
        
        // High contrast checkbox
        auto contrast_checkbox = std::make_unique<checkbox_element>();
        contrast_checkbox->set_position({10, 320});
        contrast_checkbox->set_text("High Contrast");
        contrast_checkbox->set_checked(state.ui_state.high_contrast);
        contrast_checkbox->on_change = [&](bool checked) {
            state.ui_state.high_contrast = checked;
            state.ui_state.theme_manager.apply_theme(state, 
                state.ui_state.theme_manager.get_current_theme());
        };
        add_child(std::move(contrast_checkbox));
        
        // Apply button
        auto apply_button = std::make_unique<button_element>();
        apply_button->set_position({10, 360});
        apply_button->set_size({100, 30});
        apply_button->set_text("Apply");
        apply_button->on_click = [&]() {
            apply_theme(state);
        };
        add_child(std::move(apply_button));
    }
    
    void apply_theme(sys::state& state) {
        auto theme = state.ui_state.theme_manager.get_current_theme();
        state.ui_state.theme_manager.apply_theme(state, theme);
    }
};
```

#### Scalable Fonts
**File:** `src/gui/gui_font.cpp`
**Add:**
```cpp
class font_scaler {
public:
    static float scale_font_size(float base_size, float multiplier) {
        return base_size * multiplier;
    }
    
    static float scale_for_accessibility(float base_size, bool high_contrast) {
        if (high_contrast) {
            return base_size * 1.2f; // Slightly larger for high contrast
        }
        return base_size;
    }
};

// Apply to text elements
class scalable_text_element : public text_element {
public:
    void render(sys::state& state) override {
        // Calculate scaled font size
        float scaled_size = font_scaler::scale_font_size(
            base_font_size, 
            state.ui_state.font_size_multiplier
        );
        
        scaled_size = font_scaler::scale_for_accessibility(
            scaled_size,
            state.ui_state.high_contrast
        );
        
        // Render with scaled size
        draw_text(position.x, position.y, text, scaled_size, color);
    }
    
private:
    float base_font_size = 12.0f;
};
```

### Step 4: Keyboard Navigation

#### Global Keyboard Handler
**File:** `src/gui/keyboard_handler.cpp`
**Add:**
```cpp
class keyboard_handler {
public:
    void handle_key(sys::state& state, key_code key, key_modifiers mods) {
        // Check if UI has focus
        if (state.ui_state.focus_manager.get_current_focus()) {
            handle_ui_key(state, key, mods);
        } else {
            handle_global_key(state, key, mods);
        }
    }
    
    void handle_ui_key(sys::state& state, key_code key, key_modifiers mods) {
        auto& focus_manager = state.ui_state.focus_manager;
        
        switch (key) {
            case key_code::Tab:
                if (mods & key_modifiers::Shift) {
                    focus_manager.tab_backward(state);
                } else {
                    focus_manager.tab_forward(state);
                }
                break;
                
            case key_code::Enter:
            case key_code::Space:
                focus_manager.activate_focus();
                break;
                
            case key_code::Escape:
                focus_manager.clear_focus();
                break;
                
            case key_code::Up:
                focus_manager.navigate_up();
                break;
                
            case key_code::Down:
                focus_manager.navigate_down();
                break;
                
            case key_code::Left:
                focus_manager.navigate_left();
                break;
                
            case key_code::Right:
                focus_manager.navigate_right();
                break;
                
            default:
                // Pass to focused element
                if (focus_manager.get_current_focus()) {
                    focus_manager.get_current_focus()->on_key_press(state, key, mods);
                }
                break;
        }
    }
    
    void handle_global_key(sys::state& state, key_code key, key_modifiers mods) {
        // Global shortcuts
        switch (key) {
            case key_code::F1:
                show_help(state);
                break;
                
            case key_code::F2:
                show_economy_panel(state);
                break;
                
            case key_code::F3:
                show_military_panel(state);
                break;
                
            case key_code::F4:
                show_diplomacy_panel(state);
                break;
                
            case key_code::F5:
                show_politics_panel(state);
                break;
                
            case key_code::F6:
                show_technology_panel(state);
                break;
                
            case key_code::F7:
                show_market_panel(state);
                break;
                
            case key_code::F8:
                show_settings(state);
                break;
                
            case key_code::Space:
                // Pause/unpause
                state.game_speed = (state.game_speed == 0) ? 1 : 0;
                break;
                
            case key_code::Plus:
            case key_code::NumpadPlus:
                // Speed up
                state.game_speed = std::min(state.game_speed + 1, 5);
                break;
                
            case key_code::Minus:
            case key_code::NumpadMinus:
                // Slow down
                state.game_speed = std::max(state.game_speed - 1, 0);
                break;
                
            case key_code::M:
                // Toggle map mode
                state.map_mode = (state.map_mode + 1) % num_map_modes;
                break;
                
            case key_code::S:
                // Quick save
                quick_save(state);
                break;
                
            case key_code::L:
                // Quick load
                quick_load(state);
                break;
                
            case key_code::P:
                // Pause
                state.game_speed = 0;
                break;
        }
    }
};
```

#### Keyboard Shortcuts UI
**File:** `src/gui/gui_keyboard_shortcuts.cpp`
**Add:**
```cpp
class keyboard_shortcuts_window : public window_element_base {
public:
    void on_create(sys::state& state) override {
        window_element_base::on_create(state);
        
        // Shortcuts list
        auto shortcuts_label = std::make_unique<text_element>();
        shortcuts_label->set_text("Keyboard Shortcuts:");
        shortcuts_label->set_position({10, 30});
        add_child(std::move(shortcuts_label));
        
        auto shortcuts_list = std::make_unique<shortcuts_listbox>();
        shortcuts_list->set_position({10, 50});
        shortcuts_list->set_size({600, 300});
        add_child(std::move(shortcuts_list));
        
        // Custom shortcuts
        auto custom_label = std::make_unique<text_element>();
        custom_label->set_text("Custom Shortcuts:");
        custom_label->set_position({10, 360});
        add_child(std::move(custom_label));
        
        auto custom_list = std::make_unique<custom_shortcuts_listbox>();
        custom_list->set_position({10, 380});
        custom_list->set_size({600, 100});
        add_child(std::move(custom_list));
        
        // Add custom shortcut button
        auto add_button = std::make_unique<button_element>();
        add_button->set_position({620, 380});
        add_button->set_size({80, 30});
        add_button->set_text("Add");
        add_button->on_click = [&]() {
            show_add_shortcut_dialog();
        };
        add_child(std::move(add_button));
    }
    
    void show_add_shortcut_dialog() {
        auto dialog = std::make_unique<add_shortcut_dialog>();
        dialog->set_position({200, 200});
        dialog->set_size({300, 200});
        // ... show modal ...
    }
};
```

### Step 5: Controller Support

#### Controller Input Handler
**File:** `src/gui/controller_handler.cpp`
**Add:**
```cpp
class controller_handler {
public:
    void handle_controller_input(sys::state& state, controller_input input) {
        switch (input.button) {
            case controller_button::A:
                // Activate / Select
                if (state.ui_state.focus_manager.get_current_focus()) {
                    state.ui_state.focus_manager.activate_focus();
                }
                break;
                
            case controller_button::B:
                // Cancel / Back
                state.ui_state.focus_manager.clear_focus();
                break;
                
            case controller_button::X:
                // Secondary action (e.g., pin tooltip)
                if (state.ui_state.tooltip_visible) {
                    state.ui_state.pinned_panel->pin_current_tooltip(state);
                }
                break;
                
            case controller_button::Y:
                // Context menu
                show_context_menu(state);
                break;
                
            case controller_button::DPadUp:
                state.ui_state.focus_manager.navigate_up();
                break;
                
            case controller_button::DPadDown:
                state.ui_state.focus_manager.navigate_down();
                break;
                
            case controller_button::DPadLeft:
                state.ui_state.focus_manager.navigate_left();
                break;
                
            case controller_button::DPadRight:
                state.ui_state.focus_manager.navigate_right();
                break;
                
            case controller_button::LeftTrigger:
                // Zoom out
                state.map_zoom = std::max(state.map_zoom - 0.1f, 0.1f);
                break;
                
            case controller_button::RightTrigger:
                // Zoom in
                state.map_zoom = std::min(state.map_zoom + 0.1f, 5.0f);
                break;
                
            case controller_button::Start:
                // Pause / Menu
                show_pause_menu(state);
                break;
                
            case controller_button::Back:
                // Toggle UI visibility
                state.ui_state.visible = !state.ui_state.visible;
                break;
        }
    }
    
    void handle_controller_axis(sys::state& state, controller_axis axis, float value) {
        switch (axis) {
            case controller_axis::LeftStickX:
                // Pan map horizontally
                if (std::abs(value) > 0.2f) {
                    state.map_pan_x += value * 10.0f;
                }
                break;
                
            case controller_axis::LeftStickY:
                // Pan map vertically
                if (std::abs(value) > 0.2f) {
                    state.map_pan_y += value * 10.0f;
                }
                break;
                
            case controller_axis::RightStickX:
                // Fine pan
                if (std::abs(value) > 0.1f) {
                    state.map_pan_x += value * 2.0f;
                }
                break;
                
            case controller_axis::RightStickY:
                // Fine pan
                if (std::abs(value) > 0.1f) {
                    state.map_pan_y += value * 2.0f;
                }
                break;
        }
    }
};
```

#### Controller UI Adaptations
**File:** `src/gui/gui_controller_adaptations.cpp`
**Add:**
```cpp
class controller_friendly_element : public element_base {
public:
    void render(sys::state& state) override {
        element_base::render(state);
        
        // Show controller hint if controller is active
        if (state.input_state.controller_connected) {
            draw_controller_hints();
        }
    }
    
    void draw_controller_hints() {
        // Draw button hints at bottom of element
        auto rect = get_screen_rect();
        
        // A button hint
        draw_text(rect.x + 5, rect.y + rect.height - 20, 
                 "A: Select", 10, {1.0f, 1.0f, 1.0f});
        
        // B button hint
        draw_text(rect.x + 60, rect.y + rect.height - 20,
                 "B: Cancel", 10, {1.0f, 1.0f, 1.0f});
    }
    
    // Make elements larger for controller
    virtual void on_controller_connect() {
        // Increase hitbox size
        set_hitbox_size({hitbox.width + 10, hitbox.height + 10});
        
        // Add visual padding
        set_padding({padding.left + 5, padding.top + 5, 
                    padding.right + 5, padding.bottom + 5});
    }
};
```

### Step 6: Screen Reader Support

#### Accessibility Tree
**File:** `src/gui/accessibility_tree.cpp`
**Add:**
```cpp
class accessibility_node {
public:
    enum class role {
        button,
        text,
        checkbox,
        slider,
        list,
        list_item,
        window,
        panel,
        label
    };
    
    role node_role;
    std::string name;  // Accessible name
    std::string description;  // Accessible description
    std::string value;  // Current value
    bool focusable;
    bool focused;
    
    std::vector<std::unique_ptr<accessibility_node>> children;
    
    // Generate accessibility tree from UI tree
    static std::unique_ptr<accessibility_node> build_tree(element_base* root) {
        if (!root) return nullptr;
        
        auto node = std::make_unique<accessibility_node>();
        
        // Determine role
        if (dynamic_cast<button_element*>(root)) {
            node->node_role = role::button;
        } else if (dynamic_cast<text_element*>(root)) {
            node->node_role = role::text;
        } else if (dynamic_cast<checkbox_element*>(root)) {
            node->node_role = role::checkbox;
        } else if (dynamic_cast<slider_element*>(root)) {
            node->node_role = role::slider;
        } else if (dynamic_cast<listbox_base*>(root)) {
            node->node_role = role::list;
        } else if (dynamic_cast<window_element_base*>(root)) {
            node->node_role = role::window;
        } else {
            node->node_role = role::panel;
        }
        
        // Get accessible name
        node->name = root->get_accessible_name();
        node->description = root->get_accessible_description();
        node->value = root->get_accessible_value();
        node->focusable = root->is_focusable();
        node->focused = root->get_focused();
        
        // Build children
        for (auto& child : root->children) {
            auto child_node = build_tree(child.get());
            if (child_node) {
                node->children.push_back(std::move(child_node));
            }
        }
        
        return node;
    }
    
    // Generate speech output for screen reader
    std::string to_speech_output() const {
        std::string output;
        
        // Role
        output += role_to_string(node_role) + ": ";
        
        // Name
        output += name;
        
        // Value (if applicable)
        if (!value.empty()) {
            output += ", value: " + value;
        }
        
        // State
        if (focused) {
            output += ", focused";
        }
        
        if (focusable) {
            output += ", focusable";
        }
        
        // Children (for navigation)
        if (!children.empty()) {
            output += ", " + std::to_string(children.size()) + " items";
        }
        
        return output;
    }
    
    std::string role_to_string(role r) const {
        switch (r) {
            case role::button: return "Button";
            case role::text: return "Text";
            case role::checkbox: return "Checkbox";
            case role::slider: return "Slider";
            case role::list: return "List";
            case role::list_item: return "List item";
            case role::window: return "Window";
            case role::panel: return "Panel";
            case role::label: return "Label";
            default: return "Unknown";
        }
    }
};
```

#### Screen Reader Integration
**File:** `src/gui/screen_reader.cpp`
**Add:**
```cpp
class screen_reader {
public:
    screen_reader() : enabled(false) {}
    
    void enable() {
        enabled = true;
        
        // Initialize screen reader API (platform-specific)
        #ifdef _WIN32
        initialize_windows_screen_reader();
        #elif defined(__linux__)
        initialize_linux_screen_reader();
        #endif
    }
    
    void disable() {
        enabled = false;
    }
    
    void announce(const std::string& text) {
        if (!enabled) return;
        
        #ifdef _WIN32
        windows_speak(text);
        #elif defined(__linux__)
        linux_speak(text);
        #endif
    }
    
    void update_focus(element_base* element) {
        if (!enabled || !element) return;
        
        // Build accessibility tree
        auto tree = accessibility_node::build_tree(state.ui_state.root);
        
        // Find focused node
        auto focused_node = find_focused_node(tree.get());
        
        if (focused_node) {
            // Announce focused element
            announce(focused_node->to_speech_output());
        }
    }
    
    void handle_navigation(element_base* from, element_base* to) {
        if (!enabled) return;
        
        // Announce navigation
        std::string announcement;
        
        if (from) {
            announcement += "Leaving " + from->get_accessible_name() + ". ";
        }
        
        if (to) {
            announcement += "Entering " + to->get_accessible_name();
            
            if (to->is_focusable()) {
                announcement += ", focusable";
            }
        }
        
        announce(announcement);
    }
    
private:
    bool enabled;
    sys::state* state;
    
    accessibility_node* find_focused_node(accessibility_node* node) {
        if (!node) return nullptr;
        
        if (node->focused) {
            return node;
        }
        
        for (auto& child : node->children) {
            auto found = find_focused_node(child.get());
            if (found) return found;
        }
        
        return nullptr;
    }
    
    #ifdef _WIN32
    void initialize_windows_screen_reader() {
        // Use UI Automation API
        // ... implementation ...
    }
    
    void windows_speak(const std::string& text) {
        // Use Windows Speech API
        // ... implementation ...
    }
    #endif
    
    #ifdef __linux__
    void initialize_linux_screen_reader() {
        // Use AT-SPI
        // ... implementation ...
    }
    
    void linux_speak(const std::string& text) {
        // Use speech-dispatcher
        // ... implementation ...
    }
    #endif
};
```

### Step 7: Success Criteria for Focus & Accessibility

- [ ] Keyboard navigation works across all UI screens
- [ ] Visual focus indicators are clear and visible
- [ ] Theme manager supports color-blind modes
- [ ] Font scaling works for accessibility
- [ ] Controller support is functional
- [ ] Screen reader integration provides useful announcements
- [ ] Players can customize UI to their needs
- [ ] Accessibility features don't hinder normal gameplay

---

## Week 11: Map & Visualization Modernization

### Objective
Improve map interface with better overlays, interactions, and visual feedback

### Step 1: Dynamic Overlays

#### Overlay System Architecture
**File:** `src/map/map_overlays.cpp`
**Add:**
```cpp
class map_overlay_system {
public:
    enum class overlay_type {
        supply,
        trade,
        politics,
        economy,
        military,
        infrastructure,
        population,
        technology
    };
    
    struct overlay_layer {
        overlay_type type;
        bool enabled;
        float opacity;  // 0.0 to 1.0
        color tint;
        
        std::function<void(sys::state&, overlay_layer*)> render_function;
    };
    
    std::vector<overlay_layer> layers;
    
    void initialize() {
        // Create default layers
        layers.push_back(create_supply_layer());
        layers.push_back(create_trade_layer());
        layers.push_back(create_politics_layer());
        layers.push_back(create_economy_layer());
        layers.push_back(create_military_layer());
        layers.push_back(create_infrastructure_layer());
        layers.push_back(create_population_layer());
        layers.push_back(create_technology_layer());
    }
    
    overlay_layer create_supply_layer() {
        overlay_layer layer;
        layer.type = overlay_type::supply;
        layer.enabled = false;
        layer.opacity = 0.5f;
        layer.tint = {1.0f, 0.5f, 0.0f}; // Orange
        
        layer.render_function = [](sys::state& state, overlay_layer* layer) {
            for (auto province : state.world.province_range()) {
                float utilization = calculate_supply_utilization(state, province);
                
                if (utilization > 0.0f) {
                    color overlay_color;
                    if (utilization > 0.9f) {
                        overlay_color = {1.0f, 0.0f, 0.0f}; // Red
                    } else if (utilization > 0.7f) {
                        overlay_color = {1.0f, 0.5f, 0.0f}; // Orange
                    } else if (utilization > 0.5f) {
                        overlay_color = {1.0f, 1.0f, 0.0f}; // Yellow
                    } else {
                        overlay_color = {0.0f, 1.0f, 0.0f}; // Green
                    }
                    
                    draw_province_overlay(province, overlay_color, layer->opacity);
                }
            }
        };
        
        return layer;
    }
    
    overlay_layer create_trade_layer() {
        overlay_layer layer;
        layer.type = overlay_type::trade;
        layer.enabled = false;
        layer.opacity = 0.4f;
        layer.tint = {0.0f, 0.5f, 1.0f}; // Blue
        
        layer.render_function = [](sys::state& state, overlay_layer* layer) {
            // Draw trade routes as lines
            for (auto& route : state.economy.trade_routes) {
                if (route.throughput > 0) {
                    float width = route.throughput / 100.0f;
                    draw_line(route.from, route.to, layer->tint, width);
                }
            }
            
            // Color provinces by trade volume
            for (auto province : state.world.province_range()) {
                float trade_volume = calculate_trade_volume(state, province);
                if (trade_volume > 0) {
                    float intensity = std::min(trade_volume / 1000.0f, 1.0f);
                    color overlay_color = layer->tint;
                    overlay_color.r *= intensity;
                    overlay_color.g *= intensity;
                    overlay_color.b *= intensity;
                    draw_province_overlay(province, overlay_color, layer->opacity);
                }
            }
        };
        
        return layer;
    }
    
    overlay_layer create_politics_layer() {
        overlay_layer layer;
        layer.type = overlay_type::politics;
        layer.enabled = false;
        layer.opacity = 0.6f;
        layer.tint = {0.5f, 0.0f, 1.0f}; // Purple
        
        layer.render_function = [](sys::state& state, overlay_layer* layer) {
            for (auto province : state.world.province_range()) {
                auto owner = province.get_owner();
                if (!owner) continue;
                
                // Color by political alignment
                auto alignment = calculate_political_alignment(state, owner);
                
                color overlay_color;
                if (alignment > 0.6f) {
                    overlay_color = {0.0f, 0.0f, 1.0f}; // Conservative (blue)
                } else if (alignment > 0.4f) {
                    overlay_color = {0.5f, 0.5f, 0.5f}; // Neutral (gray)
                } else {
                    overlay_color = {1.0f, 0.0f, 0.0f}; // Liberal (red)
                }
                
                draw_province_overlay(province, overlay_color, layer->opacity);
            }
        };
        
        return layer;
    }
    
    overlay_layer create_economy_layer() {
        overlay_layer layer;
        layer.type = overlay_type::economy;
        layer.enabled = false;
        layer.opacity = 0.5f;
        layer.tint = {0.0f, 1.0f, 0.0f}; // Green
        
        layer.render_function = [](sys::state& state, overlay_layer* layer) {
            for (auto province : state.world.province_range()) {
                float gdp = calculate_province_gdp(state, province);
                float max_gdp = get_max_province_gdp(state);
                
                if (max_gdp > 0) {
                    float intensity = gdp / max_gdp;
                    color overlay_color = {intensity, 1.0f - intensity, 0.0f};
                    draw_province_overlay(province, overlay_color, layer->opacity);
                }
            }
        };
        
        return layer;
    }
    
    overlay_layer create_military_layer() {
        overlay_layer layer;
        layer.type = overlay_type::military;
        layer.enabled = false;
        layer.opacity = 0.5f;
        layer.tint = {1.0f, 0.0f, 0.0f}; // Red
        
        layer.render_function = [](sys::state& state, overlay_layer* layer) {
            for (auto province : state.world.province_range()) {
                float military_presence = calculate_military_presence(state, province);
                
                if (military_presence > 0) {
                    color overlay_color = {1.0f, 0.0f, 0.0f};
                    draw_province_overlay(province, overlay_color, layer->opacity * military_presence);
                }
            }
        };
        
        return layer;
    }
    
    overlay_layer create_infrastructure_layer() {
        overlay_layer layer;
        layer.type = overlay_type::infrastructure;
        layer.enabled = false;
        layer.opacity = 0.5f;
        layer.tint = {0.5f, 0.5f, 0.5f}; // Gray
        
        layer.render_function = [](sys::state& state, overlay_layer* layer) {
            for (auto province : state.world.province_range()) {
                float infrastructure = calculate_infrastructure(state, province);
                
                if (infrastructure > 0) {
                    color overlay_color = {infrastructure, infrastructure, infrastructure};
                    draw_province_overlay(province, overlay_color, layer->opacity);
                }
            }
        };
        
        return layer;
    }
    
    overlay_layer create_population_layer() {
        overlay_layer layer;
        layer.type = overlay_type::population;
        layer.enabled = false;
        layer.opacity = 0.5f;
        layer.tint = {1.0f, 1.0f, 0.0f}; // Yellow
        
        layer.render_function = [](sys::state& state, overlay_layer* layer) {
            for (auto province : state.world.province_range()) {
                float population = calculate_population_density(state, province);
                float max_population = get_max_population_density(state);
                
                if (max_population > 0) {
                    float intensity = population / max_population;
                    color overlay_color = {intensity, intensity, 0.0f};
                    draw_province_overlay(province, overlay_color, layer->opacity);
                }
            }
        };
        
        return layer;
    }
    
    overlay_layer create_technology_layer() {
        overlay_layer layer;
        layer.type = overlay_type::technology;
        layer.enabled = false;
        layer.opacity = 0.5f;
        layer.tint = {0.0f, 1.0f, 1.0f}; // Cyan
        
        layer.render_function = [](sys::state& state, overlay_layer* layer) {
            for (auto province : state.world.province_range()) {
                auto owner = province.get_owner();
                if (!owner) continue;
                
                float tech_level = calculate_technology_level(state, owner);
                float max_tech = get_max_technology_level(state);
                
                if (max_tech > 0) {
                    float intensity = tech_level / max_tech;
                    color overlay_color = {0.0f, intensity, intensity};
                    draw_province_overlay(province, overlay_color, layer->opacity);
                }
            }
        };
        
        return layer;
    }
    
    void render_overlays(sys::state& state) {
        for (auto& layer : layers) {
            if (layer.enabled && layer.render_function) {
                layer.render_function(state, &layer);
            }
        }
    }
    
    void toggle_overlay(overlay_type type) {
        for (auto& layer : layers) {
            if (layer.type == type) {
                layer.enabled = !layer.enabled;
                break;
            }
        }
    }
    
    void set_overlay_opacity(overlay_type type, float opacity) {
        for (auto& layer : layers) {
            if (layer.type == type) {
                layer.opacity = std::max(0.0f, std::min(1.0f, opacity));
                break;
            }
        }
    }
};
```

#### Overlay Control UI
**File:** `src/gui/gui_map_overlays.cpp`
**Add:**
```cpp
class map_overlay_window : public window_element_base {
public:
    void on_create(sys::state& state) override {
        window_element_base::on_create(state);
        
        // Overlay toggles
        auto overlay_label = std::make_unique<text_element>();
        overlay_label->set_text("Map Overlays:");
        overlay_label->set_position({10, 30});
        add_child(std::move(overlay_label));
        
        auto supply_toggle = std::make_unique<overlay_toggle>();
        supply_toggle->set_position({10, 50});
        supply_toggle->set_text("Supply");
        supply_toggle->set_overlay_type(map_overlay_system::overlay_type::supply);
        supply_toggle->on_toggle = [&](bool enabled) {
            state.map_overlays.toggle_overlay(map_overlay_system::overlay_type::supply);
        };
        add_child(std::move(supply_toggle));
        
        auto trade_toggle = std::make_unique<overlay_toggle>();
        trade_toggle->set_position({10, 80});
        trade_toggle->set_text("Trade");
        trade_toggle->set_overlay_type(map_overlay_system::overlay_type::trade);
        trade_toggle->on_toggle = [&](bool enabled) {
            state.map_overlays.toggle_overlay(map_overlay_system::overlay_type::trade);
        };
        add_child(std::move(trade_toggle));
        
        auto politics_toggle = std::make_unique<overlay_toggle>();
        politics_toggle->set_position({10, 110});
        politics_toggle->set_text("Politics");
        politics_toggle->set_overlay_type(map_overlay_system::overlay_type::politics);
        politics_toggle->on_toggle = [&](bool enabled) {
            state.map_overlays.toggle_overlay(map_overlay_system::overlay_type::politics);
        };
        add_child(std::move(politics_toggle));
        
        auto economy_toggle = std::make_unique<overlay_toggle>();
        economy_toggle->set_position({10, 140});
        economy_toggle->set_text("Economy");
        economy_toggle->set_overlay_type(map_overlay_system::overlay_type::economy);
        economy_toggle->on_toggle = [&](bool enabled) {
            state.map_overlays.toggle_overlay(map_overlay_system::overlay_type::economy);
        };
        add_child(std::move(economy_toggle));
        
        auto military_toggle = std::make_unique<overlay_toggle>();
        military_toggle->set_position({10, 170});
        military_toggle->set_text("Military");
        military_toggle->set_overlay_type(map_overlay_system::overlay_type::military);
        military_toggle->on_toggle = [&](bool enabled) {
            state.map_overlays.toggle_overlay(map_overlay_system::overlay_type::military);
        };
        add_child(std::move(military_toggle));
        
        auto infrastructure_toggle = std::make_unique<overlay_toggle>();
        infrastructure_toggle->set_position({10, 200});
        infrastructure_toggle->set_text("Infrastructure");
        infrastructure_toggle->set_overlay_type(map_overlay_system::overlay_type::infrastructure);
        infrastructure_toggle->on_toggle = [&](bool enabled) {
            state.map_overlays.toggle_overlay(map_overlay_system::overlay_type::infrastructure);
        };
        add_child(std::move(infrastructure_toggle));
        
        auto population_toggle = std::make_unique<overlay_toggle>();
        population_toggle->set_position({10, 230});
        population_toggle->set_text("Population");
        population_toggle->set_overlay_type(map_overlay_system::overlay_type::population);
        population_toggle->on_toggle = [&](bool enabled) {
            state.map_overlays.toggle_overlay(map_overlay_system::overlay_type::population);
        };
        add_child(std::move(population_toggle));
        
        auto technology_toggle = std::make_unique<overlay_toggle>();
        technology_toggle->set_position({10, 260});
        technology_toggle->set_text("Technology");
        technology_toggle->set_overlay_type(map_overlay_system::overlay_type::technology);
        technology_toggle->on_toggle = [&](bool enabled) {
            state.map_overlays.toggle_overlay(map_overlay_system::overlay_type::technology);
        };
        add_child(std::move(technology_toggle));
        
        // Opacity sliders
        auto opacity_label = std::make_unique<text_element>();
        opacity_label->set_text("Opacity:");
        opacity_label->set_position({120, 30});
        add_child(std::move(opacity_label));
        
        auto opacity_slider = std::make_unique<slider_element>();
        opacity_slider->set_position({120, 50});
        opacity_slider->set_range(0.1f, 1.0f);
        opacity_slider->set_value(0.5f);
        opacity_slider->on_change = [&](float value) {
            // Apply to all enabled overlays
            for (auto type : {map_overlay_system::overlay_type::supply,
                              map_overlay_system::overlay_type::trade,
                              map_overlay_system::overlay_type::politics,
                              map_overlay_system::overlay_type::economy,
                              map_overlay_system::overlay_type::military,
                              map_overlay_system::overlay_type::infrastructure,
                              map_overlay_system::overlay_type::population,
                              map_overlay_system::overlay_type::technology}) {
                state.map_overlays.set_overlay_opacity(type, value);
            }
        };
        add_child(std::move(opacity_slider));
        
        // Legend
        auto legend_label = std::make_unique<text_element>();
        legend_label->set_text("Legend:");
        legend_label->set_position({120, 90});
        add_child(std::move(legend_label));
        
        auto legend = std::make_unique<legend_element>();
        legend->set_position({120, 110});
        legend->set_size({200, 150});
        add_child(std::move(legend));
    }
};
```

### Step 2: Zoom & Pan Improvements

#### Smooth Zoom & Pan
**File:** `src/gui/gui_map_navigation.cpp`
**Add:**
```cpp
class map_navigation {
public:
    struct navigation_state {
        float zoom = 1.0f;
        float target_zoom = 1.0f;
        float pan_x = 0.0f;
        float pan_y = 0.0f;
        float target_pan_x = 0.0f;
        float target_pan_y = 0.0f;
        
        // Smooth interpolation
        void update(float dt) {
            // Smooth zoom
            float zoom_diff = target_zoom - zoom;
            if (std::abs(zoom_diff) > 0.001f) {
                zoom += zoom_diff * std::min(1.0f, dt * 10.0f);
            }
            
            // Smooth pan
            float pan_x_diff = target_pan_x - pan_x;
            float pan_y_diff = target_pan_y - pan_y;
            
            if (std::abs(pan_x_diff) > 0.01f) {
                pan_x += pan_x_diff * std::min(1.0f, dt * 10.0f);
            }
            
            if (std::abs(pan_y_diff) > 0.01f) {
                pan_y += pan_y_diff * std::min(1.0f, dt * 10.0f);
            }
        }
    };
    
    navigation_state state;
    
    void zoom_to(float target_zoom, float x, float y) {
        // Zoom toward cursor position
        float old_zoom = state.zoom;
        state.target_zoom = std::max(0.1f, std::min(5.0f, target_zoom));
        
        // Adjust pan to keep cursor at same screen position
        float zoom_factor = state.target_zoom / old_zoom;
        state.target_pan_x = x - (x - state.pan_x) * zoom_factor;
        state.target_pan_y = y - (y - state.pan_y) * zoom_factor;
    }
    
    void pan_to(float target_x, float target_y) {
        state.target_pan_x = target_x;
        state.target_pan_y = target_y;
    }
    
    void pan_by(float dx, float dy) {
        state.target_pan_x += dx;
        state.target_pan_y += dy;
    }
    
    void zoom_in() {
        state.target_zoom = std::min(state.target_zoom * 1.2f, 5.0f);
    }
    
    void zoom_out() {
        state.target_zoom = std::max(state.target_zoom / 1.2f, 0.1f);
    }
    
    void zoom_to_fit(sys::state& state, dcon::province_id province) {
        // Calculate bounding box of province
        auto rect = get_province_screen_rect(state, province);
        
        // Calculate zoom to fit
        float zoom_x = state.ui_state.screen_width / rect.width;
        float zoom_y = state.ui_state.screen_height / rect.height;
        float target_zoom = std::min(zoom_x, zoom_y) * 0.8f; // 80% to leave margin
        
        // Center on province
        float center_x = rect.x + rect.width / 2;
        float center_y = rect.y + rect.height / 2;
        
        zoom_to(target_zoom, center_x, center_y);
    }
    
    void update(float dt) {
        state.update(dt);
    }
};
```

#### Zoom Controls UI
**File:** `src/gui/gui_zoom_controls.cpp`
**Add:**
```cpp
class zoom_controls : public panel_base {
public:
    void on_create(sys::state& state) override {
        panel_base::on_create(state);
        
        // Zoom in button
        auto zoom_in = std::make_unique<button_element>();
        zoom_in->set_position({0, 0});
        zoom_in->set_size({40, 40});
        zoom_in->set_text("+");
        zoom_in->on_click = [&]() {
            state.map_navigation.zoom_in();
        };
        add_child(std::move(zoom_in));
        
        // Zoom out button
        auto zoom_out = std::make_unique<button_element>();
        zoom_out->set_position({0, 50});
        zoom_out->set_size({40, 40});
        zoom_out->set_text("-");
        zoom_out->on_click = [&]() {
            state.map_navigation.zoom_out();
        };
        add_child(std::move(zoom_out));
        
        // Zoom slider
        auto zoom_slider = std::make_unique<slider_element>();
        zoom_slider->set_position({0, 100});
        zoom_slider->set_size({40, 100});
        zoom_slider->set_range(0.1f, 5.0f);
        zoom_slider->set_value(state.map_navigation.state.zoom);
        zoom_slider->on_change = [&](float value) {
            state.map_navigation.state.target_zoom = value;
        };
        add_child(std::move(zoom_slider));
        
        // Zoom to fit button
        auto fit_button = std::make_unique<button_element>();
        fit_button->set_position({0, 210});
        fit_button->set_size({40, 30});
        fit_button->set_text("Fit");
        fit_button->on_click = [&]() {
            if (state.map.selected_province) {
                state.map_navigation.zoom_to_fit(state, state.map.selected_province);
            }
        };
        add_child(std::move(fit_button));
        
        // Reset button
        auto reset_button = std::make_unique<button_element>();
        reset_button->set_position({0, 250});
        reset_button->set_size({40, 30});
        reset_button->set_text("Reset");
        reset_button->on_click = [&]() {
            state.map_navigation.state.target_zoom = 1.0f;
            state.map_navigation.state.target_pan_x = 0.0f;
            state.map_navigation.state.target_pan_y = 0.0f;
        };
        add_child(std::move(reset_button));
    }
};
```

### Step 3: Strategic View

#### Province Aggregation
**File:** `src/map/strategic_view.cpp`
**Add:**
```cpp
class strategic_view {
public:
    struct aggregated_province {
        std::vector<dcon::province_id> provinces;
        dcon::nation_id owner;
        float total_gdp;
        float total_population;
        float total_military;
        point center;
        float radius;
    };
    
    std::vector<aggregated_province> aggregated_regions;
    
    void aggregate_regions(sys::state& state, float zoom_level) {
        aggregated_regions.clear();
        
        if (zoom_level > 2.0f) {
            // Don't aggregate when zoomed in
            return;
        }
        
        // Group provinces by owner and proximity
        std::map<dcon::nation_id, std::vector<dcon::province_id>> by_owner;
        
        for (auto province : state.world.province_range()) {
            auto owner = province.get_owner();
            if (owner) {
                by_owner[owner].push_back(province);
            }
        }
        
        // Create aggregated regions
        for (auto& [owner, provinces] : by_owner) {
            if (provinces.size() < 5) {
                // Too few provinces to aggregate
                continue;
            }
            
            aggregated_region region;
            region.owner = owner;
            region.provinces = provinces;
            
            // Calculate center
            point center = {0.0f, 0.0f};
            for (auto prov : provinces) {
                auto pos = get_province_position(state, prov);
                center.x += pos.x;
                center.y += pos.y;
            }
            center.x /= provinces.size();
            center.y /= provinces.size();
            region.center = center;
            
            // Calculate radius
            float max_dist = 0.0f;
            for (auto prov : provinces) {
                auto pos = get_province_position(state, prov);
                float dist = std::sqrt(std::pow(pos.x - center.x, 2) + 
                                      std::pow(pos.y - center.y, 2));
                max_dist = std::max(max_dist, dist);
            }
            region.radius = max_dist;
            
            // Calculate totals
            region.total_gdp = 0.0f;
            region.total_population = 0.0f;
            region.total_military = 0.0f;
            
            for (auto prov : provinces) {
                region.total_gdp += calculate_province_gdp(state, prov);
                region.total_population += calculate_province_population(state, prov);
                region.total_military += calculate_province_military(state, prov);
            }
            
            aggregated_regions.push_back(region);
        }
    }
    
    void render_aggregated_regions(sys::state& state) {
        for (auto& region : aggregated_regions) {
            // Draw region circle
            draw_circle(region.center.x, region.center.y, region.radius,
                       get_nation_color(region.owner), 0.3f);
            
            // Draw owner name
            draw_text(region.center.x - 10, region.center.y - 10,
                     get_nation_name(region.owner), 12, {1.0f, 1.0f, 1.0f});
            
            // Draw stats on hover
            if (is_hovered(region.center, region.radius)) {
                draw_tooltip(region.center.x, region.center.y,
                            format_region_stats(region));
            }
        }
    }
    
    std::string format_region_stats(const aggregated_region& region) {
        std::string stats;
        stats += "Provinces: " + std::to_string(region.provinces.size()) + "\n";
        stats += "GDP: " + format_gdp(region.total_gdp) + "\n";
        stats += "Population: " + format_population(region.total_population) + "\n";
        stats += "Military: " + format_military(region.total_military);
        return stats;
    }
};
```

### Step 4: Visual Feedback System

#### Animation & Transitions
**File:** `src/gui/gui_animations.cpp`
**Add:**
```cpp
class animation_system {
public:
    struct animation {
        enum class type {
            fade_in,
            fade_out,
            slide_in,
            slide_out,
            scale,
            pulse,
            color_change
        };
        
        type anim_type;
        element_base* target;
        float duration;  // seconds
        float elapsed;
        
        // Start/end values
        float start_value;
        float end_value;
        
        // Color animation
        color start_color;
        color end_color;
        
        bool completed;
        
        void update(float dt) {
            elapsed += dt;
            
            if (elapsed >= duration) {
                elapsed = duration;
                completed = true;
            }
            
            // Apply animation to target
            apply_animation();
        }
        
        void apply_animation() {
            if (!target) return;
            
            float progress = elapsed / duration;
            
            switch (anim_type) {
                case type::fade_in:
                    target->set_opacity(progress);
                    break;
                    
                case type::fade_out:
                    target->set_opacity(1.0f - progress);
                    break;
                    
                case type::slide_in:
                    target->set_position({
                        target->get_position().x + (start_value - end_value) * (1.0f - progress),
                        target->get_position().y
                    });
                    break;
                    
                case type::slide_out:
                    target->set_position({
                        target->get_position().x + (end_value - start_value) * progress,
                        target->get_position().y
                    });
                    break;
                    
                case type::scale:
                    target->set_scale(start_value + (end_value - start_value) * progress);
                    break;
                    
                case type::pulse:
                    float pulse = std::sin(progress * 3.14159f * 2.0f) * 0.5f + 0.5f;
                    target->set_scale(1.0f + pulse * 0.1f);
                    break;
                    
                case type::color_change:
                    color current = {
                        start_color.r + (end_color.r - start_color.r) * progress,
                        start_color.g + (end_color.g - start_color.g) * progress,
                        start_color.b + (end_color.b - start_color.b) * progress
                    };
                    target->set_foreground_color(current);
                    break;
            }
        }
    };
    
    std::vector<animation> active_animations;
    
    void update(float dt) {
        for (auto it = active_animations.begin(); it != active_animations.end();) {
            it->update(dt);
            
            if (it->completed) {
                it = active_animations.erase(it);
            } else {
                ++it;
            }
        }
    }
    
    void fade_in(element_base* element, float duration = 0.3f) {
        animation anim;
        anim.anim_type = animation::type::fade_in;
        anim.target = element;
        anim.duration = duration;
        anim.elapsed = 0.0f;
        anim.start_value = 0.0f;
        anim.end_value = 1.0f;
        anim.completed = false;
        
        element->set_opacity(0.0f);
        active_animations.push_back(anim);
    }
    
    void fade_out(element_base* element, float duration = 0.3f) {
        animation anim;
        anim.anim_type = animation::type::fade_out;
        anim.target = element;
        anim.duration = duration;
        anim.elapsed = 0.0f;
        anim.start_value = 1.0f;
        anim.end_value = 0.0f;
        anim.completed = false;
        
        active_animations.push_back(anim);
    }
    
    void pulse(element_base* element, float duration = 1.0f) {
        animation anim;
        anim.anim_type = animation::type::pulse;
        anim.target = element;
        anim.duration = duration;
        anim.elapsed = 0.0f;
        anim.completed = false;
        
        active_animations.push_back(anim);
    }
    
    void color_change(element_base* element, color from, color to, float duration = 0.5f) {
        animation anim;
        anim.anim_type = animation::type::color_change;
        anim.target = element;
        anim.duration = duration;
        anim.elapsed = 0.0f;
        anim.start_color = from;
        anim.end_color = to;
        anim.completed = false;
        
        active_animations.push_back(anim);
    }
};

// Usage in UI
class feedback_element : public element_base {
public:
    void on_click(sys::state& state, int x, int y) override {
        // Visual feedback
        state.ui_state.animation_system.pulse(this, 0.5f);
        
        // Color feedback
        color original = get_foreground_color();
        state.ui_state.animation_system.color_change(
            this, original, {1.0f, 1.0f, 0.0f}, 0.2f);
        
        // Restore color after delay
        state.ui_state.animation_system.color_change(
            this, {1.0f, 1.0f, 0.0f}, original, 0.2f);
    }
};
```

#### Event Notifications
**File:** `src/gui/gui_notifications.cpp`
**Add:**
```cpp
class notification_system {
public:
    struct notification {
        std::string title;
        std::string message;
        notification_type type;
        float duration;  // seconds (0 = persistent)
        float elapsed;
        bool dismissed;
        
        // Visual properties
        point position;
        color bg_color;
        color text_color;
    };
    
    std::vector<notification> active_notifications;
    
    enum class notification_type {
        info,
        success,
        warning,
        error,
        event
    };
    
    void show_notification(const std::string& title, const std::string& message, 
                          notification_type type, float duration = 5.0f) {
        notification notif;
        notif.title = title;
        notif.message = message;
        notif.type = type;
        notif.duration = duration;
        notif.elapsed = 0.0f;
        notif.dismissed = false;
        
        // Set colors based on type
        switch (type) {
            case notification_type::info:
                notif.bg_color = {0.2f, 0.4f, 0.8f}; // Blue
                notif.text_color = {1.0f, 1.0f, 1.0f};
                break;
            case notification_type::success:
                notif.bg_color = {0.2f, 0.6f, 0.2f}; // Green
                notif.text_color = {1.0f, 1.0f, 1.0f};
                break;
            case notification_type::warning:
                notif.bg_color = {0.8f, 0.6f, 0.2f}; // Orange
                notif.text_color = {0.0f, 0.0f, 0.0f};
                break;
            case notification_type::error:
                notif.bg_color = {0.8f, 0.2f, 0.2f}; // Red
                notif.text_color = {1.0f, 1.0f, 1.0f};
                break;
            case notification_type::event:
                notif.bg_color = {0.6f, 0.2f, 0.8f}; // Purple
                notif.text_color = {1.0f, 1.0f, 1.0f};
                break;
        }
        
        // Position (stack from top-right)
        notif.position = {
            static_cast<float>(state.ui_state.screen_width - 320),
            static_cast<float>(30 + active_notifications.size() * 80)
        };
        
        active_notifications.push_back(notif);
        
        // Animate in
        state.ui_state.animation_system.fade_in(this, 0.3f);
    }
    
    void update(float dt) {
        for (auto it = active_notifications.begin(); it != active_notifications.end();) {
            it->elapsed += dt;
            
            if (it->duration > 0 && it->elapsed >= it->duration) {
                // Fade out before removing
                state.ui_state.animation_system.fade_out(this, 0.3f);
                it = active_notifications.erase(it);
            } else {
                ++it;
            }
        }
    }
    
    void render(sys::state& state) {
        for (auto& notif : active_notifications) {
            // Draw background
            draw_rectangle(notif.position.x, notif.position.y, 
                          300, 70, notif.bg_color, 0.9f);
            
            // Draw title
            draw_text(notif.position.x + 10, notif.position.y + 10,
                     notif.title, 14, notif.text_color);
            
            // Draw message
            draw_text(notif.position.x + 10, notif.position.y + 30,
                     notif.message, 11, notif.text_color);
            
            // Draw progress bar if timed
            if (notif.duration > 0) {
                float progress = notif.elapsed / notif.duration;
                draw_rectangle(notif.position.x, notif.position.y + 68,
                              300 * progress, 2, {1.0f, 1.0f, 1.0f}, 1.0f);
            }
            
            // Draw close button
            if (notif.duration == 0) {
                draw_text(notif.position.x + 280, notif.position.y + 10,
                         "X", 12, notif.text_color);
            }
        }
    }
    
    void dismiss_notification(size_t index) {
        if (index < active_notifications.size()) {
            active_notifications.erase(active_notifications.begin() + index);
        }
    }
};

// Usage in game
void show_event_notification(sys::state& state, const std::string& title, 
                            const std::string& message) {
    state.ui_state.notification_system.show_notification(
        title, message, notification_system::notification_type::event, 8.0f);
}

void show_error_notification(sys::state& state, const std::string& title,
                            const std::string& message) {
    state.ui_state.notification_system.show_notification(
        title, message, notification_system::notification_type::error, 0.0f); // Persistent
}
```

### Step 5: Map Search

#### Search Functionality
**File:** `src/gui/gui_map_search.cpp`
**Add:**
```cpp
class map_search_window : public window_element_base {
public:
    void on_create(sys::state& state) override {
        window_element_base::on_create(state);
        
        // Search input
        auto search_label = std::make_unique<text_element>();
        search_label->set_text("Search:");
        search_label->set_position({10, 30});
        add_child(std::move(search_label));
        
        auto search_input = std::make_unique<text_input_element>();
        search_input->set_position({10, 50});
        search_input->set_size({300, 30});
        search_input->set_placeholder("Enter province name, resource, or criteria...");
        search_input->on_text_change = [&](const std::string& text) {
            perform_search(state, text);
        };
        add_child(std::move(search_input));
        
        // Results list
        auto results_label = std::make_unique<text_element>();
        results_label->set_text("Results:");
        results_label->set_position({10, 90});
        add_child(std::move(results_label));
        
        auto results_list = std::make_unique<search_results_listbox>();
        results_list->set_position({10, 110});
        results_list->set_size({300, 200});
        add_child(std::move(results_list));
        
        // Search criteria
        auto criteria_label = std::make_unique<text_element>();
        criteria_label->set_text("Criteria:");
        criteria_label->set_position({320, 30});
        add_child(std::move(criteria_label));
        
        // Resource filter
        auto resource_label = std::make_unique<text_element>();
        resource_label->set_text("Resource:");
        resource_label->set_position({320, 50});
        add_child(std::move(resource_label));
        
        auto resource_selector = std::make_unique<commodity_selector>();
        resource_selector->set_position({320, 70});
        resource_selector->set_size({200, 30});
        add_child(std::move(resource_selector));
        
        // Population filter
        auto pop_label = std::make_unique<text_element>();
        pop_label->set_text("Min Population:");
        pop_label->set_position({320, 110});
        add_child(std::move(pop_label));
        
        auto pop_slider = std::make_unique<slider_element>();
        pop_slider->set_position({320, 130});
        pop_slider->set_range(0.0f, 1000000.0f);
        pop_slider->set_value(0.0f);
        pop_slider->on_change = [&](float value) {
            perform_search(state, search_input->get_text());
        };
        add_child(std::move(pop_slider));
        
        // GDP filter
        auto gdp_label = std::make_unique<text_element>();
        gdp_label->set_text("Min GDP:");
        gdp_label->set_position({320, 170});
        add_child(std::move(gdp_label));
        
        auto gdp_slider = std::make_unique<slider_element>();
        gdp_slider->set_position({320, 190});
        gdp_slider->set_range(0.0f, 1000000.0f);
        gdp_slider->set_value(0.0f);
        gdp_slider->on_change = [&](float value) {
            perform_search(state, search_input->get_text());
        };
        add_child(std::move(gdp_slider));
        
        // Search button
        auto search_button = std::make_unique<button_element>();
        search_button->set_position({320, 230});
        search_button->set_size({100, 30});
        search_button->set_text("Search");
        search_button->on_click = [&]() {
            perform_search(state, search_input->get_text());
        };
        add_child(std::move(search_button));
    }
    
    void perform_search(sys::state& state, const std::string& query) {
        std::vector<dcon::province_id> results;
        
        // Search by name
        if (!query.empty()) {
            for (auto province : state.world.province_range()) {
                std::string name = get_province_name(province);
                if (name.find(query) != std::string::npos) {
                    results.push_back(province);
                }
            }
        }
        
        // Apply filters
        auto resource = get_selected_resource();
        auto min_pop = get_min_population();
        auto min_gdp = get_min_gdp();
        
        if (resource || min_pop > 0 || min_gdp > 0) {
            std::vector<dcon::province_id> filtered;
            
            for (auto province : results) {
                bool include = true;
                
                if (resource && !province.produces_commodity(resource)) {
                    include = false;
                }
                
                if (min_pop > 0 && calculate_province_population(state, province) < min_pop) {
                    include = false;
                }
                
                if (min_gdp > 0 && calculate_province_gdp(state, province) < min_gdp) {
                    include = false;
                }
                
                if (include) {
                    filtered.push_back(province);
                }
            }
            
            results = filtered;
        }
        
        // Update results list
        update_results_list(state, results);
        
        // Highlight on map
        highlight_results(state, results);
    }
    
    void update_results_list(sys::state& state, const std::vector<dcon::province_id>& results) {
        auto results_list = get_child_by_index(2); // Results listbox
        results_list->clear_items();
        
        for (auto province : results) {
            auto item = std::make_unique<search_result_item>();
            item->set_province(province);
            item->on_click = [&]() {
                // Zoom to province
                state.map_navigation.zoom_to_fit(state, province);
                // Select province
                state.map.selected_province = province;
            };
            results_list->add_item(std::move(item));
        }
    }
    
    void highlight_results(sys::state& state, const std::vector<dcon::province_id>& results) {
        // Clear previous highlights
        state.map.highlighted_provinces.clear();
        
        // Add new highlights
        for (auto province : results) {
            state.map.highlighted_provinces.push_back(province);
        }
    }
};

class search_result_item : public listbox_item_base {
public:
    void set_province(dcon::province_id province) {
        auto name = get_province_name(province);
        auto owner = get_province_owner(province);
        auto gdp = get_province_gdp(province);
        auto population = get_province_population(province);
        
        auto text = name + " (" + get_nation_name(owner) + ") - " +
                    "GDP: " + format_gdp(gdp) + ", Pop: " + format_population(population);
        
        set_text(text);
    }
};
```

### Step 6: Visual Feedback for Events

#### Event Highlighting
**File:** `src/gui/gui_event_feedback.cpp`
**Add:**
```cpp
class event_feedback_system {
public:
    struct event_highlight {
        dcon::province_id province;
        event_type type;
        float intensity;  // 0.0 to 1.0
        float duration;   // seconds
        float elapsed;
        
        color get_color() const {
            switch (type) {
                case event_type::war_declaration:
                    return {1.0f, 0.0f, 0.0f}; // Red
                case event_type::peace_treaty:
                    return {0.0f, 1.0f, 0.0f}; // Green
                case event_type::revolution:
                    return {0.5f, 0.0f, 0.5f}; // Purple
                case event_type::economic_crisis:
                    return {1.0f, 0.5f, 0.0f}; // Orange
                case event_type::technological_breakthrough:
                    return {0.0f, 1.0f, 1.0f}; // Cyan
                default:
                    return {1.0f, 1.0f, 1.0f}; // White
            }
        }
    };
    
    std::vector<event_highlight> active_highlights;
    
    enum class event_type {
        war_declaration,
        peace_treaty,
        revolution,
        economic_crisis,
        technological_breakthrough
    };
    
    void add_highlight(dcon::province_id province, event_type type, float duration = 3.0f) {
        event_highlight highlight;
        highlight.province = province;
        highlight.type = type;
        highlight.intensity = 1.0f;
        highlight.duration = duration;
        highlight.elapsed = 0.0f;
        
        active_highlights.push_back(highlight);
        
        // Animate pulse
        state.ui_state.animation_system.pulse(
            get_province_element(province), duration);
    }
    
    void update(float dt) {
        for (auto it = active_highlights.begin(); it != active_highlights.end();) {
            it->elapsed += dt;
            
            // Fade out over time
            float progress = it->elapsed / it->duration;
            it->intensity = 1.0f - progress;
            
            if (it->elapsed >= it->duration) {
                it = active_highlights.erase(it);
            } else {
                ++it;
            }
        }
    }
    
    void render_overlays(sys::state& state) {
        for (auto& highlight : active_highlights) {
            if (highlight.intensity > 0.0f) {
                color overlay_color = highlight.get_color();
                draw_province_overlay(highlight.province, overlay_color, 
                                     highlight.intensity * 0.6f);
            }
        }
    }
};
```

### Step 7: Success Criteria for Map & Visualization

- [ ] Dynamic overlays provide useful strategic information
- [ ] Smooth zoom and pan with intuitive controls
- [ ] Strategic view simplifies large-scale planning
- [ ] Visual feedback is clear and immediate
- [ ] Map search helps find locations quickly
- [ ] Event highlighting draws attention to important events
- [ ] Players can navigate map efficiently
- [ ] Visual information is informative without being overwhelming

---

## Week 12: Event & Decision System

### Objective
Modernize event presentation and decision-making with better visuals and feedback

### Step 1: Event Cards

#### Modern Event UI
**File:** `src/gui/gui_event_cards.cpp`
**Add:**
```cpp
class event_card_window : public window_element_base {
public:
    void on_create(sys::state& state) override {
        window_element_base::on_create(state);
        
        // Card background
        auto background = std::make_unique<card_background>();
        background->set_position({0, 0});
        background->set_size({500, 400});
        add_child(std::move(background));
        
        // Event title
        auto title = std::make_unique<text_element>();
        title->set_text("Event Title");
        title->set_font_size(18);
        title->set_position({20, 20});
        title->set_color({1.0f, 1.0f, 1.0f});
        add_child(std::move(title));
        
        // Event image/icon
        auto icon = std::make_unique<image_element>();
        icon->set_position({20, 50});
        icon->set_size({80, 80});
        icon->set_texture("event_icon");
        add_child(std::move(icon));
        
        // Event description
        auto description = std::make_unique<text_element>();
        description->set_text("Event description goes here...");
        description->set_position({110, 50});
        description->set_size({370, 100});
        description->set_wrap(true);
        add_child(std::move(description));
        
        // Event effects
        auto effects_label = std::make_unique<text_element>();
        effects_label->set_text("Effects:");
        effects_label->set_position({20, 150});
        effects_label->set_font_size(14);
        effects_label->set_color({1.0f, 1.0f, 0.0f});
        add_child(std::move(effects_label));
        
        auto effects_list = std::make_unique<effects_listbox>();
        effects_list->set_position({20, 170});
        effects_list->set_size({460, 100});
        add_child(std::move(effects_list));
        
        // Decision options
        auto decisions_label = std::make_unique<text_element>();
        decisions_label->set_text("Decisions:");
        decisions_label->set_position({20, 280});
        decisions_label->set_font_size(14);
        decisions_label->set_color({1.0f, 1.0f, 0.0f});
        add_child(std::move(decisions_label));
        
        auto decisions_list = std::make_unique<decisions_listbox>();
        decisions_list->set_position({20, 300});
        decisions_list->set_size({460, 80});
        add_child(std::move(decisions_list));
        
        // Close button
        auto close_button = std::make_unique<button_element>();
        close_button->set_position({200, 390});
        close_button->set_size({100, 30});
        close_button->set_text("Close");
        close_button->on_click = [&]() {
            close();
        };
        add_child(std::move(close_button));
    }
    
    void update_event(sys::state& state, dcon::event_id event) {
        auto event_obj = state.world.event_get(event);
        
        // Update title
        auto title = get_child_by_index(1);
        title->set_text(event_obj.title);
        
        // Update icon
        auto icon = get_child_by_index(2);
        icon->set_texture(event_obj.icon);
        
        // Update description
        auto description = get_child_by_index(3);
        description->set_text(event_obj.description);
        
        // Update effects
        auto effects_list = get_child_by_index(5);
        effects_list->clear_items();
        
        for (auto& effect : event_obj.effects) {
            auto item = std::make_unique<effect_list_item>();
            item->set_effect(effect);
            effects_list->add_item(std::move(item));
        }
        
        // Update decisions
        auto decisions_list = get_child_by_index(7);
        decisions_list->clear_items();
        
        for (auto& decision : event_obj.decisions) {
            auto item = std::make_unique<decision_list_item>();
            item->set_decision(decision);
            item->on_click = [&]() {
                execute_decision(state, event, decision);
                close();
            };
            decisions_list->add_item(std::move(item));
        }
    }
};

class card_background : public element_base {
public:
    void render(sys::state& state) override {
        // Draw rounded rectangle
        draw_rounded_rectangle(position.x, position.y, size.width, size.height,
                              10.0f, // corner radius
                              {0.1f, 0.1f, 0.2f}, // dark blue background
                              0.95f); // opacity
        
        // Draw border
        draw_rounded_rectangle(position.x - 2, position.y - 2,
                              size.width + 4, size.height + 4,
                              12.0f,
                              {0.3f, 0.3f, 0.6f}, // lighter blue border
                              1.0f);
        
        // Draw shadow
        draw_rectangle(position.x + 5, position.y + 5,
                      size.width, size.height,
                      {0.0f, 0.0f, 0.0f}, // black
                      0.3f); // semi-transparent
    }
};

class effect_list_item : public listbox_item_base {
public:
    void set_effect(const event_effect& effect) {
        std::string text;
        
        switch (effect.type) {
            case effect_type::change_gdp:
                text = "GDP: " + format_change(effect.value);
                set_color({0.0f, 1.0f, 0.0f}); // Green
                break;
            case effect_type::change_population:
                text = "Population: " + format_change(effect.value);
                set_color({0.0f, 1.0f, 0.0f});
                break;
            case effect_type::change_stability:
                text = "Stability: " + format_change(effect.value);
                set_color({effect.value > 0 ? 0.0f : 1.0f, 
                          effect.value > 0 ? 1.0f : 0.0f, 0.0f});
                break;
            case effect_type::change_war_support:
                text = "War Support: " + format_change(effect.value);
                set_color({effect.value > 0 ? 0.0f : 1.0f,
                          effect.value > 0 ? 1.0f : 0.0f, 0.0f});
                break;
            default:
                text = effect.description;
                set_color({1.0f, 1.0f, 1.0f});
        }
        
        set_text(text);
    }
};

class decision_list_item : public listbox_item_base {
public:
    void set_decision(const event_decision& decision) {
        set_text(decision.name);
        
        // Color based on consequences
        if (decision.consequences == consequence_type::positive) {
            set_color({0.0f, 1.0f, 0.0f}); // Green
        } else if (decision.consequences == consequence_type::negative) {
            set_color({1.0f, 0.0f, 0.0f}); // Red
        } else {
            set_color({1.0f, 1.0f, 1.0f}); // White
        }
    }
};
```

#### Event Queue
**File:** `src/gui/gui_event_queue.cpp`
**Add:**
```cpp
class event_queue_window : public window_element_base {
public:
    void on_create(sys::state& state) override {
        window_element_base::on_create(state);
        
        // Queue header
        auto header = std::make_unique<text_element>();
        header->set_text("Event Queue");
        header->set_font_size(16);
        header->set_position({10, 10});
        add_child(std::move(header));
        
        // Event list
        auto event_list = std::make_unique<event_listbox>();
        event_list->set_position({10, 30});
        event_list->set_size({300, 200});
        add_child(std::move(event_list));
        
        // Event details panel
        auto details_panel = std::make_unique<event_details_panel>();
        details_panel->set_position({320, 30});
        details_panel->set_size({300, 200});
        add_child(std::move(details_panel));
        
        // Auto-process checkbox
        auto auto_process = std::make_unique<checkbox_element>();
        auto_process->set_position({10, 240});
        auto_process->set_text("Auto-process events");
        auto_process->set_checked(state.events.auto_process);
        auto_process->on_change = [&](bool checked) {
            state.events.auto_process = checked;
        };
        add_child(std::move(auto_process));
        
        // Process all button
        auto process_all = std::make_unique<button_element>();
        process_all->set_position({10, 270});
        process_all->set_size({100, 30});
        process_all->set_text("Process All");
        process_all->on_click = [&]() {
            process_all_events(state);
        };
        add_child(std::move(process_all));
    }
    
    void update_event_list(sys::state& state) {
        auto event_list = get_child_by_index(1);
        event_list->clear_items();
        
        for (auto event : state.events.queue) {
            auto item = std::make_unique<event_queue_item>();
            item->set_event(event);
            item->on_click = [&]() {
                show_event_card(state, event);
            };
            event_list->add_item(std::move(item));
        }
    }
};

class event_queue_item : public listbox_item_base {
public:
    void set_event(dcon::event_id event) {
        auto event_obj = state.world.event_get(event);
        
        auto text = event_obj.title + " - " + format_date(event_obj.trigger_date);
        set_text(text);
        
        // Color based on urgency
        if (event_obj.urgency == event_urgency::high) {
            set_color({1.0f, 0.0f, 0.0f}); // Red
        } else if (event_obj.urgency == event_urgency::medium) {
            set_color({1.0f, 1.0f, 0.0f}); // Yellow
        } else {
            set_color({1.0f, 1.0f, 1.0f}); // White
        }
    }
};
```

### Step 2: Decision Previews

#### Show Consequences Before Choosing
**File:** `src/gui/gui_decision_previews.cpp`
**Add:**
```cpp
class decision_preview_window : public window_element_base {
public:
    void on_create(sys::state& state) override {
        window_element_base::on_create(state);
        
        // Decision name
        auto name_label = std::make_unique<text_element>();
        name_label->set_text("Decision:");
        name_label->set_position({10, 30});
        add_child(std::move(name_label));
        
        auto name_display = std::make_unique<text_element>();
        name_display->set_position({10, 50});
        name_display->set_font_size(14);
        add_child(std::move(name_display));
        
        // Immediate effects
        auto immediate_label = std::make_unique<text_element>();
        immediate_label->set_text("Immediate Effects:");
        immediate_label->set_position({10, 80});
        immediate_label->set_color({1.0f, 1.0f, 0.0f});
        add_child(std::move(immediate_label));
        
        auto immediate_list = std::make_unique<effects_listbox>();
        immediate_list->set_position({10, 100});
        immediate_list->set_size({280, 100});
        add_child(std::move(immediate_list));
        
        // Long-term effects
        auto longterm_label = std::make_unique<text_element>();
        longterm_label->set_text("Long-term Effects:");
        longterm_label->set_position({300, 80});
        longterm_label->set_color({1.0f, 1.0f, 0.0f});
        add_child(std::move(longterm_label));
        
        auto longterm_list = std::make_unique<effects_listbox>();
        longterm_list->set_position({300, 100});
        longterm_list->set_size({280, 100});
        add_child(std::move(longterm_list));
        
        // Risk assessment
        auto risk_label = std::make_unique<text_element>();
        risk_label->set_text("Risk Assessment:");
        risk_label->set_position({10, 210});
        risk_label->set_color({1.0f, 1.0f, 0.0f});
        add_child(std::move(risk_label));
        
        auto risk_display = std::make_unique<risk_display>();
        risk_display->set_position({10, 230});
        risk_display->set_size({570, 50});
        add_child(std::move(risk_display));
        
        // Accept/Reject buttons
        auto accept_button = std::make_unique<button_element>();
        accept_button->set_position({10, 290});
        accept_button->set_size({100, 30});
        accept_button->set_text("Accept");
        accept_button->on_click = [&]() {
            accept_decision(state);
        };
        add_child(std::move(accept_button));
        
        auto reject_button = std::make_unique<button_element>();
        reject_button->set_position({120, 290});
        reject_button->set_size({100, 30});
        reject_button->set_text("Reject");
        reject_button->on_click = [&]() {
            close();
        };
        add_child(std::move(reject_button));
    }
    
    void update_preview(sys::state& state, dcon::decision_id decision) {
        auto decision_obj = state.world.decision_get(decision);
        
        // Update name
        auto name_display = get_child_by_index(1);
        name_display->set_text(decision_obj.name);
        
        // Calculate immediate effects
        auto immediate = calculate_immediate_effects(state, decision);
        auto immediate_list = get_child_by_index(3);
        immediate_list->clear_items();
        
        for (auto& effect : immediate) {
            auto item = std::make_unique<effect_list_item>();
            item->set_effect(effect);
            immediate_list->add_item(std::move(item));
        }
        
        // Calculate long-term effects
        auto longterm = calculate_longterm_effects(state, decision);
        auto longterm_list = get_child_by_index(5);
        longterm_list->clear_items();
        
        for (auto& effect : longterm) {
            auto item = std::make_unique<effect_list_item>();
            item->set_effect(effect);
            longterm_list->add_item(std::move(item));
        }
        
        // Calculate risk
        auto risk = calculate_risk(state, decision);
        auto risk_display = get_child_by_index(7);
        risk_display->set_risk(risk);
    }
};
```

### Step 3: Event History

#### Timeline of Past Events
**File:** `src/gui/gui_event_history.cpp`
**Add:**
```cpp
class event_history_window : public window_element_base {
public:
    void on_create(sys::state& state) override {
        window_element_base::on_create(state);
        
        // Timeline chart
        auto timeline_chart = std::make_unique<timeline_chart_element>();
        timeline_chart->set_position({10, 30});
        timeline_chart->set_size({600, 200});
        add_child(std::move(timeline_chart));
        
        // Event list
        auto event_list = std::make_unique<event_history_listbox>();
        event_list->set_position({10, 240});
        event_list->set_size({600, 150});
        add_child(std::move(event_list));
        
        // Filter controls
        auto filter_label = std::make_unique<text_element>();
        filter_label->set_text("Filter:");
        filter_label->set_position({620, 30});
        add_child(std::move(filter_label));
        
        auto filter_checkboxes = std::make_unique<event_type_filter>();
        filter_checkboxes->set_position({620, 50});
        filter_checkboxes->set_size({100, 150});
        add_child(std::move(filter_checkboxes));
        
        // Search
        auto search_label = std::make_unique<text_element>();
        search_label->set_text("Search:");
        search_label->set_position({620, 210});
        add_child(std::move(search_label));
        
        auto search_input = std::make_unique<text_input_element>();
        search_input->set_position({620, 230});
        search_input->set_size({100, 30});
        search_input->on_text_change = [&](const std::string& text) {
            filter_events(text);
        };
        add_child(std::move(search_input));
    }
    
    void update_history(sys::state& state) {
        auto& history = state.events.history;
        
        // Update timeline chart
        auto timeline_chart = get_child_by_index(1);
        timeline_chart->update(state, history);
        
        // Update event list
        auto event_list = get_child_by_index(2);
        event_list->clear_items();
        
        for (auto& event : history) {
            auto item = std::make_unique<event_history_item>();
            item->set_event(event);
            event_list->add_item(std::move(item));
        }
    }
};

class event_history_item : public listbox_item_base {
public:
    void set_event(const event_record& record) {
        auto text = record.title + " - " + format_date(record.date) + 
                   " - " + record.outcome;
        set_text(text);
        
        // Color based on outcome
        if (record.outcome == "positive") {
            set_color({0.0f, 1.0f, 0.0f}); // Green
        } else if (record.outcome == "negative") {
            set_color({1.0f, 0.0f, 0.0f}); // Red
        } else {
            set_color({1.0f, 1.0f, 1.0f}); // White
        }
    }
};
```

### Step 4: Event Suggestions

#### AI-Recommended Decisions
**File:** `src/ai/ai_event_suggestions.cpp`
**Add:**
```cpp
class event_suggester {
public:
    struct decision_suggestion {
        dcon::decision_id decision;
        float confidence;  // 0.0 to 1.0
        std::string rationale;
        std::vector<std::string> predicted_outcomes;
    };
    
    std::vector<decision_suggestion> suggest_decisions(
        sys::state& state, 
        dcon::event_id event
    ) {
        std::vector<decision_suggestion> suggestions;
        
        auto event_obj = state.world.event_get(event);
        
        for (auto& decision : event_obj.decisions) {
            decision_suggestion suggestion;
            suggestion.decision = decision.id;
            
            // Calculate confidence based on historical success
            suggestion.confidence = calculate_confidence(state, event, decision);
            
            // Generate rationale
            suggestion.rationale = generate_rationale(state, event, decision);
            
            // Predict outcomes
            suggestion.predicted_outcomes = predict_outcomes(state, event, decision);
            
            if (suggestion.confidence > 0.5f) {
                suggestions.push_back(suggestion);
            }
        }
        
        // Sort by confidence
        std::sort(suggestions.begin(), suggestions.end(),
            [](const auto& a, const auto& b) { return a.confidence > b.confidence; });
        
        return suggestions;
    }
    
    float calculate_confidence(sys::state& state, dcon::event_id event, 
                              const event_decision& decision) {
        float confidence = 0.5f; // Base confidence
        
        // Check historical success rate
        auto success_rate = get_historical_success_rate(state, decision.id);
        confidence += success_rate * 0.3f;
        
        // Check alignment with national strategy
        auto alignment = calculate_strategy_alignment(state, decision);
        confidence += alignment * 0.2f;
        
        // Check risk level
        auto risk = calculate_decision_risk(state, event, decision);
        confidence -= risk * 0.1f;
        
        return std::max(0.0f, std::min(1.0f, confidence));
    }
    
    std::string generate_rationale(sys::state& state, dcon::event_id event,
                                  const event_decision& decision) {
        std::string rationale;
        
        // Check for economic benefits
        if (decision.effects.economic > 0) {
            rationale += "Economic benefit: " + format_value(decision.effects.economic) + ". ";
        }
        
        // Check for stability
        if (decision.effects.stability > 0) {
            rationale += "Increases stability. ";
        } else if (decision.effects.stability < 0) {
            rationale += "Decreases stability. ";
        }
        
        // Check for war support
        if (decision.effects.war_support > 0) {
            rationale += "Increases war support. ";
        }
        
        return rationale;
    }
    
    std::vector<std::string> predict_outcomes(sys::state& state, dcon::event_id event,
                                             const event_decision& decision) {
        std::vector<std::string> outcomes;
        
        // Based on historical data
        auto historical = get_historical_outcomes(state, decision.id);
        
        for (auto& outcome : historical) {
            outcomes.push_back(outcome.description);
        }
        
        // Add predicted short-term outcomes
        if (decision.effects.economic > 10) {
            outcomes.push_back("Short-term: Economic growth expected");
        }
        
        if (decision.effects.stability < -5) {
            outcomes.push_back("Short-term: Possible unrest");
        }
        
        return outcomes;
    }
};
```

#### Event Suggestion UI
**File:** `src/gui/gui_event_suggestions.cpp`
**Add:**
```cpp
class event_suggestion_window : public window_element_base {
public:
    void on_create(sys::state& state) override {
        window_element_base::on_create(state);
        
        // Suggestion header
        auto header = std::make_unique<text_element>();
        header->set_text("AI Recommendations");
        header->set_font_size(16);
        header->set_position({10, 10});
        add_child(std::move(header));
        
        // Suggestion list
        auto suggestion_list = std::make_unique<suggestion_listbox>();
        suggestion_list->set_position({10, 30});
        suggestion_list->set_size({500, 200});
        add_child(std::move(suggestion_list));
        
        // Confidence indicator
        auto confidence_label = std::make_unique<text_element>();
        confidence_label->set_text("Confidence:");
        confidence_label->set_position({520, 30});
        add_child(std::move(confidence_label));
        
        auto confidence_display = std::make_unique<confidence_display>();
        confidence_display->set_position({520, 50});
        confidence_display->set_size({100, 30});
        add_child(std::move(confidence_display));
        
        // Rationale
        auto rationale_label = std::make_unique<text_element>();
        rationale_label->set_text("Rationale:");
        rationale_label->set_position({10, 240});
        add_child(std::move(rationale_label));
        
        auto rationale_display = std::make_unique<rationale_display>();
        rationale_display->set_position({10, 260});
        rationale_display->set_size({610, 80});
        add_child(std::move(rationale_display));
        
        // Predicted outcomes
        auto outcomes_label = std::make_unique<text_element>();
        outcomes_label->set_text("Predicted Outcomes:");
        outcomes_label->set_position({10, 350});
        add_child(std::move(outcomes_label));
        
        auto outcomes_list = std::make_unique<outcomes_listbox>();
        outcomes_list->set_position({10, 370});
        outcomes_list->set_size({610, 80});
        add_child(std::move(outcomes_list));
        
        // Apply suggestion button
        auto apply_button = std::make_unique<button_element>();
        apply_button->set_position({10, 460});
        apply_button->set_size({100, 30});
        apply_button->set_text("Apply");
        apply_button->on_click = [&]() {
            apply_suggestion(state);
        };
        add_child(std::move(apply_button));
        
        // Auto-suggest checkbox
        auto auto_suggest = std::make_unique<checkbox_element>();
        auto_suggest->set_position({120, 460});
        auto_suggest->set_text("Auto-suggest");
        auto_suggest->set_checked(state.events.auto_suggest);
        auto_suggest->on_change = [&](bool checked) {
            state.events.auto_suggest = checked;
        };
        add_child(std::move(auto_suggest));
    }
    
    void update_suggestions(sys::state& state, dcon::event_id event) {
        auto suggestions = state.ai.event_suggester.suggest_decisions(state, event);
        
        auto suggestion_list = get_child_by_index(1);
        suggestion_list->clear_items();
        
        for (auto& suggestion : suggestions) {
            auto item = std::make_unique<suggestion_list_item>();
            item->set_suggestion(suggestion);
            item->on_click = [&]() {
                show_suggestion_details(suggestion);
            };
            suggestion_list->add_item(std::move(item));
        }
        
        if (!suggestions.empty()) {
            // Update confidence
            auto confidence_display = get_child_by_index(3);
            confidence_display->set_confidence(suggestions[0].confidence);
            
            // Update rationale
            auto rationale_display = get_child_by_index(5);
            rationale_display->set_rationale(suggestions[0].rationale);
            
            // Update outcomes
            auto outcomes_list = get_child_by_index(7);
            outcomes_list->clear_items();
            
            for (auto& outcome : suggestions[0].predicted_outcomes) {
                auto item = std::make_unique<outcome_list_item>();
                item->set_text(outcome);
                outcomes_list->add_item(std::move(item));
            }
        }
    }
};

class suggestion_list_item : public listbox_item_base {
public:
    void set_suggestion(const event_suggester::decision_suggestion& suggestion) {
        auto text = "Recommended: " + get_decision_name(suggestion.decision) +
                   " (Confidence: " + format_percentage(suggestion.confidence) + ")";
        set_text(text);
        
        // Color based on confidence
        if (suggestion.confidence > 0.8f) {
            set_color({0.0f, 1.0f, 0.0f}); // Green (high confidence)
        } else if (suggestion.confidence > 0.6f) {
            set_color({1.0f, 1.0f, 0.0f}); // Yellow (medium confidence)
        } else {
            set_color({1.0f, 0.5f, 0.0f}); // Orange (low confidence)
        }
    }
};
```

### Step 5: Event Filtering

#### Filter by Type, Importance, etc.
**File:** `src/gui/gui_event_filter.cpp`
**Add:**
```cpp
class event_filter_window : public window_element_base {
public:
    void on_create(sys::state& state) override {
        window_element_base::on_create(state);
        
        // Filter by type
        auto type_label = std::make_unique<text_element>();
        type_label->set_text("Event Types:");
        type_label->set_position({10, 30});
        add_child(std::move(type_label));
        
        auto type_checkboxes = std::make_unique<event_type_checkboxes>();
        type_checkboxes->set_position({10, 50});
        type_checkboxes->set_size({200, 150});
        add_child(std::move(type_checkboxes));
        
        // Filter by importance
        auto importance_label = std::make_unique<text_element>();
        importance_label->set_text("Importance:");
        importance_label->set_position({220, 30});
        add_child(std::move(importance_label));
        
        auto importance_slider = std::make_unique<slider_element>();
        importance_slider->set_position({220, 50});
        importance_slider->set_range(0.0f, 1.0f);
        importance_slider->set_value(0.5f);
        importance_slider->on_change = [&](float value) {
            apply_filters(state);
        };
        add_child(std::move(importance_slider));
        
        // Filter by date range
        auto date_label = std::make_unique<text_element>();
        date_label->set_text("Date Range:");
        date_label->set_position({10, 210});
        add_child(std::move(date_label));
        
        auto start_date = std::make_unique<date_input>();
        start_date->set_position({10, 230});
        start_date->set_size({100, 30});
        add_child(std::move(start_date));
        
        auto end_date = std::make_unique<date_input>();
        end_date->set_position({120, 230});
        end_date->set_size({100, 30});
        add_child(std::move(end_date));
        
        // Apply filters button
        auto apply_button = std::make_unique<button_element>();
        apply_button->set_position({10, 270});
        apply_button->set_size({100, 30});
        apply_button->set_text("Apply");
        apply_button->on_click = [&]() {
            apply_filters(state);
        };
        add_child(std::move(apply_button));
        
        // Reset button
        auto reset_button = std::make_unique<button_element>();
        reset_button->set_position({120, 270});
        reset_button->set_size({100, 30});
        reset_button->set_text("Reset");
        reset_button->on_click = [&]() {
            reset_filters(state);
        };
        add_child(std::move(reset_button));
    }
    
    void apply_filters(sys::state& state) {
        // Get filter values
        auto types = get_selected_types();
        auto importance = get_importance_value();
        auto start = get_start_date();
        auto end = get_end_date();
        
        // Apply to event list
        state.events.filter_events(types, importance, start, end);
    }
    
    void reset_filters(sys::state& state) {
        // Reset all filters to default
        state.events.reset_filters();
    }
};
```

### Step 6: Success Criteria for Event & Decision System

- [ ] Event cards are visually appealing and informative
- [ ] Decision previews show consequences before choosing
- [ ] Event history timeline is useful for learning
- [ ] AI suggestions provide helpful recommendations
- [ ] Event filtering helps manage information overload
- [ ] Players understand event implications clearly
- [ ] Decision-making is strategic rather than reactive
- [ ] Event system is engaging and immersive

---

## Phase 4 Success Criteria

### Tooltip & Information System
- [ ] Tooltip caching reduces rebuilds by 50%+
- [ ] Pinned panel allows users to keep information visible
- [ ] Summary vs full tooltips reduce information overload
- [ ] Async computation prevents UI blocking
- [ ] Context-sensitive tooltips show relevant information

### Focus Management & Accessibility
- [ ] Keyboard navigation works across all UI screens
- [ ] Visual focus indicators are clear and visible
- [ ] Theme manager supports color-blind modes
- [ ] Font scaling works for accessibility
- [ ] Controller support is functional
- [ ] Screen reader integration provides useful announcements
- [ ] Players can customize UI to their needs

### Map & Visualization
- [ ] Dynamic overlays provide useful strategic information
- [ ] Smooth zoom and pan with intuitive controls
- [ ] Strategic view simplifies large-scale planning
- [ ] Visual feedback is clear and immediate
- [ ] Map search helps find locations quickly
- [ ] Event highlighting draws attention to important events

### Event & Decision System
- [ ] Event cards are visually appealing and informative
- [ ] Decision previews show consequences before choosing
- [ ] Event history timeline is useful for learning
- [ ] AI suggestions provide helpful recommendations
- [ ] Event filtering helps manage information overload

### Overall UI/UX
- [ ] UI is intuitive and discoverable for new players
- [ ] Tooltips provide relevant information without overwhelming
- [ ] Accessibility features support diverse player needs
- [ ] UI performance is smooth even in late-game
- [ ] Visual feedback is clear and immediate
- [ ] Players can customize UI to their preferences

---

## Quick Reference Commands

### Build & Test:
```bash
mkdir build && cd build
cmake -S .. -B . -G "Ninja" -DCMAKE_BUILD_TYPE=RelWithDebInfo
cmake --build . -- -j$(nproc)
ctest --output-on-failure --verbose
```

### Run UI Tests:
```bash
./build/tests/ui_tests --scenario late_game.sav --test_ui_performance
```

### Measure UI Performance:
```bash
# Run UI benchmark
./build/src/AliceExecutable --benchmark --ui --ticks 1000
```

---

## Phase 4 Summary

**Week 9:** Tooltip & Information System
- Tooltip caching with smart invalidation
- Pinned info panel for persistent information
- Summary vs full tooltips (two-level system)
- Async tooltip computation
- Context-sensitive tooltips

**Week 10:** Focus Management & Accessibility
- Central focus manager for keyboard navigation
- Visual focus indicators (focus rectangles)
- Theme manager with color-blind modes
- Font scaling for accessibility
- Controller support
- Screen reader integration

**Week 11:** Map & Visualization Modernization
- Dynamic overlays (supply, trade, politics, economy, etc.)
- Smooth zoom and pan with interpolation
- Strategic view with province aggregation
- Visual feedback system (animations, transitions)
- Event notifications
- Map search functionality
- Event highlighting

**Week 12:** Event & Decision System
- Modern event cards with visuals
- Decision previews with consequences
- Event history timeline
- AI decision suggestions
- Event filtering by type/importance/date

**Success:** UI is intuitive, accessible, and performant

---

**Phase 4 starts now. Let's modernize the UI/UX.**
