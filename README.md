# Government Construction Cap

A Victoria 3 (1.15) mod that adds a setting capping how much of your construction
capacity **government** construction may use, without touching the private sector's share.

It implements this request on r/victoria3:
[A slider for maximum used construction](https://old.reddit.com/r/victoria3/comments/1xzeria/a_slider_for_maximum_used_construction/).

> Say you set the slider at 50%, you would not go above 50% capacity even if private
> construction is completely stopped. This would naturally not override the private
> construction share, so if private construction share is 75%, you would be capped at
> 25% (when private sector is fully queued), unless you set the slider at, say 10%,
> which would then be your self-imposed cap.

## Usage

The **Government cap** control appears in two places: the **Budget** panel under **Goods
for Construction** (below Pause/Resume All Construction), and **Construction > National
Queue** (below the construction sector row with the pause button):

```
Government cap: 60%   (−) [■■■■■■■■■■■■□□□□□□□□] (+)
```

Click a point on the bar to set the cap (20 steps of 5%), or use `−`/`+` to change it by
5%. At 100% (the default, shown as "Off") the mod does nothing. Hovering the control shows
the live numbers: the government limit, how much construction the government and the
private sector are using, and how much is held back.

Vic3's engine has draggable sliders, but their value can only be sent to built-in code,
not to mod script, so the bar is made of clickable segments instead.

## How it works

Script cannot pause or limit the government queue directly. It can, however, change the
amount of construction produced and the private construction allocation. Once a week
(and the day after you change the cap) the mod computes:

| Symbol | Meaning |
| --- | --- |
| `C` | nominal construction (what you would produce without the mod) |
| `a` | nominal private construction allocation (from laws etc.) |
| `X` | the cap |
| `D` | estimated private demand = queued private projects × Max Weekly Construction Progress |

```
government limit   G  = X · C
government floor   F  = min(G, (1 − a) · C)    # what vanilla guarantees the government, capped
private reserve    P  = min(D, C − F)
active capacity    C' = min(C, G + P)          # the rest (C − C') is held back
private allocation a' = clamp(P / C', a, 1)
```

Two static modifiers apply this:

- `lcs_construction_throttle`: −100% Construction Sector throughput, applied with a
  multiplier that brings total construction down to `C'`. As far as observed in-game, a
  sector's goods cost is its full input (set by workforce and throughput) times the share of
  its produced construction that is actually used: nothing built, no goods bought. A flat
  −construction modifier shrinks output but not inputs, so the cost per point rises and
  spending doesn't fall (an earlier version of this mod made that mistake). Throughput
  scales inputs and output together, so the cost per point stays the same and spending
  falls with output. Construction sector wages are still paid.
- `lcs_private_construction_reserved`: +100% private allocation, applied with multiplier
  `a' − a`. Private projects get their reserve first, so the government is left with at
  most `C' − P ≤ G`.

The throttle is computed from the current state each week. Sector output is assumed
proportional to the throughput factor `1 + T`, where `T` is the country's construction
sector throughput bonus:

```
non-sector construction  N = GDP-based country_gdp_construction (≤ 10)
nominal construction     C = N + (current − N) · (1 + T) / (1 + T − throttle)
new throttle               = (1 + T) − (1 + T − throttle) · (C' − N) / (current − N)
```

If construction is already on target, the throttle stays put. So throughput the mod can't
see (per-building or state bonuses) gets corrected over the next weeks. The throttle never
takes sector output below 5% of nominal (`lcs_min_throughput_fraction`).

You keep your sectors and can lift the cap at any time.

### Limitations

- Throttling relies on Construction Sector throughput scaling construction output linearly,
  which is what vanilla techs giving sector throughput rely on. If the engine clamps
  throughput at some minimum, very low caps won't be reached.
- Lower throughput may also lower the small State Construction Efficiency bonus that
  Construction Sectors give their own state.

- `D` is an estimate. Script cannot read actual private usage (only the UI can), so the
  mod assumes every queued private project can absorb Max Weekly Construction Progress
  per week. If private construction looks throttled, raise
  `lcs_private_demand_per_project_factor` in `common/script_values/lcs_script_values.txt`.
  If the government overshoots the cap, lower it.
- When the private sector uses less than its reserve, the government gets the difference
  until the next weekly update, so it can briefly go over the cap.
- The total construction shown in the UI is the reduced (active) amount; the cap
  control's tooltip shows the nominal amount.
- Only player countries can set a cap. When a country stops being player controlled, its
  modifiers are removed at the next weekly or monthly update.
- **Compatibility:** the control is added by overriding two vanilla files, each changed
  only by a short hook marked `# limit_cons_share`:
  - `gui/budget_panel.gui`: `lcs_construction_cap_control = {}` at the end of the
    `second_line` block of the Goods for Construction row.
  - `gui/construction_panel.gui`: `lcs_construction_cap_control = { parentanchor = hcenter }`
    after the `construction_sector_or_no_sector` flowcontainer of the domestic queue tab.

  The control itself lives in `gui/lcs_construction_cap.gui`. Other mods that override
  either file conflict with this one. After a game patch, copy the new vanilla files and
  re-add the hooks.

### Files

| File | Purpose |
| --- | --- |
| `gui/budget_panel.gui` | vanilla copy + one-line hook |
| `gui/construction_panel.gui` | vanilla copy + hook in the National Queue tab |
| `gui/lcs_construction_cap.gui` | the cap control (label, −/+ buttons, clickable bar) |
| `common/scripted_guis/lcs_scripted_guis.txt` | set / lower / raise the cap from the UI |
| `common/scripted_effects/lcs_scripted_effects.txt` | the cap calculation |
| `common/script_values/lcs_script_values.txt` | formula terms, tuning factor |
| `common/static_modifiers/lcs_static_modifiers.txt` | the two modifiers applied |
| `events/lcs_events.txt` | `lcs.1` recalculates the day after a change, `lcs.2` weekly |
| `common/on_actions/lcs_on_actions.txt` | monthly safety net for the weekly chain |

## Development

This repository follows the
[Victoria 3 Modding Co-op mod template](https://github.com/Victoria-3-Modding-Co-op/Mod-Template).
The mod itself lives in [mod](mod); link that folder into the game's mod folder:

```
ln -s "$PWD/mod" "$HOME/.local/share/Paradox Interactive/Victoria 3/mod/limit_cons_share"
```

Mod prefix: `lcs`.

### Folder structure

- [mod](mod): the mod itself
- [documentation](documentation): documentation that is not shipped with the mod; start with [DEVELOPMENT.md](documentation/DEVELOPMENT.md)
- [script](script): helper scripts; `generate-localization.sh` copies the English
  localization to the other languages listed in `script/languages.txt`
