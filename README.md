# Home Assistant Smart Energy Arbitrage (Octopus + Enphase + Solcast)

This repository contains an advanced, highly automated energy management system for Home Assistant. It is designed to optimize battery storage by making dynamic export and grid-charging decisions based on live energy prices, solar forecasts, and historical house consumption.

It is specifically tailored for UK users on **Octopus Energy** dynamic tariffs (like Agile, Intelligent, or Flux) with an **Enphase** solar/battery ecosystem.

## ✨ Key Features
* **Smart Peak Export**: Automatically exports battery power to the grid during the most profitable slots (e.g., 16:30-19:00) while ensuring enough reserve is kept to power your home through the evening.
* **Winter / Summer Modes**: Dynamically adjusts battery reserve floors and evening consumption estimates based on the season.
* **Predictive Grid Charging**: Calculates if it is mathematically profitable to charge the battery overnight from the grid, based on tomorrow's solar forecast and arbitrage margins.
* **EV Charging Protection**: Hard blocks exporting when your car is charging on a cheap overnight rate.
* **Daily ROI Audits**: Sends detailed daily emails breaking down your solar generation, peak avoidance savings, and net profit/loss.
* **Failsafe Logic**: Safely suspends operations and alerts you if the Enphase Gateway drops offline to prevent data corruption.

## 📋 Prerequisites
To use this setup, you must have the following custom integrations installed in Home Assistant (available via HACS):
1. [Octopus Energy Integration](https://github.com/BottlecapDave/HomeAssistant-OctopusEnergy) (by BottlecapDave)
2. [Solcast PV Forecast](https://github.com/BJReplay/ha-solcast-solar) (by BJReplay)
3. Native Enphase Envoy Integration
4. [hacs_enphase_envoy_cloud](https://github.com/chinedu40/hacs_enphase_envoy_cloud) (by chinedu40)

## ⚙️ Installation & Setup

Because Home Assistant requires hardcoded entity IDs for some integrations and dashboards, you must replace my system's specific hardware IDs with your own for the templates and dashboard. However, the automations are provided as easy-to-use **Blueprints**!

### Step 1: Add the System Variables
Add the following into your `configuration.yaml`, replacing `EnvoyMAC` with `YOUR_ENVOY_MAC`:

```yaml
utility_meter:
  daily_battery_discharged:
    source: sensor.envoy_EnvoyMAC_lifetime_battery_energy_discharged
    name: "Daily Battery Energy Discharged"
    cycle: daily

  grid_energy_import_today:
    source: sensor.envoy_EnvoyMAC_lifetime_net_energy_consumption
    name: "Grid Energy Import Today"
    cycle: daily

  grid_energy_export_today:
    source: sensor.envoy_EnvoyMAC_lifetime_net_energy_production
    name: "Grid Energy Export Today"
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

### Step 2: Configure Templates & Dashboard
Open `templates.yaml` and `Smart Export Controller Dashboard.yaml` in a text editor. Use **Find and Replace** to swap the following strings with your actual entity IDs:

| Find (My System) | Replace With (Your System) | What it is |
| :--- | :--- | :--- |
| `EnvoyMAC` | `YOUR_ENVOY_MAC` | Your Enphase Gateway MAC address. |
| `OctopusExportMPAN` | `YOUR_EXPORT_MPAN` | Your Octopus Export MPAN. |
| `OctopusImportMPAN` | `YOUR_IMPORT_MPAN` | Your Octopus Import MPAN. |

### Step 3: Import Blueprints & Create Automations
The core logic of this system is powered by Home Assistant Blueprints. This means you do not need to manually edit YAML to configure the automations.

1. Download the `.yaml` files from the `/blueprints` folder of this repository.
2. Place them into your Home Assistant `/config/blueprints/automation/` directory.
3. In Home Assistant, navigate to **Developer Tools > YAML** and click **Reload Automations** (or **Reload Blueprints**).
4. Go to **Settings > Automations & Blueprints > Blueprints**.
5. Find the new Blueprints (e.g., "Energy: Smart Export Controller") and click **Create Automation**.
6. A user interface will appear with dropdown menus. Select your specific Battery, Export, and Import sensors from the dropdowns, then click **Save**. 

*Repeat this process for the Grid Charge Controller, ROI Audit, and Gateway Connection Alert blueprints.*

## 🎛️ Understanding the UI Variables

Once installed, you will have several sliders in your Home Assistant UI. Here is what they control:

* **System Battery Capacity**: Your total usable battery storage in kWh (e.g., 5.0).
* **Battery Round Trip Efficiency (RTE)**: The percentage of energy retained during a full charge/discharge cycle (usually ~0.83 or 83% for Enphase). Used to calculate true arbitrage profitability.
* **Grid Charge Forecast Threshold**: If tomorrow's Solcast predicted generation is *below* this number (kWh), the system will force a cheap overnight grid charge.
* **Minimum SOC Floor (Winter/Summer)**: The absolute lowest percentage the battery is allowed to reach during a forced export.
* **Evening Usage Estimate (Winter/Summer)**: How much power your house typically uses between 16:30 and 23:30. The system uses this to dynamically hold back enough battery power so you don't export everything and end up importing at peak rates later.