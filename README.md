# Home Assistant Smart Energy Arbitrage (Octopus + Enphase + Solcast)

This repository contains an advanced, highly automated energy management system for Home Assistant. It is designed to optimize battery storage by making dynamic export and grid-charging decisions based on live energy prices, solar forecasts, and historical house consumption.

It is specifically tailored for UK users on **Octopus Energy** dynamic tariffs (like Agile, Intelligent, or Flux) with an **Enphase** solar/battery ecosystem.

## ✨ Key Features
* **Smart Peak Export**: Automatically exports battery power to the grid during the most profitable slots (e.g., 16:30-19:00) while ensuring enough reserve is kept to power your home through the evening.
* **Predictive Evening Reserve**: Learns your actual household consumption over a 7-day rolling average to dynamically calculate exactly how much battery to hold back for the evening, automatically falling back to manual estimates if data is unavailable.
* **Holiday Mode**: Maximizes export profits while you are away by dropping the reserve floor, and automatically freezes the 7-day predictive AI from learning your artificially low vacation usage.
* **Winter / Summer Modes**: Dynamically adjusts battery reserve floors based on the season.
* **Predictive Grid Charging**: Calculates if it is mathematically profitable to charge the battery overnight from the grid, based on tomorrow's solar forecast and arbitrage margins.
* **EV Charging Protection**: Hard blocks exporting when your car is charging on a cheap overnight rate.
* **Dynamic Battery Health Tracking**: Automatically reads the true, live capacity of your Enphase battery to perfectly scale arbitrage math without needing manual slider adjustments.
* **Daily ROI Audits**: Sends detailed daily notifications (via email or mobile app) breaking down your solar generation, peak avoidance savings, and net profit/loss.
* **Failsafe Logic**: Safely suspends operations and alerts you if the Enphase Gateway drops offline to prevent data corruption.

