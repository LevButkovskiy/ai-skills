---
name: 3d-modeling
description: >
  Use this skill whenever the user wants to create, design, or generate 3D models programmatically.
  Includes: STL/STEP/3MF/IGES/OBJ creation, 3D printing prep, mechanical parts (gears, threads,
  enclosures, brackets, flanges, shafts, housings), assemblies, loft/sweep/revolve, and B-rep
  solid modeling. Trigger on: STL, STEP, IGES, 3MF, CAD, 3D print, gear, шестерёнка, крышка,
  резьба, thread, extrude, revolve, loft, sweep, fillet, chamfer, parametric, enclosure, housing,
  bracket, flange, assembly, сборка, деталь. Also trigger for КОМПАС/Fusion 360/SolidWorks/FreeCAD
  requests. Use even if user says сделай модель, спроектируй деталь, нарисуй деталь, сделай сборку.
---

# 3D Modeling Skill

## Overview

This skill enables Claude to create professional-quality 3D solid models programmatically using
Python and the CadQuery/OpenCASCADE stack. Models are exported as STEP (preferred for CAD),
STL (for 3D printing), 3MF, AMF, IGES, and OBJ files.

## Critical Workflow

**ALWAYS follow this sequence for every 3D modeling request:**

1. **Understand** — Parse all dimensions, tolerances, and functional requirements
2. **Research** — If the request involves standards (GPI bottle necks, ISO threads, gear modules,
   DIN flanges): try `web_search` first; if unavailable, use tables in `references/REFERENCE.md`;
   if neither works, use approximate values and flag them clearly to the user.
3. **Calculate** — Derive all dependent geometry from input parameters. Print a parameter table.
4. **Validate inputs** — Check for geometric conflicts (e.g., tooth width > pitch, wall too thin,
   thread pitch too fine for FDM printing)
5. **Model** — Build the solid using CadQuery
6. **Verify** — Compute bounding box, volume, center of mass. Compare against requirements.
7. **Export** — Save to requested format(s) and copy to `/mnt/user-data/outputs/`

## Environment Setup

**CadQuery is NOT pre-installed** — always install it first with pip before any import.
`pypi.org` is in the allowed network list, so installation is fast (~1–2 seconds, silent).

### Step 1: Install cadquery (ALWAYS run this first, unconditionally)

```bash
pip install cadquery --break-system-packages -q
```

### Step 2: Verify import

```python
import cadquery as cq
print("CadQuery OK:", cq.__version__)
```

**Important notes:**
- The OCP package on PyPI is named `cadquery-ocp` (not `OCP`) — cadquery pulls it automatically
- Supports Python 3.9–3.12 only; Python 3.13+ may fail
- Installation is idempotent — running pip install again when already installed is instant and harmless

### Step 3: If pip install fails

Network is blocked or pip fails. In this case:
1. Tell the user: "CadQuery is not available in this environment"
2. Offer to generate the Python modeling script as a `.py` file the user can run locally
3. Provide installation instructions: `pip install cadquery` (requires Python 3.9–3.12)

### For STL-only tasks (no B-rep needed)

`numpy-stl` may not be pre-installed, but is also rarely needed since CadQuery handles STL:

```bash
pip install numpy-stl --break-system-packages -q
```

## Format Selection Guide

| Target | Best Format | Notes |
|--------|-------------|-------|
| КОМПАС-3D | STEP (.step) | File → Open, save as .m3d. IGES as backup. |
| Fusion 360 | STEP (.step) | Full B-rep import, editable geometry |
| SolidWorks | STEP (.step) | Universal exchange format |
| FreeCAD | STEP (.step) | Native STEP support |
| 3D Printing (FDM/SLA) | STL (.stl) | Mesh only, no parametric data |
| 3D Printing (modern) | 3MF (.3mf) | Better than STL: preserves units, color, metadata |
| Web/Visual | OBJ (.obj) | Lightweight mesh |
| Multi-material printing | AMF (.amf) | XML-based, supports color/material |
| Any CAD | STEP (.step) | Always the safest default |

**Proprietary formats** (.m3d, .sldprt, .f3d, .ipt) cannot be created programmatically —
they require their respective CAD software. Explain this to the user and provide STEP instead,
with instructions on how to convert in the target CAD.

