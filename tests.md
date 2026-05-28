# circuitcore test suite

713 test cases across 105 files. Every Catch2 TEST_CASE in the
repository, grouped by source file. The grouping mirrors the kit
the tests cover.

---

## board (circuitcore::board core)

### pdnkit/tests/hit_test.cpp

- **hittest: empty board misses** `[hittest]` -- calls `at_point` on a default Board; asserts the returned Hit kind is None.

- **hittest: zone polygon hit** `[hittest]` -- builds a board with one F.Cu layer and a 100x100mm zone on net 5; asserts a probe at (0.05, 0.05) returns a Zone hit with net_id 5 and layer 0, and a probe outside returns None.

- **hittest: zone hole is treated as not-inside** `[hittest]` -- adds a zone with a square hole; asserts a probe inside the hole misses and a probe outside the hole but inside the outline hits the zone.

- **hittest: segment hit within width/2 + pick radius** `[hittest]` -- places a 1mm-wide segment; asserts an on-centerline probe hits, a 0.6mm-off probe with zero pick radius misses, and the same probe with a 0.2mm pick radius hits with the segment's net.

- **hittest: via hit within outer radius** `[hittest]` -- adds a via with 0.8mm outer diameter; asserts probes within the outer radius return Via and a probe beyond it returns None.

- **hittest: pad takes priority over zone** `[hittest]` -- overlays a pad on a covering zone; asserts the hit at the pad center returns Pad with the pad's net_id, not the zone's.

- **hittest: name() handles all kinds** `[hittest]` -- asserts `hittest::name()` returns "pad", "via", "segment", "zone" and "" for None.

- **hittest: rect pad respects width/height + rotation** `[hittest]` -- places a 4x1mm rect pad at origin; asserts hits inside the wide and tall extents, misses beyond them.

- **hittest: rect pad rotated 90deg swaps W/H sensitivity** `[hittest]` -- rotates the same pad 90 degrees; asserts the long axis now follows Y and the short axis follows X.

### pdnkit/tests/model_test.cpp

- **model: empty board defaults** `[model]` -- constructs an empty Board; asserts nets, segments, vias, zones and stackup layers are empty and total_thickness defaults to 1.6mm.

- **model: find_net by id and name** `[model]` -- adds three nets; asserts `find_net(1)` and `find_net_by_name("+3V3")` return the expected entries and lookups for missing ids/names return nullptr.

- **model: find_layer and is_copper** `[model]` -- adds F.Cu, B.Cu and F.SilkS layers; asserts `find_layer` returns the right names by ordinal, a missing ordinal returns nullptr, and `is_copper()` is true for F.Cu/B.Cu and false for F.SilkS.

- **model: polygon with holes** `[model]` -- builds a zone polygon with one square hole; asserts the polygon retains one filled region with four outline vertices and one hole of four vertices.

### pdnkit/tests/package_defaults_test.cpp

- **package defaults: chip resistors / caps recognised** `[package-defaults]` -- looks up R_0402; asserts body height lands in 0.30..0.60 mm and that C_0805 is taller than C_0402.

- **package defaults: ICs differentiated by family** `[package-defaults]` -- looks up QFN-32, SOIC-8 and DIP-14; asserts QFN > 0 and SOIC > QFN and DIP > SOIC.

- **package defaults: connectors get a tall body** `[package-defaults]` -- looks up a 2.54mm pin header; asserts body height is greater than 5mm.

- **package defaults: unknown footprint returns generic default** `[package-defaults]` -- queries an unknown footprint name; asserts body height is exactly 1.0mm and mass is 0.1g.

- **package defaults: mass scales roughly with size** `[package-defaults]` -- asserts mass(R_0402) < mass(R_1206) and mass(SOIC-8) < mass(DIP-14).

---

## sexpr (circuitcore::sexpr s-expression parser)

### sexpr/tests/sexpr_test.cpp

- **sexpr: empty list** `[sexpr]` -- parses "()"; asserts the node is a list with no children.

- **sexpr: single symbol in list** `[sexpr]` -- parses "(kicad_pcb)"; asserts the child is a symbol with text "kicad_pcb" and `tag()` returns the same.

- **sexpr: numbers (int, float, negative)** `[sexpr]` -- parses a list of seven mixed numerics; asserts each child carries the expected double value (incl. 1e3 -> 1000).

- **sexpr: strings with escapes** `[sexpr]` -- parses a net form with escaped quotes and tab; asserts the unescaped strings round-trip into `text`.

- **sexpr: nested lists** `[sexpr]` -- parses "(a (b (c 1) (d 2)) e)"; asserts each nesting level reports the right tag and the deepest number is 1.0.

- **sexpr: KiCad-flavored snippet** `[sexpr]` -- parses a representative kicad_pcb fragment; asserts root tag is kicad_pcb, version is 20240108, layers contain two rows and the empty net name parses as an empty string atom.

- **sexpr: line comments are ignored** `[sexpr]` -- parses a list with a `;` comment to end of line; asserts the comment is skipped and three children remain.

- **sexpr: error on unterminated list** `[sexpr]` -- parses "(a (b c"; asserts `parse` throws ParseError.

- **sexpr: error on unterminated string** `[sexpr]` -- parses an input with an unterminated quoted string; asserts ParseError is thrown.

- **sexpr: trailing junk after the main form is tolerated** `[sexpr]` -- feeds three inputs with extra forms or stray tokens after the main form; asserts all three still return a node with tag "kicad_pcb".

- **sexpr: error on empty input** `[sexpr]` -- parses ""; asserts ParseError is thrown.

- **sexpr: line/col tracking** `[sexpr]` -- parses "(a\n  b\n  c)"; asserts each child reports the expected line and column for its position.

### sexpr/tests/sexpr_emit_test.cpp

- **emit: simple inline form** `[sexpr][emit]` -- emits "(foo 1 2 3)"; asserts the output matches the source plus a trailing newline.

- **emit: nested lists break onto their own lines** `[sexpr][emit]` -- emits a setup/stackup form; asserts output contains at least one newline and re-parsing reproduces the original tree.

- **emit: strings stay quoted, escapes round-trip** `[sexpr][emit]` -- emits and re-parses a form with escaped quotes; asserts `tree_equal` of original and re-parsed trees.

- **emit: numbers use the shortest round-trippable form** `[sexpr][emit]` -- emits coords 1.5, 0.51, 100 and pi; asserts output contains shortened tokens and no "1.500" cruft.

- **emit: empty list emits ()** `[sexpr][emit]` -- emits a form with embedded empty list; asserts re-parsed tree matches original.

- **emit: round-trips a kicad_pcb-shaped tree** `[sexpr][emit][validation]` -- emits a multi-layer stackup form and re-parses; asserts structural equality with the source.

---

## netlist (circuitcore::netlist + kicad netlist parser)

### formats/kicad/tests/netlist_parser_test.cpp

- **netlist: parses components and nets** `[netlist]` -- parses a tiny three-component, three-net export; asserts source_sheet matches and components.size() == 3 and nets.size() == 3.

- **netlist: component lookup** `[netlist]` -- looks up component "C1"; asserts its value is "100nF", footprint is "Capacitor_SMD:C_0402" and a nonexistent ref returns nullptr.

- **netlist: net lookup carries node list** `[netlist]` -- looks up net "VDD"; asserts code is 1, two nodes are present, the first node is C1.1 with pin_function "VDD" and pin_type "passive".

- **netlist: rejects non-export root** `[netlist]` -- parses "(kicad_pcb)"; asserts the optional return value is empty.

- **netlist: handles missing optional sections** `[netlist]` -- parses an export with empty design/components/nets; asserts parsing succeeds and both lists are empty.

---

## formats (kicad pcb + gmsh msh writers)

### formats/kicad/tests/parser_test.cpp

- **parser: top-level requires kicad_pcb** `[parser]` -- parses "(other)"; asserts the optional result is empty.

- **parser: parses stackup and thickness** `[parser]` -- parses the tiny board; asserts three layers, correct names and ordinals, total_thickness == 1.6e-3 and `is_copper` is true for F.Cu and false for F.SilkS.

- **parser: parses nets** `[parser]` -- parses the tiny board; asserts three nets including an unnamed net 0, "GND" at id 1 and "+3V3" at id 2.

- **parser: parses segment with mm-to-m conversion** `[parser]` -- parses one segment; asserts coordinates and width are converted from mm to meters and layer/net are filled in.

- **parser: parses via with from/to layer ordinals** `[parser]` -- parses a via with size 0.8 drill 0.4 spanning F.Cu to B.Cu; asserts the layer ordinals are 0 and 31.

- **parser: parses zone with outline and filled polygons** `[parser]` -- parses a GND zone; asserts net_id 1, layer 0, four-point outline and one filled polygon with four points starting at 5mm.

- **parser: footprint pads transformed by footprint origin** `[parser]` -- parses an R_0402 footprint at (40, 50); asserts the two pads end up at the world-space positions (39.5, 50) and (40.5, 50) with the right nets.

- **parser: unknown layer name in segment is an error** `[parser]` -- parses a board where a segment references layer "Mystery"; asserts the parse fails.

- **parser: pad shape and size extracted** `[parser]` -- parses the tiny board pads; asserts shape is Rect and size is 0.5x0.5mm.

- **parser: parent_ref populated from footprint Reference property** `[parser]` -- parses the tiny board; asserts both pads have parent_ref "R1".

- **parser: graphic items on non-copper layers** `[parser][graphics]` -- parses a board with five board-level graphics plus one fp_line; asserts at least one of each kind (Line/Arc/Circle/Polygon/Text) plus the fp_line on F.CrtYd is transformed to world (40, 50)-(41, 50).

- **parser: graphic items skip Edge.Cuts and copper layers** `[parser][graphics]` -- iterates graphics; asserts every graphic's layer is non-copper and not Edge.Cuts.

- **parser: emits Component per footprint** `[parser][components]` -- parses the tiny board; asserts one Component with name "Resistor_SMD:R_0402", reference R1, value 10k, position (40, 50) mm and a zero courtyard bbox.

- **parser: component courtyard bbox from fp_line on F.CrtYd** `[parser][components]` -- parses the graphics board; asserts the U1 component's courtyard bbox collapses to the (40, 50)-(41, 50) fp_line extent.

### formats/gmsh/tests/msh_writer_test.cpp

- **msh writer: empty grid is rejected** `[msh]` -- writes a 0x0x0 grid; asserts the writer returns an empty optional.

- **msh writer: writes correct node + element counts** `[msh]` -- writes a 4x3x2 uniform copper grid; asserts MeshFormat header, "Nodes\n60", "Elements\n24" and 24 hex8 element lines.

- **msh writer: air voxels are skipped from elements** `[msh]` -- writes a 2x2x2 grid with one copper voxel; asserts only 1 element is emitted but the full 3x3x3 = 27 vertex lattice is written.

- **msh writer: distinct material ids become $PhysicalNames entries** `[msh]` -- writes a 3x1x1 grid with copper, substrate and a custom id 7; asserts PhysicalNames section lists three names including "copper", "substrate" and "material_7".

- **msh writer: hex8 node ordering matches gmsh convention** `[msh]` -- writes a 1x1x1 copper grid; asserts the element row ends with corner ids "1 2 4 3 5 6 8 7".

---

## ui (circuitcore::ui camera + meshers)

### pdnkit/tests/camera_test.cpp

- **camera: defaults centered at origin, 1px=1mm** `[camera]` -- builds a default Camera2D; asserts center is (0,0) and `pixels_per_meter` is 1000.

- **camera: screen center maps to world center** `[camera]` -- sets center to (0.05, 0.07); asserts `world_to_screen` of the center returns the viewport midpoint and `screen_to_world` is its inverse.

- **camera: world<->screen round-trip** `[camera]` -- with center (0.01, -0.02) and 5000 px/m, iterates nine screen points; asserts each survives a screen->world->screen round-trip within 1e-9.

- **camera: pan moves center by inverse of drag** `[camera]` -- calls `pan_pixels(100, 50)` at 1000 px/m; asserts the new center is (-0.1, -0.05).

- **camera: zoom_at keeps anchor pixel pointing at same world** `[camera]` -- captures the world point at (250, 175), zooms 2.5x at that anchor; asserts the same screen position still maps to the same world point and pixels_per_meter becomes 2500.

- **camera: fit_to_bounds centers and zooms to fit** `[camera]` -- fits a 100x50mm box into 800x600; asserts center is (0.05, 0.025) and pixels_per_meter is the width-limited 8000.

- **camera: ortho matrix has Y-flip and centers world** `[camera]` -- builds the ortho matrix for an origin-centered camera; asserts m[12]/m[13] are zero and a positive world Y maps to negative NDC Y.

### pdnkit/tests/segment_mesher_test.cpp

- **segments: horizontal segment yields 4 vertices, 2 triangles** `[segments]` -- meshes a 10mm-long, 1mm-wide horizontal segment; asserts one layer mesh with 54 vertices and 50 triangles and the first rim vertex sits at (0, +0.5mm).

- **segments: vertical segment width offset along X** `[segments]` -- meshes a 2mm-wide vertical segment; asserts 54 vertices and the first vertex is at (4mm, 0).

- **segments: zero-length and zero-width are skipped** `[segments]` -- adds one zero-length and one zero-width segment; asserts the mesher returns empty.

- **segments: non-copper layer is skipped** `[segments]` -- adds a segment on F.SilkS (ordinal 32); asserts the mesher returns empty.

- **segments: many segments on same layer aggregate** `[segments]` -- adds five segments on F.Cu; asserts one mesh with 270 vertices and 250 triangles.

- **build_all_meshes: combines zones and segments on same layer** `[segments]` -- mixes one square zone and one segment on F.Cu; asserts a single combined mesh with 58 vertices and 52 triangles.

- **build_all_meshes: zones-only and segments-only layers stay separate** `[segments]` -- puts a zone on F.Cu and a segment on B.Cu; asserts two distinct meshes are produced.

### pdnkit/tests/via_mesher_test.cpp

- **via: 2-layer through-via produces a disk on each copper layer** `[via]` -- builds a through-via with drill 0.4mm spanning F.Cu to B.Cu; asserts three meshes (two copper + one drill pseudo-layer) each with 33 verts and 32 triangles.

- **via: 4-layer through-via covers all intermediate copper** `[via]` -- spans F.Cu, In1, In2 and B.Cu with drill 0; asserts four meshes (no drill pseudo-layer).

- **via: blind via covers only its endpoint range** `[via]` -- adds a blind via from F.Cu to In1 only; asserts two meshes.

- **via: zero outer_diameter is skipped** `[via]` -- adds a via with outer_diameter 0; asserts the mesher returns empty.

- **pad: pads draw a disk on each listed copper layer** `[pad]` -- places a pad covering F.Cu and F.SilkS; asserts only the copper layer mesh is produced.

- **pad: multiple pads on same layer aggregate** `[pad]` -- adds three pads on F.Cu; asserts one mesh with 99 vertices and 96 triangles.

### pdnkit/tests/zone_mesher_test.cpp

- **mesher: empty board produces no meshes** `[mesher]` -- meshes a default Board; asserts the result is empty.

- **mesher: single square zone yields 4 vertices and 2 triangles** `[mesher]` -- meshes one 100x100mm square zone on F.Cu; asserts one mesh with 4 vertices, 2 triangles and all indices below 4.

- **mesher: square with square hole yields 8 vertices, 8 triangles** `[mesher]` -- meshes a square zone with a square hole; asserts 8 vertices and 8 earcut triangles.

- **mesher: two zones on different layers -> two meshes** `[mesher]` -- adds zones on F.Cu and B.Cu; asserts two meshes in insertion order.

- **mesher: two zones on same layer share one mesh** `[mesher]` -- adds two F.Cu zones; asserts one combined mesh with 8 verts and 4 triangles.

- **mesher: non-copper layer is skipped** `[mesher]` -- adds a zone on F.SilkS; asserts the mesher returns empty.

- **mesher: degenerate (< 3 pts) outline is skipped without crashing** `[mesher]` -- meshes a polygon with only two outline points; asserts the result is empty.

### sikit/tests/camera_test.cpp

- **camera: defaults centered at origin, 1px=1mm** `[camera]` -- builds a default Camera2D; asserts center is (0,0) and pixels_per_meter is 1000.

- **camera: screen center maps to world center** `[camera]` -- sets center to (0.05, 0.07); asserts world_to_screen returns the viewport midpoint and screen_to_world inverts it.

- **camera: world<->screen round-trip** `[camera]` -- iterates nine screen points at 5000 px/m; asserts each round-trips within 1e-9.

- **camera: pan moves center by inverse of drag** `[camera]` -- pans 100x50 px at 1000 px/m; asserts center becomes (-0.1, -0.05).

