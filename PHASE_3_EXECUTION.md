# Phase 3 Execution: Gameplay Refinements & Modern Features

**Generated:** 2026-01-16T00:00:00Z
**Commit:** a90dd964a
**Branch:** main
**Phase:** 3 of 16-week improvement plan

---

## Phase 3 Overview

**Goal:** Modernize core gameplay systems (economy, military, diplomacy, politics, technology) with better feedback, automation, and player tools

**Duration:** 4 weeks (Weeks 5-8)

**Priority:** HIGH

**Success Criteria:**
- Players can understand economic health at a glance
- Military logistics and combat outcomes are explainable
- Diplomatic relationships are visible and interactive
- Political changes are understandable and predictable
- Technology progression is strategic and meaningful
- Late-game management is less tedious with automation tools

---

## Week 5: Economy Modernization

### Objective
Improve economic gameplay with better feedback, visualization, and automation

### Step 1: Market Overlays & Visualization

#### Price History Tracking
**File:** `src/economy/economy.cpp`
**Add:**
```cpp
// Add price history to market data
struct market_history {
    std::vector<float> price_history;  // Last 100 ticks
    std::vector<float> supply_history; // Last 100 ticks
    std::vector<float> demand_history; // Last 100 ticks
    size_t history_index = 0;
    
    void record_tick(float price, float supply, float demand) {
        if (history_index >= price_history.size()) {
            history_index = 0;
        }
        price_history[history_index] = price;
        supply_history[history_index] = supply;
        demand_history[history_index] = demand;
        history_index++;
    }
    
    float get_average_price(size_t window = 10) const {
        float sum = 0.0f;
        size_t count = 0;
        for (size_t i = 0; i < window && i < price_history.size(); i++) {
            size_t idx = (history_index + price_history.size() - 1 - i) % price_history.size();
            sum += price_history[idx];
            count++;
        }
        return count > 0 ? sum / count : 0.0f;
    }
};

// Add to market data
struct market_data {
    dcon::commodity_id commodity;
    float current_price;
    float current_supply;
    float current_demand;
    market_history history;
    
    // For UI
    float get_price_trend(size_t window = 10) const {
        if (history.price_history.size() < window) return 0.0f;
        float old_avg = history.get_average_price(window);
        float new_avg = history.get_average_price(1);
        return new_avg - old_avg;
    }
};
```

#### Market Overlay UI
**File:** `src/gui/gui_market_overlay.cpp`
**Add:**
```cpp
// Market overlay window
class market_overlay_window : public window_element_base {
public:
    void on_create(sys::state& state) override {
        window_element_base::on_create(state);
        
        // Create commodity list
        auto commodity_list = std::make_unique<commodity_listbox>();
        commodity_list->set_position({10, 30});
        commodity_list->set_size({400, 300});
        add_child(std::move(commodity_list));
        
        // Create price chart
        auto price_chart = std::make_unique<price_chart_element>();
        price_chart->set_position({420, 30});
        price_chart->set_size({300, 200});
        add_child(std::move(price_chart));
        
        // Create supply/demand chart
        auto sd_chart = std::make_unique<supply_demand_chart>();
        sd_chart->set_position({420, 240});
        sd_chart->set_size({300, 150});
        add_child(std::move(sd_chart));
    }
};

// Commodity list item
class commodity_list_item : public listbox_item_base {
public:
    void update(sys::state& state, dcon::commodity_id commodity) override {
        auto& market = state.economy.markets[commodity];
        
        // Update text fields
        price_text->set_text(format_price(market.current_price));
        trend_text->set_text(format_trend(market.get_price_trend()));
        supply_text->set_text(format_quantity(market.current_supply));
        demand_text->set_text(format_quantity(market.current_demand));
        
        // Color coding
        if (market.get_price_trend() > 0.1f) {
            price_text->set_color({1.0f, 0.3f, 0.3f}); // Red for rising
        } else if (market.get_price_trend() < -0.1f) {
            price_text->set_color({0.3f, 1.0f, 0.3f}); // Green for falling
        }
    }
};
```

#### Price Chart Element
**File:** `src/gui/gui_market_charts.cpp`
**Add:**
```cpp
class price_chart_element : public element_base {
public:
    void render(sys::state& state) override {
        auto& market = state.economy.markets[selected_commodity];
        
        // Draw price history line
        draw_line_chart(market.history.price_history, 
            market.history.history_index,
            {0.0f, 1.0f, 0.0f}); // Green line
        
        // Draw supply/demand as bars
        draw_bar_chart(market.history.supply_history,
            market.history.demand_history,
            market.history.history_index);
        
        // Draw current price indicator
        draw_current_price(market.current_price);
    }
    
    void on_click(sys::state& state, int x, int y) override {
        // Click to select commodity
        selected_commodity = get_commodity_at_position(x, y);
        invalidate();
    }
};
```

### Step 2: Supply Chain Visualization

#### Sankey Diagram for Goods Flow
**File:** `src/gui/gui_supply_chain.cpp`
**Add:**
```cpp
class supply_chain_sankey : public element_base {
public:
    void update(sys::state& state, dcon::commodity_id commodity) {
        // Build supply chain graph
        auto chain = build_supply_chain(state, commodity);
        
        // Calculate node positions
        calculate_positions(chain);
        
        // Draw flows
        for (auto& flow : chain.flows) {
            draw_flow(flow);
        }
        
        // Draw nodes
        for (auto& node : chain.nodes) {
            draw_node(node);
        }
    }
    
    struct supply_chain {
        struct node {
            std::string name;
            dcon::province_id province;
            float quantity;
            node_type type; // producer, consumer, factory, market
            point position;
        };
        
        struct flow {
            node* from;
            node* to;
            float quantity;
            float width; // Visual width based on quantity
        };
        
        std::vector<node> nodes;
        std::vector<flow> flows;
    };
    
    supply_chain build_supply_chain(sys::state& state, dcon::commodity_id commodity) {
        supply_chain chain;
        
        // Find producers
        for (auto province : state.world.province_range()) {
            if (province.produces_commodity(commodity)) {
                supply_chain::node node;
                node.name = "Producer: " + get_province_name(province);
                node.province = province;
                node.quantity = province.get_production(commodity);
                node.type = node_type::producer;
                chain.nodes.push_back(node);
            }
        }
        
        // Find factories that use this commodity
        for (auto factory : state.world.factory_range()) {
            if (factory.consumes_commodity(commodity)) {
                supply_chain::node node;
                node.name = "Factory: " + get_factory_name(factory);
                node.province = factory.province;
                node.quantity = factory.get_consumption(commodity);
                node.type = node_type::factory;
                chain.nodes.push_back(node);
            }
        }
        
        // Find consumers
        for (auto pop : state.world.pop_range()) {
            if (pop.consumes_commodity(commodity)) {
                supply_chain::node node;
                node.name = "Consumers: " + get_pop_type_name(pop);
                node.province = pop.province;
                node.quantity = pop.get_consumption(commodity);
                node.type = node_type::consumer;
                chain.nodes.push_back(node);
            }
        }
        
        // Build flows between nodes
        // ... flow calculation logic ...
        
        return chain;
    }
};
```

### Step 3: Economic Advisors

#### AI-Run Economic Policies
**File:** `src/economy/economy_advisor.cpp`
**Add:**
```cpp
class economic_advisor {
public:
    enum class policy_aggressiveness {
        conservative,  // Minimal changes, stable
        moderate,       // Balanced approach
        aggressive,     // Rapid changes for growth
    };
    
    struct advisor_settings {
        policy_aggressiveness aggressiveness;
        bool auto_build_factories;
        bool auto_upgrade_factories;
        bool auto_adjust_tariffs;
        bool auto_adjust_taxes;
    };
    
    advisor_settings settings;
    
    void update(sys::state& state) {
        if (!settings.auto_build_factories) return;
        
        // Analyze economic situation
        auto analysis = analyze_economy(state);
        
        // Make recommendations
        auto recommendations = generate_recommendations(state, analysis);
        
        // Execute based on aggressiveness
        execute_recommendations(state, recommendations);
    }
    
    struct economic_analysis {
        float gdp_growth_rate;
        float unemployment_rate;
        float inflation_rate;
        std::vector<dcon::commodity_id> shortages;
        std::vector<dcon::commodity_id> surpluses;
        std::vector<dcon::factory_type_id> needed_factories;
    };
    
    economic_analysis analyze_economy(sys::state& state) {
        economic_analysis analysis;
        
        // Calculate GDP growth
        auto& gdp_history = state.economy.gdp_history;
        if (gdp_history.size() >= 10) {
            float old_gdp = gdp_history[gdp_history.size() - 10];
            float new_gdp = gdp_history.back();
            analysis.gdp_growth_rate = (new_gdp - old_gdp) / old_gdp;
        }
        
        // Calculate unemployment
        analysis.unemployment_rate = calculate_unemployment_rate(state);
        
        // Find shortages
        for (auto commodity : state.world.commodity_range()) {
            auto& market = state.economy.markets[commodity];
            if (market.current_demand > market.current_supply * 1.5f) {
                analysis.shortages.push_back(commodity);
            }
        }
        
        // Find needed factories
        analysis.needed_factories = identify_needed_factories(state, analysis.shortages);
        
        return analysis;
    }
    
    std::vector<factory_recommendation> generate_recommendations(
        sys::state& state, 
        const economic_analysis& analysis
    ) {
        std::vector<factory_recommendation> recommendations;
        
        for (auto factory_type : analysis.needed_factories) {
            factory_recommendation rec;
            rec.factory_type = factory_type;
            rec.priority = calculate_priority(state, factory_type, analysis);
            rec.expected_roi = estimate_roi(state, factory_type);
            
            // Filter by aggressiveness
            if (settings.aggressiveness == policy_aggressiveness::conservative) {
                if (rec.expected_roi < 0.15f) continue; // Require high ROI
            } else if (settings.aggressiveness == policy_aggressiveness::aggressive) {
                if (rec.expected_roi < 0.05f) continue; // Accept lower ROI
            }
            
            recommendations.push_back(rec);
        }
        
        // Sort by priority
        std::sort(recommendations.begin(), recommendations.end(),
            [](const auto& a, const auto& b) { return a.priority > b.priority; });
        
        return recommendations;
    }
    
    void execute_recommendations(sys::state& state, const std::vector<factory_recommendation>& recommendations) {
        // Limit number of factories to build based on budget
        size_t max_builds = calculate_max_builds(state);
        
        for (size_t i = 0; i < std::min(max_builds, recommendations.size()); i++) {
            auto& rec = recommendations[i];
            
            // Find suitable province
            auto province = find_suitable_province(state, rec.factory_type);
            
            if (province) {
                // Queue factory construction
                queue_factory_construction(state, province, rec.factory_type);
            }
        }
    }
};
```

