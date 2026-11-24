# Development guide

This explains how the mod is put together, written for someone who has never made a
Victoria 3 mod. For what the mod does and the maths behind it, see the [README](../README.md).

## 1. What a Victoria 3 mod is

A mod is a folder of plain text files (plus images) that the game loads **on top of** its
own files. The game's files live in `.../steamapps/common/Victoria 3/game/`. The mod mirrors
that folder layout:

- A mod file at a **new path** adds something. For example
  `common/scripted_effects/lcs_scripted_effects.txt` adds new effects next to the game's own.
- A mod file at the **same path** as a game file **replaces** it completely. This mod does
  that for two interface files (`gui/budget_panel.gui`, `gui/construction_panel.gui`).

There is no compiling. You edit text, restart the game (or reload), and it reads the files.
The launcher finds the mod through `mod/.metadata/metadata.json`.

## 2. Paradox script in five minutes

Almost every `.txt` file uses the same `key = value` format, with `{ }` for blocks and `#`
for comments:

```
lcs_cap_percent = {          # a named definition
    value = 100              # start at 100
    if = {
        limit = { has_variable = lcs_cap }   # condition
        value = var:lcs_cap                  # use the stored variable instead
    }
}
```

The building blocks used in this mod:

| Term | What it is | Example here |
| --- | --- | --- |
| **Scope** | "Who" a piece of script runs on: a country, a state, a building… Everything in this mod runs on the player's country. | `scope = country` |
| **Trigger** | A yes/no condition. | `is_player = yes`, `lcs_cap_percent > 0` |
| **Effect** | Something that changes the game. | `set_variable`, `add_modifier`, `trigger_event` |
| **Script value** | A named formula that computes a number. | `lcs_target_construction` |
| **Scripted effect** | A named, reusable list of effects (like a function). | `lcs_update_construction_cap` |
| **Variable** | A number the mod stores on the country between days. | `var:lcs_cap` (the cap you set) |
| **Modifier** | A bundle of bonuses/penalties attached to the country, like the ones you see in tooltips. | `lcs_construction_throttle` |
| **Event** | Normally a popup; a `hidden = yes` event runs script silently, which is how the mod schedules work for "tomorrow" or "in 7 days". | `lcs.1`, `lcs.2` |
| **On action** | A hook the game calls at fixed times (game start, every month…). | `on_monthly_pulse_country` |
| **Scripted GUI** | Script that interface buttons can call, so clicks can run effects. | `lcs_set_cap_sgui` |
| **Localization** | The text shown to the player, looked up by key. | `LCS_CAP_LABEL` |

Interface files (`.gui`) use the same `{ }` syntax but describe widgets: boxes, text,
buttons. Text inside `"[ ... ]"` is a **data function**: a live call into the game, such as
`[GetPlayer.CalcUsedGovernmentConstruction]`.

## 3. Repository layout

