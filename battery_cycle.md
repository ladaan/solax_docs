# Battery Cycle Counter for Home Assistant

## Overview

This guide sets up a **battery cycle counter** for a SolaX PV system with battery storage in Home Assistant using the SolaX Modbus integration.

**System specs:**
- Battery capacity: **11.6 kWh** (nominal)
- Usable capacity: 90% (10.44 kWh) — managed by BMS
- Rated lifetime: **6000 cycles** or **10 years**
- Installation date: **August 2023** (adjust to your actual date)

**What is a cycle?**
One full cycle = discharging the full nominal capacity (11.6 kWh) **and** charging it back.
Partial cycles count proportionally — two 50% discharges + recharges = 1 cycle.
We use **11.6 kWh** (not 10.44 kWh) because the manufacturer's 6000-cycle rating already accounts for the 90% depth of discharge.

---

## Step 1: Integration Sensors (Cumulative Energy)

Add these to your `configuration.yaml` (or an included file) to accumulate total charged and discharged energy over time.

> **Important:** Replace `sensor.solax_battery_charge_power` and `sensor.solax_battery_discharge_power` with your actual entity names. Check **Developer Tools → States** and search for "battery" to find the correct power sensors (in Watts).

```yaml
sensor:
  # Cumulative energy charged into battery (kWh)
  - platform: integration
    source: sensor.solax_battery_charge_power
    name: "Battery Total Energy Charged"
    unit_prefix: k
    unit_time: h
    round: 2
    method: left

  # Cumulative energy discharged from battery (kWh)
  - platform: integration
    source: sensor.solax_battery_discharge_power
    name: "Battery Total Energy Discharged"
    unit_prefix: k
    unit_time: h
    round: 2
    method: left
```

> **Note:** If your inverter uses a single sensor where positive = charging and negative = discharging (or vice versa), you need template sensors to split them first. See [Appendix A](#appendix-a-splitting-a-combined-batterypower-sensor).

---

## Step 2: Template Sensors (Cycle Count & Health)

Add this as a separate file (e.g., `templates/battery_cycles.yaml`) or directly in `configuration.yaml` under `template:`.

### If included via `template: !include templates/battery_cycles.yaml`:

```yaml
- sensor:
    - name: "Battery Cycle Count"
      unit_of_measurement: "cycles"
      state_class: total_increasing
      state: >
        {% set charged = states('sensor.battery_total_energy_charged') | float(0) %}
        {% set discharged = states('sensor.battery_total_energy_discharged') | float(0) %}
        {% set nominal_capacity = 11.6 %}
        {% set past_cycles = 400 %}
        {{ (past_cycles + (charged + discharged) / 2 / nominal_capacity) | round(2) }}
      icon: mdi:battery-sync

    - name: "Battery Life Used (Cycles)"
      unit_of_measurement: "%"
      state: >
        {% set cycles = states('sensor.battery_cycle_count') | float(0) %}
        {{ ((cycles / 6000) * 100) | round(2) }}
      icon: mdi:battery-heart-variant

    - name: "Battery Life Used (Age)"
      unit_of_measurement: "%"
      state: >
        {% set install_date = strptime('2023-08-01', '%Y-%m-%d').replace(tzinfo=now().tzinfo) %}
        {% set years = (now() - install_date).days / 365.25 %}
        {{ ((years / 10) * 100) | round(2) }}
      icon: mdi:calendar-clock

    - name: "Battery Health Estimate"
      unit_of_measurement: "%"
      state: >
        {% set cycle_pct = states('sensor.battery_life_used_cycles') | float(0) %}
        {% set age_pct = states('sensor.battery_life_used_age') | float(0) %}
        {% set worst = [cycle_pct, age_pct] | max %}
        {{ (100 - worst) | round(1) }}
      icon: mdi:battery-heart
```

### What each sensor does

| Sensor | Description |
|---|---|
| **Battery Cycle Count** | Total equivalent full cycles (past estimate + measured) |
| **Battery Life Used (Cycles)** | Percentage of 6000-cycle lifetime consumed |
| **Battery Life Used (Age)** | Percentage of 10-year lifetime consumed |
| **Battery Health Estimate** | Remaining health based on whichever limit (cycles or age) is worse |

---

## Step 3: Estimating Past Cycles

Since the integration sensors only start counting from when you add them, you need to estimate cycles already completed. Set the `past_cycles` value in the Battery Cycle Count template.

**How to estimate:**

1. Calculate days since installation (e.g., Aug 2023 to now)
2. Estimate average daily discharge in kWh
3. Formula: `past_cycles = days × daily_discharge_kWh / 11.6`

**Example:** If you average ~7 kWh/day discharge over 968 days:
`968 × 7 / 11.6 ≈ 584 cycles`

**Seasonal adjustment for Central Europe:**
- Summer (Apr–Sep): battery cycles almost daily → ~0.5–0.7 cycles/day
- Winter (Oct–Mar): much less solar → ~0.1–0.3 cycles/day
- Rough estimate for 2.5 years: **~350–400 cycles**

Set `past_cycles` to your best estimate. You can refine it later.

---

## Step 4: Dashboard Cards

### Entities Card

```yaml
type: entities
title: Battery Lifecycle
entities:
  - entity: sensor.battery_cycle_count
    name: Cycle Count
  - entity: sensor.battery_life_used_cycles
    name: Life Used (Cycles)
  - entity: sensor.battery_life_used_age
    name: Life Used (Age)
  - entity: sensor.battery_health_estimate
    name: Health Estimate
```

### Gauge Card (Battery Health)

```yaml
type: gauge
entity: sensor.battery_health_estimate
name: Battery Health
min: 0
max: 100
severity:
  green: 60
  yellow: 30
  red: 0
```

### ApexCharts Card (History Graph — requires HACS)

Install `apexcharts-card` from HACS, then:

```yaml
type: custom:apexcharts-card
header:
  title: Battery Cycle History
  show: true
graph_span: 30d
series:
  - entity: sensor.battery_cycle_count
    name: Cycles
    stroke_width: 2
```

---

## Appendix A: Splitting a Combined Battery Power Sensor

If your SolaX integration provides a single `sensor.solax_battery_power` where positive = charging and negative = discharging (or vice versa), create two helper template sensors first:

```yaml
template:
  - sensor:
      - name: "SolaX Battery Charge Power"
        unit_of_measurement: "W"
        device_class: power
        state: >
          {% set power = states('sensor.solax_battery_power') | float(0) %}
          {{ power if power > 0 else 0 }}

      - name: "SolaX Battery Discharge Power"
        unit_of_measurement: "W"
        device_class: power
        state: >
          {% set power = states('sensor.solax_battery_power') | float(0) %}
          {{ (power | abs) if power < 0 else 0 }}
```

> **Check the sign convention of your inverter!** Some SolaX models use positive for discharge and negative for charge. Swap the logic above if needed.

Then use `sensor.solax_battery_charge_power` and `sensor.solax_battery_discharge_power` as the sources in Step 1.

---

## Checklist

- [ ] Identify correct battery power entity names in Developer Tools → States
- [ ] Determine sign convention (positive = charge or discharge?)
- [ ] Add integration sensors (Step 1)
- [ ] Add template sensors (Step 2)
- [ ] Set `past_cycles` estimate (Step 3)
- [ ] Set correct `install_date` in the Age sensor
- [ ] Add dashboard cards (Step 4)
- [ ] Restart Home Assistant
- [ ] Verify all sensors show values in Developer Tools → States