#### Advisor UI
**File:** `src/gui/gui_economic_advisor.cpp`
**Add:**
```cpp
class economic_advisor_window : public window_element_base {
public:
    void on_create(sys::state& state) override {
        window_element_base::on_create(state);
        
        // Settings panel
        auto settings_panel = std::make_unique<advisor_settings_panel>();
        settings_panel->set_position({10, 30});
        settings_panel->set_size({300, 200});
        add_child(std::move(settings_panel));
        
        // Recommendations list
        auto recommendations_list = std::make_unique<recommendations_listbox>();
        recommendations_list->set_position({320, 30});
        recommendations_list->set_size({400, 300});
        add_child(std::move(recommendations_list));
        
        // Economic dashboard
        auto dashboard = std::make_unique<economic_dashboard>();
        dashboard->set_position({10, 240});
        dashboard->set_size({710, 200});
        add_child(std::move(dashboard));
    }
};

class advisor_settings_panel : public panel_base {
public:
    void on_create(sys::state& state) override {
        // Aggressiveness slider
        auto aggressiveness_label = std::make_unique<text_element>();
        aggressiveness_label->set_text("Advisor Aggressiveness:");
        add_child(std::move(aggressiveness_label));
        
        auto aggressiveness_slider = std::make_unique<slider_element>();
        aggressiveness_slider->set_position({0, 20});
        aggressiveness_slider->set_range(0.0f, 1.0f);
        aggressiveness_slider->set_value(state.economy.advisor.settings.aggressiveness);
        aggressiveness_slider->on_change = [&](float value) {
            state.economy.advisor.settings.aggressiveness = 
                static_cast<economic_advisor::policy_aggressiveness>(value);
        };
        add_child(std::move(aggressiveness_slider));
        
        // Checkboxes for automation
        auto build_checkbox = std::make_unique<checkbox_element>();
        build_checkbox->set_position({0, 50});
        build_checkbox->set_text("Auto-build factories");
        build_checkbox->set_checked(state.economy.advisor.settings.auto_build_factories);
        build_checkbox->on_change = [&](bool checked) {
            state.economy.advisor.settings.auto_build_factories = checked;
        };
        add_child(std::move(build_checkbox));
        
        // ... more checkboxes ...
    }
};
```

### Step 4: Data-Driven Factory Recipes

#### JSON Factory Definitions
**File:** `data/factories/recipes.json`
```json
{
  "factories": [
    {
      "id": "steel_factory",
      "name": "Steel Mill",
      "inputs": [
        {"commodity": "iron_ore", "amount": 2.0},
        {"commodity": "coal", "amount": 1.5}
      ],
      "outputs": [
        {"commodity": "steel", "amount": 1.0}
      ],
      "throughput_multiplier": 1.0,
      "output_multiplier": 1.0,
      "base_cost": 10000,
      "upkeep": 100,
      "required_technology": "basic_steel_production",
      "production_methods": [
        {
          "name": "basic",
          "inputs_multiplier": 1.0,
          "output_multiplier": 1.0,
          "unlock_technology": null
        },
        {
          "name": "bessemer",
          "inputs_multiplier": 0.8,
          "output_multiplier": 1.2,
          "unlock_technology": "bessemer_process"
        }
      ]
    }
  ]
}
```

#### Recipe Loader
**File:** `src/economy/factory_recipes.cpp`
**Add:**
```cpp
struct factory_recipe {
    dcon::factory_type_id id;
    std::string name;
    std::vector<input_output> inputs;
    std::vector<input_output> outputs;
    float throughput_multiplier;
    float output_multiplier;
    float base_cost;
    float upkeep;
    dcon::technology_id required_technology;
    std::vector<production_method> production_methods;
};

std::vector<factory_recipe> load_factory_recipes(sys::state& state) {
    std::vector<factory_recipe> recipes;
    
    // Load from JSON
    auto json = load_json_file("data/factories/recipes.json");
    
    for (auto& factory_json : json["factories"]) {
        factory_recipe recipe;
        
        // Parse fields
        recipe.id = get_factory_type_id(factory_json["id"]);
        recipe.name = factory_json["name"];
        
        // Parse inputs
        for (auto& input_json : factory_json["inputs"]) {
            input_output input;
            input.commodity = get_commodity_id(input_json["commodity"]);
            input.amount = input_json["amount"];
            recipe.inputs.push_back(input);
        }
        
        // Parse outputs
        for (auto& output_json : factory_json["outputs"]) {
            input_output output;
            output.commodity = get_commodity_id(output_json["commodity"]);
            output.amount = output_json["amount"];
            recipe.outputs.push_back(output);
        }
        
        // Parse other fields
        recipe.throughput_multiplier = factory_json["throughput_multiplier"];
        recipe.output_multiplier = factory_json["output_multiplier"];
        recipe.base_cost = factory_json["base_cost"];
        recipe.upkeep = factory_json["upkeep"];
        recipe.required_technology = get_technology_id(factory_json["required_technology"]);
        
        // Parse production methods
        for (auto& method_json : factory_json["production_methods"]) {
            production_method method;
            method.name = method_json["name"];
            method.inputs_multiplier = method_json["inputs_multiplier"];
            method.output_multiplier = method_json["output_multiplier"];
            method.unlock_technology = get_technology_id(method_json["unlock_technology"]);
            recipe.production_methods.push_back(method);
        }
        
        recipes.push_back(recipe);
    }
    
    return recipes;
}
```

### Step 5: Automation Tools

#### Factory Build Queue
**File:** `src/economy/factory_queue.cpp`
**Add:**
```cpp
struct factory_queue_item {
    dcon::province_id province;
    dcon::factory_type_id factory_type;
    dcon::production_method_id production_method;
    float priority; // 0.0 to 1.0
    bool auto_upgrade;
    size_t quantity; // Number of factories to build
};

class factory_queue {
public:
    std::vector<factory_queue_item> queue;
    
    void add_item(const factory_queue_item& item) {
        queue.push_back(item);
        // Sort by priority
        std::sort(queue.begin(), queue.end(),
            [](const auto& a, const auto& b) { return a.priority > b.priority; });
    }
    
    void process_queue(sys::state& state) {
        size_t processed = 0;
        size_t max_per_tick = 3; // Limit to prevent overwhelming economy
        
        for (auto& item : queue) {
            if (processed >= max_per_tick) break;
            
            if (can_build_factory(state, item)) {
                build_factory(state, item);
                item.quantity--;
                
                if (item.quantity == 0) {
                    // Remove from queue
                    queue.erase(std::remove(queue.begin(), queue.end(), item), queue.end());
                }
                
                processed++;
            }
        }
    }
    
    bool can_build_factory(sys::state& state, const factory_queue_item& item) {
        // Check budget
        auto cost = calculate_factory_cost(state, item.factory_type);
        if (state.economy.budget < cost) return false;
        
        // Check technology
        auto recipe = get_factory_recipe(item.factory_type);
        if (recipe.required_technology && !state.tech.is_researched(recipe.required_technology)) {
            return false;
        }
        
        // Check province suitability
        if (!is_province_suitable(state, item.province, item.factory_type)) {
            return false;
        }
        
        return true;
    }
    
    void build_factory(sys::state& state, const factory_queue_item& item) {
        // Deduct budget
        auto cost = calculate_factory_cost(state, item.factory_type);
        state.economy.budget -= cost;
        
        // Create factory
        auto factory = state.world.create_factory();
        factory.province = item.province;
        factory.type = item.factory_type;
        factory.production_method = item.production_method;
        factory.construction_progress = 0.0f;
        
        // Log construction
        log_construction(state, factory, cost);
    }
};
```

#### Factory Queue UI
**File:** `src/gui/gui_factory_queue.cpp`
**Add:**
```cpp
class factory_queue_window : public window_element_base {
public:
    void on_create(sys::state& state) override {
        window_element_base::on_create(state);
        
        // Queue list
        auto queue_list = std::make_unique<queue_listbox>();
        queue_list->set_position({10, 30});
        queue_list->set_size({400, 300});
        add_child(std::move(queue_list));
        
        // Add factory button
        auto add_button = std::make_unique<button_element>();
        add_button->set_position({420, 30});
        add_button->set_size({100, 30});
        add_button->set_text("Add Factory");
        add_button->on_click = [&]() {
            show_factory_selector();
        };
        add_child(std::move(add_button));
        
        // Priority slider
        auto priority_slider = std::make_unique<slider_element>();
        priority_slider->set_position({420, 70});
        priority_slider->set_range(0.0f, 1.0f);
        priority_slider->set_value(0.5f);
        add_child(std::move(priority_slider));
    }
    
    void show_factory_selector() {
        auto selector = std::make_unique<factory_selector_window>();
        selector->set_position({100, 100});
        selector->set_size({500, 400});
        // ... show modal ...
    }
};

class factory_selector_window : public window_element_base {
public:
    void on_create(sys::state& state) override {
        // List all available factory types
        for (auto factory_type : state.world.factory_type_range()) {
            auto recipe = get_factory_recipe(factory_type);
            
            auto item = std::make_unique<factory_type_list_item>();
            item->set_factory_type(factory_type);
            item->set_text(recipe.name);
            item->on_click = [&]() {
                // Add to queue
                factory_queue_item queue_item;
                queue_item.factory_type = factory_type;
                queue_item.priority = 0.5f;
                queue_item.quantity = 1;
                state.economy.factory_queue.add_item(queue_item);
                close();
            };
            add_child(std::move(item));
        }
    }
};
```

### Step 6: Economic Dashboard

#### Real-Time Metrics
**File:** `src/gui/gui_economic_dashboard.cpp`
**Add:**
```cpp
class economic_dashboard : public panel_base {
public:
    void on_create(sys::state& state) override {
        // GDP chart
        auto gdp_chart = std::make_unique<gdp_chart_element>();
        gdp_chart->set_position({0, 0});
        gdp_chart->set_size({200, 100});
        add_child(std::move(gdp_chart));
        
        // Unemployment indicator
        auto unemployment = std::make_unique<metric_element>();
        unemployment->set_position({210, 0});
        unemployment->set_size({100, 50});
        unemployment->set_label("Unemployment");
        unemployment->set_value_calculator([&]() {
            return calculate_unemployment_rate(state);
        });
        add_child(std::move(unemployment));
        
        // Inflation indicator
        auto inflation = std::make_unique<metric_element>();
        inflation->set_position({320, 0});
        inflation->set_size({100, 50});
        inflation->set_label("Inflation");
        inflation->set_value_calculator([&]() {
            return calculate_inflation_rate(state);
        });
        add_child(std::move(inflation));
        
        // Budget balance
        auto budget = std::make_unique<metric_element>();
        budget->set_position({430, 0});
        budget->set_size({100, 50});
        budget->set_label("Budget");
        budget->set_value_calculator([&]() {
            return state.economy.budget;
        });
        add_child(std::move(budget));
        
        // Shortage list
        auto shortage_list = std::make_unique<shortage_listbox>();
        shortage_list->set_position({0, 110});
        shortage_list->set_size({530, 90});
        add_child(std::move(shortage_list));
    }
};

class gdp_chart_element : public element_base {
public:
    void render(sys::state& state) override {
        auto& gdp_history = state.economy.gdp_history;
        
        if (gdp_history.size() < 2) return;
        
        // Draw GDP line
        draw_line_chart(gdp_history, 
            gdp_history.size() - 1,
            {0.0f, 0.5f, 1.0f}); // Blue line
        
        // Draw growth rate
        if (gdp_history.size() >= 10) {
            float old_gdp = gdp_history[gdp_history.size() - 10];
            float new_gdp = gdp_history.back();
            float growth_rate = (new_gdp - old_gdp) / old_gdp;
            
            // Display growth rate
            draw_text(10, 10, format_percentage(growth_rate));
        }
    }
};
```

### Step 7: Success Criteria for Economy Modernization