- **camera: zoom_at keeps anchor pixel pointing at same world** `[camera]` -- zooms 2.5x at screen (250, 175); asserts the anchor's world point is preserved and pixels_per_meter is 2500.

- **camera: fit_to_bounds centers and zooms to fit** `[camera]` -- fits 100x50mm into 800x600; asserts center (0.05, 0.025) and width-limited 8000 px/m.

- **camera: ortho matrix has Y-flip and centers world** `[camera]` -- builds the ortho matrix; asserts m[12]/m[13] are zero and positive world Y maps to negative NDC Y.

### sikit/tests/camera3d_test.cpp

- **camera3d: eye_position derives spherical -> Cartesian correctly** `[cam3d]` -- sets azimuth=0, elevation=0, distance=0.10 and a target; asserts `eye_position` returns target + (distance, 0, 0).

- **camera3d: orbit_pixels clamps elevation away from poles** `[cam3d]` -- orbits by 8000 px up and -16000 px down; asserts elevation stays inside (-pi/2, pi/2) but saturates near the caps.

- **camera3d: orbit_pixels wraps azimuth into [-pi, pi]** `[cam3d]` -- orbits 20000 px horizontally; asserts the resulting azimuth lies in [-pi, pi].

- **camera3d: pan_pixels moves target perpendicular to view** `[cam3d]` -- pans 100 px right while looking along +X; asserts target_x is unchanged but target_y moves.

- **camera3d: zoom clamps distance to a sane band** `[cam3d]` -- zooms in 60 times then out 80 times; asserts distance stays in [1e-4, 10.0].

- **camera3d: fit_to_bounds centers target on the box** `[cam3d]` -- fits a 100x50mm box; asserts target lands at (0.05, 0.025, 0.0008) and distance is in (0, 10).

- **camera3d: view_projection has 16 entries and is non-degenerate** `[cam3d]` -- fits a 50x50mm box and builds the view-projection matrix; asserts 16 entries, at least one differs from identity, and all are finite.

- **camera3d: zero widget size returns identity matrix safely** `[cam3d]` -- calls view_projection(0, 0); asserts the diagonal entries are 1.

### sikit/tests/segment_mesher_test.cpp

- **segments: horizontal segment yields 4 vertices, 2 triangles** `[segments]` -- meshes a 10mm horizontal segment; asserts 54 verts, 50 tris and the first vertex sits at (0, +0.5mm).

- **segments: vertical segment width offset along X** `[segments]` -- meshes a 2mm-wide vertical segment; asserts 54 verts and first vertex at (4mm, 0).

- **segments: zero-length and zero-width are skipped** `[segments]` -- adds zero-length and zero-width segments; asserts the mesher returns empty.

- **segments: non-copper layer is skipped** `[segments]` -- adds a segment on F.SilkS; asserts the mesher returns empty.

- **segments: many segments on same layer aggregate** `[segments]` -- adds five F.Cu segments; asserts one mesh with 270 verts and 250 tris.

- **build_all_meshes: combines zones and segments on same layer** `[segments]` -- mixes one zone and one segment on F.Cu; asserts a single 58-vert, 52-tri mesh.

- **build_all_meshes: zones-only and segments-only layers stay separate** `[segments]` -- splits one zone on F.Cu and one segment on B.Cu; asserts two meshes.

### sikit/tests/via_mesher_test.cpp

- **via: 2-layer through-via produces a disk on each copper layer** `[via]` -- meshes a through-via with 0.4mm drill spanning F.Cu/B.Cu; asserts three meshes (two copper + drill) each 33 verts / 32 tris.

- **via: 4-layer through-via covers all intermediate copper** `[via]` -- spans F.Cu, In1, In2, B.Cu with drill 0; asserts four meshes.

- **via: blind via covers only its endpoint range** `[via]` -- adds a blind via F.Cu -> In1; asserts two meshes.

- **via: zero outer_diameter is skipped** `[via]` -- adds a via with zero outer_diameter; asserts the mesher returns empty.

- **pad: pads draw a disk on each listed copper layer** `[pad]` -- places a pad covering F.Cu and F.SilkS; asserts only the copper mesh is produced.

- **pad: multiple pads on same layer aggregate** `[pad]` -- adds three pads on F.Cu; asserts one mesh with 99 verts / 96 tris.

### sikit/tests/zone_mesher_test.cpp

- **mesher: empty board produces no meshes** `[mesher]` -- meshes a default Board; asserts the result is empty.

- **mesher: single square zone yields 4 vertices and 2 triangles** `[mesher]` -- meshes a single square zone on F.Cu; asserts 4 verts, 2 tris and indices below 4.

- **mesher: square with square hole yields 8 vertices, 8 triangles** `[mesher]` -- meshes a square with a square hole; asserts 8 verts and 8 earcut tris.

- **mesher: two zones on different layers -> two meshes** `[mesher]` -- adds zones on F.Cu and B.Cu; asserts two meshes in insertion order.

- **mesher: two zones on same layer share one mesh** `[mesher]` -- adds two F.Cu zones; asserts one merged mesh with 8 verts / 4 tris.

- **mesher: non-copper layer is skipped** `[mesher]` -- adds a zone on F.SilkS; asserts the mesher returns empty.

- **mesher: degenerate (< 3 pts) outline is skipped without crashing** `[mesher]` -- meshes a polygon with two outline points; asserts the result is empty.

---

## sikit (signal integrity)

### sikit/tests/ami_test.cpp

- **ami parser: model name + sections** `[ami]` -- parses a stub AMI definition; asserts model_name is `My_FFE_Model` and both reserved and model_specific sections have 3 entries.

- **ami parser: usage + type per parameter** `[ami]` -- parses the sample; asserts AMI_Version is String/Info, Init_Returns_Impulse is Boolean/Info, Tap1 is Float/In and Mode is Integer.

- **ami parser: ranges parsed (typ, min, max)** `[ami]` -- parses Tap1 with range (0.0 -0.5 0.5); asserts has_range and the typ/min/max values are extracted, plus Tap2 typ=0.1 min=-0.3.

- **ami parser: defaults captured** `[ami]` -- asserts Tap2 default is "0.1" and Mode default is "1".

- **ami parser: description field captured** `[ami]` -- asserts the Mode parameter's description contains "normal".

- **ami parser: missing file throws** `[ami]` -- calls `read_file("/nonexistent/path.ami")`; asserts AmiParseError is thrown.

- **ami loader: stub library round-trips through Init + GetWave** `[ami]` -- searches for `stub_ami.so` next to the test binary, skips if missing; otherwise asserts init/close/get_wave are available, the impulse is scaled by 0.8 by `init`, and the stub adds 0.05 to each sample in `get_wave`.

### sikit/tests/bathtub_test.cpp

- **bathtub: non-trivial waveform produces a non-flat curve** `[bathtub]` -- builds a heavily ISI-ridden PRBS7+RC waveform, computes the bathtub; asserts ber size matches ui_offset, all BER values are in [0,1], some BER is non-zero, and ui_offset is monotonically increasing.

- **bathtub: clean NRZ has near-zero BER across the UI** `[bathtub]` -- builds a clean NRZ waveform and bathtub; asserts every BER value is approximately 0.

- **bathtub: timing_margin_at returns sensible offsets** `[bathtub]` -- computes the bathtub of a mildly filtered PRBS7; asserts margin at BER 10% is >= margin at BER 1% when both are positive.

- **bathtub: fully empty eye returns empty curve** `[bathtub]` -- feeds a 64x64 EyeGrid of zeros; asserts ber is empty and `timing_margin_at(0.5)` returns -1.

- **bathtub: ui_offset covers the full UI** `[bathtub]` -- builds a bathtub from a filtered PRBS7; asserts ui_offset starts below 0.05 and ends above 0.95.

### sikit/tests/bus_group_test.cpp

- **bus: groups DDR_DQ0..3 as a single bus** `[bus]` -- adds four DDR_DQ nets to a board; asserts a single DDR_DQ group with four members.

- **bus: min / max length report the bus extremes** `[bus]` -- adds four DQ nets with varying lengths; asserts min_length_m == 49.5mm, max == 50.3mm and skew == 0.8mm.

- **bus: skew_ps + budget flag set correctly** `[bus]` -- adds two DQ nets with 10mm spread; asserts skew_ps > 50 and `exceeds_budget` is true.

- **bus: single-member group is not reported** `[bus]` -- adds just DDR_DQ0; asserts no group is returned.

- **bus: nets without a trailing integer are skipped** `[bus]` -- adds USB_DP/DN and SCK/MOSI; asserts the groups list is empty.

- **bus: diff pairs collapse to a single member** `[bus]` -- adds three PCIE_TX_LANE differential pairs; asserts the PCIE_TX_LANE group has 3 members all flagged as diff pairs with indices 0/1/2.

- **bus: empty board returns empty result** `[bus]` -- runs `compute_bus_groups` on an empty Board; asserts the result is empty.

- **bus: zero-length skew gives zero ps + no budget flag** `[bus]` -- adds three identical-length DQ nets; asserts skew_m and skew_ps are ~0 and `exceeds_budget` is false.

- **bus: members come back sorted by index** `[bus]` -- adds DQ nets out of order; asserts member indices come back sorted (0, 1, 3, 7).

### sikit/tests/channel_dispersion_test.cpp

- **dispersion: ChannelSpec without model matches v0 behavior** `[disp]` -- synthesises a channel with and without a dispersion_model (none); asserts both produce identical 2x2 S-matrices at 1/5/10 GHz within 1e-12.

- **dispersion: model attached -> results differ from constant epsilon_r** `[disp]` -- synthesises with a Djordjevic-Sarkar model vs constant epsilon_r at 10 GHz; asserts |S21| differs by more than 1e-4.

- **dispersion: at f0 the two paths agree closely** `[disp]` -- evaluates at the model's reference frequency 1 GHz; asserts |S21| matches the constant-model magnitude within 2%.

- **dispersion: |S21| still drops monotonically at higher freq** `[disp]` -- sweeps 1/3/7/15 GHz with the DS model; asserts |S21| is strictly decreasing.

### sikit/tests/channel_synthesis_test.cpp

- **synthesize: zero-length channel is the identity 2-port** `[synth]` -- synthesises a length=0 channel at 1/5/10 GHz; asserts num_ports==2, frequencies.size()==3, |S11|=|S22|=0 and |S12|=|S21|=1.

- **synthesize: matched lossless line has |S21|=1 and S11=0** `[synth]` -- uses tan_delta=0 and sigma_copper=1e30 with Zref=Z0; asserts |S11|=0 and |S21|=1.

- **synthesize: mismatched line has non-zero |S11|** `[synth]` -- synthesises with Zref=75 ohms; asserts at least one frequency has |S11| > 0.01.

- **synthesize: reciprocal -- S12 == S21** `[synth]` -- sweeps four frequencies; asserts S12 == S21 at each.

- **synthesize: phase delay matches v_phase * l** `[synth]` -- runs a lossless 2 GHz synthesis with Zref=Z0; asserts the measured atan2 phase equals the analytic -beta*l wrapped to (-pi, pi].

- **synthesize: throws on invalid geometry** `[synth]` -- sets trace_width=0; asserts `synthesize_channel` throws.

- **synthesize: lossy matched line has |S21| < 1 at high frequency** `[synth]` -- runs a 30cm tan_delta=0.02 channel at 1/5/10 GHz; asserts |S21| < 1 and strictly decreases.

- **synthesize: zero loss reproduces lossless behavior** `[synth]` -- runs a 30cm lossless line at 5 GHz; asserts |S21| == 1 within 1e-6.

- **synthesize: FDM engine produces a similar Z0/loss to closed-form** `[synth][fdm]` -- runs the same spec through ClosedForm and Fdm engines at 1/5 GHz; asserts |S21| values agree within 5%.

- **synthesize: FDM engine returns 2-port with the requested grid** `[synth][fdm]` -- runs the FDM engine at 1/3/7 GHz; asserts num_ports==2 and three S-matrices come back.

### sikit/tests/channel_test.cpp

- **channel: interpolate_s21 at exact grid points** `[channel]` -- builds a 3-point S21 table; asserts the interpolator returns each tabulated value at its exact frequency.

- **channel: interpolate_s21 linear between points** `[channel]` -- linear S21 from 1.0 at 1 GHz to 0 at 2 GHz; asserts the value at 1.5 GHz is 0.5.

- **channel: interpolate_s21 below range is passthrough** `[channel]` -- queries 0 Hz on a 1-2 GHz table; asserts |S21|=1, imag=0 (DC passthrough).

- **channel: interpolate_s21 above range clamps to last value** `[channel]` -- queries 10 GHz on a 1-2 GHz table ending at 0.2; asserts the returned value is 0.2.

- **channel: identity S21 reproduces TX waveform** `[channel]` -- applies an S21=1 channel to a 1 GHz sine at 32 GS/s; asserts the output equals the input within 1e-8.

- **channel: low-pass S21 attenuates a high-frequency tone** `[channel]` -- applies a 1->10 GHz roll-off to 500 MHz and 8 GHz sines; asserts the LF peak stays > 0.8 and the HF peak drops below 0.3.

- **channel: empty input returns empty** `[channel]` -- calls `apply_channel({})`; asserts the result is empty.

- **channel: rejects non-2-port file** `[channel]` -- supplies a 1-port Touchstone; asserts `apply_channel` throws.

### sikit/tests/compliance_test.cpp

- **compliance: every shipped spec carries a usable mask** `[compliance]` -- iterates all_compliance_specs(); asserts non-empty name/family/source, baud_hz>0, ui_seconds==1/baud, mask polygon has >=3 vertices.

- **compliance: name lookup retrieves each shipped spec** `[compliance]` -- looks up every spec by name; asserts the pointer round-trips and a bogus name returns nullptr.

- **compliance: spec names are unique** `[compliance]` -- inserts every spec name into a set; asserts no duplicates.

- **compliance: PCIe generations have strictly increasing baud** `[compliance][pcie]` -- asserts Gen3 < Gen4 < Gen5 baud and Gen6 >= Gen5 (PAM4 stays at Gen5 symbol rate).

- **compliance: PAM4 specs are flagged is_pam4=true** `[compliance][pam4]` -- asserts PCIe Gen6 and 50G-KR are flagged is_pam4 while Gen5, DDR4-3200 and USB 3.1 Gen2 are not.

- **compliance: spec families cover the listed standards** `[compliance]` -- collects unique family strings; asserts PCIe, DDR, USB, HDMI and Ethernet are all represented.

- **compliance: mask polygons are non-degenerate** `[compliance]` -- iterates every spec; asserts consecutive vertices differ and shoelace area > 1e-6.

- **compliance: every mask centre is at the eye-centre keep-out** `[compliance]` -- asserts (0.5 UI, 0 V) lies inside every shipped mask polygon.

- **compliance: a clean centred eye passes every NRZ spec** `[compliance]` -- builds a synthetic +-1V eye with no mid-voltage transitions; asserts `passes()` is true for every non-PAM4 spec.

### sikit/tests/connector_test.cpp

- **connector: SMA edge launch generates a low-loss 2-port** `[connector]` -- generates a 64-point log sweep from 100 MHz to 30 GHz; asserts |S21| > 0.95 at 1 GHz, > 0.70 at 30 GHz, and monotonically decreasing.

- **connector: 2-port output is reciprocal and symmetric** `[connector]` -- uses the SMA panel-mount preset; asserts S11==S22 and S21==S12 at every frequency.

- **connector: USB-C diff connector emits a 4-port file** `[connector]` -- generates a USB-C diff pair sweep; asserts num_ports==4, 16 entries per matrix, and the P_near->P_far through |S31| > 0.9 at low freq.

- **connector: USB-C notch produces a measurable dip near 18 GHz** `[connector]` -- finds the minimum |S31| across a 128-point sweep; asserts the notch frequency is between 10 and 25 GHz.

- **connector: preset registry is consistent** `[connector]` -- iterates `available_connector_presets()`; asserts each preset name round-trips, num_ports >= 2 and il_slope > 0.

- **connector: preset_by_name rejects unknown names** `[connector]` -- queries a bogus name; asserts an exception is thrown.

- **connector: rejects degenerate frequency input** `[connector]` -- supplies empty, [0.0] and decreasing frequency lists; asserts all three throw.

- **connector: harsher RL preset produces louder |S11|** `[connector]` -- compares edge-launch (RL 25 dB) and panel-mount (RL 20 dB); asserts panel-mount's |S11| is larger.

### sikit/tests/crosstalk_test.cpp

- **crosstalk: zero-aggressor scenario reproduces clean victim eye** `[crosstalk]` -- simulates a 5 Gbaud eye with no aggressors and a passthrough victim; asserts the eye has height > 1.5 V and width > 0.5 UI.

- **crosstalk: a single coupled aggressor closes the eye more than no aggressor** `[crosstalk]` -- compares clean vs one aggressor at 0.35 coupling; asserts the noisy eye is shorter.