The structure follows the
[Victoria 3 Modding Co-op mod template](https://github.com/Victoria-3-Modding-Co-op/Mod-Template).
Only `mod/` is the actual mod. Everything else stays in the repository and is not uploaded
to the Workshop. (`image/templates/` is collapsed below; it only holds template images.)

```
limit_cons_share/
├── README.md                     what the mod does, the formula, limitations
├── .gitattributes                disables line-ending conversion (see section 7)
├── .gitignore                    from the template
├── documentation/
│   └── DEVELOPMENT.md            this file
├── image/                        from the template: GIMP/PNG templates for icons (unused)
│   ├── buildings.xcf
│   ├── production_methods.xcf
│   └── templates/…
├── script/                       from the template
│   ├── generate-localization.sh  copies English text files to the other languages
│   └── languages.txt
└── mod/                          ← the mod itself (this folder is linked into the game)
    ├── .metadata/
    │   ├── metadata.json         name, id, version, supported game version
    │   └── thumbnail.png         launcher picture (template placeholder)
    ├── thumbnail.png             Steam Workshop picture (template placeholder)
    ├── common/
    │   ├── on_actions/
    │   │   └── lcs_on_actions.txt          monthly safety net
    │   ├── scripted_effects/
    │   │   └── lcs_scripted_effects.txt    the update logic
    │   ├── scripted_guis/
    │   │   └── lcs_scripted_guis.txt       what the buttons do
    │   ├── script_values/
    │   │   └── lcs_script_values.txt       all the formulas
    │   └── static_modifiers/
    │       └── lcs_static_modifiers.txt    the two modifiers applied to the country
    ├── events/
    │   └── lcs_events.txt                  hidden events: "tomorrow" and "every week"
    ├── gfx/                                (empty; no custom images)
    ├── gui/
    │   ├── budget_panel.gui                vanilla copy + 2-line hook
    │   ├── construction_panel.gui          vanilla copy + 4-line hook
    │   └── lcs_construction_cap.gui        the cap control itself
    └── localization/
        └── english/
            ├── lcs_l_english.yml           all player-facing text
            └── replace/                    (empty; for overriding vanilla text)
```

## 4. What each file does

### `mod/.metadata/metadata.json`
Tells the launcher the mod's name, id (`com.github.lucasfa.limit_cons_share`), version and
supported game version (`1.15.*`). `multiplayer_synchronized` means every player in a
multiplayer game must have it.

### `mod/gui/lcs_construction_cap.gui` (new)
Defines one reusable widget, `lcs_construction_cap_control`:

```
Government cap: 60%   (−) [■■■■■■■■■■■■□□□□□□□□] (+)
```

- the label, whose text comes from localization;
- a `−` and a `+` button;
- a progress bar showing the cap, with 20 invisible buttons laid over it (5%, 10% … 100%).

Each button calls a scripted GUI. The bar is made of segments because the game's draggable
sliders can only report their position to built-in code, not to mod script.

### `mod/gui/budget_panel.gui` and `mod/gui/construction_panel.gui` (vanilla copies)
Full copies of the game's files, needed because a mod can't insert into a game file, only
replace it. Each differs from vanilla by a hook marked `# limit_cons_share` that places the
widget:

- Budget panel: under Goods for Construction, below Pause/Resume All Construction.
- Construction panel: in the National Queue tab, below the construction sector row.

**After a game patch**, these copies are outdated. Copy the new vanilla files and re-add
the hooks; search for `limit_cons_share` in the old copies to find them. Any other mod
that replaces these files conflicts with this one.

### `mod/common/scripted_guis/lcs_scripted_guis.txt` (new)
What the buttons do:

- `lcs_set_cap_sgui`: sets the cap to the value the bar segment passes in.
- `lcs_lower_cap_sgui` / `lcs_raise_cap_sgui`: −5% / +5%. Disabled at 0% / 100%.

Each stores the cap in the variable `lcs_cap` and calls `lcs_on_cap_changed`.

### `mod/common/scripted_effects/lcs_scripted_effects.txt` (new)
The logic, as named effects:

| Effect | Does |
| --- | --- |
| `lcs_on_cap_changed` | Schedules a recalculation for tomorrow (event `lcs.1`) and makes sure the weekly one is running. |
| `lcs_ensure_weekly_update` | Starts the weekly event `lcs.2` if it isn't already scheduled. |
| `lcs_change_cap` | Adds/subtracts from the cap, clamped to 0–100. |
| `lcs_update_construction_cap` | Runs the calculation, at most once per day. |
| `lcs_update_construction_cap_now` | The calculation itself: snapshots the current numbers into variables, computes the target, then replaces the two modifiers. |
| `lcs_remove_construction_cap_modifiers` / `lcs_clear_construction_cap` | Clean-up when the cap is off or the country is no longer player-controlled. |

### `mod/common/script_values/lcs_script_values.txt` (new)
Every formula, each a small named step so it can be read and tuned:

- **Reading the current state:** `lcs_cap_percent`, `lcs_non_sector_construction`,
  `lcs_current_sector_construction`, throughput factors, `lcs_nominal_construction` (what
  you'd produce without the mod).
- **Estimating private demand:** `lcs_private_demand_estimate` = queued private projects ×
  Max Weekly Construction Progress × `lcs_private_demand_per_project_factor` (the tuning
  knob, default 1).
- **The target:** `lcs_government_limit`, `lcs_government_floor`,
  `lcs_private_reservation`, `lcs_target_construction`, `lcs_target_private_allocation`.
- **The throttle:** `lcs_target_throttle`, how much construction sector throughput to remove.
- **Display:** `lcs_display_government_limit`, used in the tooltip.

### `mod/common/static_modifiers/lcs_static_modifiers.txt` (new)
The two modifiers. Each is defined as a "unit" and applied with a multiplier:

- `lcs_construction_throttle`: −100% construction sector throughput per unit. Lowers both
  the construction produced and the goods the sectors buy.
- `lcs_private_construction_reserved`: +100% private construction allocation per unit, so
  private projects keep their share.

### `mod/events/lcs_events.txt` (new)
- `lcs.1`: hidden, fires the day after you change the cap and recalculates.
- `lcs.2`: hidden, fires every 7 days and recalculates, then schedules itself again while a
  cap is set.
- `lcs_error_suppression.0001`: from the template; never fires, it just keeps the
  `lcs_is_loaded` variable from being reported as unused.

### `mod/common/on_actions/lcs_on_actions.txt` (new, based on the template)
- At game start: sets `lcs_is_loaded` (from the template).
- Every month: restarts the weekly event if it has stopped, and removes the modifiers from
  countries that are no longer player-controlled.

### `mod/localization/english/lcs_l_english.yml` (new)
All text: the label, the big tooltip with live numbers, button tooltips and modifier names.
The file name must end in `_l_english.yml`, start with `l_english:`, and be saved as UTF-8
**with BOM**, or the game ignores it.

## 5. How it fits together

```
 You click the bar / − / +
        │
        ▼
 scripted GUI (lcs_scripted_guis.txt)
        │  stores the cap in var:lcs_cap
        ▼
 lcs_on_cap_changed ──► event lcs.1 tomorrow ──┐
        │                                      │
        └──► weekly event lcs.2 (every 7 days) ┤
                                               ▼
                          lcs_update_construction_cap_now
                          1. read current construction, throughput, private allocation
                          2. compute target (script values)
                          3. replace the two modifiers with new multipliers
                                               │
                                               ▼
              game applies the modifiers: sectors produce less and buy fewer goods,
              private projects get their share first, the government gets the rest (≤ cap)

 Monthly on_action: safety net (restart the weekly chain, clean up AI countries)
```

Why recalculate **tomorrow** and not immediately: after a modifier changes, the game only
updates the country's totals at the next daily tick. Reading them the same day would give
stale numbers, so the mod also refuses to run twice on one day.

## 6. Conventions

- **Prefix `lcs`** (limit_cons_share) on every file and name, so nothing clashes with the
  game or other mods.
- **UTF-8 with BOM** for script and localization files (the files start with the bytes
  `EF BB BF`).
- Script files are indented with tabs, like vanilla.

## 7. Working on the mod

1. **Link it into the game** (on the host):
   ```
   ln -s ~/dev/vic3-mods/limit_cons_share/mod "$HOME/.local/share/Paradox Interactive/Victoria 3/mod/limit_cons_share"
   ```
   Then enable it in a playset in the launcher.
2. **Debug mode:** add `-debug_mode` to the Steam launch options. The `` ` `` key opens the
   console; `event lcs.1` forces a recalculation.
3. **Errors** go to `~/.local/share/Paradox Interactive/Victoria 3/logs/error.log`. Search
   it for `lcs`, `budget_panel` and `construction_panel`.
4. **Lint** with [vic3-tiger](https://github.com/amtep/tiger/releases):
   `vic3-tiger --game "<Victoria 3 install>" mod`. It catches typos and unknown keys.
   It's built for 1.13.5, so it reports 2 errors and 2 warnings in vanilla parts of
   `construction_panel.gui`; those can be ignored.
5. **Version control:** `.gitattributes` contains `* -text`, so git never converts line
   endings (on Windows it may otherwise switch files to CRLF). The template used
   `* binary`, which also hid all diffs and blocked merges; that was changed.

## 8. Things learned the hard way

- **Script can't see construction usage.** Only the interface can read how much
  construction the government or private sector is using. That's why private demand is
  estimated (see the README's limitations).
- **Construction goods aren't charged per point.** A sector's goods cost is its full input
  times the share of its output that is used. A flat −construction modifier raised the cost
  per point and saved nothing; lowering throughput scales input and output together.
- **Per-building caps apply before efficiency.** A project showing less than its cap can
  still be at its cap; the leftover then goes to the next project. That's vanilla.
- **Interface clicks can pass fixed values only.** Each bar segment carries its own value
  (5, 10 … 100) because a slider's position can't be passed to script.

## 9. What I wrote vs. what came from the template

| | Origin |
| --- | --- |
| `README.md` | Rewritten for this mod |
| `documentation/DEVELOPMENT.md` | New |
| `image/`, `script/`, `.gitignore`, `.gitattributes`, thumbnails | Template, unchanged |
| `mod/.metadata/metadata.json` | Template, filled in |
| `mod/events/lcs_events.txt`, `mod/common/on_actions/lcs_on_actions.txt` | Template placeholders renamed; mod logic added |
| `mod/gui/budget_panel.gui`, `mod/gui/construction_panel.gui` | Vanilla 1.15.2 copies + hooks |
| Everything else under `mod/` | New |

The mod's own code is about 750 lines, a third of it the generated bar segments in
`lcs_construction_cap.gui`.
