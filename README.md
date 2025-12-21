# Documentation: Insurgency Growth Model Simulation

## 1. Overview

The model simulates the interaction between two opposing forces (Government and Insurgents) competing for the allegiance of a neutral population through:
1.  **Propaganda:** Psychological operations to convert the population.
2.  **Military Conflict:** Combat attrition and troop generation.

---

## 2. Model Architecture

The simulation uses **Euler integration** with a time step (`dt`) of 1.0 (representing 1 month). The system is closed (population moves between stocks but total population is generally conserved, minus combat deaths).

### 2.1 Sectors
The code is divided into three logical sectors, mirroring the thesis structure:

1.  **Population Sector:** Tracks the flow of people between three stocks:
    *   **Government (`GC`):** Loyal citizens.
    *   **Insurgent (`INS`):** Active insurgents and supporters.
    *   **Neutral (`NC`):** Uncommitted population.
2.  **Propaganda Sector:** Determines the rate of flow between population stocks based on "Propaganda Effort" (`PI` and `PG`).
3.  **Military Sector:** Handles troop recruitment, combat attrition (using Lanchester equations), and logistical delays.

### 2.2 Key Mechanics
*   **Table Lookups:** Non-linear relationships (e.g., *Effect of Propaganda vs. Probability of Conversion*) are defined via lookup tables, using linear interpolation.
*   **Delays:** Information and material delays (e.g., training time for troops) are modeled using first-order exponential smoothing (`Delay1` class).
*   **Phase Change:** The combat model switches from "Guerrilla Warfare" to "Conventional Warfare" when Insurgent Troops (`TI`) exceed **1,000**.

---

## 3. Code Structure Analysis

### 3.1 Dependencies
*   `numpy`: For numerical operations and random number generation (simulating intelligence errors).
*   `pandas`: For data storage and DataFrame management.
*   `matplotlib`: For plotting simulation results.
*   `IPython.display`: For rendering HTML tables and formatted headers in a notebook environment.

### 3.2 Helper Functions
| Function | Description |
| :--- | :--- |
| `table_lookup` | Performs linear interpolation on the static data tables (e.g., `TPIT`, `TFCI`) given an input value `x`. Handles clamping at boundaries. |
| `Delay1` | A class implementing first-order exponential smoothing. Used to simulate time delays in troop reinforcement (Training/Bureaucracy). |
| `_clamp01` | Constrains values between 0.0 and 1.0. |
| `_nice_header` | HTML styling for notebook output. |
| `_style_table_...` | formatting functions to match the visual style of the thesis tables (integers for people, floats for ratios). |

### 3.3 Data Tables (`TABLE_DATA`)
The dictionary `TABLE_DATA` contains the exact coordinates derived from the graphs in the thesis.
*   **Propaganda Tables:** `TPIT`, `TPGT` (Indicated Effort); `TPRI`, `TPRG` (Reach probability); `TPFHI`, `TPFGI`, etc. (Conversion probabilities).
*   **Military Tables:** `TTMG` (Target Military Govt); `TFCI`, `TFCG` (Fighting Constants); `TEENC` (Expected Encounters); `TFST` (Force Size).

---

## 4. Simulation Logic (`run_simulation`)

The core function `run_simulation` executes the time-stepped loop. Below is the step-by-step logic within the loop:

### Step 1: Initialization
*   Sets initial populations (Default: `PT`=200k, `INS`=800, `GC`=user_defined).
*   Calculates initial propaganda efforts (`PIPRE`, `PGPRE`) based on the initial population ratios.

### Step 2: Propaganda Sector (Inside Loop)
1.  **Intelligence Noise:** Random noise is added to the calculation of population ratios (`GCR`, `IR`) to simulate imperfect intelligence gathering by both sides (`rng.uniform`).
2.  **Effort Calculation:**
    *   Indicated Effort (`PIT`, `PGT`) is looked up based on the perceived population ratio.
    *   Actual Effort (`PI`, `PG`) is an exponentially smoothed version of Indicated Effort (representing decision-making inertia).
3.  **Probability Calculation:**
    *   `PRI` / `PRG`: Probability propaganda *reaches* a citizen.
    *   `PNHI`, `PMGI`, etc.: Probability a citizen *converts* (e.g., Neutral to Insurgent). These are derived from multiplying Reach × Population Share × Conversion Table.

### Step 3: Combat Sector (Inside Loop)
1.  **Troop Allocation:**
    *   **Insurgents:** Troop levels (`TI`) are a percentage of the total Insurgent population (`INS`), determined by `TPINS`.
    *   **Government:** Total troops (`TG`) is the sum of Regulars (`TGR`) and Police (`TGP`).
