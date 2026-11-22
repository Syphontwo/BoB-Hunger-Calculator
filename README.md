# BoB-Hunger-Calculator
A dependency-free, single-page calculator that estimates elapsed time to four food-level milestones in Beasts of Bermuda, given a dinosaur-specific multiplier (M), growth (G), and current food remaining (H). Calculations run locally in the browser; no game connection, backend, cookies, telemetry, accounts, or external libraries are required.

Data is provisional, approximate, and under active improvement. This is not authoritative.

**[Launch the Hunger Calculator](https://syphontwo.github.io/BoB-Hunger-Calculator/)**

## Quick start

- Open `index.html` directly in a modern browser; it also works offline.
- Select a dinosaur from the 28 named/form presets, or select **Custom multiplier**.
- Set `M` (editable even when a preset is selected), growth `G`, and current food remaining as a percentage `H` from `0` to `100`.
- Read the predicted **tick counts**, **times**, and **food-over-time plot**. Use **Copy results** for a plain-text record.
- After deploying, **Copy shareable link** creates a URL with the selected inputs.

The site is currently one file (`index.html`) with embedded HTML, CSS, JavaScript, and SVG. Included `preview.png` and `preview-mobile.png` are reference screenshots, not dependencies.

## Mathematical model

### Inputs and units

| Symbol | Meaning | Units |
| --- | --- | --- |
| `M` | Species/form-specific hunger multiplier | Model parameter; estimated from measurements or provisionally assumed |
| `G` | Actual growth level | Game growth value, e.g., `4` or `10` |
| `P_t` | Food remaining immediately **before** tick `t` | Percentage points on a nominal `0..100` bar |
| `H_t = P_t / 100` | Remaining food fraction | `0..1` |
| `D_t` | Food lost on tick `t` | Percentage points per **15-second** tick |
| `G_eff` | Effective growth used for drain | `G` when `P_t >= 35`, otherwise `1.32` |

The proposed combined high-food formula is:

```text
D_t = (M * G_eff - 1) * (0.25 + 0.75 * H_t)
    = (M * G_eff - 1) * (0.25 + 0.0075 * P_t)
```

The **fixed `-1` intercept** is part of the hypothesis; it has not been established as a game constant for all species and only confirmed through experimentation. The hunger scaling term `0.25 + 0.75*H` was inferred from measured Zupaysaurus food-level curves and reconfirmed through spot checking with other species. The **1.32 low-food growth substitution** is an additional, less-validated hypothesis. The in-game elder/gauntlet growth transition around 1.32 motivated that choice; it is **not proof** of the calculation performed by the game.

### Tick-by-tick algorithm (the code's actual behavior)

The app **simulates ticks** rather than using a closed-form time estimate. The order of operations matters:

```text
remaining = starting_food_percentage
record any milestones already satisfied at tick 0

for tick = 1 to 100000:
    effective_growth = 1.32 if remaining < 35 else entered_growth
    base_drain = M * effective_growth - 1
    if base_drain <= 0: stop (no further decay predicted)

    drain = base_drain * (0.25 + 0.0075 * remaining)
    remaining = max(0, remaining - drain)
    record first tick reaching each as-yet-unreached milestone
    if all milestones reached: stop

elapsed_seconds = tick_count * 15
```

**At exactly 35.000...%**, the next tick still uses the entered actual growth. Only a subsequent tick that *begins below 35%* uses 1.32. The amount of overshoot below 35% is preserved.

### Calibration of a species multiplier

From **one measured 15-second drain** at growth `10`, food `100%`, and the controlled test conditions described below:

```text
D_10,100 = 10*M - 1
M = (D_10,100 + 1) / 10
```

Examples: Zupaysaurus observed `17.8` -> `M = 1.88`; Sai observed `16.6` -> `M = 1.76`; Elas observed `16.9` -> `M = 1.79`.

**Precision caveat:** The game exposes drain to **one decimal place**, so the inferred `M` is not exact.

## Source measurements and collection method

**Primary source:** My workbook where I've stored the raw data as I've collected it - [https://docs.google.com/spreadsheets/d/13PnO1ziA9xUgqEugiZcK97yFO6MtfuQSh6r0lkZbdfU](https://docs.google.com/spreadsheets/d/13PnO1ziA9xUgqEugiZcK97yFO6MtfuQSh6r0lkZbdfU/edit?usp=sharing)

**Hidden sheets are `DEPRECATED` were explicitly excluded from the accepted research dataset.** Earlier Utahraptor readings were found non-reproducible so all measurements were retaken.

### Standardized test setup recorded in the workbook

Used the following commands on every test at each level to assure the most controlled environment possible:

```text
teleportat 471705.781250, 526305.875000, -11915.712891              # TERRESTRIAL ONLY
teleportat 167789.781250, 598364.312500, -15215.770508              # AQUATIC / SEMI-AQUATIC ONLY

settimeofday 12
setdayduration 9999999
setweather 0 0 0 0 0 0 0 0 0 0
pauseweather true
setstats 1 1 1 1 1 0.5 0 0 0 0 0 0 0 0 0     # change the 5th parameter to desired hunger for each trial
setgrowth 10              # change to the desired growth for each trial
```

I ensured the values were correctly applied **before the tick**, and recorded the **first drain instance**. I then reset the growth and hunger values for the **same** test a **second time** and recorded the value only if it matched the first observation. Further limitations ensured no buffs, shrines, events, or weather changes, with comfort held at 50.

These are observations from controlled in-game tests. The exact sequence of internal game calculations, display rounding method, synchronization with the server's tick clock, and effects of every possible environment or trait are not yet established.

### Where the model is not established or demonstrably differs

- **Low-food growth clamp is hypothetical.** Example: using `M=1.88`, `G_eff=1.32`, and `34%` food predicts approximately **0.748** food lost/tick, while the growth-10 Zupay hunger-level test observed **0.6** near 34%. This is larger than the typical one-decimal rounding discrepancy. Treat times to the 15% and 10% milestones as especially provisional approximations.
- **Lower growth behaves differently.** The growth-1 and growth-1.321 food curves do not exhibit the pronounced 35% drain reduction seen at growth 10. At growth 1.321 and 100% food, the `M=1.88` high-growth formula predicts approximately **1.48**, versus **1.1** observed. The model is principally supported at **growth 1.321–10**, and may not apply unchanged to juveniles, adults just below but near 1.32, or very high growth outside the measured range. The lack of a dramatic drain reduction at 35% with 1.321 growth and the nearly identical observed values for that and growth 10 are in support of using 1.32 as the "standard" drain when below 35% hunger.
- **Not every species fits a universal `-1` intercept exactly.** The Apato growth 10 100% food observation was `8.7` resulting in `M=0.97`. Apatosaurus growth-4 was `3.2`, whereas predicting the drain using `M=0.97` it should have been `2.88`.  This is a bit too far from what would be acceptable as rounding errors and is one of the data points that reinforces there is more to the hunger drain than being accounted for.  Other expected species to have issues are Kapro due to it's `Lazy` trait, Rex due to having hunger above 100, and Krono due to having a surprisingly low `M` value calculated.
- **Health damage and death are not modeled.** Health damage was observed to have a sharp curve highly dependent on growth and hunger, as well as potentially many other unaccounted for variables. Due to this, the only `death percentages` noted were for Zupay at a few different growths through letting it starve to death all the way from 10% and simply the percentage it was at on death. Damage started small, then grew as it got worse.

## Preset catalog and status

These are **research estimates or assumed values**, not 28 independently validated implementations of the complete formula. Names mirror the research workbook; they are short labels, not necessarily the game's full taxonomic names. The user may override `M` manually at any time.

| Dinosaur / form | M | Dinosaur / form | M |
| --- | ---: | --- | ---: |
| Apato * | 0.97 | Arch | 1.88 |
| Arco | 1.38 | Bronto | 1.53 |
| Carch | 1.26 | Changy | 1.88 |
| Coah | 1.54 | Elas | 1.79 |
| Icthy | 1.88 | Kapra * | 1.35 |
| Krono | 1.06 | Lurdo | 1.88 |
| Megarat | 1.79 | Mosa | 1.08 |
| Ory | 2.45 | Pachy | 2.32 |
| Palae (aqua) | 1.35 | Palae (semi) | 1.52 |
| Palae (terra) | 1.52 | Para | 2.32 |
| PT | 2.14 | Rex * | 1.06 |
| Sai | 1.76 | Trope | 2.05 |
| Utah | 1.88 | Velo | 1.88 |
| Wieh | 1.58 | Zupay | 1.88 |

`*` = species observed values stray further from predictions than generally acceptable.  Still somewhat close, but considerably less accurate for variables not yet accounted for

## Replication, corrections, and next experiments

**High Priority Followups**
1. Resolve species specific discrepancies for Apato, Kapro, and Rex. This is expected to new variables to account for species specific modifiers as the model here is not representative of the full game system.
2. Spot check all species at various growth and hunger levels to establish a level of variance under the currently established test conditions. Acceptable variance is currently considered `± 0.1` between the calculated and observed drain.
3. Establish a full curve for other *typical* species in addition to the Zupay curve
4. Establish a full curve for all *atypical* species such as kapro

## Attribution and status

*Beasts of Bermuda* is the game studied; this calculator is an **unofficial** community resource and not endorsed by the game's developers. Raw experiments and hypothesis development were provided by me. No game code, internal engine calculations, or developer confirmation was used to derive the calculator or collect any data points. Research and formulas may change with new observations or game updates.

**License:** Unlicense as listed in the repository.