- [ ] Market overlays show price history and trends
- [ ] Supply chain visualization (Sankey diagrams) for goods flow
- [ ] Economic advisors provide actionable recommendations
- [ ] Data-driven factory recipes (JSON) for moddability
- [ ] Factory build queue with priority system
- [ ] Real-time economic dashboard with key metrics
- [ ] Players can understand economic health at a glance
- [ ] Late-game economic management is less tedious

---

## Week 6: Military Modernization

### Objective
Improve military gameplay with better logistics, combat feedback, and AI war planning

### Step 1: Supply Overlays

#### Visual Supply Indicators
**File:** `src/gui/gui_supply_overlay.cpp`
**Add:**
```cpp
class supply_overlay : public map_overlay_base {
public:
    void render(sys::state& state) override {
        for (auto province : state.world.province_range()) {
            float supply_limit = calculate_supply_limit(state, province);
            float current_supply = calculate_current_supply(state, province);
            float utilization = current_supply / supply_limit;
            
            // Color code based on utilization
            color color;
            if (utilization > 0.9f) {
                color = {1.0f, 0.0f, 0.0f}; // Red (critical)
            } else if (utilization > 0.7f) {
                color = {1.0f, 0.5f, 0.0f}; // Orange (warning)
            } else if (utilization > 0.5f) {
                color = {1.0f, 1.0f, 0.0f}; // Yellow (moderate)
            } else {
                color = {0.0f, 1.0f, 0.0f}; // Green (good)
            }
            
            // Draw province overlay
            draw_province_overlay(province, color, 0.3f); // 30% opacity
        }
    }
    
    float calculate_supply_limit(sys::state& state, dcon::province_id province) {
        float base = 100.0f; // Base supply capacity
        
        // Railroad bonus
        if (province.has_railroad()) {
            base *= 1.5f;
        }
        
        // Naval base bonus
        if (province.has_naval_base()) {
            base *= 2.0f;
        }
        
        // Terrain penalty
        if (province.is_mountainous()) {
            base *= 0.5f;
        }
        
        return base;
    }
    
    float calculate_current_supply(sys::state& state, dcon::province_id province) {
        float supply = 0.0f;
        
        // Sum of all units in province
        for (auto unit : state.world.province_get_army_units(province)) {
            supply += unit.get_supply_consumption();
        }
        
        return supply;
    }
};
```

#### Supply Route Visualization
**File:** `src/gui/gui_supply_routes.cpp`
**Add:**
```cpp
class supply_route_overlay : public map_overlay_base {
public:
    void render(sys::state& state) override {
        // Draw supply routes as lines
        for (auto& route : state.military.supply_routes) {
            draw_supply_route(route);
        }
    }
    
    void draw_supply_route(const supply_route& route) {
        // Calculate line width based on throughput
        float width = route.throughput / 100.0f;
        
        // Color based on efficiency
        color line_color;
        if (route.efficiency > 0.8f) {
            line_color = {0.0f, 1.0f, 0.0f}; // Green
        } else if (route.efficiency > 0.5f) {
            line_color = {1.0f, 1.0f, 0.0f}; // Yellow
        } else {
            line_color = {1.0f, 0.0f, 0.0f}; // Red
        }
        
        // Draw line between provinces
        draw_line(route.from, route.to, line_color, width);
        
        // Draw throughput indicator
        draw_text_at_midpoint(route, format_quantity(route.throughput));
    }
};
```

### Step 2: Battle Reports

#### Combat Outcome Analysis
**File:** `src/military/battle_report.cpp`
**Add:**
```cpp
struct battle_report {
    dcon::battle_id battle;
    dcon::province_id location;
    dcon::nation_id attacker;
    dcon::nation_id defender;
    
    // Combat factors
    struct combat_factor {
        std::string name;
        float value; // e.g., 1.2 for 20% bonus
        std::string description;
    };
    
    std::vector<combat_factor> attacker_factors;
    std::vector<combat_factor> defender_factors;
    
    // Results
    struct casualty_report {
        dcon::nation_id nation;
        int32_t losses;
        int32_t remaining;
    };
    
    std::vector<casualty_report> casualties;
    
    // Outcome
    enum class outcome {
        attacker_victory,
        defender_victory,
        stalemate,
        retreat
    };
    
    outcome result;
    
    // Generate report
    static battle_report generate(sys::state& state, dcon::battle_id battle) {
        battle_report report;
        report.battle = battle;
        
        // Get battle data
        auto battle_obj = state.world.battle_get(battle);
        report.attacker = battle_obj.attacker;
        report.defender = battle_obj.defender;
        report.location = battle_obj.location;
        
        // Analyze combat factors
        report.attacker_factors = analyze_factors(state, battle_obj, true);
        report.defender_factors = analyze_factors(state, battle_obj, false);
        
        // Calculate casualties
        report.casualties = calculate_casualties(state, battle_obj);
        
        // Determine outcome
        report.result = determine_outcome(state, battle_obj);
        
        return report;
    }
    
    static std::vector<combat_factor> analyze_factors(
        sys::state& state, 
        dcon::battle battle, 
        bool is_attacker
    ) {
        std::vector<combat_factor> factors;
        
        // Technology advantage
        auto tech_advantage = calculate_tech_advantage(state, battle, is_attacker);
        if (tech_advantage != 1.0f) {
            factors.push_back({
                "Technology",
                tech_advantage,
                format_tech_advantage(tech_advantage)
            });
        }
        
        // Morale
        auto morale = calculate_morale(state, battle, is_attacker);
        factors.push_back({
            "Morale",
            morale,
            format_morale(morale)
        });
        
        // Terrain
        auto terrain_bonus = calculate_terrain_bonus(state, battle, is_attacker);
        if (terrain_bonus != 1.0f) {
            factors.push_back({
                "Terrain",
                terrain_bonus,
                format_terrain_bonus(terrain_bonus)
            });
        }
        
        // Leadership
        auto leadership = calculate_leadership(state, battle, is_attacker);
        factors.push_back({
            "Leadership",
            leadership,
            format_leadership(leadership)
        });
        
        // Entrenchment (for defender)
        if (!is_attacker) {
            auto entrenchment = calculate_entrenchment(state, battle);
            factors.push_back({
                "Entrenchment",
                entrenchment,
                format_entrenchment(entrenchment)
            });
        }
        
        return factors;
    }
};
```

#### Battle Report UI
**File:** `src/gui/gui_battle_report.cpp`
**Add:**
```cpp
class battle_report_window : public window_element_base {
public:
    void on_create(sys::state& state) override {
        window_element_base::on_create(state);
        
        // Header
        auto header = std::make_unique<text_element>();
        header->set_text("Battle Report");
        header->set_font_size(16);
        header->set_position({10, 10});
        add_child(std::move(header));
        
        // Attacker factors
        auto attacker_label = std::make_unique<text_element>();
        attacker_label->set_text("Attacker Factors:");
        attacker_label->set_position({10, 40});
        add_child(std::move(attacker_label));
        
        auto attacker_factors = std::make_unique<factors_listbox>();
        attacker_factors->set_position({10, 60});
        attacker_factors->set_size({300, 150});
        add_child(std::move(attacker_factors));
        
        // Defender factors
        auto defender_label = std::make_unique<text_element>();
        defender_label->set_text("Defender Factors:");
        defender_label->set_position({320, 40});
        add_child(std::move(defender_label));
        
        auto defender_factors = std::make_unique<factors_listbox>();
        defender_factors->set_position({320, 60});
        defender_factors->set_size({300, 150});
        add_child(std::move(defender_factors));
        
        // Casualties
        auto casualties_label = std::make_unique<text_element>();
        casualties_label->set_text("Casualties:");
        casualties_label->set_position({10, 220});
        add_child(std::move(casualties_label));
        
        auto casualties_list = std::make_unique<casualties_listbox>();
        casualties_list->set_position({10, 240});
        casualties_list->set_size({610, 100});
        add_child(std::move(casualties_list));
        
        // Outcome
        auto outcome = std::make_unique<text_element>();
        outcome->set_position({10, 350});
        outcome->set_font_size(14);
        outcome->set_color({1.0f, 1.0f, 1.0f});
        add_child(std::move(outcome));
        
        // Close button
        auto close_button = std::make_unique<button_element>();
        close_button->set_position({250, 380});
        close_button->set_size({100, 30});
        close_button->set_text("Close");
        close_button->on_click = [&]() {
            close();
        };
        add_child(std::move(close_button));
    }
    
    void update(sys::state& state, dcon::battle_id battle) {
        auto report = battle_report::generate(state, battle);
        
        // Update factors lists
        attacker_factors->update(state, report.attacker_factors);
        defender_factors->update(state, report.defender_factors);
        
        // Update casualties
        casualties_list->update(state, report.casualties);
        
        // Update outcome
        auto outcome_text = format_outcome(report.result);
        get_child_by_index(5)->set_text(outcome_text); // Outcome element
    }
};

class factors_listbox : public listbox_base {
public:
    void update(sys::state& state, const std::vector<battle_report::combat_factor>& factors) {
        clear_items();
        
        for (auto& factor : factors) {
            auto item = std::make_unique<factor_list_item>();
            item->set_factor(factor);
            add_item(std::move(item));
        }
    }
};

class factor_list_item : public listbox_item_base {
public:
    void set_factor(const battle_report::combat_factor& factor) {
        auto text = factor.name + ": " + format_factor(factor.value) + " (" + factor.description + ")";
        set_text(text);
        
        // Color code
        if (factor.value > 1.0f) {
            set_color({0.3f, 1.0f, 0.3f}); // Green (bonus)
        } else if (factor.value < 1.0f) {
            set_color({1.0f, 0.3f, 0.3f}); // Red (penalty)
        } else {
            set_color({1.0f, 1.0f, 1.0f}); // White (neutral)
        }
    }
};
```

### Step 3: Automated Logistics

#### Transport Suggestions
**File:** `src/military/logistics_suggestions.cpp`
**Add:**
```cpp
class logistics_suggester {
public:
    struct transport_suggestion {
        dcon::province_id from;
        dcon::province_id to;
        dcon::commodity_id commodity;
        float quantity;
        float priority; // 0.0 to 1.0
        std::string reason;
    };
    
    std::vector<transport_suggestion> generate_suggestions(sys::state& state) {
        std::vector<transport_suggestion> suggestions;
        
        // Find supply shortages
        for (auto province : state.world.province_range()) {
            auto shortages = find_supply_shortages(state, province);
            
            for (auto& shortage : shortages) {
                // Find sources
                auto sources = find_supply_sources(state, province, shortage.commodity);
                
                for (auto& source : sources) {
                    transport_suggestion suggestion;
                    suggestion.from = source.province;
                    suggestion.to = province;
                    suggestion.commodity = shortage.commodity;
                    suggestion.quantity = shortage.quantity;
                    suggestion.priority = calculate_priority(state, province, shortage);
                    suggestion.reason = shortage.reason;
                    
                    suggestions.push_back(suggestion);
                }
            }
        }
        
        // Sort by priority
        std::sort(suggestions.begin(), suggestions.end(),
            [](const auto& a, const auto& b) { return a.priority > b.priority; });
        
        return suggestions;
    }
    
    struct supply_shortage {
        dcon::commodity_id commodity;
        float quantity;
        std::string reason;
    };
    
    std::vector<supply_shortage> find_supply_shortages(sys::state& state, dcon::province_id province) {
        std::vector<supply_shortage> shortages;
        
        // Check unit supply needs
        for (auto unit : state.world.province_get_army_units(province)) {
            auto needs = unit.get_supply_needs();
            for (auto& need : needs) {
                float available = get_available_supply(state, province, need.commodity);
                if (available < need.amount) {
                    shortages.push_back({
                        need.commodity,
                        need.amount - available,
                        "Unit supply deficit"
                    });
                }
            }
        }
        
        return shortages;
    }
    
    struct supply_source {
        dcon::province_id province;
        float available;
    };
    
    std::vector<supply_source> find_supply_sources(
        sys::state& state, 
        dcon::province_id target, 
        dcon::commodity_id commodity
    ) {
        std::vector<supply_source> sources;
        
        // Find nearby provinces with surplus
        for (auto province : state.world.province_range()) {
            float available = get_available_supply(state, province, commodity);
            if (available > 0) {
                sources.push_back({province, available});
            }
        }
        
        // Sort by distance
        std::sort(sources.begin(), sources.end(),
            [&](const auto& a, const auto& b) {
                float dist_a = province::distance(state, a.province, target);
                float dist_b = province::distance(state, b.province, target);
                return dist_a < dist_b;
            });
        
        return sources;
    }
};
```

