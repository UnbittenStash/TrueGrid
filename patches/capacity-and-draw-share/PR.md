# Machines offer their input's capacity and pay only for their share; Conversion Loss costs power

Patch `capacity-and-draw-share`, rev 1, for TrueGrid 1.3.5 (upstream commit `459e423`).
Asset `/TrueGrid/Buildables/-Shared/Build_TrueGridMachine_Base`, EventGraph only.

Files:
- [editor.md](https://github.com/UnbittenStash/TrueGrid/blob/main/patches/capacity-and-draw-share/editor.md): the change as Unreal Editor steps.
- [diff.txt](https://github.com/UnbittenStash/TrueGrid/blob/main/patches/capacity-and-draw-share/diff.txt): pseudo-code of the graph before and after.
- [patch.json](https://github.com/UnbittenStash/TrueGrid/blob/main/patches/capacity-and-draw-share/patch.json): the same change in machine-readable form (75 graph operations).

All three fixes are in the same Blueprint, so they come as one change.

## Problem

Every TrueGrid machine (converter, transformer, substation) runs the same update: it reads its input
circuit(s), offers power on its output(s) as base production, and sets its input draw from what its
output circuits consume. 1.3.5 changed this update, and three things in it go wrong.

1. **Machines only offer what upstream is producing right now (new in 1.3.5).**
   - `Upstream Production Capacity` now reads the input circuit's last *produced* point instead of its
     production *capacity* (1.3.4 used `GetPowerProductionCapacity`).
   - Fuel, coal, biomass and nuclear generators produce only what is being consumed. A machine fed by
     them therefore offers only the current load.
   - When demand downstream rises, it exceeds that offer and the downstream circuit's fuse blows. With
     the fuse blown the machine draws nothing, upstream production drops further, and resetting the fuse
     repeats the cycle.
   - After loading a save every machine first offers its full rating, then drops to whatever upstream
     happened to produce, so many circuits trip at once.
   - Solar-style producers are not affected: their output counts as produced whether it's used or not.
   - Reported in #50: an idle biomass grid showing 0.5 MW, and an AC→DC converter on it reporting
     0.5 MW available instead of 40 MW.
2. **Machines in parallel each pay for the whole output circuit (new in 1.3.5).**
   - The draw is the sum of each output circuit's *whole* consumption. 1.3.4 summed each output's own
     regulated production.
   - Two converters feeding one circuit each draw the full load, so upstream is asked for double.
   - A substation with several outputs on one grid adds that grid once per output: the SV→MV substation
     has six. This fits #48's "MV substations adding a lot of consumption to their SV side".
3. **Conversion Loss makes power instead of costing it (also in 1.3.4).**
   - The setting's tooltip describes a power loss. The code computes `Conversion % Penalty = 1 − loss/100`
     (0.9 at 10 %) and **multiplies the input draw by it**, so a converter draws 90 % of what it delivers.
   - At 100 %, converters draw nothing at all.

**Measured in game, 1.3.5 unmodified.** A chain of solar (DC) → DC→AC converter → AC circuit → AC→DC
converter → a 20 MW Smelter Mk3; draws read from each circuit's power screen.

| Setup | Smelter circuit | AC circuit | Solar circuit |
|---|---|---|---|
| One converter per step, Conversion Loss off | 20 MW | 20 MW | 20 MW |
| One converter per step, 10 % | 20 MW | 18 MW | 16.2 MW |
| One converter per step, 100 % | 20 MW | 0 MW | 0 MW |
| Two DC→AC converters in parallel, 10 % | 20 MW | 18 MW | **32.4 MW** (2 × 16.2) |
| Two DC→AC converters in parallel, off | 20 MW | 20 MW | **40 MW** |

Fixes 2 and 3 partly cancel in some setups, so they need to go in **together**. The conversion fix
alone would raise the parallel 10 % case from 32.4 to 48.4 MW.

## Original code (1.3.5)

`Update TrueGrid Machine`:

```
Upstream Production Capacity = 0
for each input power info whose circuit is active:
    Upstream Production Capacity += circuit.GetProducedPoints(stats)[last]   ← current production
for each output power info:
    Final Production Value = clamp(Upstream Production Capacity, 0, Max ÷ outputs × penalties) + hoverpacks
    SetBaseProduction(Final Production Value)

Combined Production = 0
for each output power info:
    Combined Production += circuit.GetConsumedPoints(stats)[last]            ← whole circuit
Combined Production = max(floor, Combined × Conversion % Penalty              ← 0.9: a gain
                                       × (2 − Distance % Penalty) × Alien Power Boost)
```

- `Factory_ReceiveTick` → `Update Power Stats` sets the input's target consumption to
  `Combined Production` (÷ inputs on a substation).
- With Phase Shift on and an inductor on the output circuit, the draw comes from `Inductor Power`
  instead. The `Smoothing Timer` rebuilds `Combined Production` with the same whole-circuit sum, without
  multipliers.

**Probably written this way because:**
- **Upstream read:** the switch from dynamic to base production came with reading circuit stats. The
  variable is still named *Capacity*, and the release notes don't mention a change to what is read.
- **Whole-circuit sum:** with one converter per circuit, "what the circuit consumes" *is* "what this
  machine delivers". Parallel machines and shared grids break that, and the fixed per-output cap makes
  those layouts common.
- **Conversion:** the comment on the block says "Extra Power *Production*". Multiplying *output* by an
  efficiency is right, but this block computes the *input* from the output, where the efficiency has to
  divide (or use `1 + loss`). Distance loss in the same block uses `2 − penalty` and points the right
  way.

## Fix

**U · upstream capacity.**
- `Upstream Production Capacity` adds each active input circuit's `GetPowerProductionCapacity()`, as in
  1.3.4.
- The four nodes that read the produced point are removed.
- A machine now offers what its input *can* deliver (still capped by its rating and penalties).
  Generators upstream ramp up as the machine's draw rises.

**C1 · share, main loop.**
- In the `Combined Production` loop, each output's consumed value is multiplied by
  `clamp(Final Production Value ÷ circuit produced, 0, 1)` before it's added.
- `Final Production Value` is the base production this machine sets on that output; "circuit produced"
  is `GetProducedPoints(stats)[last]` from the same stats sample.
- Base production counts as produced in full even when unused, so the ratio is this machine's part of
  everything feeding the circuit:
  - **Two equal converters:** each pays half.
  - **Several substation outputs on one grid:** each output pays its share, and the shares add up to
    the grid's consumption.
  - **A grid with its own generators:** the generators throttle down while base production covers the
    load, and the machine pays for the part it supplies.
  - **The clamp:** keeps a machine from paying for more than the whole circuit while the stats lag one
    sample behind.

**C2 · conversion.**
- The draw multiplier takes `2 − Conversion % Penalty` (1.1 at 10 %) instead of the penalty.
- The subsystem variable keeps its meaning (an efficiency), so the scanner's "loss" display stays correct.
- `2 − P` is slightly milder than an exact loss at high settings (1.5× at 50 % instead of 2×) and stays
  bounded at 100 %.

**C3 · share, Smoothing Timer.** The same share as C1 in the inductor smoothing loop.

**Nodes:** 16 new and 3 comment boxes; 4 removed. Four links are replaced:
- the upstream sum's `B` (U);
- the two draw sums' `B` (C1, C3);
- the multiplier's `B` (C2).

## Side effects: what this change causes, and what was already there

Legend: **new** = caused by this change; **larger** = already there, but this change makes it bigger or more
common; **unchanged** = already there, and this change neither causes nor fixes it.

| Effect | 1.3.4 | 1.3.5 unmodified | With this change |
|---|---|---|---|
| Machines offer their input's full capacity | yes | no (current production) | **new vs 1.3.5** (back to 1.3.4) |
| More draw upstream with Conversion Loss on (0.9× → 1.1× per stage) | no (0.9×) | no (0.9×) | **new** |
| A machine supplies a grid that has its own generators before those generators, and can overload its input | no (load shared with local generators) | only when the input's producers count in full, e.g. the modded solar panels | **larger** (now also behind fuel generators) |
| Parallel machines each advertise the input's full capacity | yes | yes (barely visible behind fuel generators, where offers were tiny) | **unchanged** (visible again, as in 1.3.4) |
| Batteries on a machine-fed circuit charge from unused offer, paid or free | no (full blast, paid) | yes (full blast removed) | **larger** (larger offer) |
| Loops through the same grid build up offer and draw | yes | yes | **unchanged** |
| Augmenter multiplies the draw by 1.1×/1.4× | yes | yes | **unchanged**, but see below |
| Inductor smoothing skips the conversion, distance and augmenter multipliers | yes | yes | **unchanged** |
| Stats read from every output, wired or not | no | yes | **larger** (one more read per output) |
| Machines supply a jump in demand at once and pay a few seconds later | yes (draw follows last update) | yes | **unchanged** |

**Details:**

- **Offers back to 1.3.4 levels.** A machine offers what its input *can* deliver, capped by its rating and
  penalties, instead of what the input delivers right now. This is the fix for #50; every row marked
  *larger* follows from the bigger offer.
- **Conversion Loss (new).** Single chains go from 0.9× to 1.1× of the delivered power per stage (+22 %),
  so generators sized for the old behaviour can come up short. Parallel setups draw *less* than before
  (C1). Only saves with Conversion Loss on are affected.
- **Mixed grids (larger).**
  - The game uses base production before fuel generators. So on a grid that also has its own generators,
    the TrueGrid machine supplies the load first, up to its whole offer, and the local generators idle.
  - The machine then draws that load from its input. If the input's other consumers already use most of
    its capacity, the input is overloaded and its fuse blows.
  - Example: a converter takes from a 300 MW coal grid that runs a 250 MW factory, and feeds a grid with
    its own generators and a 100 MW load. The converter supplies all 100 MW, the coal grid is asked for
    350 MW and trips, and the downstream generators sit idle.
  - Needs all three of: generators on the fed grid, little spare capacity on the input grid, and a load
    the machine can cover. A plain chain (generators → transformer → substation → loads) can't hit it.
  - Unmodified 1.3.5 already behaves this way when the input's producers count in full (the modded solar
    panels). Behind fuel generators it instead fails earlier, by offering almost nothing (#50).
- **Parallel machines (unchanged).**
  - Two converters taking from one 40 MW grid and feeding the same grid each offer 40 MW there, so its
    power screen shows 80 MW.
  - Their draws are correct (C1). But a load above 40 MW is accepted, and it's the *input* grid that trips.
  - Measured in unmodified 1.3.5: 450 MW of solar showed as 900 MW on the AC side.
- **Batteries (larger).**
  - 1.3.4 set machine outputs to full blast when batteries were present, and the machine paid for the
    charging. 1.3.5 removed that.
  - Batteries now charge from unused base production. If charging counts as consumption in the circuit's
    stats, the machine pays for it, and its draw can rise to its whole offer while they fill, which can
    overload its input. If not, the charge is free.
  - The bigger offer makes either case larger. Untested (T7).
- **Loops (unchanged).**
  - Examples: generators → step-up transformer → substation → back into the generators' own grid; or two
    converters running in opposite directions between the same two grids.
  - Each machine reads the other's offer as capacity, and its own output feeds its own input, so offer and
    draw build up each update until something trips. No draw formula can fix this; avoid loops.
- **Augmenter (unchanged, possible interaction).**
  - `Alien Power Boost` multiplies the draw (1.1× with one built, 1.4× while one is producing). It also
    sets the outputs' production boost factor.
  - If that factor raises base production, the share (C1) cancels the draw multiplier, so this change
    would *remove* the extra draw.
  - If it doesn't, every machine draws 1.1×–1.4× what it delivers, as before. Untested.
- **Unwired substation outputs (larger).** The 1.3.5 draw loop already reads circuit stats from every
  output; this change adds a produced-points read. If an unwired output has no circuit, this can log
  warnings (`Accessed None`, or an index −1 read); the value counts as 0. Untested (T8).
- **Saves and multiplayer.** No save-format change (runtime math only); computed on the server. A save
  made with this change loads in unmodified TrueGrid.

## Testing

Measured on a build with C1–C3 only (before U was added), about 5–10 s per reading, same chain as above:

| Setup | Expected | Measured |
|---|---|---|
| Two DC→AC converters in parallel, off | solar 20 MW (was 40) | 20 / 20 / 20 MW |
| Two DC→AC converters in parallel, 10 % | 20 / 22 / 24.2 MW | 20 / 22 / 24.2 MW |

- **Mixed grid (120 MW of biomass burners added to the AC circuit):** the burners idle while the
  converters' base production covers the load; the converters keep paying for all of it, which is
  correct since they supply it. Pass.
- **Logs:** no `Accessed None` and no new warnings from `Build_TrueGridMachine_Base`.

Not yet tested in game:

| # | Setup | Expected with this change |
|---|---|---|
| T1 | One converter per step, Conversion Loss off | 20 / 20 / 20 MW |
| T2 | 2 biomass burners → AC→DC converter → 20 MW load (#50) | runs; converter offers 40 MW; burners ≈ 20 MW; no trip (unmodified: offer ≈ 0.5 MW, fuse blows) |
| T3 | Fuel generators → step-up transformer → step-down substation → load, raised in steps | each step runs up to the generators' capacity; no trips |
| T4 | Save and reload with T2 and T3 running | no trips after loading |
| T5 | SV→MV substation, 3 outputs into one MV grid, 60 MW load | SV side ≈ 60 MW (unmodified ≈ 180 MW) |
| T6 | Mixed grid: converter from a busy coal grid into a grid with its own generators (see Side effects) | the converter takes the load first; the coal grid trips once its total demand exceeds its capacity |
| T7 | Converter-fed circuit with one empty battery and a small load | whether the converter's draw rises while the battery charges (and whether its input then trips) |
| T8 | Substation with unwired outputs | check the log for `Accessed None` / index warnings from `Build_TrueGridMachine_Base` |
| T9 | Inductor path (C3) | needs inductors balanced with capacitors: one inductor alone drives phase shift to 0 % and trips LV converters with or without this change |

## Related, not changed here

All of these were already in 1.3.4 and are unchanged in 1.3.5 and by this change:
- **The augmenter multiplier on the draw** (above).
- **Parallel machines advertise the input's capacity once each** (above).
- **Phase Shift Percent is multiplied raw rather than as a percent,** so one inductor zeroes any LV or MV
  converter.
- **Substation phase shift uses only the last valid output.**
- **The HV→QV substation adds each input's reading, even when several inputs share one circuit,** so its
  offer can exceed what that circuit has.

Seen in 1.3.5, not checked in 1.3.4:
- **Max consumption doesn't pass through converters:** upstream circuits show 0 MW max consumption.

## History

- **rev 1 (2026-11-08):** replaces the earlier `converter-draw-share` patch (C1–C3) and adds U. C1–C3 are
  unchanged and were tested in game as above; U and the combination are untested in game.

---

Review aids: [diff.txt](https://github.com/UnbittenStash/TrueGrid/blob/main/patches/capacity-and-draw-share/diff.txt) (pseudo-code diff of the Blueprint graph); [editor.md](https://github.com/UnbittenStash/TrueGrid/blob/main/patches/capacity-and-draw-share/editor.md) (the same change as Unreal Editor steps).