When the user doesn't specify a format, **always export both STEP and STL**.

## CadQuery Modeling Patterns

### Basic Extrusion (Prism from Sketch)

```python
import cadquery as cq

part = (
    cq.Workplane("XY")
    .circle(10)          # outer circle r=10
    .circle(5)           # inner hole r=5 (creates a ring)
    .extrude(20)         # extrude 20mm
)
cq.exporters.export(part, "part.step", exportType="STEP")
cq.exporters.export(part, "part.stl")
```

### Revolution (Axisymmetric Parts)

For parts like bottle caps, bushings, pulleys — sketch a 2D profile and revolve:

```python
profile = (
    cq.Workplane("XZ")
    .moveTo(5, 0)
    .lineTo(10, 0)
    .lineTo(10, 15)
    .lineTo(5, 15)
    .close()
    .revolve(360, (0, 0, 0), (0, 1, 0))
)
```

### Sweep (Pipes, Tubes, Channels)

Sweep a 2D profile along a path:

```python
# Pipe: circular cross-section swept along a 3-point arc
path = (
    cq.Workplane("XZ")
    .moveTo(0, 0)
    .threePointArc((10, 5), (20, 0))
)
result = (
    cq.Workplane("YZ")
    .circle(3)            # outer radius
    .circle(2)            # inner (wall = 1mm)
    .sweep(path)
)
```

### Loft (Transitional Shapes, Connectors)

Loft between two or more cross-sections:

```python
result = (
    cq.Workplane("XY")
    .rect(20, 10)              # bottom rectangle
    .workplane(offset=15)
    .circle(8)                 # top circle
    .loft()
)
```

### Selector Strings (Face/Edge/Vertex Selection)

CadQuery uses a powerful selector DSL — always use it instead of hard-coding coordinates:

```python
# Select faces by direction (centroid-based)
part.faces(">Z")          # topmost face
part.faces("<Z")          # bottommost face
part.faces("|X")          # faces whose normal is parallel to X
part.faces("#Z")          # faces perpendicular to Z (i.e. horizontal)
part.faces("+Y or -Y")    # faces pointing +Y or -Y

# Select edges
part.edges("|Z")          # vertical edges (parallel to Z)
part.edges("%CIRCLE")     # circular edges only
part.edges(">Z")          # edges with highest Z centroid

# Combine selectors with logical operators
part.faces(">Z or <Z")    # top and bottom faces

# Apply fillets to specific edges
result = (
    cq.Workplane("XY").box(10, 10, 5)
    .edges("|Z")           # all 4 vertical edges
    .fillet(1.0)
)
```

### Tags for Complex Multi-Feature Parts

Use tags to return to an earlier workplane after further operations:

```python
result = (
    cq.Workplane("XY")
    .box(30, 30, 8)
    .tag("base_top")                        # remember this face
    .faces(">Z").workplane()
    .circle(5).extrude(10)                  # add boss
    .workplaneFromTagged("base_top")        # return to tagged face
    .faces(">Z").workplane()
    .circle(2).cutBlind(-8)                 # add hole on base
)
```

### Complex Profiles (Gears, Splines, Cams) — OCP Level

When CadQuery's sketch tools are insufficient, build geometry at the OCP level:

```python
from OCP.gp import gp_Pnt, gp_Vec
from OCP.BRepBuilderAPI import (
    BRepBuilderAPI_MakeWire,
    BRepBuilderAPI_MakeEdge,
    BRepBuilderAPI_MakeFace,
)
from OCP.BRepPrimAPI import BRepPrimAPI_MakePrism

wire_builder = BRepBuilderAPI_MakeWire()
for i in range(len(pts) - 1):
    p1 = gp_Pnt(pts[i][0], pts[i][1], 0.0)
    p2 = gp_Pnt(pts[i+1][0], pts[i+1][1], 0.0)
    wire_builder.Add(BRepBuilderAPI_MakeEdge(p1, p2).Edge())

face  = BRepBuilderAPI_MakeFace(wire_builder.Wire()).Face()
solid = BRepPrimAPI_MakePrism(face, gp_Vec(0, 0, height)).Shape()
result = cq.Shape(solid)
cq.exporters.export(result, "output.step", exportType="STEP")
```

