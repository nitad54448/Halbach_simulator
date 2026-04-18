# Halbach type simulations
In this repository there are two web based simulators for calculating and visualizing the magnetic fields in multiple dipole systems. The first one is a derivation of the known Halbach array in which multiple segments, with defined orientation, lead to a direction controlled magnetic field. The second one is an extention of the Halbach model by using a fullerene-like arrangement of dipoles, distributed along the radii of the truncated icosaedron (I have no ideea if this second setup was described before, but it's not such a big discovery as it is an extention of multiple dipoles idea... we can think of other geometries, maybe Lebedev, or even quasi-crystal like sytems to avoid harmonic perturbations... ) 

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


# 2. Magic Sphere - 48-Dipole Simulator

Mandhala, or in previous versions Magic Sphere, is an interactive 3D simulator for designing and optimizing a spherical arrangement of permanent magnets based on a truncated C60 (fullerene) lattice. It models 48 cylindrical dipoles arranged on a spherical shell and computes the resulting magnetic field using a physically accurate Biot-Savart formulation. The tool is intended for exploring high-homogeneity magnetic field configurations. The 48 poles considered are the ones that remains after removing two 6–6 fulvalene analogue atoms, located along the C2 symmetry axis. This gives some space to the core, for setting a sample inside, while preserving the C2 symmetry. Half of the 48 are North inside and the other half are South-inside configuration. We can calculate and simulate the best configuration (in terms of distances from the center) so as to obtain homogeneous field or high field, or a cocmbination of both. The main idea and purpose is to get a high field, homogeneous, in the direction fof the X axis in order to perform Hall measurement. There is a repository and a Labview program that uses this configuration, in a classical Halbach magnet. This works but there are several problems: a commercial Halbach would be from 1K to 10K -or more- and if you have to compromise between the field and the size of the space. This is also the problem here, the compromise, the difference being that you can print in 3D your setup for 100 bucks and test it...
More than the experimental part, this tool can be used for teaching and exploring ideas. 

---

## A. Features

### Physics-Based Simulation
This sytem uses the same full 3D magnetic field computation as the previous code: finite cylinder magnet model, ring integration with elliptic integrals (Furlani). It will compute total field magnitude |B| and the field components (Bx, By, Bz) and gives an homogenity metric in a given selected volume.

### Geometry and Configuration
48 dipoles of specified diameter and length are placed on a C60-derived spherical lattice, oriented toward the center. 
Some parameters are adjustable, and restricted: Inner radius (the space for the sample), Outer radius (the total volume of the assembly), the Test volume radius. The Magnet diameter and length are used to calculate a minumum diameter of te inner sphere.
For contruction purposes we can define a Minimum gap, so the magnets should not overlap. For the magnetic calculations the user is prompted to input the Remanence (Br), it is typically 1.2 to 1.4 for NdBF or N40 to N45 type.

Initially the 48 dipoles are radial orientated with symmetric polarity assignment so as to obtain a field along the X axis.

### Visualization
There is an interactive 3D view (Three.js) with camera presets: 3D, top, front, side and Optional dipole numbering. 
- Visual elements:
  - Inner and outer spheres
  - Test volume
  - Magnet polarity (color-coded)


### Field Mapping
- 2D field maps in XY, XZ, YZ planes
- Bore profile along Z axis
- Real-time computation

### Optimization (Simulated Annealing)
- Multi-threaded using Web Workers (from 4 to 12 cores, adaptable dpeending on the user CPU). Some parameters are adjustable:
  Iterations, Maximum displacement in a cycle (confined to the sphere). Objective functions are Maximize homogeneity, Maximize field strength or Combined objective of the previous two.

- Features:
  - Automatic temperature tuning
  - Local and global moves
  - Escape mechanism to avoid local minima
  - Live progress display

### Export
- PDF export of current configuration

---
last update 18 april 2026
