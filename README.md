# Halbach type simulations
In this repository there are two web based simulators for calculating and visualizing the magnetic fields in multiple dipole systems. The first one is a derivation of the known Halbach array in which multiple segments, with defined orientation, lead to a direction controlled magnetic field. The second one is an extention of the Halbach model by using a fullerene-like arrangement of dipoles, distributed along the radii of the truncated icosaedron (I have no ideea if this second setup was described before, but it's not such a big discovery as it is an extention of multiple dipoles idea... we can think of other geometries, maybe Lebedev, or even quasi-crystal like sytems to avoid harmonic perturbations. See Notes at the end.)

# 1. Halbach Ring Simulator

The Halbach.html is an interactive, web-based 3D simulator for calculating and visualizing the magnetic fields of finite-length Halbach cylinders. This program was initially created in Labview dyring several years, 2014-2018 for a Hall measuring system (see my repos) and upgraded recently with a local Qwen-Coder 70b and further improved with Gemini 3.1 and Claude Sonnet 4.6.
With the version 2, the ability to use two concentric rings makes it a powerful tool to build your own magnets.

## A. Mathematical Model

This simulator does not rely on infinite-length approximations. It computes the exact 3D magnetic field for discrete, finite-length cylindrical magnets using the **Equivalent Surface Current Method** and **Biot-Savart integration**.

### 1. Equivalent Current Solenoid
A uniformly magnetized cylinder with remanence $B_r$ can be modeled as a solenoid carrying an azimuthal surface current density:

$$K = \frac{B_r}{\mu_0} \text{ [A/m]}$$

### 2. Single Ring Field (Elliptic Integrals)
The field of a single infinitesimally thin current ring of radius $a$ is calculated using the exact closed-form expressions derived from the Biot-Savart law. For a point at radial distance $\rho$ and axial distance $\zeta$ from the ring, the field requires complete elliptic integrals of the first ($K$) and second ($E$) kind:

$$B_{axial} = \frac{\mu_0 I}{2\pi \sqrt{Q}} \left[ K(m) + E(m) \frac{a^2 - \rho^2 - \zeta^2}{D} \right]$$

$$B_{radial} = \frac{\mu_0 I}{2\pi \rho \sqrt{Q}} \left[ -K(m) + E(m) \frac{a^2 + \rho^2 + \zeta^2}{D} \right]$$

$$Q = (a+\rho)^2 + \zeta^2$, $D = (a-\rho)^2 + \zeta^2$$
and
$$m = \frac{4a\rho}{Q}$$

### 3. Finite Cylinder Integration
To find the field of a solid cylindrical magnet, the simulator numerically integrates `N_RINGS` (now set at 64) discrete current loops along the cylinder's axis from $-L/2$ to $+L/2$. This captures the severe 3D end-leakage (fringe fields) that occurs in short arrays.

### 4. Halbach Superposition
For an array of $N$ magnets, the angular position of the $i$-th magnet is $\theta = i \frac{2\pi}{N}$. Its magnetization angle $\phi$ is determined by the Halbach order $k$:
$$\phi = k \theta$$
* $k=1$: Confined internal field (no field in bore, strong outside).
* $k=2$: Uniform dipole field across the bore.
* $k=3$: Quadrupole field (linear gradient).

The global field vector is the linear superposition of the rotated field vectors from all $N$ cylinders.

## B. Core Functions

### Physics Kernel
* **`ellipticKE(m)`**
  Computes the complete elliptic integrals $K(m)$ and $E(m)$ simultaneously using the fast-converging Arithmetic-Geometric Mean (AGM) algorithm.
* **`ringField(a, I, rho, zeta)`**
  Returns the $\{B_{axial}, B_{radial}\}$ components for a single current loop in its local cylindrical frame.
* **`cylinderField(px, py, pz, cx, cy, cz, mux, muy, muz, a_mm, halfL_mm, Br)`**
  Handles the coordinate transformation. It translates the evaluation point `(px, py, pz)` into the local frame of a cylinder located at `(cx, cy, cz)` with magnetization vector `(mux, muy, muz)`. It integrates `ringField` along the length and transforms the resulting local vector back into global $\{B_x, B_y, B_z\}$.

### Global Summation
* **`fieldVectorAt(x, y, z)`**
  Iterates through all $N$ poles. Calculates each magnet's physical position in the array and its Halbach magnetization vector. Sums the contributions from `cylinderField` and returns the net 3D magnetic field vector `{Bx, By, Bz}`.
* **`fieldAt(x, y, z)`**
  A utility wrapper that returns the scalar magnitude ($||B||$) of `fieldVectorAt`.

### User Interface & Projections
* **`computeStats(...)`**
  Evaluates the field at specific geometrical points to populate the UI (e.g., center bore field at `(0,0,0)`, and circular sampling around the holder perimeter to find peak leakage).
* **`drawSingleMap(plane)`**
  Renders the 2D cross-sectional heatmaps (XY, XZ, YZ). It generates an off-screen bitmap, computes the field scalar at each pixel, maps it to a color gradient, and calculates continuous isolines. It also overlays projected 2D vectors or out-of-plane flux indicators ($\odot$, $\otimes$).


# 2. Magic Sphere — 48-Dipole Simulator (*Mandhala*)

`Mandhala.html` is an interactive 3D simulator for designing and optimizing a spherical arrangement of permanent magnets placed on a truncated C60 (fullerene) lattice. It models 48 cylindrical dipoles on a spherical shell and computes the resulting magnetic field using the same Biot–Savart formulation as the Halbach ring simulator. 

The 48 dipoles are what remains of the 60 C60 vertices after removing the two hexagonal caps closest to the ±Z poles (the "6–6 fulvalene analogue" atoms). This carves out a clear bore along the Z axis for inserting a sample, while preserving the C2 symmetry of the lattice. Half of the 48 dipoles are "North-in" and half are "South-in" — the polarity pattern is assigned by hemisphere (x ≥ 0 or x < 0) so that the net field at the center points along +X. The design intent is a strong, homogeneous field in the X direction for Hall measurements.

Compared to a commercial Halbach cylinder (typically €1k–10k, with a hard compromise between field strength and bore size), this configuration is cheap to prototype: the holder prints in PLA/PETG on a desktop 3D printer and this tool lets us iterate on the geometry before cutting material. Beyond the lab use case, the simulator is also useful for teaching — the effect of each design choice on field homogeneity and strength is immediate and visual.

---

## A. Features

### Physics-Based Simulation
Same full 3D magnetic field engine as the Halbach simulator: finite cylinder magnets modeled as stacked surface-current rings, each ring solved analytically via the complete elliptic integrals $K(m)$ and $E(m)$ (AGM algorithm). The ring integration uses `N_RINGS = 32` loops per magnet along the cylinder axis. The total field at any point is the linear superposition over all 48 dipoles.

The UI reports |B|, Bx, By, Bz at the center, and two dimensionless homogeneity metrics over the test volume:
- **Bx spread** — peak-to-peak variation of Bx as a percentage of center |Bx|
- **YZ leakage** — worst-case transverse component √(By²+Bz²) as a percentage of center |Bx|

Because each Cartesian component of **B** is harmonic in the source-free interior (∇²Bᵢ = 0), by the maximum principle its extrema lie on the boundary. Sampling the field on the surface of the test sphere therefore captures the worst-case inhomogeneity inside the whole volume — no volumetric integration needed.

### Geometry and Configuration
48 dipoles of specified diameter and length are placed on a C60-derived spherical lattice, each oriented radially toward the center. User-adjustable parameters:

- **Inner R (mm)** — radius of the bore (sample space). Auto-clamped upward if too small: given the dipole diameter and the desired minimum gap, the tool computes the geometric minimum inner radius from the smallest angle between adjacent lattice directions, and bumps the value if the user enters something smaller.
- **Outer R (mm)** — outer shell radius (bounds the full assembly). Auto-clamped to at least `Inner R + magnet length`.
- **Test R (mm)** — radius of the spherical volume over which homogeneity is evaluated.
- **Dipole diameter / length (mm)** — cylindrical magnet dimensions.
- **Min gap (mm)** — construction clearance between adjacent magnets.
- **Remanence Br (T)** — typically 1.2–1.4 for NdFeB N40–N45.

All dipoles initially sit flush against the inner sphere (offset = 0) with radial orientation. The optimizer moves each magnet radially outward (offset ∈ [0, R_outer − R_inner − L]) without changing orientation or polarity, and in a symmetrical manner: mirror pairs across the YZ plane receive the same offset, preserving the X-axis symmetry of the field

### Visualization
- Interactive 3D scene (Three.js) with orbit/pan/zoom, camera presets (3D, Top +Z, Front +Y, Side +X), and an optional dipole-numbering overlay.
- An axes HUD gizmo in the bottom-right corner shows orientation at all times.
- Visual elements rendered: inner sphere, outer shell, yellow test-volume wireframe, and the 48 dipoles with color-coded N (red) / S (blue) face caps.

### Field Mapping
- 2D |B| heatmaps in the XY, XZ, YZ planes, with colorbar and axis ticks in millimeters.
- Bore profile: Bx, By, Bz along the Z axis over the test range.
- Schlegel diagram (stereographic projection from the Z axis) showing the C60 lattice topology and current polarity of each dipole, included in the PDF report.
- All maps recompute live as parameters change, with input debouncing so the heavy field evaluation isn't triggered on every keystroke.

### Optimization (Simulated Annealing)
Parallel simulated annealing in Web Workers. The tool auto-detects `navigator.hardwareConcurrency` and spawns up to 8 independent SA threads, each seeded with a different initial temperature multiplier so they explore different regions of the search space.

- User-adjustable: **iterations** (per thread) and **max radial nudge** (mm).
- **Objective weight slider** — a single slider from pure homogeneity (left) to pure field (right). The score is a *positive residual*:

$$\text{score} = (1-w) \cdot P_{homog} + w \cdot P_{field}$$

where both penalties are positive and on a comparable percentage scale:

$$P_{homog} = \frac{\Delta B_x}{|B_x|} \cdot 100 + \frac{B_{\perp,\max}}{|B_x|} \cdot 200$$

$$P_{field} = \max\!\left(0,\; \frac{B_{ideal} - |B_x|}{B_{ideal}}\right) \cdot K$$

The field penalty is expressed as a **shortfall from the Halbach theoretical ceiling**,
$B_{ideal} = B_r \cdot \ln(R_{outer} / R_{inner})$ (Mallinson / Abele infinite-dipole limit). This is dimensionless and tops out at 100 when |Bx| = 0, making the slider's in-between values physically meaningful rather than unit-mismatched. The constant `FIELD_WEIGHT_SCALE = K = 100` sets the exchange rate between the two terms; it is exposed in the PDF report for reproducibility. A score of 0 would mean "perfect homogeneity and hit the Halbach ceiling simultaneously".

At the slider endpoints the objective reduces to the classical cases: w=0 is pure homogeneity minimization, w=1 is maximizing |Bx| (monotonically equivalent to minimizing the shortfall, since B_ideal is constant during optimization). The default position is 50/50.

- **Features of the SA engine:**
    - *Automatic T₀ tuning* — each worker samples 50 random moves at startup to estimate a typical energy gap, then sets T₀ so that these moves are accepted ~90% of the time initially. If you want "hotter", change the parameter 0.98 in __params.T0 = (-avgDe / Math.log(0.98)) * threadMult;__
    - *Symmetric-pair mutation* — because the target field is along +X, the lattice has a natural mirror symmetry across the YZ plane. Each worker identifies these mirror pairs at startup and, when nudging a dipole, applies the same radial offset to its mirror partner. This halves the effective search-space dimensionality and enforces X-axis symmetry of the solution.
    - *Mixed local/global moves* — each iteration performs a symmetric nudge on 1–5 random pairs (90% of iterations), or a uniform global radial shift of all 48 magnets (10%). The nudge amplitude shrinks as temperature cools.
    - *Parachute escape* — if no improvement is found over 20% of the total iterations, all magnet offsets are scrambled to random positions within their allowed range and temperature is reset to T₀. Prevents the search from stalling in a poor local minimum.
    - *Geometric constraints enforced per iteration* — each offset is clamped to respect both the minimum gap (magnets may not collide with one another as they move outward) and the outer-shell envelope.
    - *Live progress display* — an overlay shows current average iteration, current temperature, the running best score, and the score of one representative thread.
    - The 3D scene is updated during optimization at ~4 Hz so the user can watch the dipoles migrate.

### Export
Multi-page PDF report via jsPDF, generated from the current state:
1. **Parameters page** — geometry, computed field at center, SA settings including the objective weight decomposition (e.g. "Mixed: 70% homogeneity + 30% field-shortfall") and the computed Halbach ideal in Tesla.
2. **3D views** — Top, Front, Side renders with dipole numbering, each on its own page, rendered at 1600×1600 off-screen so the sphere fills the page cleanly regardless of browser window shape.
3. **Field maps** — bore profile, XY/XZ/YZ heatmaps, and the Schlegel diagram with per-dipole polarity color-coding.
4. **Dipole table** — for each of the 48 dipoles, the absolute radial positions of its N and S pole faces in millimeters from the center. This is the build sheet for machining or 3D-printing the holder.

---

# Notes
## Beyond $C_{60}$: Quasicrystals and Aperiodic Magnetic Geometries

Exploring geometries beyond the $C_{60}$ (truncated icosahedron) lattice makes physical sense for improving field homogeneity. While the $C_{60}$ structure is nice, its high degree of **discrete rotational symmetry** is exactly what dictates the amplitudes of the harmonic distortions in the bore. I was exploring other fullerenes, like C84 but it's quite similar, and even more difficult to analyze. So, maybe we need to avoid a too symetrical ource distribution, the field appear as specific high-order multipole moments rather than a smooth, diminishing decay.


### 1. The Harmonic Problem with $C_{60}$
In the current Mandhala setup, 48 dipoles are placed on an Ih lattice. Because the magnets are discrete and follow a regular tiling, the magnetic potential $\Phi$ inside the bore is a solution to Laplace's equation $\nabla^2 \Phi = 0$, in spherical harmonics:

$$\Phi(r, \theta, \phi) = \sum_{\ell=0}^{\infty} \sum_{m=-\ell}^{\ell} A_{\ell m} r^\ell Y_{\ell m}(\theta, \phi)$$

The coefficients $A_{\ell m}$ represent the "purity" of the field. High-symmetry lattices like the icosahedral group ($I_h$) eliminate many low-order terms, but they "stack" the remaining error into specific, high-amplitude harmonic spikes.

### 2. The Case for Fibonacci Spheres
A **Fibonacci Lattice** (or Golden Spiral) on a sphere is one of the most efficient ways to distribute $N$ points nearly equidistantly. Unlike the $C_{60}$ lattice, it is aperiodic (C60 has symmetry but it is not that bad, because we can move the magnets radially though).

* **Spectral Whitening:** By breaking the discrete rotational symmetry, the harmonic distortion is effectively "spread" across the entire spectrum. Instead of having a massive 6th-order distortion, the result should be in many tiny, negligible errors across all orders.
* **Variable Density:** I am exploring to "cluster" more dipoles near the "equator" (relative to the $X$-axis field direction) to compensate for the $1/r^3$ fall-off of individual dipoles more effectively than a rigid lattice (theoretically is possible but building one is another story). 

### 3. Lebedev Quadrature Grids
Lebedev grids are designed to integrate polynomials on the surface of a sphere with high precision. 
* If we place magnets at Lebedev points and weight their "strengths" (by varying the distance), we can decrease the multipole moments. This would turn the Mandhala simulator into a **Spherical Harmonic Whitening**. I am working currently on a Lebedev 110 solid.

### 4. Practical Implementation: The "Aperiodic" Trade-off
Fibonnacci and Quasicrystal are probably very difficult to optimize (need a lot of bumpings and large temperature ranges in SA; probably should work with a genetic algo or a swrm). Assembly for the "aperiodic" stuff is difficult (machining is a mess), 3D printing might be a solution but not sure if the PLA is strong enough. Besides, if I want high temperature, only PEEK is adequate.
Finaly, we may might find that the "best" geometry for a 1% homogeneity volume isn't a known crystal structure at all, but a "magnetic glass" configuration.
   

---

last updated 19 April 2026