- **crosstalk: stronger coupling closes the eye further** `[crosstalk]` -- compares 0.10 and 0.45 couplings; asserts the heavy-coupling eye is shorter.

- **crosstalk: zero-coupling aggressor contributes nothing** `[crosstalk]` -- compares scenario with one 0-coupling aggressor vs none; asserts heights match within 0.05 V.

- **crosstalk: three aggressors close the eye more than one** `[crosstalk]` -- one 0.20-coupling aggressor vs three; asserts the three-aggressor eye is shorter.

- **crosstalk: rejects invalid input** `[crosstalk]` -- feeds zero baud, zero bits, samples_per_ui=1, an empty victim and a wrong-port-count aggressor; asserts each throws.

- **crosstalk: diff_pair_to_scenario extracts the right slices for PNPN** `[crosstalk]` -- builds a 4-port with index-encoded entries; asserts the PNPN extraction picks S31=20 for the through and S(P_far<-N_near)=21 for the coupling.

- **crosstalk: diff_pair_to_scenario rejects non-4-port** `[crosstalk]` -- supplies a 2-port file; asserts the call throws.

### sikit/tests/deembed_test.cpp

- **deembed: passthrough fixture leaves the measurement unchanged** `[deembed]` -- de-embeds with identity fixtures on both sides; asserts the recovered DUT matches the input.

- **deembed: embed-then-deembed round-trip recovers the DUT** `[deembed]` -- cascades L*DUT*R, de-embeds; asserts the recovered DUT matches within 1e-6.

- **deembed_symmetric: same fixture on both sides round-trips** `[deembed]` -- cascades F*DUT*F and runs `deembed_symmetric`; asserts the recovered DUT matches within 1e-6.

- **deembed: frequency grid mismatch is rejected** `[deembed]` -- shifts the last frequency point on the right-side fixture; asserts SParamError is thrown.

- **deembed: non-2-port input is rejected** `[deembed]` -- supplies a 4-port measurement; asserts SParamError is thrown.

- **invert_t: T * T^-1 ~= identity for a passthrough** `[deembed][t]` -- inverts a non-trivial T matrix; asserts T*Tinv equals identity within 1e-9.

- **invert_t: singular T throws** `[deembed][t]` -- inverts a T with a zero column; asserts SParamError is thrown.

### sikit/tests/diff_pair_test.cpp

- **diff: _P/_N pair detected** `[diffpair]` -- adds USB_DP_P and USB_DP_N to a board; asserts one pair with base USB_DP and suffix style "_P/_N".

- **diff: +/- suffix style** `[diffpair]` -- adds D+ and D-; asserts one pair with base D and suffix "+/-".

- **diff: _POS/_NEG suffix style** `[diffpair]` -- adds RX_POS and RX_NEG; asserts one pair with suffix "_POS/_NEG".

- **diff: DP/DM suffix style (no separator)** `[diffpair]` -- adds USBDP and USBDM; asserts one pair with base USB and suffix "DP/DM".

- **diff: multiple pairs in one board** `[diffpair]` -- adds USB_DP/DM plus PCIE_TX_P/N and PCIE_RX_P/N; asserts three pairs are found.

- **diff: unpaired _P net is not reported** `[diffpair]` -- adds only FOO_P with no FOO_N; asserts the pair list is empty.

- **diff: empty net name skipped** `[diffpair]` -- adds an empty-name net plus FOO_P/FOO_N; asserts one pair is reported.

- **diff: prefer more-specific suffix (_POS over P)** `[diffpair]` -- adds CLK_POS/NEG; asserts the suffix is the explicit "_POS/_NEG" not bare P/N.

- **highspeed keyword: protocol names match** `[diffpair]` -- queries `looks_high_speed` on USB_DP, PCIE_TX_P, DDR4_DQ0, hdmi_clk, MIPI_CSI_D0_P, LVDS_RX0+ and SATA_RX_P; asserts each returns true.

- **highspeed keyword: non-matches** `[diffpair]` -- queries `looks_high_speed` on GND, VCC, +3V3, RESET_N and ""; asserts each returns false.

- **find_high_speed_nets includes diff-pair members and keyword hits** `[diffpair]` -- adds USB diff pair, GND, VCC and PCIE_REFCLK; asserts the returned net id list is {1, 2, 5}.

### sikit/tests/diff_synth_test.cpp

- **diff synth: returns a 4-port file with the requested freq grid** `[diffsynth]` -- synthesises a 5cm diff pair at 1/2/5 GHz; asserts num_ports==4, three frequencies and 16 entries per matrix.

- **diff synth: matrix is symmetric (reciprocal lossless network)** `[diffsynth]` -- synthesises at 1/3 GHz; asserts S_ij == S_ji for every off-diagonal entry within 1e-9.

- **diff synth: zero-length channel is the identity 4-port** `[diffsynth]` -- runs length=1e-12 at 1 GHz; asserts |Sii| < 0.01 and same-trace through |S21|, |S43| ~= 1.

- **diff synth: cross-trace coupling falls off with spacing** `[diffsynth]` -- compares 0.10mm and 1.0mm spacings at 2 GHz; asserts |S13| is larger for the closer pair.

- **diff synth: rejects bad geometry** `[diffsynth]` -- sets trace_width=0 then length=-1; asserts both throw.

### sikit/tests/djordjevic_sarkar_test.cpp

- **DS: from_reference reproduces inputs at f0** `[ds]` -- builds model from FR-4 (epsilon_r=4.5, tan_delta=0.02) at 1 GHz; asserts the model returns those exact values at f0.

- **DS: epsilon_r decreases monotonically with frequency in-band** `[ds]` -- evaluates epsilon_r at 1e7..1e11 Hz; asserts each value is strictly less than the previous.

