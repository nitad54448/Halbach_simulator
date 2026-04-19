# Halbach type simulations
In this repository there are two web based simulators for calculating and visualizing the magnetic fields in multiple dipole systems. The first one is a derivation of the known Halbach array in which multiple segments, with defined orientation, lead to a direction controlled magnetic field. The second one is an extension of the Halbach model by using a radial arrangement of dipoles distributed along spherical lattices (C60 fullerene, Lebedev grids, and Fibonacci spheres).

# 1. Halbach Ring Simulator

The Halbach.html is an interactive, web-based 3D simulator for calculating and visualizing the magnetic fields of finite-length Halbach cylinders. This program was initially created in Labview during several years, 2014-2018 for a Hall measuring system (see my repos) and upgraded recently with a local Qwen-Coder 70b and further improved with Gemini 3.1 and Claude Sonnet 4.6.
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

$$Q = (a+\rho)^2 + \zeta^2$$
$$D = (a-\rho)^2 + \zeta^2$$
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
  Iterates through all $N$ poles. Calculates each magnet's physical position in the array and its Halbach magnetization vector. Sums the contributions from `cylinderField` and returns the net 3D magnetic field vector $\{B_x, B_y, B_z\}$.
* **`fieldAt(x, y, z)`**
  A utility wrapper that returns the scalar magnitude ($||B||$) of `fieldVectorAt`.

### User Interface & Projections
* **`computeStats(...)`**
  Evaluates the field at specific geometrical points to populate the UI (e.g., center bore field at `(0,0,0)`, and circular sampling around the holder perimeter to find peak leakage).
* **`drawSingleMap(plane)`**
  Renders the 2D cross-sectional heatmaps (XY, XZ, YZ). It generates an off-screen bitmap, computes the field scalar at each pixel, maps it to a color gradient, and calculates continuous isolines. It also overlays projected 2D vectors or out-of-plane flux indicators ($\odot$, $\otimes$).


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