### Boolean Operations

```python
body   = cq.Workplane("XY").box(50, 30, 10)
hole   = cq.Workplane("XY").circle(5).extrude(10)
result = body.cut(hole)          # subtraction
# Also: body.union(other), body.intersect(other)
```

### Multi-Part Assemblies

Use `cq.Assembly` to combine parts with positional constraints:

```python
plate = cq.Workplane("XY").box(50, 50, 5).faces(">Z").workplane().hole(8)
pin   = cq.Workplane("XY").cylinder(20, 4)

assy = cq.Assembly()
assy.add(plate, name="plate", color=cq.Color("gray"))
assy.add(pin,   name="pin",   color=cq.Color("steelblue"))

# Constraint: pin bottom face centered on plate top face hole
assy.constrain("plate@faces@>Z", "pin@faces@<Z", "Point")
assy.constrain("plate@faces@>Z", "pin@faces@<Z", "Axis", param=0)
assy.solve()

# Export assembly STEP (preserves part tree)
assy.save("assembly.step")
```

### Verification Pattern

Always include at the end of every modeling script:

```python
# For CadQuery Workplane result
bb = result.val().BoundingBox()
print(f"Bounding box:")
print(f"  X: {bb.xmin:.3f}..{bb.xmax:.3f} = {bb.xmax - bb.xmin:.3f} mm")
print(f"  Y: {bb.ymin:.3f}..{bb.ymax:.3f} = {bb.ymax - bb.ymin:.3f} mm")
print(f"  Z: {bb.zmin:.3f}..{bb.zmax:.3f} = {bb.zmax - bb.zmin:.3f} mm")
```

For OCP-level shapes:

```python
from OCP.Bnd import Bnd_Box
from OCP.BRepBndLib import BRepBndLib
from OCP.GProp import GProp_GProps
from OCP.BRepGProp import BRepGProp

bbox = Bnd_Box()
BRepBndLib.Add_s(solid, bbox)
xmin, ymin, zmin, xmax, ymax, zmax = bbox.Get()
print(f"X: {xmin:.3f}..{xmax:.3f} = {xmax - xmin:.3f} mm")
print(f"Y: {ymin:.3f}..{ymax:.3f} = {ymax - ymin:.3f} mm")
print(f"Z: {zmin:.3f}..{zmax:.3f} = {zmax - zmin:.3f} mm")

props = GProp_GProps()
BRepGProp.VolumeProperties_s(solid, props)
volume = props.Mass()
cog    = props.CentreOfMass()
print(f"Volume: {volume:.2f} mm³")
print(f"Center of mass: ({cog.X():.2f}, {cog.Y():.2f}, {cog.Z():.2f})")
```

## Common Part Recipes

For detailed recipes for specific part types, read `references/REFERENCE.md`:

- **Spur gears** — profile generation, tooth geometry, module/pitch calculations
- **Threads** — helical internal/external threads (metric, GPI bottle standards)
- **Bottle caps** — GPI 24/415, 28/410, etc., with sealing rings and knurling
- **Enclosures** — box with lid, snap fits, screw bosses
- **Flanges** — bolt patterns, gasket faces
- **Sweep examples** — pipes, channels, routing paths
- **Assemblies** — constraint types, multi-part STEP export

## Standards Research

For anything involving standards (threads, bottle necks, flanges, O-rings), prefer verified
dimensions over memory. Follow this priority:

1. **Web search (preferred)** — use the `web_search` tool if available:
   - Bottle caps: `"GPI 28-410 neck finish dimensions mm"`
   - Metric threads: `"M8x1.25 ISO dimensions"`
   - Gears: `"DIN 867 gear module pressure angle"`
   - O-rings: `"AS568-110 dimensions"`
   - Flanges: `"DIN 2576 DN40 flange dimensions"`

2. **Use REFERENCE.md tables (offline fallback)** — `references/REFERENCE.md` contains
   metric thread tables (M4–M12) and common GPI bottle cap sizes. Use these when web search
   is unavailable.

3. **Flag uncertainty explicitly** — if neither web nor REFERENCE.md has the exact spec,
   tell the user: "I'm using approximate dimensions for [standard] — please verify against
   the official datasheet before manufacturing." Then proceed with best-guess values and
   add a note in the verification table.

