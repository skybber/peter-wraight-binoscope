Build me an interactive web simulator (one self-contained HTML page, all JavaScript inline, no external libraries, English UI) that predicts how the frame of a 3D-printed **offset binoscope** (the design by Peter Tinkerer / Peter Wraight, "Build a 150mm Offset Binoscope") deflects under its own weight, and how that deflection changes the **collimation difference between the two eyepieces** at high magnification. Model the **original design**: aluminium L angles at the outer corners, and at each of the two middle edges (where the upper and lower tube boxes meet) a **glued back-to-back pair of L angles**, so 2 pairs = 4 L angles in the middle in total. Validate the calculation before building the UI, then publish the page.

## Geometry (units: mm, N, MPa, kg)

- x runs along the tubes from the rear end (x = 0, primary mirrors) to the front end. y is horizontal across the frame and is the altitude axis. z is vertical when the telescope points at the horizon.
- Two rectangular boxes are stacked: **tube A** (upper, z = H … 2H) and **tube B** (lower, z = 0 … H). Both have width W (spacing of the profile centrelines in y) and height H.
- The **front ends of both tubes are always aligned** and both secondaries lie in the same plane (same x). Tube B starts later at the rear, by at least the mirror diameter plus a margin, because the light path from its secondary to its eyepiece is longer by exactly that amount.
- Inputs per tube: rear start x0 and the distance from the start to the primary. Tube A additionally takes the secondary-to-focus distance and the tube length beyond the secondary. Derive the secondary position as x_s = x_primary + f − (secondary→focus). Tube B uses the same x_s and tube end as A, so its secondary→focus = f − (x_s − x_primary,B). Show the derived positions in the UI.
- There are six longitudinal member lines: four outer corners (A top-left/top-right spanning tube A, B bottom-left/bottom-right spanning tube B) and two shared middle edges at (y = 0, z = H) and (y = W, z = H) spanning the union of both tubes.

## Members and cross-sections

- **Corners:** equal L angle a × a × t (original: L20×20×4). Compute A, I_y, I_z and J from rectangles. Use moments of inertia about axes parallel to the legs, i.e. bending restrained by bulkheads and panels, and say so in the notes.
- **Middle edges, the original glued pair:** at each middle edge the upper box's L has its vertical leg pointing up, the lower box's L has its vertical leg pointing down, and their horizontal legs are stacked and glued (cross-section like ⊣). Compute the section two ways:
  - fully bonded: one composite section about the common centroid,
  - unbonded: the sum of the two L sections, each about its own centroid.

  Add a **bond efficiency** input (0 = loose, 1 = fully bonded; default 1 for epoxy or plastic welder) that interpolates I and J linearly between those two cases. Reference values to check: L20×20×4 has I ≈ 0.50 cm⁴, the unbonded pair ≈ 1.0 cm⁴, the bonded pair ≈ 2.2 cm⁴ (vertical bending).
- Let the user switch both the corner and the middle members to other profile types for comparison: L, T (vertical part a × horizontal b), rectangular tube, vertical flat bar, custom (A, I_y, I_z, J). Aluminium E = 69 000 MPa, G = E/2.6, density 2.7 g/cm³.
- **Printed bulkheads:** rectangular rings joining the corners of each box at given x positions, modelled as beams with a rectangular section (thickness along x × width of the load-bearing band), plastic E (PLA ≈ 3000 MPa, PETG ≈ 2000 MPa) and a mass per bulkhead. Add rings automatically at the tube ends, at the bearing, at each primary and at each secondary. The rings at the primaries and secondaries carry no extra mass. Optional checkbox: end frames made of aluminium profiles with the corner section.
- **Panels** (sheet on the box faces between x = P0 and P1, on the sides and optionally on top and bottom): replace them with equivalent X diagonals (pin-ended truss bars) in every bay between nodes, with area A_eq = G·t·d³ / (2·E·a·b), where a is the bay length, b the face height and d the diagonal. This reproduces the panel's shear stiffness. Panel mass is distributed to the bay corners. Note in the UI that only shear stiffness is modelled, so the user must enter E = 2.6 × the in-plane shear modulus. For plywood that means G ≈ 600 MPa, so E ≈ 1600, not 8000.
- **Optional real X diagonals** (flat bar, area A_d) in a segment: convert them to an equivalent panel thickness t_eq = 2·E·A_d·cos²α·b / (d·G·a) and reuse the panel code.

## Structural solver

- 3D Euler–Bernoulli frame elements (12 DOF: 3 translations and 3 rotations per node), standard local stiffness matrix, transformed by an explicit local axis triad: the reference vector is global z for longitudinal members and global x for bulkhead members. Truss elements carry axial stiffness only.
- Mesh stations: every special x (tube ends, bulkheads, panel and diagonal limits, primaries, secondaries, bearing), subdivided so that no element is longer than a user setting (default 50 mm). Number the nodes station by station so the matrix is banded, and solve with a banded Gaussian elimination for two load vectors at once.
- **Support:** the altitude bearing at x_b, as two nodes on the two middle edges. Fix all three translations and the rotation about the altitude axis (y) by penalty (≈ 1e7 × max diagonal), as if held by a clutch or friction.
- **Loads:** self-weight lumped to the nodes (profiles, bulkheads, panels), plus per tube: primary with cell at the four box corners at x_primary, and secondary with holder plus focuser with eyepiece at the four corners at x_secondary. Two unit load cases: horizon (g along −z) and zenith (g along −x). For any altitude α the result is cos α · horizon + sin α · zenith.

