# Home Assistant Smart Energy Arbitrage (Octopus + Enphase + Solcast)
**Current Version:** V6.10 (The Octoplus DFS Hoarding & Free Power Edition)

This repository contains an advanced, highly automated energy management system for Home Assistant. It is designed to optimize battery storage by making dynamic export and grid-charging decisions based on live energy prices, solar forecasts, and historical house consumption.

It is specifically tailored for UK users on **Octopus Energy** dynamic tariffs (like Agile, Intelligent, or Flux) with an **Enphase** solar/battery ecosystem.

## ✨ Key Features
* **V6.10 DFS Hoarding Mode:** The system now checks if an Octopus Saving Session (DFS) is scheduled later in the day. If detected, it actively suppresses standard 16:30-19:00 peak exports, hoarding battery capacity to ensure a maximum dump during the premium DFS payout window.
* **V6.9 Dynamic Burn-Down Fix:** The evening export floor calculation has been updated to completely remove the CT clamp offset. It now relies on pure, live evening consumption baseline tracking.
* **Free Power Vacuum Mode:** Actively listens for Octopus "Free Electricity" sessions. When triggered, it forces the battery to suck 100% free power from the grid, automatically shutting off when the session concludes.
* **Lifetime ROI Ledger:** A nightly automation calculates your exact daily savings from peak-rate avoidance, direct solar usage, and export earnings, depositing the exact monetary values into a permanent dashboard vault.
* **V6.8 Pure AI Failsafes:** Grid charging is now fully commanded by the `Calculated Target SOC` AI logic rather than independent thresholds. It dynamically calculates exactly how many kWh you need to survive until the sun rises.
* **V6.2 The "Hold at 100" Winter Fix:** Cured the "Overnight Leak." During winter, if the overnight target is 100%, the grid charger intentionally hangs in an active state until 05:27 AM. This forces the Enphase inverter into "Bypass Mode," powering the house directly from the cheap grid rather than letting the battery drain before the cheap window ends.
* **Intelligent Octopus Daytime Rescue:** Actively listens for random, unpredicted daytime cheap slots triggered by your EV. It checks your dynamic reserve floor and remaining solar forecast, hijacking the cheap electricity to top up your house battery *only* if the system was struggling.

