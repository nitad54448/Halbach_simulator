# Halbach type simulations
In this repository there are two web based simulators for calculating and visualizing the magnetic fields in multiple dipole systems. The first one is a derivation of the known Halbach array in which multiple segments, with defined orientation, lead to a direction controlled magnetic field. The second one is an extension of the Halbach model by using a radial arrangement of dipoles distributed along spherical lattices (C60 fullerene, Lebedev grids, and Fibonacci spheres).

# 1. Halbach Ring Simulator

The `Halbach.html` is an interactive, web-based 3D simulator for calculating and visualizing the magnetic fields of finite-length Halbach rings. This program was initially created in LabVIEW during 2014–2018 for a Hall measuring system (see my repos) and upgraded recently with a local Qwen-Coder 70b, then further improved with Gemini 3.1 and Claude Sonnet 4.6 / Opus 4.7.

Version 2 adds two concentric rings with independent geometry and an optional inter-ring phase optimizer, making it a practical tool for designing shimmed or compound Halbach magnets. Version 2.5 extends the model to **per-ring magnet shape and pole count** — so Ring 1 can be eight cylindrical magnets while Ring 2 is twelve parallelepiped tiles — along with a **STEP (AP214) exporter** for direct import into CAD, and a proper shape-aware **collision check** with a user-defined minimum gap.

## A. Mathematical Model

This simulator does not rely on infinite-length approximations. It computes the exact 3D magnetic field for discrete, finite-size magnets using the **Equivalent Surface Current Method** with **Biot–Savart integration** for cylinders, and the **Furlani closed-form scalar-potential model** for parallelepipeds.

### 1. Cylindrical magnets — equivalent current solenoid
A uniformly magnetized cylinder with remanence $B_r$ is modeled as a solenoid carrying an azimuthal surface current density:

$$K = \frac{B_r}{\mu_0} \text{ [A/m]}$$

The field of a single infinitesimally thin current ring of radius $a$ is given by the exact closed-form Biot–Savart expressions. For a point at radial distance $\rho$ and axial distance $\zeta$ from the ring, the field uses the complete elliptic integrals of the first ($K$) and second ($E$) kind:

$$B_{axial} = \frac{\mu_0 I}{2\pi \sqrt{Q}} \left[ K(m) + E(m) \frac{a^2 - \rho^2 - \zeta^2}{D} \right]$$

$$B_{radial} = \frac{\mu_0 I}{2\pi \rho \sqrt{Q}} \left[ -K(m) + E(m) \frac{a^2 + \rho^2 + \zeta^2}{D} \right]$$

$$Q = (a+\rho)^2 + \zeta^2 \qquad D = (a-\rho)^2 + \zeta^2 \qquad m = \frac{4a\rho}{Q}$$

To find the field of a solid cylindrical magnet, the simulator numerically integrates `N_RINGS = 32` discrete current loops along the cylinder's axis from $-L/2$ to $+L/2$. This captures the severe 3D end-leakage (fringe fields) that occurs in short arrays.

### 2. Parallelepiped magnets — Furlani analytical model
For cuboid magnets, the simulator uses the closed-form expressions by E. P. Furlani (*Permanent Magnet and Electromechanical Devices*, 2001). The scalar magnetic potential of a uniformly magnetized rectangular block is integrated over its six charge faces, yielding an exact, singularity-free expression for all three components of $\vec{B}$ at any external point. No numerical integration is used — this path is strictly analytical.

### 3. Halbach superposition
For a single ring of $N$ magnets of order $k$, the angular position of the $i$-th magnet is $\theta_i = i \cdot (2\pi/N)$ and its magnetization direction lies in the XY plane at angle:

$$\varphi_i = k \cdot \theta_i$$

* $k=1$: Confined internal field (no field in bore, strong outside).
* $k=2$: Uniform dipole field across the bore.
* $k=3$: Quadrupole field (linear gradient).
* $k=4$: Sextupole.

For two rings, each ring has its own pole count $N_r$ and an optional **phase offset** $\Delta_r$ that rigidly rotates the entire pattern — positions *and* moment directions — around the Z axis:

$$\theta_{r,i} = i \cdot \frac{2\pi}{N_r} + \Delta_r \qquad \varphi_{r,i} = k \cdot \theta_{r,i}$$

The global field at any evaluation point is the linear superposition of the contributions from every magnet in every ring, computed through the appropriate kernel (cylinder or cuboid) for that ring's shape.

## B. Core Functions

