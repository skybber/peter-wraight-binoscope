Build me an interactive web simulator (one self-contained HTML page, all JavaScript inline, no external libraries, English UI) that predicts how the frame of a 3D-printed **offset binoscope** (the design by Peter Tinkerer / Peter Wraight, "Build a 150mm Offset Binoscope") deflects under its own weight, and how that deflection changes the **collimation difference between the two eyepieces** at high magnification. Model the **original design**:
- aluminium L angles at the outer corners,
- at each of the two middle edges (where the upper and lower tube boxes meet) an **epoxy-glued back-to-back pair of L angles**,
- the box closed by **printed plates**: frame-like plates at the front ends, printed panels in the middle around the bearing, printed boards at the front (top, middle, bottom) that carry the focusers and the secondary holders, and printed base plates for the mirrors at the rear ends.

There are **no transverse printed rings (bulkheads)** and **no X diagonals** in the design. Validate the calculation in Node before building the UI, then publish the page.

## 1. Geometry (units: mm, N, MPa, kg)

- x runs along the tubes from the rear end (x = 0) to the front end. z is vertical when the telescope points at the horizon. y = z × x: **looking at the front of the telescope, +y points to the right**.
- Two stacked boxes: **tube A** (upper, z = H…2H) and **tube B** (lower, z = 0…H), both y = 0…W. W and H are spacings of the profile centrelines. Estimate from a front view scaled by the 150 mm mirror: outer box about 220 × 163 mm per tube, so **W ≈ 207, H ≈ 157** (editable estimates). The plates reach out to the profiles' outer edge, i.e. beyond the centreline by the corner section's centroid distance (6.44 mm for L20×20×4; half the size for a rectangular tube).
- **Optical axes offset sideways** (measured): seen from the front, A (upper) sits left and B (lower) right, e_y = **46 mm**: y_A = W/2 − e_y/2, y_B = W/2 + e_y/2, z_A = 1.5H, z_B = 0.5H, both parallel to x.
- **Front ends aligned.** Tube B starts later at the rear (offset, default 170 mm).
- **Secondaries at different x:** x_s,B = x_s,A + Δx_s, with Δx_s ≈ 39 mm (estimate from a top view).
- Inputs: base plate → primary distance dP (both tubes, 35), sf_A (secondary → focus of A, 180), tube length beyond A's secondary (100), tube B's start offset. Derived:
  - x_PA = dP, x_sA = x_PA + f − sf_A, x_end = x_sA + beyond,
  - x0_B = offset, x_PB = x0_B + dP, x_sB = x_sA + Δx_s, sf_B = f − (x_sB − x_PB).
- **Both focusers point up (+z)** through the top of tube A. B's beam passes up through tube A beside A's secondary. Eyepiece centres: (x_sA, y_A, 1.5H + sf_A) and (x_sB, y_B, 0.5H + sf_B). Show the eyepiece separation, the focal height difference, and a clearance check of B's light cone, (sf_B − H)·D/f wide at A's axis height, against A's secondary (minor axis 35 mm, internal).
- Six longitudinal member lines: A top-left/top-right, B bottom-left/bottom-right, and the middle edges (0, H) and (W, H) spanning the union of both tubes. Where only one box exists, the middle edge is a single L.
- **Open check:** a side view of the original measures tube A ≈ 880 mm and primary-to-secondary ≈ 718 mm at this H, which suggests f ≈ 880–900 mm (F/6) rather than 750 mm. Keep f = 750 as the default, editable, and say so in the notes.

## 2. Sections and materials