2.  **Warfare Phase Check:**
    *   If `TI < 1000`: **Guerrilla Warfare**. High evasion, low engagement frequency.
    *   If `TI >= 1000`: **Conventional Warfare**. Engagement frequency (`EENC`) and Force Size (`FST`) scale up significantly.
3.  **Lanchester Equations:**
    *   `DG` (Government Attrition): Calculated using Aimed Fire logic.
    *   `DI` (Insurgent Attrition): Calculated using Area Fire logic (Guerrilla) or Aimed Fire (Conventional).
    *   *Note:* The variable `IC` (Insurgent Constant) handles the mathematical switch between Area/Aimed fire forms.

### Step 4: Troop Replacement & Delays
1.  **Police (`TGP`):** The government attempts to maintain police at 5% of the total population. Adjustments are smoothed via `del_tgp`.
2.  **Regulars (`TGR`):** The government sets a "Target" strength (`TMG`) based on perceived insurgent strength.
    *   If `TMG > Actual`: Reinforcements are ordered (subject to `Delay1`).
    *   If `TMG < Actual`: Troops are removed (subject to `Delay1`).

### Step 5: Integration (State Updates)
The rates calculated above are applied to the stocks:
*   `INS += dt * (Generations - Attrition)`
*   `GC += dt * (Generations - Attrition)`
*   `NC` is adjusted similarly.
*   Troop levels are updated based on recruitment flow and combat deaths.

---

## 5. Variable Dictionary

### Population Variables
| Variable | Meaning | Definition |
| :--- | :--- | :--- |
| `PT` | Population Total | `GC + INS + NC` |
| `GC` | Government Population | Citizens loyal to the government. |
| `INS` | Insurgent Population | Citizens loyal to the insurgency. |
| `NC` | Neutral Population | Uncommitted citizens. |
| `GCR` | Government Ratio | `GC / PT` |
| `IR` | Insurgent Ratio | `INS / PT` |
| `NCR` | Neutral Ratio | `NC / PT` |

### Propaganda Variables
| Variable | Meaning | Definition |
| :--- | :--- | :--- |
| `PI` | Insurgent Propaganda | Smoothed effort level (0-10 scale). |
| `PG` | Government Propaganda | Smoothed effort level (0-10 scale). |
| `PIT` / `PGT` | Indicated Propaganda | Target effort based on population ratios. |
| `PRI` / `PRG` | Prob. of Reach | Probability propaganda reaches a specific citizen. |
| `PNHI` | Prob. Neutral -> Ins | Probability flow from Neutral to Insurgent. |
| `PMGI` | Prob. Govt -> Ins | Probability flow from Govt to Insurgent. |
| `PMNG` | Prob. Neutral -> Govt | Probability flow from Neutral to Govt. |

### Military Variables
| Variable | Meaning | Definition |
| :--- | :--- | :--- |
| `TI` | Total Insurgents | Active insurgent combatants (subset of `INS`). |
| `TG` | Total Govt Troops | `TGR` (Regulars) + `TGP` (Police). |
| `TMG` | Target Military Govt | The perceived required troop count by Govt. |
| `FST` | Force Size | Average size of a unit in combat. |
| `EENC` | Expected Encounters | Number of battles per time period. |
| `IC` | Insurgent Constant | Switch variable for Lanchester equations (1000 or calculated). |
| `DG` | Dead Govt | Govt troops killed this step. |
| `DI` | Dead Insurgents | Insurgent troops killed this step. |
| `KILLR` | Exchange Ratio | `DG / DI` (Govt killed per 1 Insurgent killed). |

---

## 6. Outputs

The script generates specific artifacts corresponding to the original thesis results:

1.  **CSV Export:** `insurgency_standard_run.csv` containing all state variables for every time step.
2.  **DataFrames / HTML Tables:**
    *   **TABLE II:** Population Profiles (GC, INS, NC over time).
    *   **TABLE IV:** Troop Profiles (TI, TG, Required TG, Exchange Ratio).
    *   **TABLE V:** Replacement Figures (Requests vs. Actual Arrivals for Police/Regulars).
3.  **Visualizations (Matplotlib):**
    *   Population Curves (The classic "Growth, Decline, Abatement" curves).
    *   Propaganda Effort Levels.
    *   Troop Levels (showing the lag between Required and Actual troops).

## 7. Assumptions & Limitations
*   **Closed System:** No external migration.
*   **Resource Constraints:** The model assumes the Insurgents recruit strictly from the population, while the Government recruits Regulars from "outside" the system (an infinite pool, limited only by policy delays).
*   **Symmetry:** Propaganda functions are largely symmetric but weighted slightly in favor of Insurgents for reaching the Neutral population.
*   **Scale:** The "Propaganda" variable is abstract (0-10), while Troops and Population are absolute numbers.