### Physics kernels
* **`ellipticKE(m)`** — computes the complete elliptic integrals $K(m)$ and $E(m)$ simultaneously using the fast-converging Arithmetic–Geometric Mean (AGM) algorithm.
* **`ringField(a, I, rho, zeta)`** — returns the $\{B_{axial}, B_{radial}\}$ components for a single current loop in its local cylindrical frame.
* **`cylinderField(px, py, pz, cx, cy, cz, mux, muy, muz, a_mm, halfL_mm, Br)`** — handles the coordinate transformation for a cylindrical magnet. It translates the evaluation point into the local frame of a cylinder centered at `(cx, cy, cz)` with radial magnetization vector `(mux, muy, muz)`, integrates `ringField` along the cylinder length, and transforms the local result back into global $\{B_x, B_y, B_z\}$.
* **`cuboidField(px, py, pz, cx, cy, cz, mux, muy, muz, hW, hH, hL, Br)`** — Furlani analytical formula for a parallelepiped of half-widths $(h_W, h_H, h_L)$. Returns $\{B_x, B_y, B_z\}$ directly in global coordinates.

### Global summation and geometry
* **`_sumField(x, y, z, params)`** — loops over both rings (and every magnet within each ring), reading the ring-specific shape, pole count, placement radius, phase offset and dimensions, then calls the appropriate kernel and accumulates the vector field.
* **`fieldVectorAt(x, y, z)`** / **`fieldAt(x, y, z)`** — convenience wrappers returning the vector or scalar magnitude of $\vec{B}$.
* **`checkCollisions(params, minGap)`** — shape-aware clearance test based on the Separating Axis Theorem applied to each magnet's oriented bounding box (radial × tangential × axial half-extents), evaluated pairwise. Correctly handles mixed shapes and unequal pole counts between rings, honours the user's **Min Gap** setting, and reports the worst offending pair in the legend and tooltip.

### User Interface & Projections
* **`computeStats(...)`** — evaluates the field at specific geometrical points to populate the UI (center bore field at $(0,0,0)$, peak leakage around the holder perimeter, variance across a user-defined test sphere).
* **`drawSingleMap(plane)`** — renders the 2D cross-sectional heatmaps (XY, XZ, YZ). Generates an off-screen bitmap, computes the field scalar at each pixel, maps it to a color gradient, and calculates continuous isolines. Overlays projected 2D vectors or out-of-plane flux indicators ($\odot$, $\otimes$).

## C. Exports
* **PDF report** — parameters, computed stats, 3D render, 2D heatmaps and collision status in a single document.
* **STEP (AP214)** — pure-JavaScript STEP writer emitting `MANIFOLD_SOLID_BREP` entities for every visible magnet and the holder. Cylinders are built from two circular `ADVANCED_FACE` caps and a lateral `CYLINDRICAL_SURFACE`; cuboids are six planar faces; the holder is a tube with annular top and bottom faces. The file opens directly in FreeCAD, SolidWorks, and Fusion 360.

---

# 2. Mandhala — Radial Sphere Simulator

`Mandhala.html` is an interactive 3D simulator for designing and optimizing a spherical arrangement of permanent cylindrical magnets. It models radially oriented dipoles on various spherical lattices and computes the resulting magnetic field using the same exact Biot–Savart formulation as the Halbach ring simulator.

The design intent is to produce a strong, homogeneous field along the X direction (for Hall measurements) while keeping a clear bore along the Z axis for inserting a sample. To achieve this, the polarity pattern is assigned by hemisphere (e.g., $x \geq 0$ facing inward, $x < 0$ facing outward) so that the net field at the center points entirely along +X.

Compared to a commercial Halbach cylinder (typically €1k–10k, with a hard compromise between field strength and bore size), this configuration is cheap to prototype: the holder prints in PLA/PETG/PEEK on a 3D printer, and this tool lets you iterate on the geometry.

---

## A. Features

### Physics-Based Simulation
Same full 3D magnetic field engine as the Halbach simulator: finite cylinder magnets modeled as stacked surface-current rings, each ring solved analytically via complete elliptic integrals.

The UI reports $|B|$, $B_x$, $B_y$, $B_z$ at the center, and two dimensionless homogeneity metrics over the test volume:
- **Bx spread** — peak-to-peak variation of $B_x$ as a percentage of center $|B_x|$
- **YZ leakage** — worst-case transverse component $\sqrt{B_y^2+B_z^2}$ as a percentage of center $|B_x|$

### Lattices and Configuration
The simulator supports multiple spherical distributions, automatically cutting out polar regions to form the Z-axis bore:
- **C60 Mandhala (48 dipoles):** Derived from the $C_{60}$ (truncated icosahedron) lattice. Removing the two hexagonal caps closest to the $\pm Z$ poles leaves 48 dipoles. This geometry preserves $D_{2h}$ symmetry, ensuring $B_y = B_z = 0$ at the center by construction.
- **Lebedev-110:** A quadrature grid designed for precise spherical integration. After removing polar caps and near-axis points, it leaves a highly symmetric distribution of dipoles that helps suppress higher-order harmonic distortions.
- **Fibonacci Sphere (Tunable N):** A golden-angle spiral that provides an uniform, aperiodic angular distribution. This breaks exact rotational symmetries to "white" harmonic errors in the magnetic field.

