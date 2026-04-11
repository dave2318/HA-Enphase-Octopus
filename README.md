# Home Assistant Smart Energy Arbitrage (Octopus + Enphase + Solcast)
**Current Version:** V6.1 (The Wall Street Edition)

This repository contains an advanced, highly automated energy management system for Home Assistant. It is designed to optimize battery storage by making dynamic export and grid-charging decisions based on live energy prices, solar forecasts, and historical house consumption.

It is specifically tailored for UK users on **Octopus Energy** dynamic tariffs (like Agile, Intelligent, or Flux) with an **Enphase** solar/battery ecosystem.

## ✨ Key Features
* **V6.1 Wall Street Mode (Energy Arbitrage):** The system actively calculates Round Trip Efficiency (RTE) to find profitable spreads between tonight's import rates and tomorrow's export rates. If `(Tomorrow's Average Export * 0.83) > Tonight's Import`, it overrides self-consumption logic and forces a 100% grid charge to maximize next-day profit.
* **V6.1 Hardware Wear Protection (12p Floor):** Introduces a hard minimum export price floor. If peak export rates drop below your calculated profitability threshold (default 12.0p/kWh), the system cancels the export, saving the battery from unnecessary hardware degradation and using the free solar to run your house instead.
* **V5.0 Self-Tuning Morning Charge:** The system observes your house's exact energy usage every morning (including overnight phantom drain) and creates a 7-day rolling average. When calculating overnight grid charges, it scans tomorrow's half-hourly Solcast data (05:30 - 11:00) and calculates a perfect "Target SOC" to cover your specific learned routine while leaving the battery empty enough to soak up afternoon sun.
* **Smart Peak Export:** Automatically exports battery power to the grid during the most profitable slots (e.g., 16:30-19:00) while ensuring enough reserve is kept to power your home through the evening.
* **Predictive Evening Reserve:** Learns your actual household consumption over a 7-day rolling average to dynamically calculate exactly how much battery to hold back for the evening, automatically falling back to manual estimates if data is unavailable.
* **Intelligent Octopus Daytime Rescue:** Actively listens for random, unpredicted daytime cheap slots triggered by your EV. Instead of just blindly taking the slot, it checks your dynamic reserve floor and remaining solar forecast, hijacking the cheap electricity to top up your house battery *only* if the system was struggling.
* **Hardware Failsafe Watchdog:** A dedicated background automation that guarantees grid charging is forcefully shut down outside of cheap rate windows. This protects your wallet from Home Assistant reboots, cancelled `wait_templates`, or accidental manual overrides.
* **Holiday Mode:** Maximizes export profits while you are away by dropping the reserve floor, and automatically freezes the 7-day predictive AI from learning your artificially low vacation usage.
* **Winter / Summer Modes:** Dynamically adjusts battery reserve floors based on the season.
* **Advanced Visualizations:** Utilizes ApexCharts to draw 48-hour forward-looking step graphs for Octopus pricing, giving a complete visual of tomorrow's energy market.