#### Logistics Suggestions UI
**File:** `src/gui/gui_logistics.cpp`
**Add:**
```cpp
class logistics_window : public window_element_base {
public:
    void on_create(sys::state& state) override {
        window_element_base::on_create(state);
        
        // Suggestions list
        auto suggestions_list = std::make_unique<suggestions_listbox>();
        suggestions_list->set_position({10, 30});
        suggestions_list->set_size({600, 300});
        add_child(std::move(suggestions_list));
        
        // Refresh button
        auto refresh_button = std::make_unique<button_element>();
        refresh_button->set_position({620, 30});
        refresh_button->set_size({80, 30});
        refresh_button->set_text("Refresh");
        refresh_button->on_click = [&]() {
            update_suggestions(state);
        };
        add_child(std::move(refresh_button));
        
        // Auto-apply checkbox
        auto auto_apply = std::make_unique<checkbox_element>();
        auto_apply->set_position({10, 340});
        auto_apply->set_text("Auto-apply suggestions");
        auto_apply->set_checked(state.military.auto_apply_logistics);
        auto_apply->on_change = [&](bool checked) {
            state.military.auto_apply_logistics = checked;
        };
        add_child(std::move(auto_apply));
    }
    
    void update_suggestions(sys::state& state) {
        auto suggestions = state.military.logistics_suggester.generate_suggestions(state);
        suggestions_list->update(state, suggestions);
    }
};

class suggestions_listbox : public listbox_base {
public:
    void update(sys::state& state, const std::vector<logistics_suggester::transport_suggestion>& suggestions) {
        clear_items();
        
        for (auto& suggestion : suggestions) {
            auto item = std::make_unique<suggestion_list_item>();
            item->set_suggestion(suggestion);
            item->on_click = [&]() {
                // Apply suggestion
                apply_transport_suggestion(state, suggestion);
            };
            add_item(std::move(item));
        }
    }
};

class suggestion_list_item : public listbox_item_base {
public:
    void set_suggestion(const logistics_suggester::transport_suggestion& suggestion) {
        auto text = "Transport " + format_quantity(suggestion.quantity) + " " +
            get_commodity_name(suggestion.commodity) + " from " +
            get_province_name(suggestion.from) + " to " +
            get_province_name(suggestion.to) + " (Priority: " +
            format_priority(suggestion.priority) + ")";
        
        set_text(text);
        
        // Color based on priority
        if (suggestion.priority > 0.8f) {
            set_color({1.0f, 0.3f, 0.3f}); // Red (urgent)
        } else if (suggestion.priority > 0.5f) {
            set_color({1.0f, 1.0f, 0.0f}); // Yellow (important)
        } else {
            set_color({0.3f, 1.0f, 0.3f}); // Green (normal)
        }
    }
};
```

### Step 4: War Planning Tools

#### Strategic Objective Setting
**File:** `src/military/war_planning.cpp`
**Add:**
```cpp
struct strategic_objective {
    enum class type {
        conquer_province,
        defend_province,
        blockade_port,
        destroy_army,
        capture_capital
    };
    
    type objective_type;
    dcon::province_id target_province;
    dcon::nation_id target_nation;
    float priority; // 0.0 to 1.0
    std::string description;
    
    // For AI planning
    float estimated_difficulty;
    float expected_reward;
    float risk_level;
};

class war_planner {
public:
    std::vector<strategic_objective> generate_objectives(
        sys::state& state, 
        dcon::nation_id nation, 
        dcon::nation_id enemy
    ) {
        std::vector<strategic_objective> objectives;
        
        // Conquer objectives
        auto conquer_objectives = generate_conquer_objectives(state, nation, enemy);
        objectives.insert(objectives.end(), conquer_objectives.begin(), conquer_objectives.end());
        
        // Defend objectives
        auto defend_objectives = generate_defend_objectives(state, nation, enemy);
        objectives.insert(objectives.end(), defend_objectives.begin(), defend_objectives.end());
        
        // Naval objectives
        auto naval_objectives = generate_naval_objectives(state, nation, enemy);
        objectives.insert(objectives.end(), naval_objectives.begin(), naval_objectives.end());
        
        // Sort by priority
        std::sort(objectives.begin(), objectives.end(),
            [](const auto& a, const auto& b) { return a.priority > b.priority; });
        
        return objectives;
    }
    
    std::vector<strategic_objective> generate_conquer_objectives(
        sys::state& state, 
        dcon::nation_id nation, 
        dcon::nation_id enemy
    ) {
        std::vector<strategic_objective> objectives;
        
        // Find valuable provinces
        for (auto province : state.world.nation_get_provinces(enemy)) {
            strategic_objective obj;
            obj.objective_type = strategic_objective::type::conquer_province;
            obj.target_province = province;
            obj.target_nation = enemy;
            
            // Calculate value
            float value = calculate_province_value(state, province);
            float difficulty = calculate_conquest_difficulty(state, nation, province);
            
            obj.priority = value / (difficulty + 1.0f);
            obj.estimated_difficulty = difficulty;
            obj.expected_reward = value;
            obj.risk_level = calculate_risk(state, nation, province);
            
            obj.description = "Conquer " + get_province_name(province) + 
                " (Value: " + format_value(value) + ")";
            
            objectives.push_back(obj);
        }
        
        return objectives;
    }
    
    std::vector<strategic_objective> generate_defend_objectives(
        sys::state& state, 
        dcon::nation_id nation, 
        dcon::nation_id enemy
    ) {
        std::vector<strategic_objective> objectives;
        
        // Find vulnerable provinces
        for (auto province : state.world.nation_get_provinces(nation)) {
            float threat = calculate_threat_level(state, province, enemy);
            
            if (threat > 0.5f) {
                strategic_objective obj;
                obj.objective_type = strategic_objective::type::defend_province;
                obj.target_province = province;
                obj.target_nation = enemy;
                obj.priority = threat;
                obj.estimated_difficulty = threat;
                obj.expected_reward = 0.0f; // Defense doesn't give rewards
                obj.risk_level = threat;
                
                obj.description = "Defend " + get_province_name(province) + 
                    " (Threat: " + format_percentage(threat) + ")";
                
                objectives.push_back(obj);
            }
        }
        
        return objectives;
    }
};
```

#### War Planning UI
**File:** `src/gui/gui_war_planning.cpp`
**Add:**
```cpp
class war_planning_window : public window_element_base {
public:
    void on_create(sys::state& state) override {
        window_element_base::on_create(state);
        
        // Objectives list
        auto objectives_list = std::make_unique<objectives_listbox>();
        objectives_list->set_position({10, 30});
        objectives_list->set_size({500, 300});
        add_child(std::move(objectives_list));
        
        // Generate objectives button
        auto generate_button = std::make_unique<button_element>();
        generate_button->set_position({520, 30});
        generate_button->set_size({100, 30});
        generate_button->set_text("Generate");
        generate_button->on_click = [&]() {
            generate_objectives(state);
        };
        add_child(std::move(generate_button));
        
        // Assign troops button
        auto assign_button = std::make_unique<button_element>();
        assign_button->set_position({520, 70});
        assign_button->set_size({100, 30});
        assign_button->set_text("Assign");
        assign_button->on_click = [&]() {
            assign_troops(state);
        };
        add_child(std::move(assign_button));
        
        // Objective details panel
        auto details_panel = std::make_unique<objective_details_panel>();
        details_panel->set_position({10, 340});
        details_panel->set_size({610, 150});
        add_child(std::move(details_panel));
    }
    
    void generate_objectives(sys::state& state) {
        auto nation = state.local_player_nation;
        auto enemy = state.military.selected_enemy;
        
        if (!enemy) return;
        
        auto objectives = state.military.war_planner.generate_objectives(state, nation, enemy);
        objectives_list->update(state, objectives);
    }
};

class objectives_listbox : public listbox_base {
public:
    void update(sys::state& state, const std::vector<strategic_objective>& objectives) {
        clear_items();
        
        for (auto& objective : objectives) {
            auto item = std::make_unique<objective_list_item>();
            item->set_objective(objective);
            item->on_click = [&]() {
                show_objective_details(objective);
            };
            add_item(std::move(item));
        }
    }
};

class objective_list_item : public listbox_item_base {
public:
    void set_objective(const strategic_objective& objective) {
        auto text = objective.description + 
            " (Priority: " + format_priority(objective.priority) + 
            ", Difficulty: " + format_difficulty(objective.estimated_difficulty) + ")";
        
        set_text(text);
        
        // Color based on priority
        if (objective.priority > 0.8f) {
            set_color({1.0f, 0.3f, 0.3f}); // Red (high priority)
        } else if (objective.priority > 0.5f) {
            set_color({1.0f, 1.0f, 0.0f}); // Yellow (medium priority)
        } else {
            set_color({0.3f, 1.0f, 0.3f}); // Green (low priority)
        }
    }
};
```

### Step 5: AI War Planning Improvements

