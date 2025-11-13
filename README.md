# README — Agentic & Multi-Agent Housing Safety (NetLogo 3D 7.0)

## OVERVIEW

This model simulates how households, NGOs, banks, professionals, and a government interact to improve the safety of homes over time in an earthquake affected community in Nepal. It runs in **NetLogo 3D 7.0** and uses GIS layers to position homes and roads, with an optional DEM rendered as a 2.5D terrain.

Agents:

* **homes** (one per building feature)
* **residents** (0–1 per home, if a CSV record matches)
* **ngos** (1)
* **professionals** (1 coordinator agent representing a team)
* **banks** (1)
* **governments** (1)
* **road-nodes / road-segments** (optional; for drawing the road network)

Key ideas:

* Each home has 14 binary safety features (`f1`…`f14`); **safety score** is their sum (0–14).
* **Target safety** is 8. Households, professionals, loans, grants, and training all help reach/improve this.
* A monthly cycle updates incomes, budgets, loan payments, and government allocations.
* An **earthquake** can degrade random features depending on current safety.

## REQUIREMENTS

* **NetLogo 3D 7.0**.
* Extensions:

  * `gis`, `csv`, `nw`, `table`, `view2.5d`
* Data files in a local `data/` folder (see below).

## DATA FOLDER & SCHEMA

Expected paths (relative to the model file):

**Required**

* `data/Community_Buildings.shp` (+ `.dbf`, `.shx`, `.prj`)

  * Must include an ID field named **`OSM_ID`** (default).
  * (The code briefly sets `building_id` but **overwrites** to `OSM_ID` as final.)
* `data/housing_data.csv`

  * Must contain at least these headers (case-sensitive):
    `OSM_ID`, `housing_safety_score`, `income`, `isTrained`,
    `f1`…`f14` (0/1), `motivation`
  * `OSM_ID` must match the buildings shapefile IDs.
  * `isTrained`: 0/1; `motivation`: 1=no upgrades beyond 8, 2=repair 3 more features, 3=repair all.

**Optional**

* `data/Community_Roads.shp` (+ companions) — roads for display.
* `data/Community_DEM.asc` — elevation (AAIGrid).
* `data/Community_Slope.asc`, `data/Community_Landuse.asc` — optional rasters.

World sizing & draping:

* If a DEM is present, the world is resized to the DEM extent and a **2.5D surface** is built.
* If no DEM, the world envelope is derived from available vector/rasters.

## INSTALL & OPEN

1. Install the required extensions (if your NetLogo doesn’t already include them).
2. Place the model file (`.nlogo3d`) and a `data/` folder (as above) together.
3. Open the model in **NetLogo 3D** (not 2D).

## INTERFACE SETUP (Controls you should create)

**Buttons**

* `setup` (Observer) — calls `setup`.
  Loads data, spawns agents, draws roads, centers the 3D view, colors DEM, sets plots.
* `go` (Observer, forever) — calls `go`.
  Main loop (see “Model Dynamics”).
* `earthquake` (Observer, once) — calls `earthquake`.
  Randomly degrades features based on current home safety.

**(Required) Sliders / inputs**
The code expects these multipliers to exist as GUI variables:

* `buying-homes-multiplier` (e.g., 0.3)
* `ngos-money-multiplier`   (e.g., 0.2)
* `residents-money-multiplier` (e.g., 0.5)

They should typically be **fractions that sum to 1.0**; the government’s monthly budget is split into:

* buying homes for critical cases,
* funding NGOs,
* funding residents directly.

Optional (to expose/tune defaults set in code):

* `interest-rate` (default 1.05; 5% APR applied once to loan principal)
* `train-batch` (default 20)
* `teaching-time` (default 50 ticks)
* `time-to-repair-feature` (default 5)
* `loan-time-payment` (default 30 ticks; loan approvals monthly)
* `loan-payback-time` (default 30 ticks; payment cadence)
* `granting-batch` (default 50,000; *note:* code uses continuous budgeting, not a fixed batch grant)

**Plots** (create with these exact titles)

* `Mean Safety Score of All Households`
* `Repairs: Needing vs Under-Repair vs Repaired`
  Pens: `needing`, `under-repair`, `repaired`

**Helpful monitors (optional)**

* `mean-safety` → reporter
* `trained-population` → reporter
* `homes-needing-repair`, `homes-under-repair`, `homes-repaired`
* `homes-with-minimum-8-s-score`
* Long-run stats (captured at tick 90/180/360/720/1080):
  `mean-safety-3-months-after`, `mean-safety-6-months-after`, `mean-safety-1-year-after`, `mean-safety-2-years-after`, `mean-safety-3-years-after`
  and `safe-houses-…` counterparts.

## RUNNING THE MODEL

1. Ensure data files exist (see “Data Folder & Schema”). On load, the model will stop with a helpful message if something is missing.
2. Set the **multiplier sliders** so they add to 1.0 (e.g., 0.4 / 0.2 / 0.4).
3. Click **`setup`**.

   * Buildings become **homes** (circles).
   * If CSV contains a matching `OSM_ID`, a **resident** is created and attached to that home.
   * Unmatched homes are **unowned** (potential gov purchases) with a randomized safety and a `gov-cost`.
   * Roads (if present) are drawn as 3D links.
   * DEM (if present) is colored and shown in 2.5D; agents are draped.