## 📋 Prerequisites
To use this setup, you must have the following custom integrations installed in Home Assistant (available via HACS):
1. [Octopus Energy Integration](https://github.com/BottlecapDave/HomeAssistant-OctopusEnergy) (by BottlecapDave)
2. [Solcast PV Forecast](https://github.com/BJReplay/ha-solcast-solar) (by BJReplay)
3. Native Enphase Envoy Integration
4. [hacs_enphase_envoy_cloud](https://github.com/chinedu40/hacs_enphase_envoy_cloud) (by chinedu40)
5. [ApexCharts Card](https://github.com/RomRider/apexcharts-card) (by RomRider - Required for V6.1 UI Dashboards)

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
* *(This creates `input_boolean.winter_mode`, which the system uses to dynamically switch between your summer and winter battery reserve floors)*.

**2. The Holiday Mode Toggle**
* Select **Toggle**
* Name it: `Holiday Mode`
* *(This creates `input_boolean.holiday_mode`, which freezes AI tracking and maximizes grid exports while you are away)*.

**3. The Grid Flow Sensor**
* Select **Template > Template a sensor**
* Name it: `Grid Daily In Out`
* Paste this exact code into the **State template** box:
```jinja2
⬇ {{ states('sensor.grid_energy_import_today') | float(0) | round(1) }} | ⬆ {{ states('sensor.grid_energy_export_today') | float(0) | round(1) }} kWh
```
* Leave all other fields blank and click **Submit**.

### Step 3: Configure Templates & Dashboard
Open `templates.yaml` and `Smart Export Controller Dashboard.yaml` in a text editor. Use **Find and Replace** to swap the following placeholder strings with your actual entity IDs:

| Find (Placeholder in code) | Replace With (Your System) | Example Format |
| :--- | :--- | :--- |
| `EnvoyMAC` | `YOUR_ENVOY_MAC` | `482425012345` |
| `OctopusExportMPAN` | `YOUR_EXPORT_MPAN` | `electricity_1234567890_1234567890_export` |
| `OctopusImportMPAN` | `YOUR_IMPORT_MPAN` | `electricity_1234567890_1234567890` |
| `IntelligentDispatchVehicleId` | `YOUR_INTELLIGENT_DISPATCH_ID` | `IntelligentDispatchVehicleId` |
| `OctopusAccountId` | `YOUR_OCTOPUS_ACCOUNT_ID` | `OctopusAccountId` |

*(Tip: You can usually find these exact strings by looking at the entity names generated by the Octopus Energy integration)*.

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
   * **Energy: Battery Grid Charge Controller (V5.0 Self-Tuning)**
   * **Energy: IOG Daytime Rescue Charge**
   * **Energy: CFG Hard Failsafe Watchdog**
   * **Energy: Daily Battery ROI Audit**
   * **Energy: Lifetime Savings Nightly Deposit**
   * **System: Enphase Gateway Connection Alert**
   * **Notification: Octopus Greener Night Reminder**
   * **Notification: Weekly Solar Forecast Digest**
6. Use the dropdown menus to select your matching entities.
7. **⚠️ Important Notification Setup:** In the **Notification Entity** text box, enter the ID of your preferred Home Assistant notifier (e.g., `notify.notify` or `notify.my_email`).
8. Click **Save**. 

## 🧠 Understanding the Predictive Logic
The system uses "Array-Based Memory" to observe your behavior over time.
* **Evening Logic**: At 16:30 and 23:30, the system calculates how much energy you used during the peak/evening period and adds it to a 7-day rolling average. This is used to calculate your "Export Floor".
* **Morning Logic**: At 11:00 AM daily, the system logs your total consumption since midnight. This captures your base-load, phantom drain, and morning routine. This average is used to calculate the "Target SOC" for your overnight charge, ensuring you have exactly enough power to reach the sunny period of the day without overcharging.
* **The 23:30 Decision (V6.1)**: Before the cheap overnight rates begin, the brain cross-references your Morning Logic against tomorrow's Solcast data. It then polls the Octopus API for tomorrow's daytime export rates. It decides whether to stay idle, perform a protective "Survival Charge", or execute a 100% "Wall Street Arbitrage" charge based on pure financial mathematics.

## 🎛️ Dashboard Glossary

### System Stats & Weather
* **Calculated Target SOC:** The dynamically generated battery target percentage based on learned usage, solar yields, and arbitrage math. Used to prevent overcharging from the grid.
* **7-Day Average Morning Usage (Learned):** Your AI-learned morning house load (00:00 to 11:00).
* **7-Day Average Evening Usage (Learned):** Your AI-learned evening house load used to protect your peak export floor.
* **Morning Solar Yield Forecast:** Solcast's prediction of solar generation specifically between 05:30 and 11:00 AM.
* **Next Greener Night:** Displays the date and time of your next scheduled Octopus Greener Night so you know when to plug in your EV.
* **Hard Block (EV/Cheap):** Shows `On` if your EV is charging or grid import rates are exceptionally cheap, safely locking the battery from exporting.
* **Peak Period:** Shows `On` between 16:30 and 19:00.
* **Current Dynamic Reserve Target:** The "Daytime Shield" percentage currently protecting your battery from random daytime export spikes.
* **Peak Export Floor:** The mathematically calculated minimum battery percentage required to survive the evening. During the "Golden Slot", the system will export everything above this line.
* **Today's Golden Export Slot:** The specific half-hour block with the absolute highest export payout. Will display `None (>12p)` if no slots beat the hardware degradation floor.
* **Export Rates (Today & Tomorrow):** A 48-hour forward-looking ApexCharts step graph displaying exactly how Octopus is pricing the current and following day.
* **Generated Today:** Physical solar energy generated so far.
* **Solar Left Today:** Solcast's prediction of how much solar energy will still be generated before sunset.
* **Total Expected Today / Tomorrow:** Solcast's daily weather predictions.
* **Battery Hours Left / Depleted At:** A smoothed estimate of when your battery will hit its reserve limit, based on your house's average load over the last 24 hours.

### System Variables
* **Grid Charge Forecast Threshold:** If tomorrow's Solcast predicted generation is *below* this number (kWh), the system will force a cheap overnight grid charge to 100%.
* **Battery Round Trip Efficiency (RTE):** The percentage of energy retained during a full charge/discharge cycle (usually ~0.83 or 83% for Enphase). Used to calculate true arbitrage profitability.
* **Evening Usage Estimate (Winter/Summer):** Acts as a fallback "training wheel." The system natively uses its 7-day AI average to dictate export limits, but falls back to these manual sliders if historical data is unavailable (e.g., fresh install).
* **Minimum SOC Floor (Winter/Summer):** The absolute lowest percentage the battery is allowed to reach during a forced export. This is your permanent safety net.
* **Winter Mode:** A manual toggle to switch the system into a more conservative battery retention state for darker, colder months.
* **Holiday Mode:** A manual toggle that freezes the 7-day AI tracker from learning your vacation usage, while dropping the battery floor to maximize grid export profits while you are away.
* **Evening Usage Estimate (Holiday):** The estimated kWh required for baseline house consumption (fridge, router, etc.) while the house is empty.
* **Minimum SOC Floor (Holiday):** The lowest battery percentage allowed while on vacation. Usually set very low (e.g., 10%) to safely maximize exports.

*** *Disclaimer: Always double-check your own specific Enphase entity IDs when importing the templates!*.