Never silently guess standards dimensions — always flag when values are approximate.

## Functional Requirements Checklist

When the user says "strong", "watertight", "герметичный", "прочный":

**Strength:**
- Wall thickness ≥ 2mm for PP/ABS, ≥ 1.5mm for metal
- Fillet radii at stress concentrations (≥ 0.5mm)
- Avoid sharp internal corners

**Sealing / Герметичность:**
- Include a sealing lip or O-ring groove
- Interference fit: 0.1–0.3mm for plastic-on-plastic
- Thread engagement: ≥ 1.5 turns minimum

**3D Print Considerations:**
- Min wall thickness: 0.8mm (FDM), 0.5mm (SLA)
- Overhang angles ≤ 45° without supports
- Thread pitch ≥ 1.5mm for FDM printability
- Add 0.2–0.3mm clearance for mating parts
- Prefer 3MF over STL for modern slicers (preserves units and scale)

**Multi-part toys / fidget toys (КРИТИЧНО — learned from failures):**

1. **Раздельный экспорт деталей — ОБЯЗАТЕЛЬНО:**
   - Каждая подвижная или независимая деталь экспортируется в ОТДЕЛЬНЫЙ STL/STEP файл.
   - НИКОГДА не записывать все тела в один файл — слайсер не разделит детали, они
     напечатаются как монолит.
   - Схема именования: `toyname_sunGear.stl`, `toyname_planet1.stl`, `toyname_body.stl`
   - В CadQuery: отдельный `cq.exporters.export(part, filename)` для каждой детали.
   - В numpy/ручном STL: отдельный вызов `write_stl(filename, triangles)` для каждой детали.

2. **Зазор между зубьями шестерён (gear clearance):**
   - Минимальный зазор между зубьями: **0.35–0.4 мм на сторону** (0.2 мм недостаточно — зубья
     сплавятся при печати).
   - Уменьшать addendum каждой шестерни на 0.35 мм от теоретического значения.
   - Расстояние между центрами оставлять теоретическим — зазор достигается уменьшением зубьев,
     не раздвиганием центров.

3. **Нависающие элементы (overhangs):**
   - Любой декоративный выступ (бампы, рукоятки, ушки) должен иметь угол к вертикали ≤ 45°.
   - Горизонтальные цилиндрические бампы ЗАПРЕЩЕНЫ без поддержек — заменять конусами или
     каплевидными формами (self-supporting).
   - Если нависание неизбежно — явно предупредить пользователя: «Деталь X требует поддержек».

## Output Delivery

After generating models:

1. Copy files to `/mnt/user-data/outputs/`
2. Use `present_files` to share with user
3. Provide a concise verification table comparing requested vs actual dimensions
4. If the model is for a specific CAD program, include brief import instructions

## Common Mistakes to Avoid

- **Wrong OCP package name**: Use `pip install cadquery-ocp`, NOT `pip install OCP`
- **Python version**: CadQuery requires Python 3.9–3.12; Python 3.13+ may fail
- **Thread too shallow**: For bottle caps, thread depth should be ≥ 0.8mm
- **Cap too short**: For GPI 415 finishes, cap height should be ≥ 15mm
- **No sealing element**: Always add a seal ring or gasket groove for "герметичный" parts
- **Forgetting clearance**: Mating parts need 0.2–0.5mm gap depending on manufacturing
- **Gear clearance too small**: For FDM gears, use 0.35–0.4mm per side on tooth addendum — 0.2mm causes teeth to fuse during printing
- **All parts in one STL**: Multi-part toys MUST export each part as a separate file — merging all triangles into one STL makes slicer treat everything as one solid
- **Horizontal decorative bumps**: Cylindrical bumps extruding outward horizontally are unprintable without supports — use cones or teardrop shapes instead
- **Wrong units**: CadQuery defaults to mm. If user gives inches, convert first.
- **Ignoring tolerances**: Design with margin for manufacturing variation
- **Assembly not solved**: Always call `assy.solve()` before exporting constrained assemblies
- **Missing exportType**: Use `exportType="STEP"` explicitly for STEP files
- **Loft sections not pushed**: Between loft cross-sections, use `.toPending()` if chaining manually