User-adjustable parameters include inner/outer radii, test volume radius, magnet dimensions, minimum physical gap (auto-clamped to prevent physical collisions), and remanence ($B_r$).

### Visualization and Field Mapping
- **Interactive 3D Scene:** Orbit/pan/zoom via Three.js, showing the inner sphere, outer bounding shell, test volume wireframe, and color-coded dipoles (N/S faces).
- **2D Heatmaps:** Live-updating $|B|$ maps in the XY, XZ, and YZ planes.
- **Bore Profile:** $B_x$, $B_y$, $B_z$ along the Z-axis over the test range.
- **Schlegel Diagram:** A stereographic projection from the North pole (+Z axis) to a 2D plane. This diagram visualizes the lattice topology (connecting nearest neighbors) and the magnetic polarity of each dipole. The polarity is calculated dynamically via the dot product of the moment and direction vectors, color-coding the nodes (e.g., Blue for North-inward, Red for South-inward). This provides a clear, printable map of the 3D spherical arrangement and assembly sequence.

### Optimization (Simulated Annealing)
Parallel simulated annealing in Web Workers to optimize the radial position (offset) of each magnet.
- **Objective function:** A user-adjustable slider balances homogeneity ($P_{homog}$) vs. field strength ($P_{field}$).
- **Symmetry Locking:** For symmetric lattices (C60, Lebedev), the SA engine identifies mirror pairs/orbits and moves them identically. This halves the search space and enforces the structural symmetry of the resulting field.
- **Collision Checking:** Each iterative nudge is checked against the geometric `min gap` constraint to ensure the resulting model can be physically assembled.

### Export (PDF, CSV, STEP)
The tool features robust export capabilities for manufacturing:
- **PDF Report:** Contains simulation parameters, center field stats, 3D renders from multiple angles, 2D heatmaps, the Schlegel diagram, and a comprehensive **Dipole Table**. The table lists the ID, the absolute radial position of the North pole face ($r_N$), the radial position of the South pole face ($r_S$), the moment direction vector, and the symmetry orbit.
- **CSV:** Exports a build sheet for machining or 3D-printing assembly jigs.
- **STEP:** Generates an AP214 STEP file containing the exact 3D cylindrical solids of the optimized array, ready to import directly into CAD software for holder design. Only the magnets are saved in the STEP file.

---

# Notes
## Beyond $C_{60}$: Quasicrystals and Aperiodic Magnetic Geometries

Exploring geometries beyond the $C_{60}$ lattice makes physical sense for improving field homogeneity. While the $C_{60}$ structure is nice, its high degree of **discrete rotational symmetry** dictates the amplitudes of specific harmonic distortions in the bore. By moving to other distributions, we can change how the field decays.

### 1. The Harmonic Problem with $C_{60}$
In the $C_{60}$ Mandhala setup, 48 dipoles are placed on an $I_h$ lattice. The magnetic potential $\Phi$ inside the bore is a solution to Laplace's equation in spherical harmonics. High-symmetry lattices eliminate many low-order terms but "stack" the remaining error into specific, high-amplitude harmonic spikes.

### 2. Lebedev Quadrature Grids
Lebedev grids are designed to integrate polynomials on the surface of a sphere with high precision. By placing magnets at Lebedev points and allowing the SA algorithm to weight their "strengths" (by varying their radial distance), we actively decrease specific multipole moments. The Mandhala simulator now implements the Lebedev-110 solid as a base structure for this exact reason.

### 3. The Case for Fibonacci Spheres
A Fibonacci Lattice on a sphere breaks discrete rotational symmetry entirely. 
* **Spectral Whitening:** By breaking symmetry, the harmonic distortion is "spread" across the entire spectrum. Instead of massive, specific-order distortions, the error is distributed into many tiny, negligible errors across all orders. This is now fully testable in the simulator.

### 4. Practical Implementation
While Fibonacci and Lebedev distributions offer theoretical improvements, assembly is challenging. The lack of standard angles makes machining complex, maybe 3D printing is an option. A STEP file can be exported, it seems to open correctly in Solidworks and FreeCAD 1.0.0. The simulator also outputs  exact radial pole coordinates and a Schlegel diagram (a reminder of my old fullerene chemistry work).

---

last updated 19 April 2026