## 📋 Prerequisites
To use this setup, you must have the following custom integrations installed in Home Assistant (available via HACS):
1. [Octopus Energy Integration](https://github.com/BottlecapDave/HomeAssistant-OctopusEnergy) (by BottlecapDave)
2. [Solcast PV Forecast](https://github.com/BJReplay/ha-solcast-solar) (by BJReplay)
3. Native Enphase Envoy Integration
4. [hacs_enphase_envoy_cloud](https://github.com/chinedu40/hacs_enphase_envoy_cloud) (by chinedu40)

## ⚙️ Installation & Setup

Because Home Assistant requires hardcoded entity IDs for some integrations and dashboards, you must replace my system's specific hardware IDs with your own for the templates and dashboard. However, the core automations are provided as easy-to-use **Blueprints**!

### Step 1: Add the System Variables
Add the following into your `configuration.yaml`, replacing `EnvoyMAC` with `YOUR_ENVOY_MAC`:

```yaml
utility_meter:
  daily_battery_discharged:
    source: sensor.envoy_EnvoyMAC_lifetime_battery_energy_discharged
    name: "Daily Battery Energy Discharged"
    unique_id: um_daily_battery_discharged
    cycle: daily

  grid_energy_import_today:
    source: sensor.envoy_EnvoyMAC_lifetime_net_energy_consumption
    name: "Grid Energy Import Today"
    unique_id: um_grid_energy_import_today
    cycle: daily

  grid_energy_export_today:
    source: sensor.envoy_EnvoyMAC_lifetime_net_energy_production
    name: "Grid Energy Export Today"
    unique_id: um_grid_energy_export_today
    cycle: daily

  solar_production_daily:
    source: sensor.envoy_EnvoyMAC_lifetime_energy_production
    name: "Daily Solar Production"
    unique_id: um_solar_production_daily
    cycle: daily    

sensor:
  - platform: statistics
    name: "Average House Load 24h"
    entity_id: sensor.house_consumption_clean_kw
    state_characteristic: mean
    max_age:
      hours: 24
```

Copy the `system_settings.yaml` file into your Home Assistant configuration. You can do this by adding `input_number: !include system_settings.yaml` to your `configuration.yaml` file. Restart Home Assistant to generate the UI sliders.

### Step 2: Create the Dashboard UI Helpers
To ensure the custom dashboard renders perfectly and the seasonal logic works, you need to create three quick UI Helpers.
Go to **Settings > Devices & Services > Helpers** and click **+ Create Helper**.

**1. The Winter Mode Toggle**
* Select **Toggle**
* Name it: `Winter Mode`
* *(This creates `input_boolean.winter_mode`, which the system uses to dynamically switch between your summer and winter battery reserve floors).*

**2. The Holiday Mode Toggle**
* Select **Toggle**
* Name it: `Holiday Mode`
* *(This creates `input_boolean.holiday_mode`, which freezes AI tracking and maximizes grid exports while you are away).*

**3. The Grid Flow Sensor**
* Select **Template > Template a sensor**
* Name it: `Grid Daily In Out`
* Paste this exact code into the **State template** box:
```jinja2
⬇ {{ states('sensor.grid_energy_import_today') | float(0) | round(1) }} | ⬆ {{ states('sensor.grid_energy_export_today') | float(0) | round(1) }} kWh
```
* Leave all other fields blank and click **Submit**.

### Step 3: Configure Templates & Dashboard
Open `templates.yaml` and `Smart Export Controller Dashboard.yaml` in a text editor. Use **Find and Replace** to swap the following strings with your actual entity IDs:

| Find (My System) | Replace With (Your System) | What it is |
| :--- | :--- | :--- |
| `EnvoyMAC` | `YOUR_ENVOY_MAC` | Your Enphase Gateway MAC address. |
| `OctopusExportMPAN` | `YOUR_EXPORT_MPAN` | Your Octopus Export MPAN. |
| `OctopusImportMPAN` | `YOUR_IMPORT_MPAN` | Your Octopus Import MPAN. |
| `OctopusAccountId` | `YOUR_OCTOPUS_ACCOUNT_ID` | Found in your Intelligent Dispatching sensor name. |

### Step 4: Import Blueprints & Create Automations
The core logic of this system is powered by Home Assistant Blueprints. This means you do not need to manually edit YAML to configure the automations.

1. Download the `.yaml` files from the `/blueprints` folder of this repository.
2. Place them into your Home Assistant `/config/blueprints/automation/` directory.
3. In Home Assistant, navigate to **Developer Tools > YAML** and click **Reload Automations** (or **Reload Blueprints**).
4. Go to **Settings > Automations & Blueprints > Blueprints**.
5. Find the new Blueprints (e.g., "Energy: Smart Export Controller") and click **Create Automation**.
6. A user interface will appear with dropdown menus. Select your specific Battery, Export, and Import sensors from the dropdowns.
7. **⚠️ Important Notification Setup:** In the **Notification Entity** text box, enter the ID of your preferred Home Assistant notifier. 
   * For standard mobile push notifications, use: `notify.notify`
   * If you have an email integration configured, use your custom entity (e.g., `notify.my_email`).
8. Click **Save**. 

*Repeat this process for the Grid Charge Controller, ROI Audit, and Gateway Connection Alert blueprints.*

## 🧠 Understanding the Export Logic (Daytime Shield vs. Peak Dump)
This system uses a dual-target strategy to ensure you never accidentally drain your battery before the expensive evening peak.

1. **The Daytime Shield (Dynamic Reserve Target):** Used during the day. If export prices randomly spike at 14:00, the system will export *some* power, but strictly stops at this high reserve (e.g., 60%). This ensures your base load is protected until the evening.
2. **The Golden Slot Dump (Peak Export Floor):** When your most profitable 30-minute window arrives between 16:30 and 19:00, the system ignores the Daytime Shield. It calculates exactly how much energy your house needs for the rest of the evening minus any remaining predicted solar generation. It aggressively dumps everything else to the grid, sometimes pushing the battery down to 10-30% for maximum profit.

## 🎛️ Dashboard Glossary

### System Stats & Weather
* **Hard Block (EV/Cheap):** Shows `On` if your EV is charging or grid import rates are exceptionally cheap, safely locking the battery from exporting.
* **Peak Period:** Shows `On` between 16:30 and 19:00.
* **Current Dynamic Reserve Target:** The "Daytime Shield" percentage currently protecting your battery from random daytime export spikes.
* **Peak Export Floor:** The mathematically calculated minimum battery percentage required to survive the evening. During the "Golden Slot", the system will export everything above this line.
* **Today's Golden Export Slot:** The specific half-hour block with the absolute highest export payout.
* **Generated Today:** Physical solar energy generated so far.
* **7-Day Average Evening Usage (Learned):** The AI's rolling 7-day average of your actual house load between 16:30 and 23:30. Used to perfectly calculate your required Peak Export Floor.
* **Solar Left Today:** Solcast's prediction of how much solar energy will still be generated before sunset.
* **Total Expected Today / Tomorrow:** Solcast's daily weather predictions.
* **Battery Hours Left / Depleted At:** A smoothed estimate of when your battery will hit its reserve limit, based on your house's average load over the last 24 hours.
* **Discharged / Charged Today / Yesterday:** Accumulative daily cycling stats.

### System Variables
* **Grid Charge Forecast Threshold:** If tomorrow's Solcast predicted generation is *below* this number (kWh), the system will force a cheap overnight grid charge.
* **Battery Round Trip Efficiency (RTE):** The percentage of energy retained during a full charge/discharge cycle (usually ~0.83 or 83% for Enphase). Used to calculate true arbitrage profitability.
* **Evening Usage Estimate (Winter/Summer):** Acts as a fallback "training wheel." The system natively uses its 7-day AI average to dictate export limits, but falls back to these manual sliders if historical data is unavailable (e.g., fresh install).
* **Minimum SOC Floor (Winter/Summer):** The absolute lowest percentage the battery is allowed to reach during a forced export. This is your permanent safety net.
* **Winter Mode:** A manual toggle to switch the system into a more conservative battery retention state for darker, colder months.
* **Holiday Mode:** A manual toggle that freezes the 7-day AI tracker from learning your vacation usage, while dropping the battery floor to maximize grid export profits while you are away.
* **Evening Usage Estimate (Holiday):** The estimated kWh required for baseline house consumption (fridge, router, etc.) while the house is empty.
* **Minimum SOC Floor (Holiday):** The lowest battery percentage allowed while on vacation. Usually set very low (e.g., 10%) to safely maximize exports.