- Rectangles {y, z, w, h}: A, centroid, I_y (vertical bending), I_z, J = Σ b·t³(1/3 − 0.21(t/b)(1 − (t/b)⁴/12)). Use axes parallel to the legs and ignore the product of inertia (say so).
- **Corners:** L20×20×4 → A 144, I 5028 mm⁴ (0.50 cm⁴), J 661.
- **Middle edges:** two L20×20×4 bonded with epoxy, treated as **one rigid composite** (no adhesive model): rectangles {0, −20, 4, 40} + {4, −4, 16, 8} → I_y 22016 mm⁴ (2.2 cm⁴), I_z 10055, J 2675.
- Profile types selectable for corners and middle: L, T, rectangular tube, vertical flat bar, custom (A, I_y, I_z, J); the middle edge can also be the glued pair. Aluminium E = 69 000, G = E/2.6, ρ 2.7 (fixed, not in the UI).
- **Printing material** (select list, no manual E): PETG E 2000 / 1.27 g/cm³, PLA 3200 / 1.24, ABS 2100 / 1.04, ASA 2000 / 1.07. One input, the **effective solid share of printed parts** (0.6 default; about 0.3–0.4 for Peter's 20 % infill), multiplies E and ρ of all printed plates and panels, but not the holder ring (thin walls print nearly solid).

## 3. Printed plates (transverse, y–z plane)

Model every plate as a **strip grid**:
- grid lines at about 25 mm (mesh setting), plus extra lines through required points;
- strips along the grid lines as frame elements with band width w: A = w·t, out-of-plane I = w·t³/12/0.91, in-plane I = w·t³/12, J = w·t³/6; the edge bands reach out to the plate's outer edge;
- **X truss bars in every cell** with A_eq = G·t·d³/(2·E·a·b) for in-plane shear;
- mass ρ·t·area per cell, to its corners.

Each plate corner on an L profile gets **its own node, joined to the profile node by a joint spring**: rigid in translation (1e6 N/mm), rotational stiffness **k_j = 3·10⁵ N·mm/rad** ("Plate-to-profile corner joint", 0 = rigid). This keeps an ideal plate from clamping the profiles at a right angle. The plate edges between the corners are otherwise free.

- **Front plates, 10 mm**, both tubes at x_end: continuous frames around a circular light opening centred on each optical axis. Input the **narrowest rim**, 10 mm (measured); the opening = 2·(distance from the axis to the nearest outer edge − rim), about 150 mm. Each strip and cell carries only its sampled area share outside the opening (minimum sliver 2 %; interior nodes with no material are dropped). Warn if the opening is smaller than the mirror.
- **Mirror base plates, 15 mm**, solid, at each tube's rear start. **Push-pull mirror mount:**
  - a central steel bolt (Ø 6, clamped beam) on the optical axis from the plate to the mirror node;
  - three steel collimation screws (Ø 5, axial struts) on the screw radius r_c (55 mm, estimate), **two above and one below the axis** (60°, 180°, 300° from the top), from plate nodes to cell nodes;
  - the cell nodes and the mirror node, at x_P = the base plate plus dP, form a rigid body carrying the primary-with-cell mass;
  - add extra grid lines so the axis point and the three screw points are exact plate nodes.

## 4. Printed panels and boards

Up to **three panel segments**, each with: on/off, material (printed filaments only), x range, thickness, and faces (side faces, top and bottom, middle wall between the tubes). Printed panels are **full plates ("boards")** on their faces: rows at every station inside the range (stations refined to ≤ 25 mm there), across lines at about 25 mm plus lines through the optical axis (y_A for A-top, y_B for B-bottom, both for the middle wall, z_axis for the sides) and through the trunnion points on the side walls; the same strips plus X bars, with the face normal as the reference vector. Boards share nodes with the profiles, i.e. they are glued along their whole length. Defaults:
1. middle side panels, PETG **20 mm**, bearing ± 75 mm, side faces;
2. middle top and bottom plates, PETG 10 mm (estimate), same range, top and bottom;
3. front focuser boards, PETG 10 mm, the last 120 mm, **top, bottom and middle wall**.

Not modelled: the opening in the middle board for B's beam (say so).

## 5. Secondary holder (ring, original)

A printed ring in the cross-section plane (axis parallel to x): **A's ring hangs from A's top board** (hub below it, the reflected beam passes through the ring); **B's ring stands on B's bottom board** (hub above it). The wall can be selected per tube (top, bottom, left, right).

- Wall → axis distance L (H/2 = 78.5). Base pad 3 mm, hub radius r_h 16, ring centreline radius (L − 3 − r_h)/2 ≈ 29.75. Ring wall (radial) 3 mm (estimate). **Ring depth along x: A 20 mm, B 40 mm** (B twice A, because its longer light path makes it more critical). Material from the filament list, solid.
- FE as a **short cylindrical shell**: 24 nodes round × (round(depth/10) + 1) circles, circumferential and axial strips (the strip section above with t = ring wall, radial reference vector) plus X shear bars per cell.
- The shell nodes within ± the **fused arc** (30°) of the top are rigidly joined to a pad node (1 mm towards the wall, so it never coincides with a shell node). The pad joins the wall node through a 2 mm element equivalent to a rotational spring k_b. Default k_b: A 2·10⁶ N·mm/rad (glued); **B 5·10⁵** (split, interlocking base); 0 = rigid. Label it "Holder base joint" and explain it in plain words: 1 N·m tilts the joint by 57.3/(k_b/1000) degrees.
- The shell nodes within ± the fused arc of the bottom are rigidly joined to the hub node on the optical axis, plus 3 rigid helper nodes for the rigid-body fit.
- **The wall node is the board node** at (x_s, axis) on that wall (fallback when there is no board: a 10 × 120 mm strip between the two corner nodes).
- Secondary-with-holder mass at the hub.
- Keep the analytic post holder in the engine only, for the validation tests.

## 6. Altitude bearing (tilted axis)

- The trunnions sit on the middle side panels. From the front view, **the frame is turned clockwise on its axis**: the axis rises to the right at **14.5°** (measured from the trunnion silhouettes) and passes about 50 mm above the middle edge at the box centre line.
- Axis direction a = (0, cos θ, sin θ), world up at the horizon U = x × a = (0, −sin θ, cos θ).
- Support nodes where the axis meets the side walls: (x_b, 0, H + dz − (W/2)·tan θ) and (x_b, W, H + dz + (W/2)·tan θ). These are side-board nodes; fall back to the middle edges with a warning if there is no printed side panel.
- Fix the 3 translations by penalty (1e7 × the largest diagonal) and the rotation about a (penalty on a·aᵀ in the rotational block).
- x_b at the centre of mass. The button "Move bearing to centre of mass" also moves the panel segments that contain the bearing.
- Show the centre of mass relative to the axis (above it and beside it).

## 7. Solver

- 3D Euler–Bernoulli frame elements (12 DOF) with an explicit triad: e_x along the element, e_y = normalise(ref × e_x), e_z = e_x × e_y. ref = z for longitudinal members, the surface normal for plate and board strips, radial for ring strips, x for in-plane rigid links. Plus truss elements and node-to-node springs (6 DOF).
- Stations at all special x (tube starts and end, primaries, both secondaries, bearing, panel limits), merged within 0.6 mm, subdivided to ≤ 50 mm (≤ 25 mm inside board ranges; both are mesh settings in an "Advanced: calculation mesh" group).
- Number the nodes station by station and solve with a **skyline (profile) LDLᵀ** for two load vectors at once.
- **Loads:** self-weight lumped to the nodes. The primary with cell at the mirror node; the secondary with holder at the hub; the focuser with eyepiece (both tubes) at tube A's four corners at that tube's x_s.
- Two unit cases: **horizon**, g along −U (sideways in the box frame when the axis is tilted), and **zenith**, g along −x. Any altitude α is cos α·horizon + sin α·zenith.
- **"Ideal material" switches:** profiles, front plates, mirror base plates, panels, secondary holders, holder plate (the board that carries the holder). An ideal part is 1000× stiffer with unchanged mass; the corner joint springs are not affected.

## 8. Optics (per tube)

- Rigid least-squares fit (translation plus small rotation). Primary: the mirror node and the 3 cell nodes, applied at the vertex (x_P, y_axis, z_axis). Secondary base: the box corners at the tube's own x_s; the hub fit gives the secondary motion, and hub minus base is the "holder share". Focuser: tube A's corners at the tube's x_s.
- Image of the star in direction u: I = P′ + 2f·u′ − f·u. Reflect it in the plane through S′ with normal m′ = m + ω_S × m, m = normalise(u − n), n = +z. Take the residual r = I_r − E′, remove its component along n, and Δs = (I − 2mmᵀ)·r⊥ / f.
- **Sky directions:** horizontal = Δs·a (along the altitude axis), vertical = Δs·U. Apparent = sky × M. Difference A − B relative to the collimation altitude (45°): (cos α − cos α_c)·ΔH + (sin α − sin α_c)·ΔZ.
- Tolerances 4′ vertical and 10′ horizontal apparent at M = 150.

## 9. Validation (Node, also run live in the page as "self-checks")

1. Cantilever: PL³/3EI and PL²/2EI in both planes, including a transverse member.
2. Sections: L 0.50 cm⁴, bonded pair 2.2 cm⁴.
3. A rigid rotation with the real offsets gives A − B ≈ 0.
4. A holder tilt about u × n gives 2θ·sf/f; a shift along u gives δ/f.
5. A lateral primary shift gives s/f.
6. Post holders with e_y = Δx_s = 0 and equal sf: opposite walls give opposite horizontal shifts at zenith; the same wall gives zero difference.
7. A rigid translation gives zero pointing change.
8. Strip-grid plate 200 × 200 × 10, central load, against Kirchhoff (ν = 0.3): clamped 0.0056·Pa²/D, simply supported 0.0116·Pa²/D, accepted up to 20 % soft (no Poisson coupling). One cell in shear gives V·b/(G·t·a).
9. Ring pulled across a diameter: (π/4 − 2/π)·F·R³/(EI), as curved beams and as the cylindrical shell (within 2 %).
10. All parts ideal: A − B ≤ 0.5 % of the real-material value.

Also check numerical robustness (penalty ×100, rigid links ÷10: results within 0.1 %) and run a sweep of input variants with no NaN.

## 10. UI

- **Left column:**
  - a schematic with a side view and a front view (offsets, tilted axis, centre of mass, eyepieces);
  - a preset selector, Reset, Copy JSON and Paste JSON (textarea fallback), and the parameters in localStorage (try/catch) with a model version, so stale saved parameters are replaced, with a notice, when the model changes;
  - collapsible groups whose open header is clearly highlighted (tinted background, accent bar and colour). Groups: Optics and tolerances; Frame geometry (W, H, e_y, Δx_s, bearing x, tilt, axis height); Profiles; Printed plates; Panels (3 segments); Tubes and masses (shared dP and masses for both tubes, sf_A, beyond, B's offset, derived positions); Secondary holder (ideal switches first, then material, walls, ring depths A and B, ring wall, hub radius, fused arc, base joints A and B); Advanced: calculation mesh;
  - custom-only fields are truly hidden (CSS [hidden] { display: none }).
- **Right column:**
  - KPI cards: the largest vertical and horizontal apparent A − B over 0–90° with tolerance colouring, the frame sag, the mass and balance;
  - an altitude table (0–90° in 15° steps);
  - **focal-plane plots** for tube A, tube B and A − B (µm, sky-aligned with apparent-arcminute scales, 15° markers, a slider marker, the Airy disk, a tolerance box on A − B, a raw focuser-view toggle, auto or locked scale);
  - a canvas 3D view of the deformed frame (altitude and exaggeration sliders, drag to rotate; profiles, plates and boards shaded by material share, the ring holders and mirror cells in tube colours, the tilted bearing axis, light paths, the undeformed frame dashed);
  - per-tube optics detail;
  - a model section: centre of mass relative to the axis, masses, section properties, derived positions, front openings, effective plate material, a **"What each part contributes"** table (button; each part made ideal in turn), notes and limitations, and the self-check results.
- Light and dark mode, dot decimals, no alert, confirm or prompt dialogs.

## 11. Presets

1. **Peter's original 150 mm F5:** everything above, with: f 750, D 150, M 150, collimation at 45°; W 207, H 157, e_y 46, Δx_s 39; dP 35, sf_A 180, beyond 100, B offset 170; masses 1.1 / 0.08 / 0.5 kg per tube; PETG with solid share 0.6; rings 20/40 × 3 mm, hub 16, fused arc ±30°, k_b 2e6 / 5e5; tilt 14.5°, axis 50 mm above the middle; bearing at the centre of mass.
2. **200 mm F5 scaled 1:1** (same profiles and plate thicknesses): f 1000, D 200; W, H, e_y, Δx_s, r_c and the ring dimensions × 4/3 (276, 209, 61, 52, 73; rings 27/53 × 4, hub 21); sf_A 205, beyond 120, B offset 230; masses 2.3 / 0.12 / 0.6 kg; axis 67 mm above the middle.
3. **200 mm improved:** as 2, but both rings hang from the wall above their tube (B from the middle board) with rigid bases.

## 12. Facts from Peter and the user to respect

- 20 % infill; the whole unit with light covers weighs about 5 kg (the model is heavier, so the masses are estimates).
- The side, top and bottom plates holding the bearing are glued firmly to the frame, and so is the plate holding the eyepiece holder. The front end plates are glued and screwed. The plate holding the lower secondary is glued to the frame.
- The user observes a real vertical A − B about 10× smaller than the model's (about 500′ with the 20/40 mm rings). The ring holder and its board are the main suspects; mark their dimensions as estimates and show the contributions table.

Mark all unmeasured dimensions as editable estimates. When the model is built and validated, report the key results of the three presets side by side.
