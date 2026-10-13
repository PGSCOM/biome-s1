---
name: freecad-goals-json
description: Write hand-made goal JSON files for biome-s1 (FreeCAD parts). Use when the user wants a custom part built by the model, a new demo goal, or an evaluation goal outside the synthetic sampler.
---

# Writing goal JSON files for biome-s1

A goals file is a JSON object mapping a goal name to a goal spec.
It is consumed with `scripts/gui_demo.py --goals <file> --name <goal>`
(also `scripts/mesa_demo.py` / `scripts/mesa_ax.py --goals`, default
`scripts/showcase_goals.json`, the reference example).

```json
{
  "bracket": {
    "level": 3,
    "scale": 40.0,
    "features": [
      {"kind": "base_box", "params": {"w": 40, "d": 30, "h": 10}},
      {"kind": "hole", "params": {"r": 3, "x": 10, "y": 0}},
      {"kind": "mirror", "params": {}},
      {"kind": "chamfer_top", "params": {"size": 1}}
    ]
  }
}
```

## Rules

- `features[0]` must be a base (`base_box`, `base_cyl`, `base_hex`, `base_ring`).
- Order is build order. A pattern/mirror copies the feature built
  immediately before it (the PartDesign Tip), so it must directly follow
  the feature to repeat.
- `level` is the feature-count bucket used by eval splits (`goals.py`:
  levels 1-3 hold up to 5 features, level 4 holds 6+, level 5 holds 8+).
  Pick the level that matches the feature count.
- `scale` normalizes lengths for the model: set it to the largest base
  dimension (`max(w, d, h)` for a box, `max(2*r, h)` for cylinder/hexagon,
  `max(2*ro, h)` for a ring).
- Step budget is automatic (`2 * expert steps + 6`); never put it in the file.
- Keep every profile on the top face: centers must fit with margin.
  Sampler convention is centers within 80% of the half-extent minus the
  radius; on cylinders/hexagons use `0.6 * r` as the usable half-extent.

## Feature reference

Sketched features build as: select support, new sketch, profile, constrain,
leave sketch, apply. Supports are the XY plane for bases (`base_ring` uses
the XZ plane, revolved 360 degrees) and the top face (+Z) for everything else.

| kind | params | command | notes |
|---|---|---|---|
| `base_box` | `w`, `d`, `h` | Pad | rectangle centered on origin, extruded `h` |
| `base_cyl` | `r`, `h` | Pad | circle diameter `2*r`, centered, extruded `h` |
| `base_hex` | `r`, `h` | Pad | hexagon inscribed in circle `r`, extruded `h` |
| `base_ring` | `ri`, `ro`, `h` | Revolution | ring, inner radius `ri`, outer `ro`, height `h` |
| `boss_cyl` | `r`, `x`, `y`, `h` | Pad | cylinder on the top face at `(x, y)`, height `h` |
| `boss_box` | `w`, `d`, `x`, `y`, `h` | Pad | box on the top face centered at `(x, y)`, height `h` |
| `hole` | `r`, `x`, `y` | Pocket | always cuts through everything (`ThroughAll`) |
| `hole_std` | `r`, `x`, `y` | Hole | standard hole, `ThroughAll`, diameter `2*r` |
| `pocket_rect` | `w`, `d`, `x`, `y`, `depth` | Pocket | blind pocket, keep `depth` below the base height |
| `linear_pattern` | `n`, `length` | LinearPattern | repeats Tip `n` times along +X; `length` is the span from first to last copy, so spacing is `length / (n - 1)`; omit `length` only to accept the `0.4 * scale` default |
| `polar_pattern` | `n` | PolarPattern | repeats Tip `n` times around Z, full 360 degrees |
| `mirror` | `{}` | Mirrored | mirrors Tip across the YZ plane (`x` changes sign); put the original at `x > 0` |
| `fillet_top` | `r` | Fillet | rounds the top-face edges |
| `fillet_vertical` | `r` | Fillet | rounds the vertical edges |
| `chamfer_top` | `size` | Chamfer | chamfers the top-face edges |
| `shell` | `t` | Thickness | hollows the part from the top face, wall `t` |

`linear_pattern` is one-dimensional: for a 2D grid, emit one
boss + `linear_pattern` pair per row, stepping the boss `y` by the row
pitch (each pattern repeats only the boss directly before it).

## Validate before handing over

Every goal must build with the clean expert to a single valid solid.
Run headless (needs FreeCAD's `FreeCAD.so` on `PYTHONPATH`):

```bash
PYTHONPATH=<FreeCAD lib>:$PYTHONPATH .venv/bin/python -c "
import json
from freecad_s1.schema import Goal
from freecad_s1.runtime.session import HeadlessSession
from freecad_s1.runtime.episode import build_target, score
from freecad_s1.goals import StartSpec
spec = json.load(open('<file>'))['<goal>']
s = HeadlessSession()
goal = Goal.from_json(spec)
target = build_target(s, goal)  # raises GoalBuildError if infeasible
s.reset(goal, StartSpec(doc_open=True, workbench='PartDesignWorkbench'))
while True:
    exp = s.expert()
    if exp[0] == 'Done':
        s.step('Done'); break
    assert s.step(exp[0])['error'] is None
print('OK, IoU:', round(score(s, target)['iou'], 4))
s.shutdown()
"
```

Success is `IoU >= 0.99` against the target. If `build_target` raises,
the geometry is infeasible (overlapping profiles, pattern running off the
part, dressup radius too large): shrink sizes, move centers inward, or
shorten the pattern.

## Known model limits

- Weights trained before the sampler covered boss patterns struggle with a
  pattern directly on a `boss_cyl` (they loop Pad/Undo instead of selecting
  the Tip). Current `scripts/gui_demo.py` recovers via `--fallback-loops`,
  but prefer validating such goals with the expert snippet above.
- Deliberately held-out pairings (`heldout_composition` in `goals.py`):
  any pattern with `boss_box`, `mirror` after `hole_std` or `boss_box`,
  and any pattern after `pocket_rect`. The released models still build
  most of these, but treat them as adversarial and always validate.
