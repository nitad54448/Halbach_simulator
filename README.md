# Halbach Ring Simulator

This repository contains an interactive, web-based 3D simulator for calculating and visualizing the magnetic fields of finite-length Halbach cylinders. This program was initially created with a local Qwen-Coder 70b and improved with Gemini 3.1 and Claude Sonnet 4.6.

## Mathematical Model

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

## Core Functions

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
