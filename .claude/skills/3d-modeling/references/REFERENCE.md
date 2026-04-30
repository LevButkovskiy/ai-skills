# 3D Modeling Reference — Part Recipes

This file contains detailed code recipes for common part types.
Load this file when working on the corresponding part type.

## Table of Contents

1. [Spur Gears](#spur-gears)
2. [Metric Threads (Helical)](#metric-threads)
3. [Bottle Caps (GPI Standards)](#bottle-caps)
4. [Enclosures with Lid](#enclosures)
5. [Flanges with Bolt Pattern](#flanges)
6. [Sweep Examples — Pipes & Channels](#sweep-examples)
7. [Assembly Constraint Reference](#assembly-constraints)

---

## Spur Gears

```python
import cadquery as cq
import math

def involute_point(base_radius, t):
    x = base_radius * (math.cos(t) + t * math.sin(t))
    y = base_radius * (math.sin(t) - t * math.cos(t))
    return (x, y)

def gear_tooth_profile(module, num_teeth, pressure_angle_deg=20):
    """Return list of (x,y) points for one tooth profile (involute)."""
    pa   = math.radians(pressure_angle_deg)
    pitch_r  = module * num_teeth / 2
    base_r   = pitch_r * math.cos(pa)
    addend_r = pitch_r + module            # addendum = 1 * module
    deduct_r = pitch_r - 1.25 * module    # dedendum = 1.25 * module

    pts = []
    # Involute from base circle to addendum
    t_max = math.sqrt((addend_r / base_r) ** 2 - 1)
    for i in range(20):
        t = t_max * i / 19
        pts.append(involute_point(base_r, t))
    return pts, pitch_r, base_r, addend_r, deduct_r

def make_spur_gear(module=2, num_teeth=20, width=10, pressure_angle=20):
    """Create a spur gear using CadQuery + OCP."""
    from OCP.gp import gp_Pnt, gp_Vec
    from OCP.BRepBuilderAPI import (
        BRepBuilderAPI_MakeWire, BRepBuilderAPI_MakeEdge, BRepBuilderAPI_MakeFace
    )
    from OCP.BRepPrimAPI import BRepPrimAPI_MakePrism
    from OCP.BRep import BRep_Builder
    from OCP.TopoDS import TopoDS_Compound

    inv_pts, pitch_r, base_r, addend_r, deduct_r = gear_tooth_profile(
        module, num_teeth, pressure_angle
    )
    tooth_angle = 2 * math.pi / num_teeth
    half_tooth  = tooth_angle / 4

    # Build all teeth by rotating the involute profile
    all_pts = []
    for i in range(num_teeth):
        angle = i * tooth_angle
        # left flank
        for (x, y) in inv_pts:
            r   = math.hypot(x, y)
            a   = math.atan2(y, x) + angle - half_tooth
            all_pts.append((r * math.cos(a), r * math.sin(a)))
        # right flank (mirror)
        for (x, y) in reversed(inv_pts):
            r   = math.hypot(x, y)
            a   = math.atan2(y, x) + angle + half_tooth
            all_pts.append((r * math.cos(a), r * math.sin(a)))

    # Close the profile
    all_pts.append(all_pts[0])

    wire_builder = BRepBuilderAPI_MakeWire()
    for i in range(len(all_pts) - 1):
        p1 = gp_Pnt(all_pts[i][0],   all_pts[i][1],   0.0)
        p2 = gp_Pnt(all_pts[i+1][0], all_pts[i+1][1], 0.0)
        if p1.Distance(p2) > 1e-6:
            wire_builder.Add(BRepBuilderAPI_MakeEdge(p1, p2).Edge())

    face  = BRepBuilderAPI_MakeFace(wire_builder.Wire()).Face()
    solid = BRepPrimAPI_MakePrism(face, gp_Vec(0, 0, width)).Shape()

    # Bore hole
    gear = cq.Shape(solid)
    bore_r = max(2.0, deduct_r * 0.4)
    bore = cq.Workplane("XY").circle(bore_r).extrude(width).val()
    result_shape = gear.cut(cq.Shape(bore.wrapped))

    return cq.Workplane().newObject([result_shape])

gear = make_spur_gear(module=2, num_teeth=20, width=10)
cq.exporters.export(gear, "gear.step", exportType="STEP")
cq.exporters.export(gear, "gear.stl")
```

**Key gear parameters:**
- `module (m)` = pitch circle diameter / num_teeth. Common: 1, 1.5, 2, 2.5, 3
- `pressure_angle` = 20° (standard, DIN 867 / ISO 53)
- Addendum = 1 × m, Dedendum = 1.25 × m
- Face width typically 8–12 × m

---

## Metric Threads

```python
import cadquery as cq
import math

def make_external_thread(
    major_dia,    # e.g. 8.0 for M8
    pitch,        # e.g. 1.25 for M8x1.25
    length,       # thread length in mm
    thread_depth=None  # defaults to 0.6495 * pitch (ISO 68-1)
):
    """Create an external (male) metric thread as a solid."""
    if thread_depth is None:
        thread_depth = 0.6495 * pitch

    minor_dia = major_dia - 2 * thread_depth
    # Helix approximated as stacked helical wire via OCP
    from OCP.gp import gp_Pnt, gp_Ax2, gp_Dir, gp_XYZ
    from OCP.BRepBuilderAPI import BRepBuilderAPI_MakeEdge
    from OCP.BRepBuilderAPI import BRepBuilderAPI_MakeWire
    from OCP.Geom import Geom_CylindricalSurface
    from OCP.GCE2d import GCE2d_MakeSegment
    from OCP.gp import gp_Pnt2d, gp_Dir2d
    from OCP.BRepOffsetAPI import BRepOffsetAPI_MakePipeShell

    # Build body cylinder
    body = cq.Workplane("XY").circle(major_dia / 2).extrude(length)

    # Helical cut: approximate with swept triangular profile
    # For most use cases, a simplified helical solid is sufficient
    turns = length / pitch
    num_steps = int(turns * 32)
    pts = []
    for i in range(num_steps + 1):
        t = i / num_steps
        angle = 2 * math.pi * turns * t
        z = length * t
        pts.append((major_dia / 2 * math.cos(angle),
                    major_dia / 2 * math.sin(angle), z))

    # Build thread groove using wire sweep (simplified)
    groove_r = (major_dia - minor_dia) / 2
    path_pts  = [cq.Vector(*p) for p in pts]
    # Use CadQuery wire from spline
    path = cq.Wire.makeSpline(path_pts)
    circle_profile = (
        cq.Workplane("XY")
        .workplane(offset=0)
        .circle(groove_r * 0.8)
    )
    thread_groove = circle_profile.sweep(
        cq.Workplane("XY").newObject([path])
    )
    return body.cut(thread_groove)

# Example: M8x1.25, 20mm long external thread
thread_body = make_external_thread(8.0, 1.25, 20)
cq.exporters.export(thread_body, "m8_thread.step", exportType="STEP")
```

**Metric thread dimensions (ISO 261):**

| Size | Pitch (coarse) | Pitch (fine) | Major Ø | Minor Ø |
|------|----------------|--------------|---------|---------|
| M4   | 0.7            | 0.5          | 4.0     | 3.242   |
| M5   | 0.8            | 0.5          | 5.0     | 4.134   |
| M6   | 1.0            | 0.75         | 6.0     | 4.917   |
| M8   | 1.25           | 1.0          | 8.0     | 6.647   |
| M10  | 1.5            | 1.0          | 10.0    | 8.376   |
| M12  | 1.75           | 1.25         | 12.0    | 10.106  |

---

## Bottle Caps (GPI Standards)

```python
import cadquery as cq

def make_bottle_cap_gpi(
    neck_od,       # GPI outer diameter at top of neck (mm)
    neck_height,   # distance from cap top to first thread start (mm)
    thread_pitch,  # GPI thread pitch (mm)
    num_turns=1.5, # number of thread turns
    wall=1.5,      # cap wall thickness (mm)
    seal_h=1.2,    # internal sealing plug height (mm)
):
    """
    Make a GPI-standard bottle cap.
    Example GPI 28/410: neck_od=28, thread_od≈30.5, pitch=3.18, turns=1.5
    Example GPI 24/415: neck_od=24, thread_od≈26.5, pitch=3.18, turns=1.5

    Always search for exact GPI spec before modeling: 
    Search "GPI [size] neck finish dimensions" for actual T, E, S dimensions.
    """
    thread_od  = neck_od + 2.5      # approximate; verify from GPI spec
    outer_od   = thread_od + 2 * wall
    cap_height = neck_height + num_turns * thread_pitch + 2.0

    # Cap shell
    cap = (
        cq.Workplane("XY")
        .circle(outer_od / 2)
        .extrude(cap_height)
        .faces(">Z")
        .shell(-wall)              # hollow out leaving top and walls
    )

    # Sealing plug (inner plug that contacts bottle mouth)
    seal = (
        cq.Workplane("XY")
        .workplane(offset=cap_height - wall - seal_h)
        .circle(neck_od / 2 - 0.3)  # slight interference fit
        .extrude(seal_h)
    )
    cap = cap.union(seal)

    # Knurl (cosmetic vertical ribs on outside)
    num_ribs = 24
    rib_w, rib_h_val = 1.0, cap_height * 0.7
    import math
    for i in range(num_ribs):
        angle = i * 360 / num_ribs
        rib = (
            cq.Workplane("XY")
            .transformed(rotate=(0, 0, angle))
            .workplane(offset=0)
            .move(outer_od / 2 - 0.2, 0)
            .rect(0.6, rib_w)
            .extrude(rib_h_val)
        )
        cap = cap.union(rib)

    return cap

# GPI 28/410 cap
cap = make_bottle_cap_gpi(
    neck_od=28.0, neck_height=4.0,
    thread_pitch=3.18, num_turns=1.5
)
cq.exporters.export(cap, "bottle_cap_28_410.step", exportType="STEP")
cq.exporters.export(cap, "bottle_cap_28_410.stl")
```

**Common GPI sizes (search web to confirm exact T/E/S dimensions):**

| GPI Code | Neck OD (E) | Pitch | Turns | Typical Use |
|----------|-------------|-------|-------|-------------|
| 18/415   | 18 mm       | 3.18  | 1.5   | Small bottles |
| 20/415   | 20 mm       | 3.18  | 1.5   | Nail polish |
| 24/415   | 24 mm       | 3.18  | 1.5   | Shampoo |
| 28/410   | 28 mm       | 3.18  | 1.5   | Wide-mouth |
| 38/400   | 38 mm       | 3.18  | 1.5   | Jars |

---

## Enclosures

```python
import cadquery as cq

def make_enclosure(
    length=80, width=60, height=40,
    wall=2.5, lid_h=8,
    boss_dia=6, boss_h=8, screw_dia=3,
    num_bosses=4
):
    """Rectangular enclosure with snap-fit lid and screw bosses."""

    # Base box (open top)
    base = (
        cq.Workplane("XY")
        .box(length, width, height - lid_h)
        .faces(">Z")
        .shell(-wall)
    )

    # Screw bosses at corners inside base
    boss_r  = boss_dia / 2
    boss_inset = wall + boss_r + 1
    corner_positions = [
        ( length/2 - boss_inset,  width/2 - boss_inset),
        (-length/2 + boss_inset,  width/2 - boss_inset),
        ( length/2 - boss_inset, -width/2 + boss_inset),
        (-length/2 + boss_inset, -width/2 + boss_inset),
    ]
    for (x, y) in corner_positions:
        boss = (
            cq.Workplane("XY")
            .workplane(offset=wall)
            .move(x, y)
            .circle(boss_r)
            .extrude(boss_h)
            .faces(">Z")
            .circle(screw_dia / 2)
            .cutBlind(-boss_h)
        )
        base = base.union(boss)

    # Lid
    lid = (
        cq.Workplane("XY")
        .workplane(offset=height - lid_h)
        .box(length, width, lid_h)
        .faces("<Z")
        .shell(-wall)
    )

    return base, lid

base, lid = make_enclosure()
cq.exporters.export(base, "enclosure_base.step", exportType="STEP")
cq.exporters.export(lid,  "enclosure_lid.step",  exportType="STEP")
cq.exporters.export(base, "enclosure_base.stl")
cq.exporters.export(lid,  "enclosure_lid.stl")
```

---

## Flanges

```python
import cadquery as cq
import math

def make_flange(
    pipe_od=48.3,    # pipe outer diameter (e.g. DN40 = 48.3mm)
    flange_od=150,   # flange outer diameter
    flange_t=14,     # flange thickness
    bolt_pcd=110,    # bolt hole pitch circle diameter
    num_bolts=4,
    bolt_dia=14,     # bolt hole diameter
    pipe_length=30,  # stub length below flange
):
    """Flat face flange, slip-on style."""
    # Flange disk
    flange = (
        cq.Workplane("XY")
        .circle(flange_od / 2)
        .circle(pipe_od / 2)       # bore
        .extrude(flange_t)
    )

    # Bolt holes on PCD
    for i in range(num_bolts):
        angle = i * 360 / num_bolts
        x = bolt_pcd / 2 * math.cos(math.radians(angle))
        y = bolt_pcd / 2 * math.sin(math.radians(angle))
        flange = (
            flange
            .faces(">Z").workplane()
            .move(x, y)
            .circle(bolt_dia / 2)
            .cutBlind(-flange_t)
        )

    # Pipe stub
    stub = (
        cq.Workplane("XY")
        .workplane(offset=-pipe_length)
        .circle(pipe_od / 2)
        .extrude(pipe_length)
    )
    result = flange.union(stub)

    # Bore through entire assembly
    bore = cq.Workplane("XY").workplane(offset=-pipe_length).circle(pipe_od/2 - 4).extrude(flange_t + pipe_length)
    result = result.cut(bore)

    return result

flange = make_flange()
cq.exporters.export(flange, "flange.step", exportType="STEP")
cq.exporters.export(flange, "flange.stl")
```

---

## Sweep Examples

### Elbow Pipe

```python
import cadquery as cq

# 90° pipe elbow: 1" nominal (OD=33.4mm, wall=2.9mm)
pipe_od, pipe_wall, bend_r = 33.4, 2.9, 50.0
pipe_id = pipe_od - 2 * pipe_wall

path = (
    cq.Workplane("XZ")
    .moveTo(0, 0)
    .radiusArc((bend_r, bend_r), bend_r)   # quarter-circle
)
elbow = (
    cq.Workplane("YZ")
    .circle(pipe_od / 2)
    .circle(pipe_id / 2)
    .sweep(path)
)
cq.exporters.export(elbow, "pipe_elbow.step", exportType="STEP")
```

### Cable Channel (Rectangular Sweep along Spline)

```python
import cadquery as cq

# Routing path as spline
path_pts = [
    cq.Vector(0, 0, 0),
    cq.Vector(20, 10, 0),
    cq.Vector(50, 10, 5),
    cq.Vector(80, 0, 10),
]
path_wire = cq.Wire.makeSpline(path_pts, includeCurrent=True)

# Rectangular channel cross-section
channel = (
    cq.Workplane("YZ")
    .rect(8, 5)                  # 8mm wide, 5mm tall channel
    .sweep(cq.Workplane("XY").newObject([path_wire]))
)
cq.exporters.export(channel, "cable_channel.step", exportType="STEP")
```

---

## Assembly Constraints Reference

CadQuery Assembly supports these constraint types:

| Constraint | Description | param |
|------------|-------------|-------|
| `"Point"` | Centers of selected faces/vertices coincide | — |
| `"Axis"` | Normals of selected faces are parallel | `param=0`: same dir; `param=180`: opposed |
| `"PointInPlane"` | Point is constrained to a plane | optional offset |
| `"PointOnLine"` | Point lies on a line/edge | optional distance |
| `"FixedPoint"` | Part fixed at absolute position | — |
| `"FixedAxis"` | Part orientation fixed | — |

**Constraint selector syntax:** `"part_name@faces@>Z"` = top face of part named "part_name"

```python
# Full example: bolt through plate
plate = cq.Workplane("XY").box(40, 40, 5).faces(">Z").workplane().hole(6)
bolt  = cq.Workplane("XY").cylinder(20, 3)

assy = cq.Assembly()
assy.add(plate, name="plate")
assy.add(bolt,  name="bolt")
assy.constrain("plate@faces@>Z", "bolt@faces@<Z", "Point")   # center
assy.constrain("plate@faces@>Z", "bolt@faces@<Z", "Axis", param=0)  # co-axial
assy.solve()
assy.save("plate_bolt.step")
```

---

## Offline Standards Tables

These tables are provided as a fallback when web search is unavailable.
**Always verify against official datasheets before manufacturing.**

### Metric Threads — ISO 261 (Coarse Series)

| Size | Pitch | Major Ø | Pitch Ø | Minor Ø | Thread Depth |
|------|-------|---------|---------|---------|--------------|
| M3   | 0.5   | 3.000   | 2.675   | 2.459   | 0.271        |
| M4   | 0.7   | 4.000   | 3.545   | 3.242   | 0.379        |
| M5   | 0.8   | 5.000   | 4.480   | 4.134   | 0.433        |
| M6   | 1.0   | 6.000   | 5.350   | 4.917   | 0.541        |
| M8   | 1.25  | 8.000   | 7.188   | 6.647   | 0.677        |
| M10  | 1.5   | 10.000  | 9.026   | 8.376   | 0.812        |
| M12  | 1.75  | 12.000  | 10.863  | 10.106  | 0.947        |
| M16  | 2.0   | 16.000  | 14.701  | 13.835  | 1.083        |
| M20  | 2.5   | 20.000  | 18.376  | 17.294  | 1.353        |
| M24  | 3.0   | 24.000  | 22.051  | 20.752  | 1.624        |

Thread depth = 0.6495 × pitch

### Metric Threads — Fine Series

| Size | Pitch | Minor Ø |
|------|-------|---------|
| M6×0.75  | 0.75 | 5.188 |
| M8×1.0   | 1.0  | 6.917 |
| M10×1.0  | 1.0  | 8.917 |
| M10×1.25 | 1.25 | 8.647 |
| M12×1.25 | 1.25 | 10.647|
| M12×1.5  | 1.5  | 10.376|

### GPI Bottle Neck Finishes (approximate — verify from GPI datasheet)

| GPI Code | E (neck OD, mm) | T (thread OD, mm) | Pitch (mm) | Turns | H (height, mm) |
|----------|-----------------|-------------------|-----------|-------|----------------|
| 18/415   | 18.0            | 20.0              | 3.18      | 1.5   | 11.1           |
| 20/415   | 20.0            | 22.0              | 3.18      | 1.5   | 11.1           |
| 24/415   | 24.0            | 26.5              | 3.18      | 1.5   | 11.1           |
| 28/400   | 28.0            | 30.5              | 3.18      | 1.0   | 7.9            |
| 28/410   | 28.0            | 30.5              | 3.18      | 1.5   | 11.1           |
| 33/400   | 33.0            | 35.9              | 3.18      | 1.0   | 7.9            |
| 38/400   | 38.0            | 41.3              | 3.18      | 1.0   | 7.9            |
| 38/415   | 38.0            | 41.3              | 3.18      | 1.5   | 11.1           |

E = neck outer diameter; T = thread outer diameter (cap inner dia ≈ T + 0.3mm clearance)

### O-Ring Sizes — AS568 (common sizes)

| Dash No. | ID (mm)  | CS (mm) | OD (mm)  |
|----------|----------|---------|----------|
| -008     | 3.69     | 1.78    | 7.25     |
| -010     | 6.07     | 1.78    | 9.63     |
| -012     | 9.25     | 1.78    | 12.81    |
| -014     | 12.42    | 1.78    | 15.98    |
| -110     | 12.37    | 2.62    | 17.61    |
| -112     | 17.12    | 2.62    | 22.36    |
| -210     | 12.37    | 3.53    | 19.43    |
| -214     | 24.99    | 3.53    | 32.05    |

CS = cross-section diameter; groove depth ≈ 0.74 × CS; groove width ≈ 1.3 × CS

### Gear Module / Tooth Count Reference

| Module | Pitch circle for Z teeth = m×Z | Addendum | Dedendum | Min teeth (no undercut) |
|--------|-------------------------------|----------|----------|------------------------|
| 1.0    | Z mm                           | 1.0 mm   | 1.25 mm  | 17                     |
| 1.5    | 1.5×Z mm                      | 1.5 mm   | 1.875 mm | 17                     |
| 2.0    | 2×Z mm                        | 2.0 mm   | 2.5 mm   | 17                     |
| 2.5    | 2.5×Z mm                      | 2.5 mm   | 3.125 mm | 17                     |
| 3.0    | 3×Z mm                        | 3.0 mm   | 3.75 mm  | 17                     |

Standard pressure angle: 20° (ISO 53 / DIN 867)
Face width rule of thumb: 8–12 × module