#### Utility-Based War Planning
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
        // Generate strategic objectives
        auto objectives = state.military.war_planner.generate_objectives(state, nation, target);
        
        // Calculate overall utility
        float total_utility = 0.0f;
        for (auto& obj : objectives) {
            total_utility += obj.priority * obj.expected_reward;
        }
        
        // Apply personality modifiers
        total_utility *= personality.params.aggression;
        
        if (total_utility > 0.5f && personality.will_declare_wars) {
            candidates.push_back({dcon::decision_id{}, total_utility, true});
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

### Step 6: Success Criteria for Military Modernization

- [ ] Supply overlays show utilization and constraints
- [ ] Supply route visualization with efficiency indicators
- [ ] Battle reports explain combat outcomes with factors
- [ ] Automated logistics suggestions with priority system
- [ ] War planning tools with strategic objectives
- [ ] AI war planning uses utility-based decisions
- [ ] Players understand supply constraints visually
- [ ] Combat outcomes are explainable

---

## Week 7: Diplomacy & Politics Modernization

### Objective
Improve diplomatic gameplay with better visibility, interaction, and political feedback

### Step 1: Influence Heatmaps

#### Visual Sphere Representation
**File:** `src/gui/gui_influence_heatmap.cpp`
**Add:**
```cpp
class influence_heatmap : public map_overlay_base {
public:
    void render(sys::state& state) override {
        for (auto province : state.world.province_range()) {
            auto owner = province.get_owner();
            
            if (!owner) continue;
            
            // Calculate influence level
            float influence = calculate_influence_level(state, province);
            
            // Color based on influence
            color overlay_color;
            if (influence > 0.8f) {
                overlay_color = {0.0f, 0.5f, 1.0f}; // Blue (strong influence)
            } else if (influence > 0.5f) {
                overlay_color = {0.5f, 0.5f, 1.0f}; // Light blue (moderate)
            } else if (influence > 0.2f) {
                overlay_color = {0.8f, 0.8f, 1.0f}; // Very light blue (weak)
            } else {
                continue; // No overlay
            }
            
            // Draw province overlay
            draw_province_overlay(province, overlay_color, 0.4f);
        }
    }
    
    float calculate_influence_level(sys::state& state, dcon::province_id province) {
        auto owner = province.get_owner();
        float max_influence = 0.0f;
        
        // Check influence from all nations
        for (auto nation : state.world.nation_range()) {
            if (nation == owner) continue;
            
            float influence = state.diplomacy.get_influence(nation, owner);
            max_influence = std::max(max_influence, influence);
        }
        
        return max_influence;
    }
};
```

#### Influence Cost Visualization
**File:** `src/gui/gui_influence_cost.cpp`
**Add:**
```cpp
class influence_cost_window : public window_element_base {
public:
    void on_create(sys::state& state) override {
        window_element_base::on_create(state);
        
        // Target nation selector
        auto target_label = std::make_unique<text_element>();
        target_label->set_text("Target Nation:");
        target_label->set_position({10, 30});
        add_child(std::move(target_label));
        
        auto target_selector = std::make_unique<nation_selector>();
        target_selector->set_position({10, 50});
        target_selector->set_size({200, 30});
        add_child(std::move(target_selector));
        
        // Action selector
        auto action_label = std::make_unique<text_element>();
        action_label->set_text("Action:");
        action_label->set_position({220, 30});
        add_child(std::move(action_label));
        
        auto action_selector = std::make_unique<action_selector>();
        action_selector->set_position({220, 50});
        action_selector->set_size({200, 30});
        add_child(std::move(action_selector));
        
        // Cost display
        auto cost_label = std::make_unique<text_element>();
        cost_label->set_text("Cost:");
        cost_label->set_position({430, 30});
        add_child(std::move(cost_label));
        
        auto cost_display = std::make_unique<cost_display>();
        cost_display->set_position({430, 50});
        cost_display->set_size({200, 30});
        add_child(std::move(cost_display));
        
        // Execute button
        auto execute_button = std::make_unique<button_element>();
        execute_button->set_position({10, 90});
        execute_button->set_size({100, 30});
        execute_button->set_text("Execute");
        execute_button->on_click = [&]() {
            execute_influence_action(state);
        };
        add_child(std::move(execute_button));
        
        // Predicted reaction
        auto reaction_label = std::make_unique<text_element>();
        reaction_label->set_text("Predicted Reaction:");
        reaction_label->set_position({10, 130});
        add_child(std::move(reaction_label));
        
        auto reaction_display = std::make_unique<reaction_display>();
        reaction_display->set_position({10, 150});
        reaction_display->set_size({600, 100});
        add_child(std::move(reaction_display));
    }
    
    void update_cost_display(sys::state& state) {
        auto target = target_selector->get_selected();
        auto action = action_selector->get_selected();
        
        if (!target || !action) return;
        
        // Calculate cost
        auto cost = state.diplomacy.calculate_influence_cost(state.local_player_nation, target, action);
        
        // Update display
        cost_display->set_cost(cost);
        
        // Predict reaction
        auto reaction = state.diplomacy.predict_reaction(state.local_player_nation, target, action);
        reaction_display->set_reaction(reaction);
    }
};
```

### Step 2: Diplomatic Opportunities

#### AI-Suggested Actions
**File:** `src/diplomacy/diplomatic_opportunities.cpp`
**Add:**
```cpp
class diplomatic_opportunities {
public:
    struct opportunity {
        enum class type {
            form_alliance,
            declare_war,
            offer_peace,
            trade_agreement,
            sphere_influence
        };
        
        type opportunity_type;
        dcon::nation_id target;
        float priority; // 0.0 to 1.0
        std::string description;
        std::string rationale;
    };
    
    std::vector<opportunity> find_opportunities(sys::state& state, dcon::nation_id nation) {
        std::vector<opportunity> opportunities;
        
        // Find alliance opportunities
        auto alliance_opp = find_alliance_opportunities(state, nation);
        opportunities.insert(opportunities.end(), alliance_opp.begin(), alliance_opp.end());
        
        // Find war opportunities
        auto war_opp = find_war_opportunities(state, nation);
        opportunities.insert(opportunities.end(), war_opp.begin(), war_opp.end());
        
        // Find peace opportunities
        auto peace_opp = find_peace_opportunities(state, nation);
        opportunities.insert(opportunities.end(), peace_opp.begin(), peace_opp.end());
        
        // Sort by priority
        std::sort(opportunities.begin(), opportunities.end(),
            [](const auto& a, const auto& b) { return a.priority > b.priority; });
        
        return opportunities;
    }
    
    std::vector<opportunity> find_alliance_opportunities(sys::state& state, dcon::nation_id nation) {
        std::vector<opportunity> opportunities;
        
        for (auto target : state.world.nation_range()) {
            if (target == nation) continue;
            if (nation.is_player_controlled() && target.is_player_controlled()) continue;
            
            // Check if alliance makes sense
            float utility = calculate_alliance_utility(state, nation, target);
            
            if (utility > 0.6f) {
                opportunity opp;
                opp.opportunity_type = opportunity::type::form_alliance;
                opp.target = target;
                opp.priority = utility;
                opp.description = "Form alliance with " + get_nation_name(target);
                opp.rationale = calculate_alliance_rationale(state, nation, target);
                
                opportunities.push_back(opp);
            }
        }
        
        return opportunities;
    }
    
    std::vector<opportunity> find_war_opportunities(sys::state& state, dcon::nation_id nation) {
        std::vector<opportunity> opportunities;
        
        for (auto target : state.world.nation_range()) {
            if (target == nation) continue;
            
            // Check if war is feasible
            float utility = calculate_war_utility(state, nation, target);
            
            if (utility > 0.7f) {
                opportunity opp;
                opp.opportunity_type = opportunity::type::declare_war;
                opp.target = target;
                opp.priority = utility;
                opp.description = "Declare war on " + get_nation_name(target);
                opp.rationale = calculate_war_rationale(state, nation, target);
                
                opportunities.push_back(opp);
            }
        }
        
        return opportunities;
    }
    
    float calculate_alliance_utility(sys::state& state, dcon::nation_id nation, dcon::nation_id target) {
        float utility = 0.0f;
        
        // Mutual defense benefit
        auto common_enemies = find_common_enemies(state, nation, target);
        utility += common_enemies.size() * 0.2f;
        
        // Economic benefit
        auto trade_potential = calculate_trade_potential(state, nation, target);
        utility += trade_potential * 0.3f;
        
        // Geographic benefit
        auto geographic_value = calculate_geographic_value(state, nation, target);
        utility += geographic_value * 0.2f;
        
        // Risk factors
        auto risk = calculate_alliance_risk(state, nation, target);
        utility -= risk * 0.3f;
        
        return std::max(0.0f, std::min(1.0f, utility));
    }
};
```

#### Opportunities UI
**File:** `src/gui/gui_diplomatic_opportunities.cpp`
**Add:**
```cpp
class diplomatic_opportunities_window : public window_element_base {
public:
    void on_create(sys::state& state) override {
        window_element_base::on_create(state);
        
        // Opportunities list
        auto opportunities_list = std::make_unique<opportunities_listbox>();
        opportunities_list->set_position({10, 30});
        opportunities_list->set_size({600, 300});
        add_child(std::move(opportunities_list));
        
        // Refresh button
        auto refresh_button = std::make_unique<button_element>();
        refresh_button->set_position({620, 30});
        refresh_button->set_size({80, 30});
        refresh_button->set_text("Refresh");
        refresh_button->on_click = [&]() {
            update_opportunities(state);
        };
        add_child(std::move(refresh_button));
        
        // Auto-suggest checkbox
        auto auto_suggest = std::make_unique<checkbox_element>();
        auto_suggest->set_position({10, 340});
        auto_suggest->set_text("Auto-suggest opportunities");
        auto_suggest->set_checked(state.diplomacy.auto_suggest_opportunities);
        auto_suggest->on_change = [&](bool checked) {
            state.diplomacy.auto_suggest_opportunities = checked;
        };
        add_child(std::move(auto_suggest));
    }
    
    void update_opportunities(sys::state& state) {
        auto opportunities = state.diplomacy.opportunities.find_opportunities(
            state.local_player_nation);
        opportunities_list->update(state, opportunities);
    }
};

class opportunities_listbox : public listbox_base {
public:
    void update(sys::state& state, const std::vector<diplomatic_opportunities::opportunity>& opportunities) {
        clear_items();
        
        for (auto& opportunity : opportunities) {
            auto item = std::make_unique<opportunity_list_item>();
            item->set_opportunity(opportunity);
            item->on_click = [&]() {
                execute_opportunity(state, opportunity);
            };
            add_item(std::move(item));
        }
    }
};

class opportunity_list_item : public listbox_item_base {
public:
    void set_opportunity(const diplomatic_opportunities::opportunity& opportunity) {
        auto text = opportunity.description + 
            " (Priority: " + format_priority(opportunity.priority) + ")";
        
        set_text(text);
        
        // Color based on priority
        if (opportunity.priority > 0.8f) {
            set_color({1.0f, 0.3f, 0.3f}); // Red (high priority)
        } else if (opportunity.priority > 0.5f) {
            set_color({1.0f, 1.0f, 0.0f}); // Yellow (medium priority)
        } else {
            set_color({0.3f, 1.0f, 0.3f}); // Green (low priority)
        }
    }
};
```

### Step 3: Interactive Negotiation

#### Trade Agreement UI
**File:** `src/gui/gui_trade_agreement.cpp`
**Add:**
```cpp
class trade_agreement_window : public window_element_base {
public:
    void on_create(sys::state& state) override {
        window_element_base::on_create(state);
        
        // Target nation
        auto target_label = std::make_unique<text_element>();
        target_label->set_text("Partner:");
        target_label->set_position({10, 30});
        add_child(std::move(target_label));
        
        auto target_selector = std::make_unique<nation_selector>();
        target_selector->set_position({10, 50});
        target_selector->set_size({200, 30});
        add_child(std::move(target_selector));
        
        // Offered goods
        auto offer_label = std::make_unique<text_element>();
        offer_label->set_text("You Offer:");
        offer_label->set_position({220, 30});
        add_child(std::move(offer_label));
        
        auto offer_list = std::make_unique<goods_listbox>();
        offer_list->set_position({220, 50});
        offer_list->set_size({180, 150});
        add_child(std::move(offer_list));
        
        // Requested goods
        auto request_label = std::make_unique<text_element>();
        request_label->set_text("You Request:");
        request_label->set_position({410, 30});
        add_child(std::move(request_label));
        
        auto request_list = std::make_unique<goods_listbox>();
        request_list->set_position({410, 50});
        request_list->set_size({180, 150});
        add_child(std::move(request_list));
        
        // Tariff slider
        auto tariff_label = std::make_unique<text_element>();
        tariff_label->set_text("Tariff:");
        tariff_label->set_position({10, 210});
        add_child(std::move(tariff_label));
        
        auto tariff_slider = std::make_unique<slider_element>();
        tariff_slider->set_position({10, 230});
        tariff_slider->set_range(0.0f, 0.3f);
        tariff_slider->set_value(0.1f);
        tariff_slider->on_change = [&](float value) {
            update_agreement_value();
        };
        add_child(std::move(tariff_slider));
        
        // Agreement value
        auto value_label = std::make_unique<text_element>();
        value_label->set_text("Agreement Value:");
        value_label->set_position({220, 210});
        add_child(std::move(value_label));
        
        auto value_display = std::make_unique<value_display>();
        value_display->set_position({220, 230});
        value_display->set_size({200, 30});
        add_child(std::move(value_display));
        
        // Accept/Reject buttons
        auto accept_button = std::make_unique<button_element>();
        accept_button->set_position({10, 270});
        accept_button->set_size({100, 30});
        accept_button->set_text("Accept");
        accept_button->on_click = [&]() {
            accept_agreement(state);
        };
        add_child(std::move(accept_button));
        
        auto reject_button = std::make_unique<button_element>();
        reject_button->set_position({120, 270});
        reject_button->set_size({100, 30});
        reject_button->set_text("Reject");
        reject_button->on_click = [&]() {
            close();
        };
        add_child(std::move(reject_button));
    }
    
    void update_agreement_value() {
        auto target = target_selector->get_selected();
        if (!target) return;
        
        // Calculate agreement value
        auto value = calculate_agreement_value();
        value_display->set_value(value);
        
        // Predict acceptance
        auto will_accept = predict_acceptance(target, value);
        value_display->set_acceptance(will_accept);
    }
};
```

### Step 4: Political Timeline

#### Visual History of Political Changes
**File:** `src/gui/gui_political_timeline.cpp`
**Add:**
```cpp
class political_timeline : public panel_base {
public:
    void on_create(sys::state& state) override {
        panel_base::on_create(state);
        
        // Timeline chart
        auto timeline_chart = std::make_unique<timeline_chart_element>();
        timeline_chart->set_position({10, 30});
        timeline_chart->set_size({600, 200});
        add_child(std::move(timeline_chart));
        
        // Event list
        auto event_list = std::make_unique<timeline_event_listbox>();
        event_list->set_position({10, 240});
        event_list->set_size({600, 150});
        add_child(std::move(event_list));
        
        // Filter controls
        auto filter_label = std::make_unique<text_element>();
        filter_label->set_text("Filter:");
        filter_label->set_position({620, 30});
        add_child(std::move(filter_label));
        
        auto filter_checkboxes = std::make_unique<filter_checkboxes>();
        filter_checkboxes->set_position({620, 50});
        filter_checkboxes->set_size({100, 150});
        add_child(std::move(filter_checkboxes));
    }
};

class timeline_chart_element : public element_base {
public:
    void render(sys::state& state) override {
        auto& timeline = state.politics.timeline;
        
        if (timeline.events.empty()) return;
        
        // Draw timeline axis
        draw_axis();
        
        // Draw events as points
        for (auto& event : timeline.events) {
            float x = timeline_to_x(event.date);
            float y = event_to_y(event);
            
            // Color based on event type
            color event_color = get_event_color(event.type);
            
            // Draw point
            draw_circle(x, y, 5.0f, event_color);
            
            // Draw label on hover
            if (is_hovered(x, y)) {
                draw_tooltip(event);
            }
        }
        
        // Draw trend lines
        draw_trend_lines(timeline);
    }
    
    color get_event_color(timeline_event::type event_type) {
        switch (event_type) {
            case timeline_event::type::election:
                return {0.0f, 0.5f, 1.0f}; // Blue
            case timeline_event::type::reform:
                return {0.5f, 0.0f, 1.0f}; // Purple
            case timeline_event::type::revolution:
                return {1.0f, 0.0f, 0.0f}; // Red
            case timeline_event::type::policy_change:
                return {0.0f, 1.0f, 0.0f}; // Green
            default:
                return {1.0f, 1.0f, 1.0f}; // White
        }
    }
};
```

### Step 5: Reform Simulator

#### Preview Effects of Political Reforms
**File:** `src/gui/gui_reform_simulator.cpp`
**Add:**
```cpp
class reform_simulator_window : public window_element_base {
public:
    void on_create(sys::state& state) override {
        window_element_base::on_create(state);
        
        // Reform selector
        auto reform_label = std::make_unique<text_element>();
        reform_label->set_text("Reform:");
        reform_label->set_position({10, 30});
        add_child(std::move(reform_label));
        
        auto reform_selector = std::make_unique<reform_selector>();
        reform_selector->set_position({10, 50});
        reform_selector->set_size({200, 30});
        add_child(std::move(reform_selector));
        
        // Immediate effects
        auto immediate_label = std::make_unique<text_element>();
        immediate_label->set_text("Immediate Effects:");
        immediate_label->set_position({220, 30});
        add_child(std::move(immediate_label));
        
        auto immediate_effects = std::make_unique<effects_listbox>();
        immediate_effects->set_position({220, 50});
        immediate_effects->set_size({200, 150});
        add_child(std::move(immediate_effects));
        
        // Long-term effects
        auto longterm_label = std::make_unique<text_element>();
        longterm_label->set_text("Long-term Effects:");
        longterm_label->set_position({430, 30});
        add_child(std::move(longterm_label));
        
        auto longterm_effects = std::make_unique<effects_listbox>();
        longterm_effects->set_position({430, 50});
        longterm_effects->set_size({200, 150});
        add_child(std::move(longterm_effects));
        
        // POP reactions
        auto pop_label = std::make_unique<text_element>();
        pop_label->set_text("POP Reactions:");
        pop_label->set_position({10, 210});
        add_child(std::move(pop_label));
        
        auto pop_reactions = std::make_unique<pop_reaction_listbox>();
        pop_reactions->set_position({10, 230});
        pop_reactions->set_size({620, 100});
        add_child(std::move(pop_reactions));
        
        // Execute button
        auto execute_button = std::make_unique<button_element>();
        execute_button->set_position({10, 340});
        execute_button->set_size({100, 30});
        execute_button->set_text("Execute");
        execute_button->on_click = [&]() {
            execute_reform(state);
        };
        add_child(std::move(execute_button));
        
        // Cancel button
        auto cancel_button = std::make_unique<button_element>();
        cancel_button->set_position({120, 340});
        cancel_button->set_size({100, 30});
        cancel_button->set_text("Cancel");
        cancel_button->on_click = [&]() {
            close();
        };
        add_child(std::move(cancel_button));
    }
    
    void update_simulator(sys::state& state) {
        auto reform = reform_selector->get_selected();
        if (!reform) return;
        
        // Calculate immediate effects
        auto immediate = calculate_immediate_effects(state, reform);
        immediate_effects->update(state, immediate);
        
        // Calculate long-term effects
        auto longterm = calculate_longterm_effects(state, reform);
        longterm_effects->update(state, longterm);
        
        // Calculate POP reactions
        auto pop_reactions = calculate_pop_reactions(state, reform);
        pop_reactions_list->update(state, pop_reactions);
    }
};
```

### Step 6: Political Preference Vectors

#### Rich Political Representation
**File:** `src/politics/political_preferences.cpp`
**Add:**
```cpp
struct political_preference {
    // Economic axis (left vs right)
    float economic_left_right; // -1.0 (left) to 1.0 (right)
    
    // Cultural axis (liberal vs conservative)
    float cultural_liberal_conservative; // -1.0 (liberal) to 1.0 (conservative)
    
    // Authoritarian vs libertarian
    float authoritarian_libertarian; // -1.0 (libertarian) to 1.0 (authoritarian)
    
    // Military vs pacifist
    float militarist_pacifist; // -1.0 (pacifist) to 1.0 (militarist)
    
    // Nationalist vs internationalist
    float nationalist_internationalist; // -1.0 (internationalist) to 1.0 (nationalist)
    
    // Calculate distance from another preference
    float distance_to(const political_preference& other) const {
        float sum = 0.0f;
        sum += (economic_left_right - other.economic_left_right) * (economic_left_right - other.economic_left_right);
        sum += (cultural_liberal_conservative - other.cultural_liberal_conservative) * 
               (cultural_liberal_conservative - other.cultural_liberal_conservative);
        sum += (authoritarian_libertarian - other.authoritarian_libertarian) * 
               (authoritarian_libertarian - other.authoritarian_libertarian);
        sum += (militarist_pacifist - other.militarist_pacifist) * 
               (militarist_pacifist - other.militarist_pacifist);
        sum += (nationalist_internationalist - other.nationalist_internationalist) * 
               (nationalist_internationalist - other.nationalist_internationalist);
        return std::sqrt(sum);
    }
    
    // Calculate alignment with a party
    float alignment_with_party(const party_platform& platform) const {
        return 1.0f / (1.0f + distance_to(platform.preference));
    }
};

struct party_platform {
    dcon::political_party_id party;
    political_preference preference;
    std::vector<dcon::policy_id> promised_policies;
    
    // Calculate party appeal to a POP
    float appeal_to_pop(sys::state& state, dcon::pop_id pop, const political_preference& pop_preference) const {
        float distance = preference.distance_to(pop_preference);
        float appeal = 1.0f / (1.0f + distance);
        
        // Adjust for promises
        for (auto policy : promised_policies) {
            if (pop.prefers_policy(policy)) {
                appeal += 0.2f;
            }
        }
        
        return std::min(1.0f, appeal);
    }
};
```

#### Political Preference UI
**File:** `src/gui/gui_political_preferences.cpp`
**Add:**
```cpp
class political_preference_window : public window_element_base {
public:
    void on_create(sys::state& state) override {
        window_element_base::on_create(state);
        
        // Political compass
        auto compass = std::make_unique<political_compass>();
        compass->set_position({10, 30});
        compass->set_size({200, 200});
        add_child(std::move(compass));
        
        // Party alignment
        auto party_label = std::make_unique<text_element>();
        party_label->set_text("Party Alignments:");
        party_label->set_position({220, 30});
        add_child(std::move(party_label));
        
        auto party_list = std::make_unique<party_alignment_listbox>();
        party_list->set_position({220, 50});
        party_list->set_size({400, 200});
        add_child(std::move(party_list));
        
        // POP preferences
        auto pop_label = std::make_unique<text_element>();
        pop_label->set_text("POP Preferences:");
        pop_label->set_position({10, 240});
        add_child(std::move(pop_label));
        
        auto pop_list = std::make_unique<pop_preference_listbox>();
        pop_list->set_position({10, 260});
        pop_list->set_size({610, 150});
        add_child(std::move(pop_list));
    }
};

class political_compass : public element_base {
public:
    void render(sys::state& state) override {
        // Draw axes
        draw_axis(-1.0f, 1.0f, -1.0f, 1.0f);
        
        // Draw party positions
        for (auto party : state.world.political_party_range()) {
            auto platform = get_party_platform(party);
            float x = platform.preference.economic_left_right;
            float y = platform.preference.cultural_liberal_conservative;
            
            draw_point(x, y, get_party_color(party), 8.0f);
            
            // Label on hover
            if (is_hovered(x, y)) {
                draw_text(x + 0.1f, y + 0.1f, get_party_name(party));
            }
        }
        
        // Draw current player preference
        auto player_pref = get_player_preference(state);
        draw_point(player_pref.economic_left_right, 
                   player_pref.cultural_liberal_conservative,
                   {1.0f, 1.0f, 1.0f}, 10.0f);
    }
};
```

### Step 7: Success Criteria for Diplomacy & Politics Modernization

- [ ] Influence heatmaps show spheres and influence levels
- [ ] Diplomatic opportunities are AI-suggested with rationale
- [ ] Interactive negotiation UI for trade and treaties
- [ ] Political timeline shows historical changes
- [ ] Reform simulator previews effects before execution
- [ ] Political preference vectors provide rich representation
- [ ] Players can see diplomatic relationships clearly
- [ ] Political changes are understandable and predictable

---

## Week 8: Technology & Research Modernization

### Objective
Improve research gameplay with better planning, feedback, and strategic depth

### Step 1: Tech Tooltips & Information

#### Detailed Tech Information
**File:** `src/gui/gui_tech_tooltips.cpp`
**Add:**
```cpp
class tech_tooltip : public tooltip_base {
public:
    void update(sys::state& state, dcon::technology_id tech) override {
        auto tech_obj = state.world.technology_get(tech);
        
        // Clear existing content
        clear_content();
        
        // Add tech name
        add_line(get_tech_name(tech), 16, {1.0f, 1.0f, 1.0f});
        
        // Add description
        add_line(tech_obj.description, 12, {0.8f, 0.8f, 0.8f});
        
        // Add effects
        add_line("Effects:", 14, {1.0f, 1.0f, 0.0f});
        for (auto& effect : tech_obj.effects) {
            add_line(format_effect(effect), 12, {0.8f, 0.8f, 0.8f});
        }
        
        // Add research time
        auto time_to_research = calculate_research_time(state, tech);
        add_line("Research Time: " + format_days(time_to_research), 12, {0.8f, 0.8f, 0.8f});
        
        // Add prerequisites
        if (!tech_obj.prerequisites.empty()) {
            add_line("Prerequisites:", 14, {1.0f, 1.0f, 0.0f});
            for (auto prereq : tech_obj.prerequisites) {
                auto status = state.tech.is_researched(prereq) ? "[X]" : "[ ]";
                add_line(status + " " + get_tech_name(prereq), 12, {0.8f, 0.8f, 0.8f});
            }
        }
        
        // Add synergy information
        auto synergies = find_synergies(state, tech);
        if (!synergies.empty()) {
            add_line("Synergies:", 14, {0.0f, 1.0f, 1.0f});
            for (auto& synergy : synergies) {
                add_line("+" + format_percentage(synergy.bonus) + " with " + get_tech_name(synergy.tech), 
                        12, {0.8f, 0.8f, 0.8f});
            }
        }
        
        // Add current research status
        auto status = state.tech.get_research_status(tech);
        add_line("Status: " + format_status(status), 12, {1.0f, 1.0f, 1.0f});
    }
    
    struct synergy_info {
        dcon::technology_id tech;
        float bonus;
    };
    
    std::vector<synergy_info> find_synergies(sys::state& state, dcon::technology_id tech) {
        std::vector<synergy_info> synergies;
        
        auto tech_obj = state.world.technology_get(tech);
        
        // Check for techs in same category
        for (auto other_tech : state.world.technology_range()) {
            if (other_tech == tech) continue;
            
            if (tech_obj.category == other_tech.category) {
                // Check if both are researched
                if (state.tech.is_researched(tech) && state.tech.is_researched(other_tech)) {
                    synergies.push_back({other_tech, 0.1f}); // 10% bonus
                }
            }
        }
        
        return synergies;
    }
};
```

### Step 2: Research Queue

#### Queue System with Auto-Prioritization
**File:** `src/technology/research_queue.cpp`
**Add:**
```cpp
struct research_queue_item {
    dcon::technology_id tech;
    float priority; // 0.0 to 1.0
    bool auto_prioritize;
    size_t estimated_days;
};

class research_queue {
public:
    std::vector<research_queue_item> queue;
    
    void add_item(const research_queue_item& item) {
        queue.push_back(item);
        // Sort by priority
        std::sort(queue.begin(), queue.end(),
            [](const auto& a, const auto& b) { return a.priority > b.priority; });
    }
    
    void process_queue(sys::state& state) {
        if (queue.empty()) return;
        
        // Check if current research is complete
        if (state.tech.current_research && 
            state.tech.is_researched(state.tech.current_research)) {
            state.tech.current_research = dcon::technology_id{};
        }
        
        // Start next research if none active
        if (!state.tech.current_research && !queue.empty()) {
            auto next = queue.front();
            queue.erase(queue.begin());
            
            if (can_research(state, next.tech)) {
                state.tech.current_research = next.tech;
                state.tech.research_progress = 0.0f;
            }
        }
        
        // Auto-prioritize if enabled
        for (auto& item : queue) {
            if (item.auto_prioritize) {
                item.priority = calculate_dynamic_priority(state, item.tech);
            }
        }
        
        // Re-sort
        std::sort(queue.begin(), queue.end(),
            [](const auto& a, const auto& b) { return a.priority > b.priority; });
    }
    
    float calculate_dynamic_priority(sys::state& state, dcon::technology_id tech) {
        float priority = 0.5f; // Base priority
        
        // Adjust based on current situation
        auto tech_obj = state.world.technology_get(tech);
        
        // Check for shortages
        for (auto& effect : tech_obj.effects) {
            if (effect.type == effect_type::increase_production) {
                auto commodity = effect.commodity;
                auto shortage = state.economy.get_shortage(commodity);
                if (shortage > 0.2f) {
                    priority += 0.3f;
                }
            }
        }
        
        // Check for strategic goals
        auto& board = ai::get_blackboard(state);
        for (auto& needed : board.needed_factory_types) {
            if (tech_obj.unlocks_factory == needed) {
                priority += 0.4f;
            }
        }
        
        // Adjust based on research cost
        auto cost = calculate_research_cost(state, tech);
        priority -= cost * 0.1f;
        
        return std::max(0.0f, std::min(1.0f, priority));
    }
};
```

#### Research Queue UI
**File:** `src/gui/gui_research_queue.cpp`
**Add:**
```cpp
class research_queue_window : public window_element_base {
public:
    void on_create(sys::state& state) override {
        window_element_base::on_create(state);
        
        // Current research
        auto current_label = std::make_unique<text_element>();
        current_label->set_text("Current Research:");
        current_label->set_position({10, 30});
        add_child(std::move(current_label));
        
        auto current_display = std::make_unique<current_research_display>();
        current_display->set_position({10, 50});
        current_display->set_size({400, 50});
        add_child(std::move(current_display));
        
        // Queue list
        auto queue_label = std::make_unique<text_element>();
        queue_label->set_text("Research Queue:");
        queue_label->set_position({10, 110});
        add_child(std::move(queue_label));
        
        auto queue_list = std::make_unique<queue_listbox>();
        queue_list->set_position({10, 130});
        queue_list->set_size({400, 200});
        add_child(std::move(queue_list));
        
        // Add tech button
        auto add_button = std::make_unique<button_element>();
        add_button->set_position({420, 130});
        add_button->set_size({100, 30});
        add_button->set_text("Add Tech");
        add_button->on_click = [&]() {
            show_tech_selector();
        };
        add_child(std::move(add_button));
        
        // Auto-prioritize checkbox
        auto auto_prioritize = std::make_unique<checkbox_element>();
        auto_prioritize->set_position({10, 340});
        auto_prioritize->set_text("Auto-prioritize");
        auto_prioritize->set_checked(state.tech.auto_prioritize);
        auto_prioritize->on_change = [&](bool checked) {
            state.tech.auto_prioritize = checked;
        };
        add_child(std::move(auto_prioritize));
    }
    
    void show_tech_selector() {
        auto selector = std::make_unique<tech_selector_window>();
        selector->set_position({100, 100});
        selector->set_size({500, 400});
        // ... show modal ...
    }
};
```

### Step 3: Tech Path Suggestions

#### AI Recommendations
**File:** `src/ai/ai_research.cpp`
**Add:**
```cpp
class research_suggester {
public:
    struct tech_suggestion {
        dcon::technology_id tech;
        float priority;
        std::string rationale;
        std::vector<dcon::technology_id> path; // Recommended path to this tech
    };
    
    std::vector<tech_suggestion> suggest_tech_path(
        sys::state& state, 
        dcon::nation_id nation,
        const std::string& strategy
    ) {
        std::vector<tech_suggestion> suggestions;
        
        // Find all available techs
        for (auto tech : state.world.technology_range()) {
            if (state.tech.is_researched(tech)) continue;
            if (!state.tech.can_research(tech)) continue;
            
            tech_suggestion suggestion;
            suggestion.tech = tech;
            suggestion.priority = calculate_priority(state, nation, tech, strategy);
            suggestion.rationale = calculate_rationale(state, nation, tech, strategy);
            suggestion.path = find_research_path(state, tech);
            
            if (suggestion.priority > 0.3f) {
                suggestions.push_back(suggestion);
            }
        }
        
        // Sort by priority
        std::sort(suggestions.begin(), suggestions.end(),
            [](const auto& a, const auto& b) { return a.priority > b.priority; });
        
        return suggestions;
    }
    
    float calculate_priority(
        sys::state& state, 
        dcon::nation_id nation, 
        dcon::technology_id tech,
        const std::string& strategy
    ) {
        float priority = 0.5f; // Base
        
        auto tech_obj = state.world.technology_get(tech);
        
        // Strategy-based weighting
        if (strategy == "industrial") {
            if (tech_obj.category == "industry") {
                priority += 0.3f;
            }
        } else if (strategy == "military") {
            if (tech_obj.category == "military") {
                priority += 0.3f;
            }
        } else if (strategy == "naval") {
            if (tech_obj.category == "naval") {
                priority += 0.3f;
            }
        }
        
        // Check for unlocks
        if (tech_obj.unlocks_factory) {
            priority += 0.2f;
        }
        
        // Check for bonuses
        for (auto& effect : tech_obj.effects) {
            if (effect.type == effect_type::increase_production) {
                priority += 0.1f;
            }
        }
        
        // Adjust for cost
        auto cost = calculate_research_cost(state, tech);
        priority -= cost * 0.05f;
        
        return std::max(0.0f, std::min(1.0f, priority));
    }
    
    std::vector<dcon::technology_id> find_research_path(
        sys::state& state, 
        dcon::technology_id target
    ) {
        std::vector<dcon::technology_id> path;
        
        // Find prerequisites recursively
        auto current = target;
        while (true) {
            auto tech_obj = state.world.technology_get(current);
            if (tech_obj.prerequisites.empty()) break;
            
            // Take first prerequisite (simplified)
            auto prereq = tech_obj.prerequisites[0];
            path.push_back(prereq);
            current = prereq;
        }
        
        // Reverse to get correct order
        std::reverse(path.begin(), path.end());
        
        return path;
    }
};
```

#### Tech Path UI
**File:** `src/gui/gui_tech_path.cpp`
**Add:**
```cpp
class tech_path_window : public window_element_base {
public:
    void on_create(sys::state& state) override {
        window_element_base::on_create(state);
        
        // Strategy selector
        auto strategy_label = std::make_unique<text_element>();
        strategy_label->set_text("Strategy:");
        strategy_label->set_position({10, 30});
        add_child(std::move(strategy_label));
        
        auto strategy_selector = std::make_unique<strategy_selector>();
        strategy_selector->set_position({10, 50});
        strategy_selector->set_size({200, 30});
        add_child(std::move(strategy_selector));
        
        // Suggested path
        auto path_label = std::make_unique<text_element>();
        path_label->set_text("Suggested Path:");
        path_label->set_position({220, 30});
        add_child(std::move(path_label));
        
        auto path_display = std::make_unique<path_display>();
        path_display->set_position({220, 50});
        path_display->set_size({400, 200});
        add_child(std::move(path_display));
        
        // Apply button
        auto apply_button = std::make_unique<button_element>();
        apply_button->set_position({10, 90});
        apply_button->set_size({100, 30});
        apply_button->set_text("Apply Path");
        apply_button->on_click = [&]() {
            apply_suggested_path(state);
        };
        add_child(std::move(apply_button));
        
        // Rationale
        auto rationale_label = std::make_unique<text_element>();
        rationale_label->set_text("Rationale:");
        rationale_label->set_position({10, 130});
        add_child(std::move(rationale_label));
        
        auto rationale_display = std::make_unique<rationale_display>();
        rationale_display->set_position({10, 150});
        rationale_display->set_size({610, 100});
        add_child(std::move(rationale_display));
    }
    
    void update_suggestions(sys::state& state) {
        auto strategy = strategy_selector->get_selected();
        if (!strategy) return;
        
        auto suggestions = state.ai.research_suggester.suggest_tech_path(
            state, state.local_player_nation, strategy);
        
        path_display->update(state, suggestions);
        
        if (!suggestions.empty()) {
            rationale_display->set_rationale(suggestions[0].rationale);
        }
    }
};
```

### Step 4: Invention Tracking

#### Visual Progress & Notifications
**File:** `src/gui/gui_invention_tracking.cpp`
**Add:**
```cpp
class invention_tracker : public panel_base {
public:
    void on_create(sys::state& state) override {
        panel_base::on_create(state);
        
        // Invention list
        auto invention_list = std::make_unique<invention_listbox>();
        invention_list->set_position({10, 30});
        invention_list->set_size({400, 200});
        add_child(std::move(invention_list));
        
        // Progress bars
        auto progress_label = std::make_unique<text_element>();
        progress_label->set_text("Research Progress:");
        progress_label->set_position({420, 30});
        add_child(std::move(progress_label));
        
        auto progress_bars = std::make_unique<progress_bars>();
        progress_bars->set_position({420, 50});
        progress_bars->set_size({200, 150});
        add_child(std::move(progress_bars));
        
        // Notification panel
        auto notification_label = std::make_unique<text_element>();
        notification_label->set_text("Recent Unlocks:");
        notification_label->set_position({10, 240});
        add_child(std::move(notification_label));
        
        auto notification_list = std::make_unique<notification_listbox>();
        notification_list->set_position({10, 260});
        notification_list->set_size({610, 100});
        add_child(std::move(notification_list));
    }
    
    void update_inventions(sys::state& state) {
        // Update invention list
        auto inventions = get_available_inventions(state);
        invention_list->update(state, inventions);
        
        // Update progress bars
        progress_bars->update(state);
        
        // Update notifications
        auto recent_unlocks = get_recent_unlocks(state);
        notification_list->update(state, recent_unlocks);
    }
};

class invention_list_item : public listbox_item_base {
public:
    void set_invention(sys::state& state, dcon::technology_id tech) {
        auto tech_obj = state.world.technology_get(tech);
        auto status = state.tech.get_research_status(tech);
        
        auto text = tech_obj.name + " - " + format_status(status);
        set_text(text);
        
        // Color based on status
        if (status == research_status::completed) {
            set_color({0.3f, 1.0f, 0.3f}); // Green
        } else if (status == research_status::in_progress) {
            set_color({1.0f, 1.0f, 0.0f}); // Yellow
        } else {
            set_color({0.8f, 0.8f, 0.8f}); // Gray
        }
    }
};
```

### Step 5: Research Analytics

#### Historical Performance
**File:** `src/technology/research_analytics.cpp`
**Add:**
```cpp
struct research_analytics {
    struct tech_record {
        dcon::technology_id tech;
        int32_t days_to_research;
        float research_points_spent;
        std::string researcher; // Nation or advisor
        date start_date;
        date end_date;
    };
    
    std::vector<tech_record> history;
    
    void record_research(sys::state& state, dcon::technology_id tech, 
                         int32_t days, float points, date start, date end) {
        tech_record record;
        record.tech = tech;
        record.days_to_research = days;
        record.research_points_spent = points;
        record.researcher = get_researcher_name(state);
        record.start_date = start;
        record.end_date = end;
        
        history.push_back(record);
    }
    
    // Analytics functions
    float calculate_average_efficiency() const {
        if (history.empty()) return 0.0f;
        
        float total = 0.0f;
        for (auto& record : history) {
            total += record.research_points_spent / record.days_to_research;
        }
        return total / history.size();
    }
    
    float calculate_success_rate() const {
        if (history.empty()) return 0.0f;
        
        size_t completed = 0;
        for (auto& record : history) {
            if (record.days_to_research > 0) completed++;
        }
        
        return static_cast<float>(completed) / history.size();
    }
    
    std::vector<dcon::technology_id> get_most_efficient_techs(size_t count = 5) const {
        std::vector<std::pair<dcon::technology_id, float>> efficiencies;
        
        for (auto& record : history) {
            float efficiency = record.research_points_spent / record.days_to_research;
            efficiencies.push_back({record.tech, efficiency});
        }
        
        std::sort(efficiencies.begin(), efficiencies.end(),
            [](const auto& a, const auto& b) { return a.second < b.second; });
        
        std::vector<dcon::technology_id> result;
        for (size_t i = 0; i < std::min(count, efficiencies.size()); i++) {
            result.push_back(efficiencies[i].first);
        }
        
        return result;
    }
};
```

#### Analytics UI
**File:** `src/gui/gui_research_analytics.cpp`
**Add:**
```cpp
class research_analytics_window : public window_element_base {
public:
    void on_create(sys::state& state) override {
        window_element_base::on_create(state);
        
        // Metrics
        auto metrics_label = std::make_unique<text_element>();
        metrics_label->set_text("Research Metrics:");
        metrics_label->set_position({10, 30});
        add_child(std::move(metrics_label));
        
        auto metrics_display = std::make_unique<metrics_display>();
        metrics_display->set_position({10, 50});
        metrics_display->set_size({300, 150});
        add_child(std::move(metrics_display));
        
        // Efficiency chart
        auto efficiency_label = std::make_unique<text_element>();
        efficiency_label->set_text("Efficiency Over Time:");
        efficiency_label->set_position({320, 30});
        add_child(std::move(efficiency_label));
        
        auto efficiency_chart = std::make_unique<efficiency_chart_element>();
        efficiency_chart->set_position({320, 50});
        efficiency_chart->set_size({300, 150});
        add_child(std::move(efficiency_chart));
        
        // Top efficient techs
        auto top_label = std::make_unique<text_element>();
        top_label->set_text("Most Efficient Techs:");
        top_label->set_position({10, 210});
        add_child(std::move(top_label));
        
        auto top_list = std::make_unique<top_techs_listbox>();
        top_list->set_position({10, 230});
        top_list->set_size({610, 100});
        add_child(std::move(top_list));
    }
    
    void update_analytics(sys::state& state) {
        auto& analytics = state.tech.analytics;
        
        // Update metrics
        metrics_display->set_metric("Average Efficiency", 
            format_efficiency(analytics.calculate_average_efficiency()));
        metrics_display->set_metric("Success Rate", 
            format_percentage(analytics.calculate_success_rate()));
        
        // Update chart
        efficiency_chart->update(state, analytics.history);
        
        // Update top techs
        auto top_techs = analytics.get_most_efficient_techs(10);
        top_list->update(state, top_techs);
    }
};
```

### Step 6: Success Criteria for Technology Modernization

- [ ] Tech tooltips show detailed effects and time-to-impact
- [ ] Research queue with auto-prioritization
- [ ] AI tech path suggestions based on strategy
- [ ] Invention tracking with visual progress
- [ ] Research analytics dashboard
- [ ] Players understand tech effects before research
- [ ] Research planning is strategic rather than reactive
- [ ] Tech progression feels meaningful and impactful

---

## Phase 3 Success Criteria

### Economy Modernization
- [ ] Market overlays show price history and trends
- [ ] Supply chain visualization (Sankey diagrams)
- [ ] Economic advisors provide actionable recommendations
- [ ] Data-driven factory recipes (JSON)
- [ ] Factory build queue with priority system
- [ ] Real-time economic dashboard
- [ ] Players can understand economic health at a glance
- [ ] Late-game economic management is less tedious

### Military Modernization
- [ ] Supply overlays show utilization and constraints
- [ ] Supply route visualization with efficiency indicators
- [ ] Battle reports explain combat outcomes
- [ ] Automated logistics suggestions
- [ ] War planning tools with strategic objectives
- [ ] AI war planning uses utility-based decisions
- [ ] Players understand supply constraints visually
- [ ] Combat outcomes are explainable

### Diplomacy & Politics Modernization
- [ ] Influence heatmaps show spheres and influence levels
- [ ] Diplomatic opportunities are AI-suggested
- [ ] Interactive negotiation UI
- [ ] Political timeline shows historical changes
- [ ] Reform simulator previews effects
- [ ] Political preference vectors provide rich representation
- [ ] Players can see diplomatic relationships clearly
- [ ] Political changes are understandable and predictable

### Technology Modernization
- [ ] Tech tooltips show detailed effects and time-to-impact
- [ ] Research queue with auto-prioritization
- [ ] AI tech path suggestions
- [ ] Invention tracking with visual progress
- [ ] Research analytics dashboard
- [ ] Players understand tech effects before research
- [ ] Research planning is strategic
- [ ] Tech progression feels meaningful

---

## Quick Reference Commands

### Build & Test:
```bash
mkdir build && cd build
cmake -S .. -B . -G "Ninja" -DCMAKE_BUILD_TYPE=RelWithDebInfo
cmake --build . -- -j$(nproc)
ctest --output-on-failure --verbose
```

### Run Gameplay Tests:
```bash
./build/tests/gameplay_tests --scenario late_game.sav --ticks 10000
```

### Measure Player Understanding:
```bash
# Run user testing scenarios
./build/tests/user_testing --scenario economic_crisis.sav --measure_time_to_understand
```

---

## Phase 3 Summary

**Week 5:** Economy Modernization
- Market overlays and price charts
- Supply chain visualization (Sankey diagrams)
- Economic advisors with automation
- Data-driven factory recipes
- Factory build queue
- Economic dashboard

**Week 6:** Military Modernization
- Supply overlays and route visualization
- Battle reports with factor analysis
- Automated logistics suggestions
- War planning tools
- AI war planning improvements

**Week 7:** Diplomacy & Politics Modernization
- Influence heatmaps
- Diplomatic opportunities
- Interactive negotiation UI
- Political timeline
- Reform simulator
- Political preference vectors

**Week 8:** Technology Modernization
- Tech tooltips and information
- Research queue with auto-prioritization
- AI tech path suggestions
- Invention tracking
- Research analytics

**Success:** Gameplay is more intuitive, strategic, and enjoyable

---

**Phase 3 starts now. Let's modernize the gameplay systems.**
