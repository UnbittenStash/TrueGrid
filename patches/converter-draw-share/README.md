# Converters pay only for their share, and Conversion Loss costs power

Patch `converter-draw-share`, rev 1, for TrueGrid 1.3.5 (upstream commit `459e423`).
Asset `/TrueGrid/Buildables/-Shared/Build_TrueGridMachine_Base`, EventGraph only.

Files:
- [editor.md](editor.md): the change as Unreal Editor steps.
- [diff.txt](diff.txt): pseudo-code of the graph before and after.
- [patch.json](patch.json): the same change in machine-readable form (59 graph operations).

## Problem

Every TrueGrid machine (converter, transformer, substation) sets the power it draws on its input from
what its output circuits consume. Two bugs in that one calculation:

1. **Machines in parallel each pay for the whole output circuit.** The draw is the *whole* output
   circuit's consumption, so two converters feeding one circuit each draw the full load, and upstream
   is asked for double.
   - A substation with several outputs on one grid adds that grid once per output.
   - A converter whose output grid also has its own generators pays for their power too.
2. **Conversion Loss makes power instead of costing it.** The setting's tooltip says "Converting between
   power types is inherently inefficient… 10% power loss". The code computes
   `Conversion % Penalty = 1 − loss/100` (0.9) and **multiplies the input draw by it**, so a converter
   draws 90 % of what it delivers. At 100 %, converters draw nothing at all.

**Measured in game, 1.3.5 unmodified.** A chain of solar (DC) → DC→AC converter → AC circuit → AC→DC
converter → a 20 MW Smelter Mk3; draws read from each circuit's power screen.

| Setup | Smelter circuit | AC circuit | Solar circuit |
|---|---|---|---|
| One converter per step, Conversion Loss off | 20 MW | 20 MW | 20 MW |
| One converter per step, 10 % | 20 MW | 18 MW | 16.2 MW |
| One converter per step, 100 % | 20 MW | 0 MW | 0 MW |
| Two DC→AC converters in parallel, 10 % | 20 MW | 18 MW | **32.4 MW** (2 × 16.2) |
| Two DC→AC converters in parallel, off | 20 MW | 20 MW | **40 MW** |

In some setups the two bugs partly cancel, so they need fixing **together**. The conversion fix alone
would raise the parallel 10 % case from 32.4 to 48.4 MW.

## Original code (1.3.5)