4. Click **`go`** to start the simulation.
5. (Optional) Click **`earthquake`** at any time to apply a hazard shock.

## MODEL DYNAMICS (what happens each tick)

Order inside `go`:

1. `calculate-cost` —

   * For homes with owner:

     * If `safety < 8`: computes **minimum-cost** set of features to reach 8 (cheapest damaged features).
     * If `safety ≥ 8`: uses resident `motivation`:
       1 → no extra work, 2 → repair 3 cheapest damaged features, 3 → repair **all** remaining.
2. `get-monthly-fee` —

   * Every 30 ticks: residents earn `0.5% * income`; the government refreshes its sub-budgets; NGOs receive their share.
3. `household-flow` — residents decide:

   * If `safety ≤ 1`: ask for a **gov home**.
   * If trained and not under pro repair: **self-repair** occasionally (7-tick task per feature, up to `features-fixed-manually`).
   * If wealthy (income ≥ 2× cost): request **professionals**.
   * If not wealthy (income/2 < cost): request **gov money**.
   * If `safety ≥ 8` and `motivation > 1`: may **apply for bank loan** and subsequently **pay it** monthly.
4. `ngos-flow` —

   * NGOs train up to `train-batch` **untrained** residents per cycle, taking `teaching-time` ticks and drawing from NGO funds.
5. `create-priority-list` — sorts residents who asked for gov funds by **lowest income** and affordability.
6. `gov-flow` —

   * Initially funds NGOs (once).
   * Maintains a **critical list** (people needing homes); periodically buys the **cheapest unowned homes** it can afford and assigns them 1-to-1.
   * Every `funding-time` ticks, funds as many residents from the **priority list** as the budget allows (pays exactly their computed repair cost).
7. `prof-flow` —

   * Professionals queue residents who asked for help (and whose homes have a non-empty plan), up to `num-prof` concurrent jobs.
   * Repair time per home = `time-to-repair-feature × features-to-repair`.
   * On completion, resident pays, pros collect, home features flip to 1, plan is recalculated.
8. `bank-flow` —

   * Creates a loan queue from applicants (higher income first).
   * If the bank has funds and the applicant can afford the loan (simple affordability check), the bank issues a loan for the **exact repair cost**; monthly payment = total with interest / 12.
9. Display updates — recolor homes (`red` < 8, `yellow` 8–13, `blue` = 14), refresh plots, and (if DEM) re-drape agents.
10. Milestones — at ticks **90, 180, 360, 720, 1080** the model stores mean safety and count of safe homes for later reporting.

## HAZARD: `earthquake`

* Reduces a random number of currently intact features, with **larger hits on lower-safety homes**.
* Recolors homes after the shock.

## KEY PARAMETERS & CONSTANTS (defaults in code)

* Feature repair costs: `[834 790 544 1020 940 600 850 785 850 670 970 860 1075 1485]`
* Target safety: **8**
* Interest rate: **1.05** (principal × 1.05 ⇒ paid back over 12 months)
* Monthly step: **30 ticks**
* NGO: `train-batch=20`, `cost-of-training=23200`, `teaching-time=50`
* Government: `government-money=80000` (seed), `funding-time=30`
* Bank: `banks-money=100000`, `loan-time-payment=30`, `loan-payback-time=30`
* Professionals: `num-prof=15`, `time-to-repair-feature=5`

## REPORTERS (for monitors/BehaviorSpace)

* `mean-safety`, `trained-population`
* `homes-needing-repair`, `homes-under-repair`, `homes-repaired`
* `homes-with-minimum-8-s-score`
* Milestones (mean & counts):
  `mean-safety-3-months-after`, `…-6-months-after`, `…-1-year-after`, `…-2-years-after`, `…-3-years-after`
  `safe-houses-3-months-after`, `…-6-months-after`, `…-1-year-after`, `…-2-years-after`, `…-3-years-after`

## TROUBLESHOOTING

* **“Missing …shp/CSV”**: ensure the paths exactly match the filenames above and that shapefiles include their `.dbf/.shx/.prj`.
* **No agents appear**: check the `OSM_ID` column exists in both buildings `.dbf` and CSV, and that some IDs match.
* **DEM not shown**: verify `Community_DEM.asc` exists; otherwise the model still runs in a flat world.
* **`view2.5d` not found**: install the `view2.5d` extension into NetLogo’s `extensions/` directory.
* **Undefined multipliers**: you must add sliders named `buying-homes-multiplier`, `ngos-money-multiplier`, `residents-money-multiplier`.
* **Units/projection**: keep all GIS layers in the same projected CRS; include a proper `.prj`.

## TIPS

* Use **BehaviorSpace** to explore multiplier allocations and budgets; record the reporters above.
* For reproducibility, set a **fixed random seed** in the NetLogo “Settings…” dialog.
* Start without a DEM while you validate CSV/IDs; add the DEM later for 2.5D visuals.
* You can manually trigger **`earthquake`** multiple times to test resilience and policy responses.

## CREDITS & LICENSE

* Built for an **Agentic & Multiagent Systems** course demo.
* Uses NetLogo 3D and the `gis`, `csv`, `nw`, `table`, `view2.5d` extensions.
* GIS and CSV data are user-provided.
* ChatGPT for formatting ReadMe file - Reviewed and Edited by Team
*  License: MIT License

MIT License

Copyright (c) 2025 jenkenpow76

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