- **DS: epsilon'' is positive (lossy material) across band** `[ds]` -- evaluates complex epsilon at 1e7..1e11 Hz; asserts the imaginary part is negative (since epsilon = e' - j e'').

- **DS: tan delta stays in a physically plausible range** `[ds]` -- evaluates tan_delta across 1e7..1e11 Hz; asserts each value is between 0 and 0.5.

- **DS: high tan_delta gives larger dispersion magnitude** `[ds]` -- compares FR-4 (tan=0.02) vs Rogers (tan=0.001); asserts the FR-4 epsilon_r swing across the band is larger and positive.

- **DS: rejects bad construction parameters** `[ds]` -- constructs with f1=0, f1>=f2, eps_r=-1 and f0=-1; asserts each throws.

- **DS: very low loss approaches lossless behavior** `[ds]` -- builds with tan_delta=0; asserts epsilon_r is 4.4 at 1e6 and 1e10 and tan_delta(1e9) is 0.

### sikit/tests/em2d_test.cpp

- **em2d: epsilon_r_at returns the correct layer value** `[em2d]` -- builds a 1mm FR-4 + 0.5mm Rogers stack; asserts air above/below returns 1.0 and mid-layer returns the matching epsilon_r.

- **em2d: parallel-plate capacitor recovers analytic C ~= eps0*er*W/d** `[em2d]` -- solves a wide-plate cap with W=4mm d=0.2mm at 50um cells; asserts the FDM charge per length is between 0.95x and 2x the analytic kEps0*W/d.

- **em2d: microstrip Z0 within 15% of Wadell closed-form** `[em2d]` -- solves a canonical 2.8mm/1.524mm FR-4 microstrip at 100um cells; asserts Z0 == 50 +/- 8 ohms and eps_eff between 2.0 and 4.4.

### sikit/tests/example_test.cpp

- **example: demo_board.kicad_pcb parses cleanly** `[example]` -- parses the bundled demo_board.kicad_pcb; asserts >=10 nets and 5 stackup layers.

- **example: demo_board carries the expected nets** `[example]` -- looks up GND, VCC, USB_DP, USB_DN, DDR_DQ0..3 and BAD_NET; asserts every net is found.

- **example: USB diff pair skew matches the walkthrough** `[example]` -- runs `compute_diff_pair_skews` with 5 ps budget; asserts one pair with skew_m == -0.4mm that does not exceed the budget.

- **example: DDR_DQ bus skew fails the 10 ps default budget** `[example]` -- runs `compute_bus_groups` with 10 ps budget; asserts one DDR_DQ group with four members, length range 39..43mm and `exceeds_budget` true.

- **example: BAD_NET is the only return-path violation** `[example]` -- runs `detect_return_path_violations`; asserts the top entry is BAD_NET with off_plane_fraction 1.0 and reference_layer 1.

### sikit/tests/eye_mask_test.cpp

- **eye_mask: point-in-polygon ray cast basics** `[mask]` -- tests `point_in_polygon` on the unit square; asserts inside returns true and three outside points return false.

- **eye_mask: available masks listed** `[mask]` -- queries `available_mask_names()`; asserts >=2 names, the first round-trips through `mask_by_name` and a bogus name returns nullptr.

- **eye_mask: clean NRZ passes the USB-style mask** `[mask]` -- builds a clean NRZ eye; asserts `count_violations` is 0 against `usb20_hs_template1` and `passes` is true.

- **eye_mask: heavy ISI fails the centered-opening mask** `[mask]` -- builds a heavily filtered NRZ eye; asserts violations > 0 and `passes` is false.

- **eye_mask: empty eye reports zero violations** `[mask]` -- supplies a 128x96 zeroed eye; asserts violations == 0 and `passes` is true.

- **eye_mask: degenerate polygon never reports inside** `[mask]` -- runs ray cast on a 2-point line; asserts the point is outside.

### sikit/tests/eye_metrics_test.cpp

- **metrics: clean unfiltered NRZ has a wide-open eye** `[metrics]` -- measures a clean NRZ eye; asserts height_v > 1.5, width_ui == 1.0 and jitter_pp_ui == 0.

- **metrics: heavy ISI closes the eye** `[metrics]` -- measures a heavily filtered eye; asserts height_v < 1.0.

- **metrics: bandwidth-limited ramped NRZ shows measurable jitter** `[metrics]` -- measures a 20% UI ramp NRZ eye; asserts height > 0.5, jitter > 0 and width in (0, 1).

- **metrics: empty eye returns zeros** `[metrics]` -- measures a 32x32 zeroed eye; asserts height/width/jitter are all 0.

- **metrics: threshold is midpoint of v_min/v_max** `[metrics]` -- measures a tiny NRZ pattern; asserts v_threshold == 0.5*(v_max+v_min).

### sikit/tests/eye_ramp_test.cpp

- **ramp: zero fraction reproduces step NRZ** `[ramp]` -- generates `nrz_waveform` and `nrz_with_ramp(...,0.0)` on the same bits; asserts the two outputs are identical.

- **ramp: transitions are linear across the ramp window** `[ramp]` -- generates a {0,1} ramp with 40% UI rise; asserts the four interpolated samples take values -0.5, 0.0, +0.5, +1.0.

- **ramp: identical bits produce a flat trace** `[ramp]` -- ramps {1,1,1}; asserts every sample equals 1.0.

- **ramp: fraction > 1 is clamped to 1** `[ramp]` -- requests 500% rise; asserts the waveform still has the expected sample count and the call doesn't fail.

- **ramp_fraction_from_ibis: dt_rise / UI** `[ramp]` -- supplies an IBIS model with 100 ps rise; asserts fraction at 1 GHz is 0.1 and at 10 GHz is clamped to 1.0.

- **ramp_fraction_from_ibis: empty model returns 0** `[ramp]` -- asserts a model with no ramp data returns fraction 0.

- **ramp_fraction_from_ibis: zero baud returns 0** `[ramp]` -- queries fraction at baud=0; asserts the function returns 0.

### sikit/tests/eye_test.cpp

- **prbs7: period of 127 and contains both bits** `[eye]` -- generates 254 bits; asserts the first 127 bits equal the next 127 and both 0 and 1 appear.

- **nrz_waveform: each bit becomes samples_per_ui samples at +/-1** `[eye]` -- generates a 4-bit, 4-spu waveform; asserts each sample matches its bit at +/-1.

- **rc_lowpass: DC passes; sufficiently high frequency attenuated** `[eye]` -- low-passes a DC and a Nyquist square wave; asserts DC == 1 in steady state and the HF max settles below 0.5.

- **eye: clean NRZ has wide-open eye, two horizontal bands** `[eye]` -- builds an eye on clean NRZ; asserts max_count > 0, the mid-voltage row is empty and top/bottom rows have non-zero counts.

- **eye: heavily filtered channel closes the eye middle** `[eye]` -- builds an eye on a heavily filtered NRZ; asserts the middle voltage row gains non-zero counts.

- **eye: build with empty input returns empty grid** `[eye]` -- builds with an empty waveform; asserts max_count is 0 but the grid retains its dimensions.

- **eye: warmup_uis skips initial transient samples** `[eye]` -- compares warmup=0 vs warmup=4 on a filtered waveform; asserts the warmup eye has fewer or equal max_count.

### sikit/tests/fdm_refined_test.cpp

- **refined: produces a finite result + reports both solves** `[refined]` -- runs `compute_z0_refined` at 100um cells on a 50-ohm microstrip; asserts r.ok, both iter counts > 0, both z0 values in (40, 70) and Richardson result also in (40, 70).

- **refined: 2h solve converges in fewer iterations than h** `[refined]` -- runs `compute_z0_refined`; asserts iter_coarse <= iter_fine.

- **refined: result varies smoothly with cell size** `[refined]` -- runs at 150um and 100um cells; asserts the two Z0 results differ by less than 10 ohms.

### sikit/tests/fdtd3d_test.cpp

- **fdtd3d: CFL dt formula matches the closed-form bound** `[fdtd3d]` -- builds an isotropic 1mm grid; asserts `cfl_dt(g)` equals the analytic `0.99/(c*sqrt(inv2))` within 1e-12 and the anisotropic dt is >= the smallest-cell isotropic case.

- **fdtd3d: Field3D round-trips reads and writes** `[fdtd3d]` -- writes three values into a 4x3x2 Field3D, fills with zero; asserts size==24, written cells read back and post-fill is zero.

- **fdtd3d: gaussian pulse peaks at t0** `[fdtd3d]` -- evaluates `gaussian_pulse` at t0, t0+s and 0; asserts peak == 1, one-sigma == exp(-1), and t=0 is < 1e-10.

- **fdtd3d: cavity_mode_freq matches Pozar formula** `[fdtd3d]` -- computes TE_101 for a 5x4x3 cm cavity; asserts the value matches Pozar's `c/2*sqrt(1/a^2 + 1/d^2)` within 1e-12.

- **fdtd3d: solver constructs and reports memory footprint** `[fdtd3d]` -- builds an 8^3 solver; asserts grid().nx==8, step_count==0 and bytes() lies between 6*8^3 and 6*16^3 doubles.

- **fdtd3d: step without dt set throws** `[fdtd3d]` -- calls `step` without `set_dt`; asserts an exception is thrown.

- **fdtd3d: PEC walls keep tangential E pinned at zero** `[fdtd3d]` -- drives a Gaussian Ez source at the center of an 8^3 grid; asserts Ez at every i=0/nx and j=0/ny edge stays exactly 0 after 100 steps.

- **fdtd3d: causal propagation -- probe is quiet before wave-front arrives, lit up after** `[fdtd3d]` -- drives an Ez step source at the center of a 24^3 grid, probes 10 cells in +x; asserts the probe is exactly 0 before the causal step count and > 1e-9 after.

- **fdtd3d: Mur ABC absorbs the wave -- much smaller residual energy than the PEC reflector** `[fdtd3d][mur]` -- pulses a 30^3 grid with PEC then Mur boundaries; asserts the volume's Ez^2 sum after 220 steps is < 20% of the PEC case.

- **fdtd3d: Mur ABC remains numerically stable for a long run** `[fdtd3d][mur]` -- runs 500 steps with no source and Mur boundary; asserts max|Ez| stays exactly 0.

- **fdtd3d: switching boundary mode is observable** `[fdtd3d][mur]` -- asserts the default boundary is PEC and `set_boundary(Mur1stOrder)` changes the getter.

- **fdtd3d: default material is vacuum** `[fdtd3d][material]` -- builds a 4^3 solver; asserts every epsr component returns 1.0 and sigma_x returns 0.

- **fdtd3d: set_uniform_material fills every E component** `[fdtd3d][material]` -- calls `set_uniform_material(4.2, 0.07)`; asserts epsr_x/y/z all read 4.2 and sigma_x is 0.07.

- **fdtd3d: set_material_box scopes the change to inside the box** `[fdtd3d][material]` -- sets a 2..5 cube to epsilon_r=9.8; asserts epsr_z is 1.0 outside and 9.8 inside.

- **fdtd3d: causality is slower inside a dielectric** `[fdtd3d][material]` -- compares vacuum vs epsilon_r=4 fills; asserts probe at step 25 is lit in vacuum but < 1% of vacuum in the dielectric.

- **fdtd3d: conductivity dissipates stored energy** `[fdtd3d][material]` -- compares sigma=0 and sigma=5 Gaussian-pulsed solves; asserts the lossy box's Ez^2 sum is < 60% of the lossless case.

### sikit/tests/fdtd_port_test.cpp

- **fdtd-port: gaussian-modulated sinusoid is zero at t=t0** `[fdtd-port]` -- asserts `gaussian_modulated_sinusoid(t0, t0, ..., 5 GHz)` returns 0 and the quarter-period offset is > 0.5.

- **fdtd-port: make_gms_drive produces the right length** `[fdtd-port]` -- builds a 128-sample drive; asserts size==128 and sample 0 differs from sample 64.

- **fdtd-port: soft source is additive (interior reads the SUM of curl update + soft drive)** `[fdtd-port]` -- runs a 12^3 grid with hard vs soft Ez sources; asserts after 40 steps the two source cells hold different values.

- **fdtd-port: well-matched (vacuum + Mur) port has small S11** `[fdtd-port]` -- runs two identical port solves in a Mur-bounded vacuum; asserts |S11| < 1e-9 at every 1..7 GHz step (identical runs yield exact zero).

- **fdtd-port: PEC barrier near the port produces a large |S11| compared to the unblocked case** `[fdtd-port]` -- adds a PEC slab 8 cells from the port and compares to an unblocked solve; asserts |S11| at the centre frequency is > 1e-5.

### sikit/tests/fdtd_rasterize_test.cpp

- **rasterize: default mapping records copper-layer Zs** `[fdtd-raster]` -- builds a synthetic 2-layer board; asserts F.Cu maps to z=0 and B.Cu falls between 1.0 and 1.1 mm.

- **rasterize: a segment writes PEC cells on its z-slab** `[fdtd-raster]` -- rasterizes one 10mm segment; asserts >0 cells written, the F.Cu slab cell (25,25,0) holds PEC and a higher k cell holds none.

- **rasterize: a via writes PEC through the stackup z range** `[fdtd-raster]` -- rasterizes one through-via; asserts the via cell has PEC at both k=0 and k=5.

- **rasterize: a filled zone marks every interior cell** `[fdtd-raster]` -- rasterizes a 12x10mm zone at 0.2mm grid; asserts >1000 PEC cells written.

- **rasterize: rasterize_board reports nonzero counts on a non-trivial board** `[fdtd-raster]` -- rasterizes the full synthetic board; asserts 1 segment/1 via/1 zone processed with non-zero PEC counts and non-zero substrate cells.

- **rasterize: pec_cell_count tallies the same order as the rasteriser reports** `[fdtd-raster]` -- compares the solver's pec_cell_count to the reporter's sum; asserts the solver count is >= the reporter sum.

### sikit/tests/fft_test.cpp

- **fft: next_power_of_2 basics** `[fft]` -- evaluates the helper on 0, 1, 2, 3, 1000, 1024, 1025; asserts each returns the expected next power of two.

- **fft: rejects non-power-of-2 size** `[fft]` -- calls `fft` on a 6-element vector; asserts it throws.

- **fft: trivial sizes are no-ops** `[fft]` -- runs `fft` on size 0 and size 1; asserts size 0 stays empty and size 1 is unchanged.

- **fft: impulse at index 0 maps to flat unit spectrum** `[fft]` -- transforms [1,0,...,0] of size 8; asserts every output bin is 1+0j.

- **fft: constant signal maps to spike at DC bin** `[fft]` -- transforms a constant-1 size-8 vector; asserts X[0] == 8 and all other bins are below 1e-10.

- **fft: forward then inverse round-trips with 1/N normalization** `[fft]` -- runs forward then inverse on a 64-point mixed signal; asserts each sample recovers the original within 1e-10.

- **fft: pure sinusoid produces a single spectral peak** `[fft]` -- transforms a k=17 cosine over 1024 points; asserts |X[k]| and |X[N-k]| are N/2 and all other bins are below 1e-8.

- **fft: linearity holds (sum of inputs -> sum of spectra)** `[fft]` -- transforms a, b and a+b separately; asserts F(a+b) == F(a)+F(b) within 1e-10.

### sikit/tests/headless_test.cpp

- **cli: impedance_op succeeds on a routed net** `[cli][impedance]` -- runs `impedance_op` on a 50mm DATA segment; asserts rc==0 and stdout contains Z0, v_phase and eps_eff.

- **cli: impedance_op reports missing net** `[cli][impedance]` -- runs `impedance_op` on a bogus net name; asserts rc==3.

- **cli: impedance_op reports missing layer** `[cli][impedance]` -- runs `impedance_op` on layer In7.Cu; asserts rc==4.

- **cli: impedance_op flags a net with no segments on the layer** `[cli][impedance]` -- runs DATA on B.Cu (no segments there); asserts rc==5.

- **cli: touchstone_op writes a .s2p file with expected metadata** `[cli][touchstone]` -- runs `touchstone_op` from 10 MHz to 20 GHz, 100 points; asserts rc==0, the file exists and is non-empty.

- **cli: spice_op writes a .cir file with the required directives** `[cli][spice]` -- runs `spice_op` with 8 poles, 200 points; asserts rc==0 and the output contains .subckt and .ends.

- **cli: list_nets_op prints the net table** `[cli][list]` -- runs `list_nets_op`; asserts rc==0 and the output contains "DATA", "id" and "name".

- **cli: list_specs_op prints every shipped compliance spec** `[cli][list]` -- runs `list_specs_op`; asserts rc==0 and output contains "PCIe Gen5", "DDR4", "USB" and "Ethernet".

- **cli: compliance_op reports missing spec by name** `[cli][compliance]` -- runs `compliance_op` with a minimal touchstone and a bogus spec name; asserts rc==3.

- **cli: compliance_op walks a real spec for a real touchstone** `[cli][compliance]` -- writes a four-point .s2p, runs `compliance_op` against "PCIe Gen3"; asserts rc==0 and output contains "PCIe Gen3", "baud" and "ber".

### sikit/tests/ibis_test.cpp

- **ibis: top-level fields parsed** `[ibis]` -- parses the kSample IBIS file; asserts version=="5.0", component=="DRV01", manufacturer=="Acme" and 2 models.

- **ibis: model type and C_comp parsed** `[ibis]` -- parses DRAM_OUT and DRAM_IN; asserts Output/Input types and c_comp values 3.0/2.5/3.5 pF and 1.5 pF.

- **ibis: pulldown V/I table parsed** `[ibis]` -- parses the pulldown section; asserts 4 entries with voltage -1.0/i_typ -0.020 and voltage 1.0/i_min 0.045/i_max 0.055.

- **ibis: pullup V/I table parsed** `[ibis]` -- parses the pullup section; asserts 3 entries with the first at v=0/i_typ=-0.100.

- **ibis: ramp dV/dt parsed** `[ibis]` -- parses the [Ramp] section; asserts dv_rise/dt_rise typ/min/max and dv_fall.typ all read 2.0/200ps style values.

- **ibis: NA tokens parse as NaN** `[ibis]` -- parses a snippet with NA in min/max columns; asserts c_comp.min/max and pulldown[0].i_min are NaN.

- **ibis: unit suffixes (p/n/u/m/k/M/G)** `[ibis]` -- parses C_comp 5.0p 3n 2u; asserts values are 5e-12, 3e-9, 2e-6.

- **ibis: model_type aliases recognized** `[ibis]` -- parses a model list with 3-state, I/O, Open_drain and Terminator; asserts the enum maps to Tristate/IO/OpenDrain/Terminator.

- **ibis: pipe comments stripped** `[ibis]` -- parses a sample with inline and full-line `|` comments; asserts component is still "x" and c_comp.typ == 1pF.

- **ibis: missing file throws** `[ibis]` -- calls `read_file` on a bogus path; asserts IbisParseError is thrown.

### sikit/tests/impedance_test.cpp

- **microstrip: ~50 ohm at standard FR-4 geometry** `[impedance]` -- evaluates `microstrip_z0` for W=2.8mm, H=1.524mm, T=35um, er=4.4; asserts Z0 == 50 +/- 2 and `microstrip_in_valid_range` is true.

- **microstrip: narrower trace -> higher impedance** `[impedance]` -- compares 1.5mm vs 3.0mm widths; asserts the narrower trace has higher Z0.

- **microstrip: higher epsilon_r -> lower impedance** `[impedance]` -- compares FR-4 (er=4.4) vs Rogers (er=3.0); asserts Rogers has higher Z0.

- **stripline: known geometry produces ~50 ohm** `[impedance]` -- evaluates `stripline_z0` for W=0.2mm, B=0.6mm; asserts Z0 == 50 +/- 5.

- **stripline: validity check rejects too-wide trace** `[impedance]` -- builds params with W/B-T > 0.35; asserts `stripline_in_valid_range` is false.

- **diff microstrip: 100 ohm at typical USB-style geometry** `[impedance]` -- evaluates `edge_coupled_microstrip_diff(50, s, h)`; asserts result in (60, 100), closer spacing gives lower z, and far spacing -> 2*Z0 == 100 within 1 ohm.

- **diff stripline: monotonicity vs spacing** `[impedance]` -- evaluates stripline diff at three spacings; asserts close < med < far and far -> 100 within 1 ohm.

- **microstrip: zero/negative inputs throw** `[impedance]` -- supplies zero width, zero height, negative thickness and er<1; asserts each throws ImpedanceError.

- **stripline: zero/negative inputs throw** `[impedance]` -- supplies zero width and zero spacing; asserts both throw.

- **diff: zero-spacing limit is below 2*Z0** `[impedance]` -- evaluates with s=0; asserts microstrip == 52.0 and stripline == 65.3 (i.e. analytically reduced formulas).

### sikit/tests/mesher3d_test.cpp

- **mesher3d: empty board yields empty meshes** `[m3d]` -- meshes an empty 2-layer board; asserts copper, vias and dielectric meshes are all empty.

- **mesher3d: a single segment produces a copper box (24 verts, 36 idx)** `[m3d]` -- meshes one segment; asserts 240 floats (24 verts * 10) and 36 indices in copper, plus a non-empty synthesised dielectric.

- **mesher3d: a single via produces a cylinder mesh** `[m3d]` -- meshes one through-via; asserts 146 cylinder vertices (24-sided wall + two 25-vert caps).

- **mesher3d: explicit 4-layer stackup -> multiple dielectric slabs** `[m3d]` -- meshes a 4-layer board with three dielectric items; asserts dielectric has 720 floats and 3*36 indices.

- **mesher3d: synthesized stackup fallback when items empty** `[m3d]` -- meshes a board with empty SiStackup; asserts a single 36-index dielectric slab is synthesised and copper is non-empty.

- **mesher3d: segments on non-existent layers are skipped** `[m3d]` -- adds segments on layer 0 and layer 99; asserts only 36 copper indices (one segment meshed).

- **mesher3d: copper vertices carry non-zero normals** `[m3d]` -- meshes one segment; asserts every vertex's normal squared length is approximately 1.

- **mesher3d: component with courtyard becomes an extruded body** `[m3d][components]` -- adds an SOIC-8 component with courtyard 0..10mm; asserts components mesh has 240 floats and 36 indices.

- **mesher3d: component without courtyard falls back to pad bbox** `[m3d][components]` -- adds R_0402 with zero courtyard and one pad; asserts components mesh has 240 floats.

- **mesher3d: bottom-side component extrudes downward** `[m3d][components]` -- adds a QFN-32 with pads on B.Cu; asserts every component vertex z is <= 0.

### sikit/tests/overlay_test.cpp

- **overlay: identical files give zero delta** `[overlay]` -- compares two identical flat-S21 files; asserts every delta_db is < 1e-9 and max_abs_db is < 1e-9.

- **overlay: 2x amplitude difference is 6 dB** `[overlay]` -- compares 0.5 vs 0.25 |S21|; asserts every delta_db is ~6.02 dB.

- **overlay: negative delta when a is quieter than b** `[overlay]` -- compares 0.25 vs 0.5; asserts delta_db[0] is -6.02 and max_abs_db is +6.02.

- **overlay: locates the worst-case frequency index** `[overlay]` -- injects a single 0.05 magnitude point at index 7; asserts max_index==7, max_freq_hz==8 GHz and max_abs_db ~ 20.

- **overlay: rejects port-count mismatch** `[overlay]` -- sets b.num_ports=4; asserts OverlayError is thrown.

- **overlay: rejects frequency-grid mismatch** `[overlay]` -- shifts one b.frequencies entry by 1 MHz; asserts OverlayError is thrown.

- **overlay: rejects out-of-range s_param_index** `[overlay]` -- calls with index 9 and -1; asserts both throw.

### sikit/tests/project_test.cpp

- **project: round-trip a minimal project** `[project]` -- serialises a Project with just kicad_pcb; asserts the parsed version==1, kicad_pcb matches and IBIS/AMI/observed_nets are absent.

- **project: round-trip with IBIS, AMI, observed nets, FDM on** `[project]` -- serialises a full Project; asserts every field round-trips including three observed_nets and use_fdm==true.

- **project: rejects non-sikit S-expression** `[project]` -- parses "(some-other-format ...)" and garbage; asserts both throw ProjectIoError.

- **project: file save/load round-trip** `[project]` -- saves to a temp file and loads; asserts kicad_pcb and use_fdm round-trip.

- **project: tolerates missing optional sections** `[project]` -- parses a minimal sikit-project form; asserts kicad_pcb is set and optional fields are empty.

- **project: bool field accepts true/yes/on/1** `[project]` -- parses each truthy spelling; asserts use_fdm is true, plus "false" returns false.

### sikit/tests/reference_data_test.cpp

- **ref-data: microstrip canonical geometries (IPC-2141A / Polar)** `[ref]` -- runs four canonical microstrips (50/75 ohm on FR-4 and HDI); asserts each computed Z0 is within the per-case tolerance band of the published reference.

- **ref-data: stripline canonical geometries** `[ref]` -- runs two canonical striplines (B=0.6mm and B=0.4mm); asserts each Z0 is within 10-15% of 50.

- **ref-data: differential microstrip Z_diff** `[ref]` -- runs 4/4/4 mil and 4/6/4 mil edge-coupled diff pairs; asserts the first is within 15% of 100 ohms and the second falls in (80, 110).

- **ref-data: FDM agrees with closed-form within budget** `[ref]` -- compares closed-form and FDM on a textbook 50-ohm microstrip; asserts both fall in (42, 62) and agree within 20%.

- **ref-data: trace impedance monotonicity sanity** `[ref]` -- asserts narrower trace > wider trace, thicker dielectric > thinner, and lower epsilon_r > higher epsilon_r for Z0.

### sikit/tests/report_test.cpp

- **report: build_board_report on the demo board** `[report]` -- runs the report on the bundled demo board; asserts >=10 nets, 4 copper layers, non-empty nets, 1 diff_pair, 1 bus and non-empty return-path violations.

- **report: overall_pass reflects the per-section flags** `[report]` -- builds the report; asserts `overall_pass()` is false and both bus_fails and return_path_fails are > 0.

- **report: render_html emits valid HTML scaffolding** `[report]` -- renders the HTML; asserts the doctype, title, all four section headers and an "Issues found" banner are present.

- **report: HTML-escapes net names that contain markup chars** `[report]` -- injects an "EVIL<NET>&" net name; asserts the rendered HTML contains the escaped form and not the raw angle brackets.

- **report: empty board produces a valid pass-banner report** `[report]` -- builds the report on an empty board; asserts `overall_pass()` and the HTML contains "All checks PASS".

### sikit/tests/return_path_test.cpp

- **return-path: trace fully over a covering plane has no violations** `[returnpath]` -- 50mm trace over a 100x20mm GND zone on In1.Cu; asserts the violations list is empty.

- **return-path: board with no reference plane flags the segment** `[returnpath]` -- omits the GND zone; asserts one violation with off_plane_fraction==1.0 and reference_layer==1.

- **return-path: trace half off the plane reports ~0.5 fraction** `[returnpath]` -- 100mm trace over a 0..50mm GND zone; asserts off_plane_fraction in (0.3, 0.7) and severity_m == fraction*length.

- **return-path: threshold filters out near-clean traces** `[returnpath]` -- 50mm trace over a 0..49.9mm zone with default 5% then 10% threshold; asserts the loose-threshold run returns empty.

- **return-path: violations sorted worst-first** `[returnpath]` -- adds a half-off trace plus a fully-off trace; asserts two violations with the fully-off one sorted first.

- **return-path: non-routable nets are skipped** `[returnpath]` -- sets segment net_id to 0; asserts the violations list is empty.

- **return-path: single-copper-layer board reports the segment as lacking any reference** `[returnpath]` -- erases the In1.Cu layer; asserts one violation with reference_layer==-1 and off_plane_fraction==1.

- **return-path: zero-length segment is skipped** `[returnpath]` -- adds a zero-length segment; asserts the violations list is empty.

### sikit/tests/rlgc_test.cpp

- **rlgc: 2-conductor microstrip -- C is positive on diagonal, negative off** `[rlgc]` -- runs `compute_rlgc` on two coupled microstrips; asserts r.ok, n==2, diagonal C > 0, off-diagonal C < 0 and |off| < diagonal.

- **rlgc: C matrix is approximately symmetric** `[rlgc]` -- runs `compute_rlgc`; asserts |C(0,1) - C(1,0)| / avg < 10%.

- **rlgc: L self-inductance is positive** `[rlgc]` -- runs `compute_rlgc`; asserts L(0,0) and L(1,1) are positive.

- **rlgc: closer spacing -> larger NEXT** `[rlgc]` -- compares 1.2mm vs 0.4mm centre-to-centre; asserts |k_near_end| for the closer pair is larger.

- **rlgc: same conductor crosstalk returns zero** `[rlgc]` -- calls `crosstalk_for_pair(r, 0, 0)`; asserts k_near_end == 0 and modal_mismatch == 0.

- **rlgc: 3-signal-conductor matrix is 3x3** `[rlgc]` -- runs `compute_rlgc` on three signals + ground; asserts C and L matrices are both 3x3.

### sikit/tests/roughness_test.cpp

- **roughness: None returns factor 1 at any frequency** `[roughness]` -- builds a default RoughnessSpec; asserts `roughness_factor` returns 1.0 at 1 GHz and 50 GHz.

- **roughness: skin depth shrinks as sqrt(freq)** `[roughness]` -- evaluates skin_depth at 1 GHz and 16 GHz; asserts the 16 GHz value is 1/4 of the 1 GHz value.

- **roughness: HJ factor monotonically rises with Delta/delta** `[roughness]` -- compares HJ at 0.5um and 3.0um RMS; asserts the rougher copper has a larger factor.

- **roughness: HJ factor at high freq approaches the 2x asymptote** `[roughness]` -- HJ at 100 GHz with 3.0um RMS; asserts the factor is in (1.9, 2.0].

- **roughness: HJ factor at low freq approaches 1** `[roughness]` -- HJ at 1 MHz with 0.5um RMS; asserts the factor is ~1 within 0.05.

- **roughness: Huray factor non-trivial** `[roughness]` -- Huray with 0.5um spheres, 1e12 density, 0.5 coverage at 25 GHz; asserts the factor > 1.

- **synthesize: rough copper produces more loss than smooth** `[roughness][synth]` -- compares synthesised channels with smooth vs 2um HJ roughness at 20 GHz; asserts the rough channel has lower |S21|.

### sikit/tests/schematic_topology_test.cpp

- **schematic-topology: power-net heuristic catches obvious rails** `[schematic-topology]` -- queries `looks_like_power_net` on VDD/VCC3V3/GND/AGND/+3V3/-12V; asserts each is true and USB_DP/DQ0 are false.

- **schematic-topology: driver and receiver classification** `[schematic-topology]` -- derives the TX_LANE topology; asserts net_code==2, 2 endpoints, exactly one driver (U1), one passive (R1) and no driver problem.

- **schematic-topology: bidirectional pins count as drivers** `[schematic-topology]` -- derives the DQ0 topology; asserts 3 drivers, 0 receivers and `has_driver_problem` true (multi-drop tripping the heuristic).

- **schematic-topology: zero-driver net is flagged** `[schematic-topology]` -- derives NO_DRIVER; asserts no drivers, 2 receivers and a driver problem.

- **schematic-topology: contention (two drivers) is flagged** `[schematic-topology]` -- derives CONTENTION; asserts 2 drivers and a driver problem.

- **schematic-topology: missing pintype lands in Unspecified** `[schematic-topology]` -- derives RAW; asserts every endpoint role is Unspecified and the net has a driver problem.

- **schematic-topology: unknown net returns empty** `[schematic-topology]` -- derives a bogus net; asserts endpoints empty, net_code==0 and the bogus name is preserved.

- **schematic-topology: bulk derivation skips power rails** `[schematic-topology]` -- runs `derive_all_topologies`; asserts six non-power nets are returned and VDD is filtered.

- **schematic-topology: role_name covers every enum** `[schematic-topology]` -- asserts `role_name` returns "driver", "receiver", "passive", "power", "nc" and "unspecified" for each role.

### sikit/tests/skew_test.cpp

- **skew: matched pair reports zero skew** `[skew]` -- runs `compute_diff_pair_skews` on a matched USB pair; asserts skew_m and skew_ps are < 1e-9 and `exceeds_budget` is false.

- **skew: mismatched pair reports correct sign and magnitude** `[skew]` -- 50.0 vs 50.4mm PCIE pair; asserts skew_m == -0.4mm and skew_ps in (1.5, 5.0) ps.

- **skew: budget threshold flips the FAIL flag** `[skew]` -- runs with a 1 ps budget; asserts the PCIE pair's `exceeds_budget` is true.

- **skew: empty board returns empty result** `[skew]` -- runs on an empty Board; asserts the result is empty.

- **skew: board without diff pairs returns empty result** `[skew]` -- adds single-ended SCK; asserts the result is empty.

- **skew: positive v_phase is reported back** `[skew]` -- runs on the two-pair board; asserts every row's v_phase falls in (1e8, 3e8) m/s.

- **skew: zero-length pair is skipped (no segments anywhere)** `[skew]` -- adds DATA_P/DATA_N with no segments; asserts the result is empty.

### sikit/tests/smoke_test.cpp

- **smoke: arithmetic works** `[smoke]` -- asserts 1 + 1 == 2.

### sikit/tests/sparam_test.cpp

- **sparam: s_to_t round-trip is identity** `[sparam]` -- runs `t_to_s(s_to_t(s))` on a non-trivial complex S matrix; asserts every entry matches the original within 1e-12.

- **sparam: cascading passthrough leaves network unchanged** `[sparam]` -- cascades passthrough with a 3 dB attenuator both ways; asserts |S21| equals the attenuator's |S21|.

- **sparam: cascaded attenuators add in dB** `[sparam]` -- cascades 3 dB and 6 dB attenuators; asserts the total IL is 9 dB.

- **sparam: cascading three attenuators stacks** `[sparam]` -- cascades 1, 2 and 3 dB attenuators; asserts the total IL is 6 dB.

- **sparam: zero-S21 throws on s_to_t** `[sparam]` -- supplies a zero S-matrix; asserts SParamError is thrown.

- **sparam: insertion / return loss in dB** `[sparam]` -- evaluates IL(|S21|=0.5) and RL(|S11|=0.1); asserts 6.02 dB and 20 dB.

- **sparam: touchstone cascade is frequency-by-frequency** `[sparam]` -- cascades two 2-port Touchstone files at two frequencies; asserts IL at 1 GHz is 9 dB and at 2 GHz is 3 dB.

- **sparam: touchstone cascade rejects mismatched grids** `[sparam]` -- supplies files with one differing frequency; asserts SParamError is thrown.

- **sparam: single-ended-to-differential basics** `[sparam]` -- converts a 4-port passthrough to differential; asserts |Sdd21| == 1.

- **sparam: mixed-mode 4x4 -- ideal passthrough has Sdd21=1, Sdc=0** `[sparam]` -- converts a PNPN passthrough to mixed-mode; asserts |Sdd21|=1, |Sdd11|=0, |Sdc21|=0, |Scd21|=0 and |Scc21|=1.

- **sparam: mixed-mode PPNN convention** `[sparam]` -- converts a PPNN passthrough; asserts |Sdd21|=1, |Scc21|=1 and |Sdc21|=0.

- **sparam: skewed pair leaks differential->common (Sdc != 0)** `[sparam]` -- imbalances N (0.7) vs P (1.0); asserts |Scd21| > 0.01.

- **sparam: to_mixed_mode round-trips a TouchstoneFile** `[sparam]` -- builds an ideal 4-port and converts at 1 GHz; asserts |Sdd21|=1 and |Scc21|=1.

- **sparam: to_mixed_mode rejects non-4-port files** `[sparam]` -- supplies a 2-port file; asserts SParamError is thrown.

- **sparam: TDR of zero-reflection line gives Z ~= Zref** `[sparam][tdr]` -- runs TDR on S11=0 across 128 points; asserts every Z(t) is within 1 ohm of 50.

- **sparam: TDR of |S11|=0.2 short reflection moves Z away from Zref** `[sparam][tdr]` -- runs TDR on constant S11=0.2; asserts the peak-to-peak excursion of Z(t) exceeds 2 ohms.

- **sparam: TDR rejects bad input** `[sparam][tdr]` -- supplies a single-point spectrum and a mismatched-length pair; asserts both return empty TDRs.

- **sparam: TDT of perfect passthrough is monotonically rising** `[sparam][tdt]` -- runs TDT on S21=1 across 256 points; asserts the peak |step response| exceeds 0.1.

### sikit/tests/spice_export_test.cpp

- **spice_export: subckt skeleton has the required directives** `[spice]` -- exports a 2-pole fit with subckt_name CHAN; asserts the output contains ".subckt CHAN in out", ".ends CHAN" and "B_out out 0 V=".

- **spice_export: emits one R and one C per pole** `[spice]` -- exports a 2-pole fit; asserts exactly 2 lines start with R and 2 with C.

- **spice_export: capacitor values are C_n = -1/p_n** `[spice]` -- exports poles -1e9 and -2e10; asserts "C0 ... 1.000000e-09" and "C1 ... 5.000000e-11" appear.

- **spice_export: summing source includes residues and d-term** `[spice]` -- exports a fit; asserts the output references V(in), V(n0) and V(n1).

- **spice_export: header carries the fit summary when enabled** `[spice]` -- sets include_header=true; asserts the output contains "poles      = 2", "converged  = yes" and "rms error  =".

- **spice_export: header suppressed when disabled** `[spice]` -- sets include_header=false; asserts "Vector-fit summary" is absent.

- **spice_export: rejects invalid subckt names** `[spice]` -- tries leading-digit, space and empty names; asserts each throws.

- **spice_export: roundtrip through a synthetic Touchstone** `[spice]` -- builds a 200-point Touchstone matching the 2-pole fit and runs `spice_subckt_from_touchstone`; asserts the output has ".subckt TEST in out" and 2 R/C lines.

- **spice_export: write_spice_subckt produces a readable file** `[spice]` -- writes to a temp file; asserts the file is readable and contains ".subckt TEST_FILE".

- **spice_export: throws on empty Touchstone** `[spice]` -- runs `spice_subckt_from_touchstone` on an empty file; asserts it throws.

### sikit/tests/stat_eye_test.cpp

- **stat-eye: clean pulse opens to cursor amplitude** `[stateye][pda]` -- runs `peak_distortion_eye` on a clean cursor SBR; asserts samples_per_ui==spui, eye_height==2.0, eye_width==1.0.

- **stat-eye: ISI tap closes the eye proportionally** `[stateye][pda]` -- runs PDA on a 0.3 post-cursor; asserts eye_height == 1.4 and width == 1.0.

- **stat-eye: empty SBR yields empty envelope** `[stateye]` -- runs PDA and `statistical_eye` on empty SBR; asserts both return empty maps.

- **stat-eye: statistical eye on clean pulse has 0 BER inside eye** `[stateye][stat]` -- runs `statistical_eye` on the clean pulse; asserts BER at v_mid is ~0 across every UI phase.

- **stat-eye: BER is highest at extreme thresholds** `[stateye][stat]` -- runs on the ISI SBR; asserts BER at the topmost and bottommost voltage rows is ~0.5.

- **stat-eye: bathtub_from_stat_eye returns per-UI BER** `[stateye][stat]` -- runs and extracts bathtub; asserts the result has spui entries with each value in [0, 0.5].

- **stat-eye: compute_sbr returns nonempty for a valid channel** `[stateye][sbr]` -- runs `compute_sbr` on a passthrough Touchstone at 5 Gbaud, 16 spui; asserts the SBR is non-empty and the max amplitude is > 0.01.

### sikit/tests/topology_test.cpp

- **topology: empty channel cascades to a passthrough 2-port** `[topology]` -- cascades an empty Channel at 1/5/10 GHz; asserts |S11|=|S22|=0 and |S21|=|S12|=1.

- **topology: single TraceBlock matches direct synthesize_channel** `[topology]` -- compares a Channel with one TraceBlock to direct `synthesize_channel`; asserts every S-parameter agrees within 1e-9 relative + 1e-12 abs.

- **topology: LumpedBlock series-R inserts insertion loss** `[topology][lumped]` -- adds a 10-ohm series R in a 50-ohm system; asserts |S21| == 0.909 within 0.005.

- **topology: LumpedBlock shunt-C rolls off at high frequency** `[topology][lumped]` -- adds a 1 pF shunt at 100 MHz / 1 GHz / 10 GHz; asserts |S21| is strictly decreasing.

- **topology: IdealLineBlock matched to z_ref is lossless and phase-shifts only** `[topology][line]` -- adds a 50-ohm line of 50mm at v_p=1.5e8 in a 50-ohm system; asserts |S11|=0 and |S21|=1.

- **topology: cascaded ideal line phases add** `[topology][line]` -- compares one vs two 50mm line cascades at 5 GHz; asserts 2*phase1 == phase2 mod 2*pi within 1e-6.

- **topology: ViaBlock contributes via insertion loss** `[topology][via]` -- adds a 1.6mm through-via; asserts low-freq |S21| > 0.95 and high-freq |S21| is smaller.

- **topology: TouchstoneBlock wraps an external 2-port** `[topology][touchstone]` -- wraps a 2-port file with |S21|=0.8 at 1 GHz and 0.5 at 10 GHz; asserts the cascade output matches at each frequency within 0.01.

- **topology: TouchstoneBlock rejects non-2-port input** `[topology][touchstone]` -- constructs a TouchstoneBlock from a 4-port file; asserts it throws.

- **topology: a realistic chain has compounded insertion loss** `[topology]` -- compares a trace-only channel to trace + via + trace; asserts the full chain has lower |S21| at 5 GHz.

- **topology: clear() resets the block list** `[topology]` -- adds a block, clears, cascades; asserts empty() is true and the result is a passthrough.

### sikit/tests/touchstone_csv_test.cpp

- **ts-csv: header row is present and correct** `[ts-csv]` -- exports a 2-port file to CSV; asserts the first line begins with "freq_hz,s11_mag_db,...".

- **ts-csv: matched line row has S11 = -inf dB** `[ts-csv]` -- exports a matched line; asserts the CSV contains "1.000000e+09" and "-inf".

- **ts-csv: S21 magnitudes drop with the second row's lossy entry** `[ts-csv]` -- parses the 2 GHz row; asserts s21_mag_db is in (-3, 0).

- **ts-csv: rejects 1-port files** `[ts-csv]` -- exports a 1-port file; asserts TouchstoneParseError is thrown.

- **ts-csv: rejects malformed s_matrices** `[ts-csv]` -- exports a 2-port with only one S-matrix entry; asserts TouchstoneParseError is thrown.

- **ts-csv: z_in equals Zref for perfectly matched port** `[ts-csv]` -- parses the 1 GHz row; asserts z_in_real == 50.0 and z_in_imag == 0 within 1e-6.

### sikit/tests/touchstone_test.cpp

- **touchstone: parse RI 2-port** `[touchstone]` -- parses a sample RI .s2p; asserts num_ports==2, format==RealImaginary, Zref==50, two frequencies and column-major S-matrix entries match the source.

- **touchstone: MA format converts to same complex values as RI** `[touchstone]` -- parses the same data in MA and RI formats; asserts S-matrix entries agree within 1e-6.

- **touchstone: DB format converts magnitude correctly** `[touchstone]` -- parses a DB-format file with -6.02 dB; asserts |S| == 0.5 within 1e-4 and imag == 0.

- **touchstone: frequency unit MHz scales to Hz** `[touchstone]` -- parses an MHz file; asserts frequency_scale==1e6 and frequencies[0]==1e9.

- **touchstone: GHz unit** `[touchstone]` -- parses a GHz file with Zref=75; asserts reference_impedance==75 and the 2.5 GHz frequency is parsed.

- **touchstone: 4-port file is row-major in file, column-major in storage** `[touchstone]` -- parses a 16-entry 4-port row-major matrix; asserts file value at row r, col c lands at storage index r + c*4.

- **touchstone: multi-line data record spans line breaks** `[touchstone]` -- parses a 2-port with data split across lines; asserts the S-matrix recovers correctly.

- **touchstone: comments and blank lines ignored** `[touchstone]` -- parses a file with `!` comments and blank lines; asserts two frequencies parse.

- **touchstone: missing option line throws** `[touchstone]` -- parses data without `#`; asserts TouchstoneParseError is thrown.

- **touchstone: wrong data count for num_ports throws** `[touchstone]` -- supplies 8 floats for a 2-port (needs 9); asserts it throws.

- **touchstone: unknown format throws** `[touchstone]` -- supplies "S XX" as format; asserts it throws.

- **touchstone: non-S parameter type throws (Y/Z not supported v0)** `[touchstone]` -- supplies a Y-parameter file; asserts it throws.

### sikit/tests/touchstone_v2_test.cpp

- **touchstone v2: minimal v2 file parses with version=2 set** `[touchstone][v2]` -- parses a `[Version] 2.0` 12_21 row-major file; asserts version==2, two_port_order==RowMajor, two frequencies and the column-major S-matrix lays out correctly.

- **touchstone v2: [Number of Ports] overrides filename hint** `[touchstone][v2]` -- passes num_ports=1 to a file declaring 2 ports; asserts the keyword wins (num_ports==2).

- **touchstone v2: [Reference] populates per-port impedance vector** `[touchstone][v2]` -- parses a 4-port with `[Reference] 50 50 100 100`; asserts port_impedances has 4 entries with port 2 == 100 and the scalar reference_impedance == 50.

- **touchstone v2: unknown keywords are skipped** `[touchstone][v2]` -- parses a file with `[Information]` and `[Comments]`; asserts the reader still produces a 1-port with the right data.

- **touchstone v2: v1 files still parse via the v1 path** `[touchstone][v2]` -- parses a no-`[Version]` file; asserts version==1 and the S-matrix is unchanged.

- **touchstone v2: rejects [Matrix Format] other than Full** `[touchstone][v2]` -- parses a file with `[Matrix Format] Lower`; asserts TouchstoneParseError is thrown.

- **touchstone v2: writer emits the [Version] keyword block** `[touchstone][v2]` -- writes a v2 file; asserts the output contains `[Version] 2.0`, `[Number of Ports] 2`, `[Network Data]` and `[End]`.

- **touchstone v2: writer emits [Reference] when port_impedances set** `[touchstone][v2]` -- writes a 4-port with port_impedances [50,50,100,100]; asserts the output contains `[Reference]`, ` 50` and ` 100`.

- **touchstone v2: v1 writer output omits the v2 keyword block** `[touchstone][v2]` -- writes a v1 file; asserts `[Version]` and `[Network Data]` are absent and the `# Hz S RI R` line is present.

- **touchstone v2: writer-then-reader round-trip preserves S-matrices** `[touchstone][v2]` -- writes and reads a 2-port v2 file; asserts frequencies and S-matrices match within 1e-9.

- **touchstone v2: writer round-trip carries port_impedances back** `[touchstone][v2]` -- writes a 4-port with port_impedances; asserts the reader recovers the same vector.

### sikit/tests/touchstone_writer_test.cpp

- **ts-write: round-trip 2-port preserves complex S values** `[ts-write]` -- writes and reads a 2-port file; asserts frequencies and S-matrix entries match within 1e-9.

- **ts-write: round-trip 4-port preserves row-major <-> column-major** `[ts-write]` -- writes and reads a 4-port file with entries 1..16; asserts all 16 entries round-trip.

- **ts-write: emits Hz units and RI format** `[ts-write]` -- writes a 2-port; asserts the output contains "# Hz S RI R".

- **ts-write: reference impedance preserved** `[ts-write]` -- writes with reference_impedance=75 and re-reads; asserts the loaded file reports 75.

- **ts-write: rejects port-count mismatch** `[ts-write]` -- supplies a 2-port file with only one S entry; asserts TouchstoneParseError is thrown.

- **ts-write: rejects freq vs matrix length mismatch** `[ts-write]` -- adds an extra frequency without adding a matrix; asserts TouchstoneParseError is thrown.

### sikit/tests/trace_impedance_test.cpp

- **trace impedance: F.Cu uses microstrip, inner uses stripline** `[trace]` -- runs `compute_one` on layers 0, 1 and 31; asserts all Z0 > 0, outer == bottom and outer != inner.

- **trace impedance: zero width returns invalid** `[trace]` -- runs `compute_one(0.0, ...)`; asserts z0==0 and `in_valid_range` is false.

- **trace impedance: compute_all skips non-copper segments** `[trace]` -- adds segments on F.Cu and F.SilkS; asserts only one result with the copper layer.

- **trace impedance: per-segment index lines up with input order** `[trace]` -- adds three F.Cu segments of increasing width; asserts segment_index matches input order and z0 decreases with width.

- **color_for_error: tolerance bands** `[trace]` -- evaluates 0%, 7%, 20% error; asserts green has g>r, yellow has r>b and g>b, red has r>g.

- **color_for_error: zero target returns gray** `[trace]` -- queries with target=0; asserts r==g==b==0.5.

- **from_board: empty stackup -> generic FR-4 defaults** `[trace]` -- builds AnalysisStackup from an empty board; asserts `from_real_stackup` is false, epsilon_r==4.4 and outer_dielectric_height==0.2mm.

- **from_board: picks up dielectric below F.Cu** `[trace]` -- adds an F.Cu copper + 0.15mm/er=3.8 dielectric SiStackup; asserts `from_real_stackup` is true and the dielectric values are read.

- **engine: FDM and closed-form agree to within ~25% on microstrip** `[trace]` -- runs the canonical 50-ohm microstrip; asserts both Z0 > 0 and relative difference < 25%.

- **diff FDM: Z_diff in the right neighbourhood for ~100 ohm geometry** `[trace]` -- runs `compute_diff_z0_fdm` for W=1.5mm/S=0.5mm on 1mm FR-4; asserts z_diff in (60, 150) and z_diff > 1.5 * single-ended Z0.

- **diff FDM: closer spacing yields lower Z_diff** `[trace]` -- compares S=1.0mm vs 0.2mm; asserts the closer spacing has lower z_diff.

- **engine: compute_all caches FDM results per (width, layer)** `[trace]` -- adds four identical-width segments and runs FDM `compute_all`; asserts every segment has the same z0 (cache hit) and z0 > 0.

- **diff closed-form: edge-coupled microstrip Z_diff is in expected band** `[trace]` -- runs `compute_diff_z0_closed_form` for W=1.5mm/S=0.5mm; asserts z in (60, 130) and a tighter S gives lower z.

- **compute_diff_pairs: finds pairs and computes Z_diff per pair** `[trace]` -- adds USB_DP_P/N segments with spacing 0.30mm and 0.20mm widths; asserts one pair with base USB_DP, trace_width 0.2mm, spacing 0.10mm, four segment indices.

- **compute_diff_pairs: spacing derived from parallel geometry** `[trace]` -- adds two parallel 0.15mm traces 0.30mm apart; asserts spacing == 0.15mm.

- **compute_diff_pairs: spacing falls back to S=W if no parallel pair** `[trace]` -- adds two perpendicular traces; asserts spacing == trace_width (0.20mm).

- **compute_diff_pairs: returns empty when no diff pairs exist** `[trace]` -- adds just a VCC net; asserts the result is empty.

- **from_board: stripline B = sum of dielectric above + below inner copper** `[trace]` -- builds an F.Cu/In1/B.Cu stack with 0.10mm prepreg + 0.30mm core; asserts `inner_plane_separation` == 0.40mm.

### sikit/tests/vector_fit_test.cpp

- **vector_fit: recovers a single-pole real model** `[vectorfit]` -- builds H(s) = r/(s-p) + d at 200 log-spaced points; asserts the fit finds one pole with rms_error < 1e-6.

- **vector_fit: recovers a three-pole real model** `[vectorfit]` -- builds a three-pole rational model at 300 points; asserts rms_error < 1e-3 and `evaluate_fit` matches H within 1e-3 relative.

- **vector_fit: poles come out negative (causal, stable)** `[vectorfit]` -- overparameterises (5 poles for 3 true); asserts every fitted pole is negative.

- **vector_fit: rejects malformed input** `[vectorfit]` -- supplies empty, size-mismatched, too-few-samples and unsorted-frequency inputs; asserts each throws VectorFitError.

- **vector_fit: many-pole fit of a low-pass channel converges to a sensible model** `[vectorfit]` -- fits 8 real poles spread over 4 decades at 400 points with max_iter=16; asserts rms_error < 0.02.

### sikit/tests/via_model_test.cpp

- **via_lumped: sensible L and C for canonical PCB via** `[via]` -- evaluates a 1.6mm/0.3mm/0.6mm/1.0mm stubless via; asserts L_barrel in (0.1, 5) nH, C_pad in (10 fF, 5 pF) and Z_stub in (10, 200) ohms.

- **via_lumped: L scales with ln(antipad/drill)** `[via]` -- compares 1.0mm vs 2.0mm antipad diameter; asserts the bigger antipad has larger L_barrel.

- **via_lumped: stub resonance at expected frequency** `[via]` -- adds a 1mm stub; asserts the resonance lands near 36.2 GHz within 2 GHz.

- **compute_via_s2p: rejects bad geometry** `[via]` -- sets drill=0, antipad<=drill or total_length=0; asserts each throws.

- **compute_via_s2p: stubless via passes through at low freq** `[via]` -- evaluates at 1/10/100 MHz; asserts |S21| ~ 1 within 0.005.

- **compute_via_s2p: insertion loss grows with frequency** `[via]` -- sweeps 100 MHz to 20 GHz; asserts at least 3 of 4 steps show a drop in |S21|.

- **compute_via_s2p: stub creates a deep |S21| notch near resonance** `[via]` -- sweeps a 1mm-stub via across 5..60 GHz; asserts the min |S21| is less than 50% of the off-resonance level.

- **compute_via_s2p: 2-port file shape matches grid** `[via]` -- evaluates at four frequencies; asserts num_ports==2, four matrices and 4 entries per matrix.

---

## pdnkit (power delivery network)

### pdnkit/tests/cavity_model_test.cpp

- **cavity: DC behavior matches parallel-plate capacitor** `[cavity]` -- evaluates 100x100mm FR-4 cavity at 100 Hz with tan_delta=0; asserts |Z| matches 1/(omega*C_pp) within 1e-4 and the imaginary part is negative.

- **cavity: reciprocity Z(p1,p2) == Z(p2,p1)** `[cavity]` -- evaluates a 80x60mm cavity at 100 MHz between two non-collocated ports; asserts Z12 == Z21 within 1e-12 relative.

- **cavity: |Z| peaks near first TM10 resonance** `[cavity]` -- evaluates a 100x100mm cavity at 0.5/1.0/2.0 * f10; asserts the magnitude at f10 exceeds both off-resonance points.

- **cavity: sweep returns vector of expected length** `[cavity]` -- runs a 20-point sweep; asserts the result has 20 entries, each finite and non-negative.

- **cavity: empty decap list equals plain self-impedance** `[cavity][decap]` -- compares `cavity_impedance_with_decaps` with empty vs `cavity_impedance`; asserts relative difference < 1e-12.

- **cavity: nearby decap pulls Z toward the parallel combination** `[cavity][decap]` -- adds a 100 nF cap 5mm from the observation port; asserts |Z_with_decap - Z_parallel| < |Z_with_decap - Z_plain|.

- **cavity: adding a decap reduces |Z| in the cap-dominant band** `[cavity][decap]` -- adds a 1 uF cap with low ESR/ESL at 10 MHz; asserts |Z_with| < |Z_plain|.

- **cavity: decap sweep magnitudes finite + non-negative** `[cavity][decap]` -- sweeps two decaps over 25 frequencies; asserts every magnitude is finite and >= 0.

- **cavity: peak frequency matches analytical TM10 within 5%** `[cavity][validation]` -- sweeps tightly around f10 with 401 points on a 100x60mm cavity; asserts the peak location is within 5% of c/(2a*sqrt(eps_r)).

- **cavity: TM01 peak resolves on a non-square plane** `[cavity][validation]` -- sweeps around f01 on a 100x50mm cavity; asserts the peak is within 5% of c/(2b*sqrt(eps_r)).

- **cavity: peak frequency scales 1/sqrt(eps_r)** `[cavity][validation]` -- compares peaks at eps_r=2.2 vs 4.3; asserts the ratio matches sqrt(4.3)/sqrt(2.2) within 5%.

### pdnkit/tests/cavity_mount_l_test.cpp

- **cavity-mount-L: zero mounting L matches default behavior** `[cavity-mount-l][validation]` -- evaluates with mounting_via_loop_l_h=0 and the baseline; asserts the two Z values match within numerical tolerance.

- **cavity-mount-L: nonzero mounting L lowers cap SRF** `[cavity-mount-l][validation]` -- evaluates a 1 uF cap with ESL 0.5 nH at its bare SRF, comparing bare vs +1 nH mounting; asserts the mounted |Z| exceeds the bare |Z|.

- **cavity-mount-L: SRF scales with 1/sqrt(L_eff)** `[cavity-mount-l]` -- analytic check: doubles L; asserts f2/f1 == 1/sqrt(2) within 1e-9.

### pdnkit/tests/decap_optimizer_test.cpp

- **optimizer: easy target requires no caps** `[optimizer]` -- sets target_z=1e6 ohms; asserts `target_met` is true and the decaps list is empty.

- **optimizer: meaningful target picks some decaps and reduces |Z|** `[optimizer]` -- sets target_z=50 mohm on an 80x60mm plate; asserts the decap list is non-empty, final_max_z < 50 and chosen caps cluster within 10mm of the configured position.

- **optimizer: max_caps cap is respected** `[optimizer]` -- sets target_z=1e-6 and max_caps=4; asserts the result has <=4 caps and `target_met` is false.

- **optimizer: invalid config returns no caps** `[optimizer]` -- sets target_z=-1; asserts the decaps list is empty and `target_met` is false.

### pdnkit/tests/dielectric_test.cpp

- **dj-sarkar: well below f1 -> eps_inf + delta_eps** `[dielectric][validation]` -- evaluates at 1 Hz (3 decades below f1=1 kHz); asserts eps_r' ~ eps_inf+delta_eps and eps_r'' < 0.01.

- **dj-sarkar: well above f2 -> eps_inf** `[dielectric][validation]` -- evaluates at 1 THz; asserts eps_r' ~ eps_inf within 0.05.

- **dj-sarkar: FR-4 default fit at f2 lands near eps_inf** `[dielectric][validation]` -- evaluates at 1 GHz; asserts eps_r' in (3.8, 4.0) and tan_delta in (0, 0.03).

- **dj-sarkar: loss peaks mid-band** `[dielectric]` -- evaluates at sqrt(f1*f2) and at decade-offset edges; asserts mid-band imag > both edge imags.

- **dj-sarkar: eps_r real part decreases with frequency** `[dielectric]` -- evaluates at 1 MHz, 1 GHz, 10 GHz; asserts strictly decreasing.

- **dj-sarkar: invalid inputs return eps_inf** `[dielectric]` -- queries with f=0 and a bad f1=0 model; asserts both fall back to eps_inf.

### pdnkit/tests/e2e_test.cpp

- **e2e: tiny_pdn fixture parses with expected counts** `[e2e]` -- parses the bundled tiny_pdn.kicad_pcb; asserts 11 layers, total_thickness 1.6mm, 3 nets, 4 pads, 1 segment, 1 via and 2 zones.

- **e2e: through-hole pads on *.Cu expand to all copper layers** `[e2e]` -- searches for pads that cover both F.Cu and B.Cu; asserts exactly 2 such pads (the PinHeader through-holes).

- **e2e: IR drop on +3V3 F.Cu produces sane voltage map** `[e2e]` -- meshes the +3V3 net on F.Cu at 0.5mm cells and solves with 1 A; asserts >100 nodes, non-empty resistors, source and sink nodes set, every voltage finite in [0, 1) and a positive spread.

- **e2e: board outline parsed from Edge.Cuts** `[e2e]` -- parses the outline; asserts 76 outline lines and the first segment runs from (0, 15) to (20, 15).

### pdnkit/tests/ir_mesher_test.cpp

- **mesher: no matching geometry returns empty mesh** `[irmesh]` -- runs IrMesher on a board with no zones; asserts nodes and resistors are empty.

- **mesher: 10mm square zone with 1mm cells -> ~100 nodes** `[irmesh]` -- meshes a 10x10mm zone at 1mm cells; asserts 100 nodes and 180 interior edge resistors.

- **mesher: invalid config returns empty mesh** `[irmesh]` -- sets cell_size=0; asserts the mesher returns no nodes.

- **mesher: per-resistor conductance = t/rho for square cells** `[irmesh]` -- meshes with t=35um and rho=1.68e-8; asserts the first resistor conductance ~ 2083 S.

- **mesher: mesh respects zone hole (no nodes inside)** `[irmesh]` -- meshes a 10x10mm zone with a 4x4mm hole; asserts 84 nodes (100 - 16).

- **mesher: leftmost and rightmost pads become source and sink** `[irmesh]` -- adds two pads at the ends of a zone; asserts one source and one sink with source.x < sink.x.

- **mesher: pad on wrong net or layer is ignored** `[irmesh]` -- adds pads on a different net and a different layer; asserts source and sink lists are empty.

- **mesher: bbox matches the source polygon bbox** `[irmesh]` -- meshes a 10x10mm zone; asserts bbox is (0,0)-(0.010, 0.010).

- **mesher: explicit source/sink pad names override auto-pick** `[irmesh]` -- adds three pads A/B/C and configures source=B, sink=C; asserts the picked source is near x=0.005 and sink near x=0.009.

- **mesher: missing-name explicit list falls back to auto-pick** `[irmesh]` -- supplies source_pad_names with no matches; asserts source and sink lists are still populated via auto-pick.

- **mesher: explicit pad_currents populate node_currents** `[irmesh]` -- supplies pad currents A=2, B=-0.5, C=-1.5; asserts 3 entries are present and they sum to 0.

- **mesher: multi-layer meshes both layers and wires via resistors** `[irmesh]` -- meshes F.Cu + B.Cu zones connected by one through-via; asserts 200 nodes, 361 resistors (180+180+1) and 100 nodes per layer.

- **mesher: extra_layer_ordinals duplicate of primary is harmless** `[irmesh]` -- supplies extra_layer_ordinals=[0,0,0]; asserts the result has 100 nodes (not 400).

- **mesher: prune drops disconnected component without source/sink** `[irmesh][connectivity]` -- adds two separated zones with pads only on the left one; asserts only the left 100 nodes remain.

- **mesher: prune kills everything if source + sink are on different islands** `[irmesh][connectivity]` -- adds two separated zones with source on one and sink on the other; asserts nodes is empty.

- **mesher: auto_select_layer picks the layer with the most copper** `[irmesh][autopick]` -- requests F.Cu but only B.Cu has copper, with auto_select_layer=true; asserts primary_layer_used==31 and every node is on B.Cu.

- **mesher: auto_select_layer=false enforces the requested layer** `[irmesh][autopick]` -- same setup with auto_select_layer=false; asserts the result is empty.

- **mesher: auto_select_layer is a no-op when primary layer already has copper** `[irmesh][autopick]` -- F.Cu has copper and auto_select_layer=true; asserts primary_layer_used==0.

- **track-mesher: 2-segment trace builds 3 nodes, 2 resistors** `[irmesh][track]` -- adds two 5mm segments end-to-end with two pads at the endpoints; asserts 3 nodes, 2 resistors and non-empty source/sink lists.

- **track-mesher: T-junction merges 3 segments at one node** `[irmesh][track]` -- adds three segments meeting at the origin; asserts 4 nodes and 3 resistors.

- **track-mesher: pad far from any track endpoint is ignored** `[irmesh][track]` -- adds a single segment plus two pads 50mm away; asserts source and sink lists are empty.

### pdnkit/tests/ir_result_mesh_test.cpp

- **ir-result-mesh: empty solution -> empty mesh** `[result-mesh]` -- builds the result mesh from an empty Solution; asserts vertices is empty.

- **ir-result-mesh: per-node quad with normalized voltage** `[result-mesh]` -- builds from a 2-node mesh with voltages 2mV and 0V; asserts 8 verts, 12 indices and the t coords are 1.0 for node 0 and 0.0 for node 1.

- **ir-result-mesh: zero-span solution clamps t to 0** `[result-mesh]` -- builds from a 1-node mesh with min==max; asserts the t coord is 0.

- **ir-result-mesh: hotspot points at the worst node (voltage)** `[result-mesh][hotspot]` -- builds with voltages [1.0, 0.4, 0.05]; asserts hotspot.valid, x==0.002, value==0.05 and is_current is false.

### pdnkit/tests/ir_solver_test.cpp

- **solver: empty mesh returns error** `[solver]` -- solves an empty IrMesh; asserts ok is false and error == "empty mesh".

- **solver: missing sources fails cleanly** `[solver]` -- builds a one-node mesh with no sources; asserts the solve returns ok=false.

- **solver: missing sinks fails cleanly** `[solver]` -- builds a one-node mesh with a source but no sink; asserts ok=false.

- **solver: 2-node, 1A through G=1000S gives 1mV drop** `[solver]` -- 2-node mesh with G=1000 and 1 A source; asserts V_src == 1 mV and V_sink == 0.

- **solver: 3-node series, middle is halfway** `[solver]` -- 3-node series with G=1000 and 1 A; asserts V_src == 2 mV, V_mid == 1 mV, V_sink == 0.

- **solver: total_current scales linearly** `[solver]` -- compares 1 A and 5 A solves; asserts the V_src ratio is 5.

- **solver: parallel paths cut effective resistance** `[solver]` -- 4-node square with two parallel paths; asserts V_src == 1 mV (R_eq == 1 mohm).

- **solver: end-to-end with mesher on a 10mm square zone** `[solver][e2e]` -- meshes a 10x10mm zone at 1mm cells and solves with 1 A; asserts 100 nodes, exactly one source/sink, max_v < 5 mV and min_v == 0.

- **solver: explicit node_currents (multi-source/sink balance)** `[solver]` -- 5-node line with two 0.5 A sources and one -1 A sink; asserts v[2] == 0, v[0]==v[4] and v[0] > 0.

- **solver: explicit currents must sum to zero** `[solver]` -- supplies +1 at node 0 and -0.5 at node 2 (sum != 0); asserts the solve fails.

- **solver: explicit currents need at least one sink** `[solver]` -- supplies two positive currents only; asserts the solve fails.

### pdnkit/tests/mor_test.cpp

- **mor: empty inputs return empty** `[mor]` -- runs `reduce_to_ports` on an empty mesh; asserts port_node_ids is empty.

- **mor: chain mesh reduces correctly** `[mor][validation]` -- reduces a 3-node 10+10 ohm chain to ports {0, 2}; asserts the 2x2 G has +0.05 on diagonal and -0.05 off-diagonal.

- **mor: parallel paths combine correctly** `[mor][validation]` -- reduces a 10 || 20 ohm network; asserts the off-diagonal entry is -0.15.

- **mor: keeping all nodes is identity** `[mor]` -- reduces with all nodes kept; asserts the 3x3 G has the original diagonal values (0.1, 0.2, 0.1).

- **mor: SPICE export of reduced network** `[mor]` -- exports the chain-3 reduction; asserts the output contains "R0 NP0 NP1 2.000000e+01" and ".end".

### pdnkit/tests/ohms_law_test.cpp

- **ohms-law: 100mm trace solves close to rho*L/(W*t)** `[ohms][validation]` -- parses trace_100mm.kicad_pcb, meshes at 0.5mm cells and solves 1 A; asserts V_drop lies in [0.5x, 1.5x] of the analytic 4.32 mV.

- **ohms-law: drop scales linearly with current** `[ohms][validation]` -- solves at 1 A and 3 A; asserts the ratio of drops is exactly 3.

- **ohms-law: result tightens as cell size shrinks** `[ohms][validation]` -- solves at 1.0mm vs 0.25mm cells; asserts both are within +/-50% of R_ideal, fine error <= coarse error and fine error < 5%.

- **ohms-law: source/sink_pad_indices match name-based picking** `[ohms][validation][probe-r]` -- compares auto-pick vs explicit pad-index pick on the same fixture; asserts the two V drops match within 1e-6 relative and swapping source/sink leaves |V drop| unchanged.

- **ohms-law: current-density heat-map matches I/W in mid-trace** `[ohms][validation][current-density]` -- builds the current-density render mesh and computes |J_x| at mid-trace nodes via central diff; asserts the mean |K| matches I/W within 10%.

### pdnkit/tests/power_drc_test.cpp

- **ipc2221: 1A external 1oz 10C rise -> ~0.30 mm** `[drc][validation]` -- computes `ipc2221_min_area(1, 10, true)`; asserts the required width is in (0.25, 0.40) mm and the inverse `ipc2221_max_current` recovers 1 A.

- **ipc2221: internal needs more copper than external** `[drc]` -- compares internal vs external at 2 A 10 C; asserts area_int > area_ext and the ratio ~ 2.6.

- **ipc2221: 30C rise needs less width than 10C rise** `[drc]` -- compares 10 C vs 30 C at 3 A; asserts area_30 < area_10 with ratio ~ 0.51.

- **drc: flags undersized trace on the target net** `[drc]` -- adds one 0.5mm and one 2.0mm segment on +3V3, runs check_ipc2152 at 3 A; asserts segments_checked==2 and exactly one violation on the 0.5mm segment.

- **drc: distinguishes internal from external layers** `[drc]` -- adds a 0.3mm internal segment on VCC at 1 A; asserts one violation flagged as `external=false`.

### pdnkit/tests/roughness_test.cpp

- **rough: skin depth in Cu at 1 GHz is ~2.06 um** `[roughness][validation]` -- evaluates `skin_depth_copper` at 1 GHz; asserts the result is between 2.0 and 2.15 um.

- **rough: low-freq limit gives multiplier ~ 1** `[roughness][validation]` -- evaluates HJ at 1 kHz with 1 um Rq; asserts K == 1 within 1e-6.

- **rough: high-freq limit gives multiplier ~ 2** `[roughness][validation]` -- evaluates HJ at 1 THz with 5 um Rq; asserts K in (1.95, 2.0].

- **rough: 1 um Rq at 10 GHz lands in published HJ band** `[roughness][validation]` -- evaluates 1 um Rq at 10 GHz; asserts K in (1.65, 1.95).

- **rough: rougher copper -> higher multiplier** `[roughness]` -- compares 0.4 um vs 2.0 um Rq at 5 GHz; asserts the rougher value is higher.

- **rough: zero inputs return 1** `[roughness]` -- evaluates HJ with R_q=0 and f=0; asserts both return 1.0.

### pdnkit/tests/sensitivity_test.cpp

- **sensitivity: empty inputs return empty** `[sensitivity]` -- runs sensitivity_sweep with no caps or freqs; asserts the result is empty.

- **sensitivity: ranks small bypass ahead of bulk at high f** `[sensitivity][validation]` -- sweeps two caps (10 uF bulk and 100 nF bypass) across 100 kHz to 5 GHz; asserts the bulk peak_freq < bypass peak_freq and the bypass peak is > 10 MHz.

- **sensitivity: symmetric caps have equal sensitivity** `[sensitivity]` -- adds two identical caps at the same position; asserts both have equal max_relative_change.

### pdnkit/tests/smoke_test.cpp

- **smoke: arithmetic works** `[smoke]` -- asserts 1 + 1 == 2.

### pdnkit/tests/spice_export_test.cpp

- **spice: empty mesh -> minimal netlist** `[spice]` -- exports an empty IrMesh; asserts the output contains ".end" and has zero R/I/Vsink lines.

- **spice: chain mesh has expected R / I / V lines** `[spice]` -- exports a 3-node chain with default 1 A; asserts 2 R lines at "1.000000e+01", one "I0 0 N0 DC 1.000000e+00" and "Vsink0 N2 0 DC 0", plus ".op" and ".end".

- **spice: explicit node_currents override source list** `[spice]` -- supplies node_currents [{0, 0.5}, {1, 0.25}]; asserts 2 I lines with 0.5 and 0.25 and no 1.0 A fallback.

- **spice: multiple sources split the total current** `[spice]` -- adds two sources and total_current=2; asserts each I line carries 1 A.

- **spice: title and position comments configurable** `[spice]` -- sets a custom title and enables position comments; asserts the title appears as a "* " comment and at least one "; (" position comment is emitted.

### pdnkit/tests/stackup_writer_test.cpp

- **stackup-save: same src/dst path is rejected** `[stackup-save]` -- calls `save_modified_stackup(p, p, b)`; asserts ok is false.

- **stackup-save: updates thickness of matching layers** `[stackup-save][validation]` -- saves with 70um F.Cu/B.Cu; asserts ok, layers_updated==2 and the destination contains "0.07" but not "0.035", with the 0.51mm dielectric untouched.

- **stackup-save: missing stackup block fails cleanly** `[stackup-save]` -- saves with a kicad_pcb missing the stackup section; asserts ok==false and the error mentions "stackup".

### pdnkit/tests/target_z_test.cpp

- **target-z: 3.3V 5pct 1A -> 165 mOhm** `[target-z][validation]` -- evaluates Smith's canonical case; asserts the result is 0.165 within 1e-9.

- **target-z: tighter step -> tighter target** `[target-z]` -- compares 0.5 A vs 5.0 A step at 3% ripple; asserts the ratio is 1/10.

- **target-z: 0.9V core rail at high current** `[target-z]` -- evaluates 0.9V/3%/50A; asserts the result is 0.00054 within 1e-9.

- **target-z: zero current step returns 0** `[target-z]` -- evaluates with current_step=0; asserts the result is 0.

### pdnkit/tests/thermal_test.cpp

- **thermal: trace_100mm at 5A heats ~12 C with R_theta=100** `[thermal][validation]` -- runs `solve_ir_with_thermal` at 5 A with R_theta=100; asserts converged, iterations >= 2, dT in (10, 18) C and rho_final in (1.03, 1.08) * rho_20.

- **thermal: bigger heatsink (lower R_theta) means less rise** `[thermal]` -- compares R_theta=200 vs 20; asserts the hot run has larger dT and rho.

- **thermal: no current means no heating** `[thermal]` -- solves with 0 A; asserts dT ~ 0 and rho equals the ambient-temperature corrected value.

- **thermal: convergence in a handful of iterations** `[thermal]` -- solves at 5 A; asserts iterations <= 6.

### pdnkit/tests/touchstone_test.cpp

- **touchstone: empty samples -> header only** `[touchstone]` -- writes an empty .s1p; asserts the file contains "# Hz Z RI R 50".

- **touchstone: round-trip a few samples** `[touchstone]` -- writes and re-reads 4 samples; asserts each frequency and real/imag round-trip exactly.

- **touchstone: comment lines emit with ! prefix** `[touchstone]` -- writes with a "line one\nline two" comment; asserts both lines appear as "! line one" / "! line two".

- **touchstone: option line is well-formed** `[touchstone]` -- writes one sample; asserts the option line reads "# Hz Z RI R 50".

### pdnkit/tests/transient_test.cpp

- **transient: empty mesh returns error** `[transient]` -- calls `solve_step_transient` on an empty mesh; asserts ok is false.

- **transient: missing sink returns error** `[transient]` -- builds a 1-node mesh with a source but no sink; asserts ok is false.

- **transient: two-node RC matches analytical step response** `[transient]` -- runs Backward Euler on a 2-node RC with dt=tau/100; asserts the solver tracks V(t) = I*R*(1 - exp(-t/RC)) to <5% at four sample points.

- **transient: asymptotic value matches static IR drop** `[transient]` -- runs 2000 steps; asserts the final source voltage equals I*R within 1%.

- **transient: per-node C vector overrides scalar** `[transient]` -- supplies per_node_capacitance=1e-15 (ignored) and per_node_capacitances=[1uF, 1uF]; asserts the final voltage matches the 1uF case within 1%.

- **transient: per-node vector wrong length is an error** `[transient]` -- supplies a one-element vector for a 2-node mesh; asserts ok is false.

- **transient: distributed C builder gives plane-pair value per cell** `[transient]` -- evaluates `build_distributed_capacitance` for 0.5mm cells, FR-4, 1.6mm; asserts each node holds ~5.95e-15 F within 5%.

- **transient: decap C lumps onto nearest node** `[transient]` -- adds a 1 uF decap near node 1; asserts c[1] > 0.5 uF and c[0] < 1 nF.

### pdnkit/tests/via_inductance_test.cpp

- **via-L: 0.3mm/1.6mm self-inductance is ~660 pH** `[via-l][validation]` -- evaluates `via_self_inductance(0.15mm, 1.6mm)`; asserts L in (600, 750) pH.

- **via-L: doubling length more than doubles inductance** `[via-l]` -- compares 1.6mm vs 3.2mm; asserts L2/L1 in (2.0, 3.0).

- **via-L: thinner via has higher self-L** `[via-l]` -- compares 0.15mm vs 0.075mm radius; asserts the thinner L is higher by ~222 pH (mu0*h*ln(2)/(2*pi)).

- **via-L: mutual at 0.5mm spacing is ~370 pH** `[via-l][validation]` -- evaluates `via_mutual_inductance` for 1.6mm length / 0.5mm spacing; asserts M in (340, 400) pH.

- **via-L: pair loop inductance is self - mutual** `[via-l][validation]` -- evaluates `via_pair_loop_inductance`; asserts it equals L_self - M and lies in (250, 350) pH.

- **via-L: zero radius or length returns 0** `[via-l]` -- evaluates with r=0, h=0 and d=0; asserts each returns 0.

- **via-L: mutual goes to zero as spacing grows** `[via-l]` -- compares 0.5mm vs 10mm spacing; asserts M(10mm) < M(0.5mm) and < 100 pH.

### pdnkit/tests/vrm_test.cpp

- **vrm: at DC Z = R_droop** `[vrm][validation]` -- evaluates `vrm_impedance` at omega=0; asserts real==r_droop and imag==0.

- **vrm: at R/L corner |Z| = R*sqrt(2)** `[vrm][validation]` -- evaluates at omega=R/L; asserts |Z| matches R*sqrt(2) within 1e-9.

- **vrm: well above corner Z is inductive** `[vrm]` -- evaluates at 10 MHz; asserts imag > 100*real, imag == omega*L and phase > 89 degrees.

- **vrm: custom parameters scale as expected** `[vrm]` -- evaluates with droop=0.5mohm, L=4.7uH at DC and 1 MHz; asserts the DC real matches and the 1 MHz imag matches omega*L.


---

## emikit (electromagnetic interference)

### emikit/tests/analysis_test.cpp

- **analysis: empty board returns NoData, not PASS** `[emi]` -- runs `analyze_board` on an empty Board; asserts nets is empty and verdict.status == NoData.

- **analysis: filter that matches nothing returns NoData** `[emi]` -- supplies net_filter={"DOES_NOT_EXIST"}; asserts nets is empty and status is NoData.

- **analysis: one trace produces one NetEmission record** `[emi]` -- analyses one 50mm CLK trace; asserts 1 net record with correct name, total_length_m, loop_area_m2 > 0 and e_dbuv length matching the default grid.

- **analysis: longer trace radiates more** `[emi]` -- compares 20mm vs 200mm traces; asserts the larger worst_case_dbuv.

- **analysis: net filter restricts evaluated nets** `[emi]` -- supplies net_filter={"CLK"} on a board with CLK and DATA; asserts only the CLK record is returned.

- **analysis: tight drive + giant loop fails the mask** `[emi]` -- runs 1m trace, 5mm loop, 1A/1ns/100MHz drive; asserts verdict==Fail and worst_margin_db < 0.

- **analysis: default freq grid covers 30 MHz to 1 GHz** `[emi]` -- queries `default_cispr_freq_grid(50)`; asserts size==50, first==30 MHz and last==1 GHz.

### emikit/tests/cable_cm_test.cpp

- **cable: zero length -> zero field** `[cable]` -- evaluates with length=0; asserts the result is 0.

- **cable: zero current -> zero field** `[cable]` -- evaluates with cm_current=0; asserts the result is 0.

- **cable: 20uA, 30 cm, 100 MHz, 10 m matches Paul Eq 11.5** `[cable][calibration]` -- evaluates the canonical case; asserts |E| == 7.54e-5 V/m within 0.5% and 37.55 dBuV/m within 0.05.

- **cable: short-dipole regime is linear in f, I, and L** `[cable]` -- doubles f, I, L and r independently; asserts each scales by the analytic factor (2x or 0.5x).

- **cable: TI ADS8686S working point sanity check** `[cable][calibration]` -- evaluates 30 cm / 30 uA / 480 MHz / 10 m; asserts dBuV equals 54.71 within 0.1.

### emikit/tests/calibration_test.cpp

- **calibration: 1mA, 1 cm^2, 100 MHz, 3 m matches Ott Eq 11-2** `[calibration]` -- evaluates the reference loop; asserts 12.85 dBuV/m within 0.05.

- **calibration: f=300 MHz scales by 9x vs 100 MHz reference** `[calibration]` -- evaluates the same loop at 300 MHz; asserts 31.93 dBuV/m (12.85 + 19.08) within 0.05.

- **calibration: r=10 m drops by 10/3x vs 3 m reference** `[calibration]` -- evaluates at r=10m; asserts 2.39 dBuV/m within 0.05.

- **calibration: I=10 mA gives +20 dB over reference** `[calibration]` -- evaluates with 10x current; asserts 32.85 dBuV/m within 0.05.

- **calibration: V/m result matches hand calc to 3 sig figs** `[calibration]` -- evaluates the reference V/m; asserts 4.390e-6 within 0.5%.

### emikit/tests/loop_test.cpp

- **loop: zero area -> zero E** `[loop]` -- evaluates with A=0; asserts E==0.

- **loop: zero current -> zero E** `[loop]` -- evaluates with I=0; asserts E==0.

- **loop: E scales as f^2** `[loop]` -- doubles freq; asserts E ratio is 4.

- **loop: E scales linearly in area and current** `[loop]` -- doubles A and I independently; asserts each gives 2x base.

- **loop: E falls as 1/r** `[loop]` -- evaluates at 3 m vs 10 m; asserts the ratio is 10/3.

- **loop: dBuV conversion is consistent with V/m result** `[loop]` -- evaluates both forms; asserts dBuV == 20*log10(V/m * 1e6).

- **loop: sweep returns one value per freq** `[loop]` -- runs a 4-frequency sweep; asserts result size==4.

- **loop: sweep size mismatch throws** `[loop]` -- supplies freqs of size 2 and loop heights of size 1; asserts it throws.

### emikit/tests/masks_test.cpp

- **masks: CISPR 32 Class B step at 230 MHz** `[masks]` -- evaluates limit at 100 MHz and 500 MHz; asserts 40 and 47 dBuV/m.

- **masks: CISPR 32 Class A is looser than Class B in upper band** `[masks]` -- evaluates both at 2 GHz; asserts Class A > Class B.

- **masks: FCC Part 15 Class B has its own breakpoints** `[masks]` -- evaluates 50/100/500/980 MHz; asserts 40/43.5/46/54 dBuV/m.

- **masks: limit_at clamps at endpoints** `[masks]` -- queries below 30 MHz and above 6 GHz; asserts the clamps return the first and last point values.

- **masks: margin_db sign convention** `[masks]` -- evaluates margin at 30 and 50 dBuV against a 40 dBuV limit; asserts +10 and -10.

- **masks: registry lookup** `[masks]` -- looks up CISPR 32 Class B by name; asserts the pointer matches, an unknown name returns nullptr and `all_masks().size() == 6`.

- **masks: every mask carries a usable shape** `[masks]` -- iterates all_masks; asserts non-empty name/family/source and test_distance > 0 and points non-empty.

### emikit/tests/spectrum_test.cpp

- **spectrum: fundamental of a 100 MHz / 50% / 1 ns rise is well-behaved** `[spectrum]` -- evaluates harmonic 1 of a 20 mA trapezoid; asserts magnitude in (10 mA, 15 mA).

- **spectrum: even harmonics of a 50% duty cycle vanish** `[spectrum]` -- evaluates harmonics 2 and 4 with duty=0.5; asserts both < 1e-12.

- **spectrum: corner frequencies match the closed form** `[spectrum]` -- evaluates envelope_corners for 10ns period / 1ns rise; asserts f_tau ~ 63.66 MHz and f_tr ~ 318.3 MHz.

- **spectrum: sweep returns one value per freq** `[spectrum]` -- runs a 4-frequency sweep; asserts result size==4.

- **spectrum: zero period -> zero output** `[spectrum]` -- evaluates with period=0; asserts harmonic 1 is 0 and every sweep entry is 0.


---

## mpkit (multiphysics)

### mpkit/tests/component_coupling_test.cpp

- **apply_default_metadata fills height + mass from package family** `[component-couple]` -- adds a SOIC-8 component with zero metadata; asserts `apply_default_metadata` returns 1 and a second call returns 0 (idempotent).

- **apply_default_metadata respects user-set values** `[component-couple]` -- presets body_height/mass; asserts the function returns 0 and the values are untouched.

- **component_power_to_joule_source stamps a top-side component** `[component-couple]` -- runs the coupling on a 1 W SOIC-8 on a 16x16x8 grid; asserts ok, total_power_w==1.0, the volume integral equals 1.0, k=0 stays zero and k_top has > 0 power.

- **component_power_to_joule_source skips zero-power parts** `[component-couple]` -- adds a zero-power resistor; asserts every source cell is 0.

- **compute_component_summary tallies mass + power + hottest** `[component-couple]` -- adds three components (U1, U2, R5); asserts n_components==3, total_power=2.76, hottest_reference=="U2" and hottest_power==2.50.

- **summary uses PackageDefaults mass when component has no mass set** `[component-couple]` -- adds a pin header with mass=0; asserts total_mass_kg > 0 via the package-default fallback.

### mpkit/tests/elasticity_test.cpp

- **Uniaxial compression with nu=0 reproduces sigma_xx = E u_x / L** `[no tag]` -- 8x4x4 cube with E=100 GPa, nu=0, 10um compression; asserts sigma_xx matches E*delta/L within 1% and sigma_yy/zz are < 0.1%.

- **Constrained uniform heat: sigma = -E alpha dT / (1 - 2 nu) hydrostatic** `[no tag]` -- 6x6x6 fully clamped cube with dT=50; asserts each diagonal stress is the hydrostatic value within 2% and shears < 0.1%.

- **Rejected configuration: no Dirichlet pin returns an error** `[no tag]` -- runs without BCs; asserts r.ok is false and the error mentions "Dirichlet pin".

### mpkit/tests/grid_test.cpp

- **Grid cell-centre + world_to_index round trip** `[no tag]` -- builds a 10x8x4 grid; asserts voxel_count==320, cx/cy/cz return the cell centres and every world->index call recovers the original (i, j, k).

- **Grid world_to_index returns -1 for out-of-grid axes** `[no tag]` -- queries (-1, 0.5mm, 0.5mm) and (0.5mm, 0.5mm, 99); asserts the out-of-range axis returns -1.

### mpkit/tests/joule_coupling_test.cpp

- **Single resistor across one cell deposits the right total power** `[no tag]` -- two-node mesh with G=2.5 and 1 V drop; asserts total_power_w == 2.5 and half the power lands in each endpoint voxel.

- **Resistor whose node sits outside the grid is dropped, not crashed** `[no tag]` -- places one node far outside; asserts dropped_nodes==1 and the kept endpoint still gets its half-share.

- **Node on a layer not in layer_ordinal_to_k is dropped** `[no tag]` -- places node 1 on layer 31 with no mapping; asserts dropped_nodes==1.

- **Empty mesh yields an all-zero source field, not an error** `[no tag]` -- runs on an empty IrMesh; asserts ok, total_power==0 and every source cell is 0.

### mpkit/tests/material_test.cpp

- **Copper has plausible engineering numbers** `[no tag]` -- queries `copper()`; asserts thermal_conductivity in (350, 450), density > 8000 and resistivity in (1e-8, 3e-8).

- **Air properties present and not copper** `[no tag]` -- queries `air()`; asserts conductivity < 0.1, density < 2, youngs_modulus is NaN.

- **material_by_name is case-insensitive + recognises aliases** `[no tag]` -- looks up "copper", "COPPER", "aluminum" and "aluminium"; asserts each returns the matching material and an unknown name throws.

- **material_names lists the canonical set without alias duplicates** `[no tag]` -- queries material_names; asserts copper/fr4/air are present, exactly one "aluminium" entry, no "aluminum" entry and the list is sorted.

### mpkit/tests/steady_heat_test.cpp

- **1D conduction between two Dirichlet faces matches the linear analytical T(x) = T0 + (T1 - T0) * x / L** `[no tag]` -- 32 cells of copper with walls at 0 and 100 C; asserts every cell-centre T matches the linear profile within 2 C.

- **1D conduction with a uniform volumetric source matches the parabolic T(x) = T_wall + Q L^2 / (8 k) at the centre** `[no tag]` -- 64 cells, Q=1 MW/m^3, walls at 0; asserts the centre T matches Q*L^2/(8k) within 5%.

- **Series copper / FR-4 slab respects harmonic-mean conductivity at the material interface** `[no tag]` -- half copper / half FR-4 slab with walls 100 and 0 C; asserts the cold-side interface cell matches the analytic series-resistance value within 1 C and the hot side is > 99 C.

### mpkit/tests/study_run_test.cpp

- **Single-node study: SteadyHeat with two Dirichlet walls runs and produces a temperature field** `[no tag]` -- runs a 1-node Steady heat study; asserts r.ok, 1 step, ok step, T[last] > 80 and T[0] < 20.

- **Sweep iteration patches the targeted config field per step** `[no tag]` -- adds a SweepSpec over the first bc value with three values; asserts 3 steps with sweep_index[0..2] and the boundary cell tracks the swept value.

- **Unknown PhysicsKind dispatch reports a clear error** `[no tag]` -- runs a study with PhysicsKind::Elasticity; asserts ok==false and the error mentions "not yet dispatched".

- **PdnIrDrop without a Board fails fast with a helpful message** `[no tag]` -- runs a PdnIrDrop study with board=nullptr; asserts ok==false and the error mentions "board is null".

- **Two-axis full factorial sweep walks the cartesian product and records a 2-D sweep_index per step** `[no tag]` -- adds two sweeps (2 vs 3 values); asserts 6 steps and the (i, j) indices follow odometer order with sweep 0 fastest.

### mpkit/tests/study_test.cpp

- **Study sexpr round-trip preserves every field** `[no tag]` -- serialises a Study with 2 nodes + 1 coupling + 1 sweep + 1 result file; asserts every field round-trips through `study_to_sexpr`/`study_from_sexpr`.

- **save_study + load_study on disk round-trips identically** `[no tag]` -- saves a Study to a temp dir; asserts the file exists and loading recovers the name and elasticity node.

- **FieldIO save + load preserves shape, spacing and every value** `[no tag]` -- writes a 4x3x2 Grid+Field and re-loads; asserts every dimension, dx, x0 and every cell value matches.

- **FieldIO rejects bad magic with a clear error** `[no tag]` -- writes a file with garbage magic; asserts `load_field` throws std::runtime_error.

### mpkit/tests/thermoelectric_test.cpp

- **Copper has the expected ~+1.8 uV/K room-temperature Seebeck** `[no tag]` -- queries copper().seebeck_coefficient; asserts the value is in (1e-6, 3e-6).

- **Type-K thermocouple (chromel / alumel) recovers a sensitivity near the published ~41 uV/K average** `[no tag]` -- evaluates thermocouple_emf(chromel, alumel, 100, 0); asserts the value matches 4.37 mV within 0.1 mV.

- **Seebeck EMF along a path is independent of intermediate uniform segments (Magnus principle)** `[no tag]` -- compares one-segment vs five-segment paths between 0 and 100; asserts the two EMFs match within 1e-12 and both equal S*100.

- **Closed thermocouple loop -- hot junction at T_h, cold ends at T_c -- gives (S_a - S_b)*(T_h - T_c)** `[no tag]` -- traverses chromel-alumel loop; asserts v_loop == thermocouple_emf within 1e-12.

- **Peltier coefficient Pi = S * T at the given absolute temperature** `[no tag]` -- evaluates at T=300; asserts the result equals S*T within 1e-15.

- **Peltier junction flux uses Pi_a - Pi_b times the normal current density and flips sign with current direction** `[no tag]` -- evaluates copper/bismuth junction at j=+/-1e5; asserts q(+) + q(-) == 0 within 1e-15 and |q| in (2e3, 2.5e3).

- **Thomson volumetric heat returns zero everywhere under v1's constant-Seebeck assumption (placeholder API)** `[no tag]` -- evaluates on a 2x2x1 grid; asserts dimensions match and every cell is 0.

- **New library materials are reachable through material_by_name** `[no tag]` -- queries iron/chromel/alumel/constantan/bismuth; asserts each returns the right name with a finite Seebeck coefficient.

### mpkit/tests/transient_heat_test.cpp

- **Transient bar with both ends Dirichlet relaxes to the steady linear profile** `[no tag]` -- 32-cell bar with walls 0/100 and initial uniform 50, run for 10*tau1 in 200 steps; asserts every cell matches the linear steady profile within 2 C.

- **Transient bar with insulated ends + sinusoidal initial condition decays at the analytical first-mode rate** `[no tag]` -- 64-cell bar with Neumann walls and cos(pi*x/L) initial; asserts the bar-edge ratio matches exp(-rate1*t_final) within 8%.

### mpkit/tests/voxelizer_test.cpp

- **Empty board produces a single-voxel air grid** `[no tag]` -- voxelizes an empty Board; asserts voxel_count==1 and ids[0]==kAirMaterialId.

- **Tiny board: copper appears, substrate fills the bulk** `[no tag]` -- voxelizes a 3-layer board with one F.Cu segment at 100um cells; asserts air, substrate and copper are all present in ids.

- **Voxelizer grid bbox spans the trace plus the air padding** `[no tag]` -- voxelizes with 3 padding cells; asserts the grid bbox starts before x=1mm - half-width - pad and ends after x=5mm + half-width + pad.