## Optics (per tube)

- Least-squares rigid-body fit (translation plus small rotation) of the ring nodes at the primary gives the primary vertex shift dP and tilt ω_P. Add an optional lateral primary-mount compliance (µm/N, e.g. silicone in shear) times the gravity force perpendicular to the axis. The same fit at the secondary ring gives the motion of the secondary base and of the focuser.
- **Secondary holder:** a hollow printed cylinder (outer diameter, wall thickness, E) cantilevered from a selectable wall (left, right or outer) to the tube axis. Its length is W/2 for side walls and H/2 for the outer wall. Add a rotational base stiffness k_b (N·mm/rad, 0 = rigid). For the gravity component F⊥ perpendicular to the holder axis a, use:
  - tip shift δ = F⊥·(L³/(3EI) + L²/k_b),
  - tip rotation ω = (a × F⊥)·(L²/(2EI) + L/k_b).

  The original design has the two holders on **opposite walls**, and the holder of one eyepiece sits on a split (interlocking) base, so give that tube a lower default k_b.
- **Image model:** primary paraboloid with axis u = +x and focal length f. The image of a star in direction s is P′ + 2f·u′ − f·s. Reflect it in the flat secondary (plane through S′ with normal m′ = m + ω_S × m, where m = normalise(u − n) and n is the direction from the secondary to its focuser). Take the residual r between the image and the moved eyepiece centre E′, remove its component along n, and convert it to a change of pointing on the sky: Δs = M_r·r⊥ / f, with M_r = I − 2mmᵀ.
- **Output:** the pointing change of each tube relative to the collimation altitude, and the **difference A − B**:
  - horizontal on the sky = y component, vertical on the sky = z component, in arcseconds,
  - apparent = sky × magnification, in arcminutes.

  Compare with user tolerances: default 4′ vertical and 10′ horizontal apparent, since eyes tolerate vertical misalignment worst.

## Validation (run in Node before building the UI, and report the results)

1. A single cantilever beam gives tip deflection and rotation equal to PL³/(3EI) and PL²/(2EI) in both bending planes.
2. Section values match the references above (L20×20×4, unbonded and bonded pair).
3. A rigid rotation of the whole frame gives identical pointing changes in both tubes, so the differential is zero.
4. A secondary-holder tilt θ on a rigid frame gives a sky shift of 2θ·(secondary→focus)/f combined with the shift δ/f.
5. A lateral primary shift s gives a sky shift of s/f.
6. Mirror-symmetric holders (opposite walls) give opposite horizontal shifts at zenith, and holders on the same wall cancel them (for equal light paths).

## UI

- **Left column:** a small schematic of the coordinates, a preset selector, and collapsible input groups (optics and tolerances, frame geometry, profiles including bond efficiency, printed bulkheads, panels, diagonals, tube A, tube B, secondary holder) with short hints, plus buttons to reset to the preset and to copy or paste the parameters as JSON (fall back to a textarea when the clipboard is unavailable). Remember the parameters in localStorage, wrapped in try/catch.
- **Right column:**
  - KPI cards: the largest vertical and horizontal apparent difference over 0–90°, with the altitude where it occurs and colour coding against tolerance,
  - a table by altitude (0–90° in 15° steps): H and V on the sky (″) and apparent (′),
  - a canvas 3D view of the deformed frame with an altitude slider and a deflection-exaggeration slider (profiles, bulkheads, panels or diagonals and bearing in different colours, undeformed frame dashed),
  - per-tube optics detail: primary shift and tilt, secondary shift and tilt (and the holder's share), focuser shift, image shift at focus in µm, pointing change,
  - a model section: centre of mass with a warning when the bearing is more than 30 mm away from it, masses by category, section properties, derived tube positions, and a "how the model works and its limitations" list (rigid joints, so real values are likely worse; no creep or thermal effects; panels in shear only; best used for comparing variants).
- **Presets:**
  1. Peter's original 150 mm F5: L20×20×4 corners, glued L20×20×4 pairs in the middle, printed bulkheads, a 1 mm aluminium panel around the bearing only, holders on opposite walls, and the split base on one tube. Suggested starting values to tune: f = 750, W = H ≈ 185, tube B starts 170 mm later, primary 35 mm from the tube start, secondary→focus ≈ 180, 100 mm of tube beyond the secondary, primary with cell 1.1 kg, secondary with holder 0.08 kg, focuser with eyepiece 0.5 kg, bulkhead 0.15 kg, bearing near the centre of mass.
  2. The same design scaled to 200 mm F5 1:1 (same profiles): f = 1000, W = H = 230, tube B starts 230 mm later, primary with cell 2.3 kg, secondary 0.12 kg, focuser 0.6 kg, bulkhead 0.25 kg.
  3. The 200 mm version with simple improvements: holders on the same wall with rigid bases, and X diagonals in the rear segment.
- Works in light and dark mode, uses dot decimals, and has no alert, confirm or prompt dialogs.

Mark all dimensions I have not measured as editable estimates. When the model is built and validated, tell me the key results of the three presets side by side.