`Update TrueGrid Machine` (every 2.3 s; comment boxes "Combined Production" and "Extra Power Production
from difficulty mods"):

```
Combined Production = 0
for each output power info:
    Combined Production += circuit.GetConsumedPoints(stats)[last]          ← whole circuit
Combined Production = max(floor, Combined × Conversion % Penalty           ← 0.9: a gain
                                       × (2 − Distance % Penalty) × Alien Power Boost)
```

`Factory_ReceiveTick` → `Update Power Stats` sets the input's target consumption to `Combined Production`
(÷ inputs on a substation).

With Phase Shift on and an inductor on the output circuit, the draw comes from `Inductor Power` instead.
The `Smoothing Timer` (every 0.1 s) rebuilds `Combined Production` with the same whole-circuit sum, without
multipliers, and eases `Inductor Power` towards it.

**Probably written this way because:**
- **Whole-circuit sum:** with one converter per circuit, "what the circuit consumes" *is* "what this
  converter delivers". Parallel converters and shared grids break that, and the fixed per-output cap
  makes those layouts common.
- **Conversion:** the comment on the block says "Extra Power *Production*". Multiplying *output* by an
  efficiency is right, but this block computes the *input* from the output, where the efficiency has to
  divide (or use `1 + loss`). Distance loss in the same block uses `2 − penalty` and points the right
  way. The scanner shows `1 − penalty` as "loss", so a loss was always the intent.

## Fix

**C1 · share, main loop.** In the `Combined Production` loop, each output's consumed value is multiplied by
`clamp(Final Production Value ÷ circuit produced, 0, 1)` before it's added:
- `Final Production Value` is the base production this machine sets on that output;
- "circuit produced" is `GetProducedPoints(stats)[last]`, from the same stats sample as the consumed value.

Base production counts as produced in full even when unused (an AC circuit fed by two converters showed
900 MW produced under a 20 MW load). So the ratio is this machine's part of everything feeding the circuit:
- **Two equal converters:** each pays half.
- **Several substation outputs on one grid:** each output pays its share, and the shares add up.
- **A grid with its own generators:** the generators throttle down while base production covers the
  load, and the converter pays for the part it supplies.
- **The clamp:** keeps a machine from paying for more than the whole circuit while the stats lag one
  sample behind.

**C2 · conversion.** The draw multiplier takes `2 − Conversion % Penalty` (1.1 at 10 %) instead of the
penalty. The subsystem variable keeps its meaning (an efficiency), so the scanner's "loss" display stays
correct. `2 − P` is slightly milder than an exact loss at high settings (1.5× at 50 % instead of 2×) and
stays bounded at 100 %.

**C3 · share, Smoothing Timer.** The same share as C1 in the inductor smoothing loop.

**Nodes:** 15 new (7 per share, 1 for conversion) and 2 comment boxes; nothing removed. Three links are
replaced: the two `Add` inputs (C1, C3) and the multiplier's `B` (C2). New nodes sit to the right of the
Update block (C1), below the smoothing box (C3), and in the "Extra Power Production" box (C2).

## Risks and side effects

- **Saves with Conversion Loss on draw more upstream than before:** single chains go from 0.9× to 1.1× per
  stage (+22 %). Generators sized for the old behaviour can come up short, and the vanilla fuse on the
  upstream circuit then blows. Parallel setups draw *less* than before.
- **Machines with inductors still skip the multipliers.** The smoothing path never applied conversion,
  distance or augmenter multipliers; C3 fixes its double draw but not that.
- **Augmenter (unchanged, open question).** `Alien Power Boost` still multiplies the draw. One producing
  augmenter anywhere in the world makes every TrueGrid machine draw 1.4× what it delivers, which looks
  inverted.
- **Timing.** The draw follows the circuit stats, which lag; expect a few seconds of settling after
  building, loading or rewiring.
- **Saves and multiplayer:** no save-format change (runtime math only); computed on the server.

## Testing

Same chain as above, patched build, about 5–10 s per reading:

| Setup | Expected | Measured |
|---|---|---|
| One converter per step, Conversion Loss off | 20 / 20 / 20 MW | 20 / 20 / 20 MW |
| Two DC→AC converters in parallel, off | solar 20 MW (was 40) | 20 / 20 / 20 MW |
| Two DC→AC converters in parallel, 10 % | 20 / 22 / 24.2 MW | 20 / 22 / 24.2 MW |

- **Mixed grid (120 MW of biomass burners added to the AC circuit):** the burners idle while the converters' base
  production covers the load; the converters keep paying for all of it, which is correct since they
  supply it. Pass.
- **Logs:** no `Accessed None` and no new warnings from `Build_TrueGridMachine_Base`.
- **Not tested in game: the inductor path (C3).** An inductor alone on an LV circuit drives phase shift to
  0 % (Phase Shift Percent 500 × its ~10 MW draw = 5,000, against a 500 MW converter), so the converters
  stop producing and the circuit's vanilla fuse trips, with or without this patch. Testing C3 needs
  inductors balanced with capacitors. C3 uses the same nodes as C1, and the Blueprint compiled without
  errors or warnings.
- **Not tested: a substation with two outputs on one grid.**

## Related, not changed here

- **The augmenter multiplier on the draw** (above).
- **Converters advertise all of upstream's production as their own capacity.** Two converters on a 450 MW
  circuit show 900 MW downstream. And a converter passes on only what upstream is *producing*, so idle
  generators upstream don't count until they run.
- **Max consumption doesn't pass through converters:** upstream circuits show 0 MW max consumption.
- **Phase Shift Percent is multiplied raw rather than as a percent,** so one inductor zeroes any LV or MV
  converter.

## History

- **rev 1 (2026-11-08):** created after both bugs were measured in game; tested in game as above.