## 📋 Prerequisites
To use this setup, you must have the following custom integrations installed in Home Assistant (available via HACS):
1. [Octopus Energy Integration](https://github.com/BottlecapDave/HomeAssistant-OctopusEnergy) (by BottlecapDave)
2. [Solcast PV Forecast](https://github.com/BJReplay/ha-solcast-solar) (by BJReplay)
3. Native Enphase Envoy Integration
4. [hacs_enphase_envoy_cloud](https://github.com/chinedu40/hacs_enphase_envoy_cloud) (by chinedu40)
5. [ApexCharts Card](https://github.com/RomRider/apexcharts-card) (by RomRider - Required for UI Dashboards)

## ⚙️ Installation & Setup

Because Home Assistant requires hardcoded entity IDs for some integrations and dashboards, you must replace my system's specific hardware IDs with your own for the templates and dashboard. 

### Step 1: Add the System Variables
Add the following into your `configuration.yaml`, replacing `EnvoyMAC` with your actual Enphase integration ID:

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

    template: !include templates.yaml
    input_number: !include system_settings.yaml

Copy `templates.yaml` and `system_settings.yaml` into your configuration directory and restart Home Assistant to generate the core sensors and UI sliders.

### Step 2: Create the Dashboard UI Helpers
Go to **Settings > Devices & Services > Helpers** and click **+ Create Helper**.

**1. The Winter Mode Toggle**
* Select **Toggle** and name it: `Winter Mode`

**2. The Holiday Mode Toggle**
* Select **Toggle** and name it: `Holiday Mode`

**3. The Grid Flow Sensor**
* Select **Template > Template a sensor** and name it: `Grid Daily In Out`
* Paste this exact code into the **State template** box:

        ⬇ {{ states('sensor.grid_energy_import_today') | float(0) | round(1) }} | ⬆ {{ states('sensor.grid_energy_export_today') | float(0) | round(1) }} kWh

### Step 3: Configure Templates & Dashboard
Open `templates.yaml` and the Dashboard YAML files in a text editor. Use **Find and Replace** to swap the following placeholder strings with your actual entity IDs:

| Find (Placeholder in code) | Replace With (Your System) | Example Format |
| :--- | :--- | :--- |
| `EnvoyMAC` | `YOUR_ENVOY_MAC` | `482425012345` |
| `OctopusExportMPAN` | `YOUR_EXPORT_MPAN` | `1234567890_1234567890` |
| `OctopusImportMPAN` | `YOUR_IMPORT_MPAN` | `1234567890_1234567890` |
| `e6e07484262222_1563990407` | `YOUR_GAS_MPRN` | `1234567890_1234567890` |
| `a_OctopusAccountId` | `YOUR_OCTOPUS_ACCOUNT_ID` | `a_123b4567` |

### Step 4: Disarm the Native Enphase App
To prevent the native Enphase gateway from fighting Home Assistant's commands, you must configure your Enphase app to allow HA to act as the primary brain. 
* **Option A (Recommended):** Set your Enphase battery profile to **Self-Consumption**. This stops Enphase from running its own internal market arbitrage, allowing HA to pierce through and command forced exports and charges.
* **Option B (If stuck on Savings Profile):** Set your CFG (Charge from Grid) schedule in the Enphase app to a wide-open `00:00 - 23:30`. Home Assistant will act as the "Brakes," turning the switch off immediately when your dynamic Target SOC is reached or expensive rates begin.

### Step 5: Import Blueprints & Create Automations
The core logic of this system is powered by Home Assistant Blueprints. This means you do not need to manually edit YAML to configure the automations.

1. Download the `.yaml` files from the `/blueprints` folder of this repository.
2. Place them into your Home Assistant `/config/blueprints/automation/` directory.
3. In Home Assistant, navigate to **Developer Tools > YAML** and click **Reload Automations** (or **Reload Blueprints**).
4. Go to **Settings > Automations & Blueprints > Blueprints**.
5. Find the new Blueprints and click **Create Automation**. The available blueprints include:
   * **Energy: Smart Export Controller**
   * **Energy: Battery Grid Charge Controller (V6.7 Smart Briefings)**
   * **Energy: IOG Daytime Rescue Charge**
   * **Energy: CFG Hard Failsafe Watchdog**
   * **Energy: Daily Battery ROI Audit**
   * **Energy: Lifetime Savings Nightly Deposit**
   * **Energy: Free Power Vacuum Mode**
   * **System: Enphase Gateway Connection Alert**
   * **Notification: Octopus Greener Night Reminder**
   * **Notification: Octoplus Saving Session (DFS) Alert**
   * **Notification: Weekly Solar Forecast Digest**
6. Use the dropdown menus to select your matching entities.
7. **⚠️ Important Notification Setup:** In the **Notification Entity** text box, enter the ID of your preferred Home Assistant notifier (e.g., `notify.notify` or `notify.my_email`).
8. Click **Save**. 

## 🧠 Understanding the Predictive Logic: Two Brains
The system treats morning charging and evening discharging entirely differently using an asymmetrical memory architecture:
* **Morning Charge (The 24-Hour Macro):** Uses a rolling 24-hour average to calculate the overnight charge target. This makes the system highly reactive to sudden changes (e.g., weekend laundry days) to ensure you never run out of power the following morning.
* **Evening Export (The 7-Day Micro):** Uses a rolling 7-day average of specific 16:30-19:00 usage to calculate the export floor. This makes the system highly smoothed and stable, ignoring random one-off spikes (like baking a cake on a Tuesday) to ensure maximum safe Agile profits without draining the battery too low.

## 🎛️ Dashboard Glossary

### System Stats & Weather
* **Calculated Target SOC:** The dynamically generated battery target percentage based on learned usage, solar yields, and arbitrage math. Used to prevent overcharging from the grid.
* **7-Day Average Morning/Evening Usage:** Your AI-learned house loads used to protect your baseline requirements.
* **Morning Solar Yield Forecast:** Solcast's prediction of solar generation specifically between 05:30 and 11:00 AM.
* **Next Greener Night:** Displays the date and time of your next scheduled Octopus Greener Night so you know when to plug in your EV.
* **Next Octoplus Saving Session:** Displays the date and time of incoming premium DFS grid events.
* **Next Free Electricity Session:** Displays the date and time of incoming free electricity events.
* **Hard Block (EV/Cheap):** Shows `On` if your EV is charging or grid import rates are exceptionally cheap, safely locking the battery from exporting.
* **Current Dynamic Reserve Target:** The "Daytime Shield" percentage currently protecting your battery from random daytime export spikes.
* **Peak Export Floor:** The mathematically calculated minimum battery percentage required to survive the evening. During the "Golden Slot", the system will export everything above this line.
* **Today's Golden Export Slot:** The specific half-hour block(s) with the absolute highest export payout. Will display `None` if no slots beat the hardware degradation floor.

### System Variables
* **Grid Charge Forecast Threshold:** If tomorrow's Solcast predicted generation is *below* this number (kWh), the system will force a cheap overnight grid charge to 100%.
* **Battery Round Trip Efficiency (RTE):** The percentage of energy retained during a full charge/discharge cycle (usually ~0.83 or 83% for Enphase). Used to calculate true arbitrage profitability.
* **Agile Spike Threshold (God Mode):** The price in pence (e.g., 50.0p) that will override all standard peak-window logic and instantly dump battery power at any time of day to capture extreme wholesale spikes.
* **Minimum Export Slot Price:** The price in pence (e.g., 12.0p) required to even consider dumping power during the evening peak. Prevents unnecessary hardware wear and tear for pennies.
* **Evening Usage Estimate (Winter/Summer):** Acts as a fallback "training wheel." The system natively uses its 7-day AI average to dictate export limits, but falls back to these manual sliders if historical data is corrupted or unavailable.
* **Minimum SOC Floor (Winter/Summer):** The absolute lowest percentage the battery is allowed to reach during a forced export. This is your permanent safety net.
* **Holiday Mode:** A manual toggle that freezes the 7-day AI tracker from learning your vacation usage, while dropping the battery floor to maximize grid export profits while you are away.
* **Morning/Evening Usage Estimate (Holiday):** The estimated kWh required for baseline house consumption (fridge, router, etc.) while the house is empty.
* **Minimum SOC Floor (Holiday):** The lowest battery percentage allowed while on vacation. Usually set very low (e.g., 10%) to safely maximize exports.
