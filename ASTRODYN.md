# ASTRODYN.md

## Project vision

`astrodyn` is a Go-native trajectory-design and astrodynamics engine.

Its goal is to provide a rigorous, production-grade foundation for computing, propagating, targeting, optimizing, validating, and analyzing spacecraft trajectories across multiple dynamical regimes:

* Earth satellite orbits
* Earth–Moon transfers
* Earth–Moon libration-point trajectories
* cislunar invariant-manifold transfers
* interplanetary transfers such as Earth–Mars
* asteroid rendezvous trajectories
* gravity-assist trajectories
* high-fidelity ephemeris-driven mission analysis
* future low-thrust and optimal-control trajectories

`astrodyn` is not intended to replace `astrogo`.

The intended relationship is:

```text
astrogo:
    precision astronomy, time scales, reference frames, ephemerides,
    physical constants, observational astronomy, catalogs, validation

astrodyn:
    dynamics, propagation, orbital mechanics, targeting, transfer design,
    trajectory optimization, maneuvers, mission-analysis workflows
```

In short:

```text
astrogo answers: where are the bodies?
astrodyn answers: what trajectory should the spacecraft fly?
```

---

## Motivation

Most Go astronomy packages focus on observation, coordinates, catalogs, or ephemerides. Most mature astrodynamics and mission-design ecosystems are written in Python, C/C++, MATLAB, Julia, or Fortran.

Go has strong advantages for this domain:

* simple deployment
* excellent concurrency
* fast batch evaluation
* static binaries
* strong typing
* clean APIs
* production-friendly tooling
* good integration with cloud infrastructure
* deterministic high-throughput search pipelines

`astrodyn` should exploit those advantages while being scientifically credible.

The long-term objective is not just to implement isolated formulas. The objective is to build a coherent engine where different trajectory-design methods can interoperate through shared concepts:

* state vectors
* epochs
* reference frames
* force models
* propagators
* events
* maneuvers
* boundary-value solvers
* optimization backends
* ephemeris adapters
* validation datasets

---

## Non-goals

`astrodyn` should avoid becoming a vague "everything about space" package.

The following are not initial goals:

* launch vehicle ascent guidance
* atmospheric entry, descent, and landing
* real-time spacecraft flight software
* finite-element spacecraft modeling
* attitude dynamics as a primary focus
* spacecraft thermal modeling
* ground-station scheduling
* telescope observation planning
* FITS/WCS/catalog processing
* astrometry pipelines

Some of these may eventually connect to `astrodyn`, but they should not drive the first architecture.

---

## Design principles

### 1. Dynamical models must be explicit

A trajectory result is meaningless unless the force model is known.

The API must make the model visible:

```go
type Dynamics interface {
    Deriv(t float64, y State) State
}
```

Examples:

```go
type TwoBody struct {
    Mu float64
}

type CR3BP struct {
    Mu1      float64
    Mu2      float64
    Distance float64
}

type NBody struct {
    Bodies    []MassiveBody
    Ephemeris EphemerisProvider
}
```

A user should never confuse:

* two-body propagation
* patched-conic propagation
* circular restricted three-body propagation
* ephemeris-driven N-body propagation
* low-thrust propagation

These are different models with different assumptions.

---

### 2. Units must be impossible to ignore

Astrodynamics code is highly vulnerable to silent unit errors.

`astrodyn` must clearly distinguish:

* SI units
* kilometers versus meters
* seconds versus days
* canonical units
* angular units
* normalized CRTBP units
* physical dimensional states

Initial implementations may use `float64` for performance, but APIs must document the expected units aggressively.

Recommended rule:

```text
Public mission-level APIs should use SI unless explicitly marked otherwise.
Specialized internal CRTBP APIs may use canonical units.
Conversions must be explicit.
```

Example:

```go
type CanonicalSystem struct {
    LengthUnit float64 // meters
    TimeUnit   float64 // seconds
    MassUnit   float64 // optional
}

func (u CanonicalSystem) ToCanonicalState(s State) State
func (u CanonicalSystem) FromCanonicalState(s State) State
```

---

### 3. Ephemeris and dynamics are separate concerns

An ephemeris provider tells us where massive bodies are.

A dynamics model tells us how a spacecraft accelerates.

Do not merge these concepts.

```go
type EphemerisProvider interface {
    State(body BodyID, epoch Epoch, frame Frame) (State, error)
}
```

Then an N-body model may use an ephemeris provider:

```go
type NBody struct {
    Central   BodyID
    Bodies    []BodyID
    Ephemeris EphemerisProvider
}
```

This allows `astrodyn` to depend on `astrogo` for JPL ephemerides without duplicating the ephemeris layer.

---

### 4. Solvers must not own the model

A propagator should accept a model.

A solver should accept a propagator or a dynamics interface.

Bad design:

```go
func SolveMarsTransferWithJPLDE440(...)
```

Better design:

```go
func SolveLambert(problem LambertProblem, opts LambertOptions) (LambertSolution, error)

func Propagate(dyn Dynamics, initial State, span TimeSpan, opts IntegratorOptions) (Trajectory, error)
```

Mission-level convenience wrappers can exist later, but the core must remain model-composable.

---

### 5. Reproducibility comes before beauty

The first versions should prioritize reproducible benchmark cases over elegant abstractions.

A beautiful API that cannot reproduce known reference trajectories is useless.

Every major module must include reference tests:

* two-body energy conservation
* Keplerian period tests
* Lambert solver known cases
* CRTBP Jacobi conservation
* Lagrange-point locations
* Lyapunov orbit periodicity
* monodromy matrix consistency
* invariant-manifold branch validation
* Earth–Moon L1 transfer benchmark
* Earth–Mars Lambert launch-window sanity checks

---

### 6. Optimization backends must be optional

`astrodyn` may support numerical backends such as:

* pure Go
* Gonum
* GoMLX
* OpenXLA through GoMLX
* future external solvers

However, the public scientific API should not expose backend-specific tensor types.

Good:

```go
type SolverBackend interface {
    SolveLeastSquares(problem LeastSquaresProblem, initial []float64) (LeastSquaresResult, error)
}
```

Bad:

```go
func SolveTransfer(x *gomlx.Node) *gomlx.Node
```

Backends should accelerate the implementation, not dominate the library design.

---

## Repository structure

Recommended initial repository:

```text
github.com/TuSKan/astrodyn
```

Proposed layout:

```text
astrodyn/
  README.md
  ASTRODYN.md
  go.mod

  state/
    state.go
    vector.go
    elements.go
    maneuver.go

  units/
    canonical.go
    si.go

  time/
    epoch.go
    duration.go

  frame/
    frame.go
    transform.go

  bodies/
    body.go
    constants.go

  ephemeris/
    provider.go
    astrogo_adapter.go

  dynamics/
    dynamics.go
    two_body/
      twobody.go
      energy.go
    crtbp/
      system.go
      equations.go
      jacobi.go
      lagrange.go
      canonical.go
      variational.go
    nbody/
      nbody.go
      acceleration.go
    low_thrust/
      low_thrust.go

  propagate/
    integrator.go
    rk45.go
    dop853.go
    event.go
    dense_output.go
    trajectory.go

  solve/
    lambert/
      problem.go
      izzo.go
      universal.go
      porkchop.go
    shooting/
      single.go
      multiple.go
    correction/
      differential.go
    leastsq/
      lm.go
      problem.go
    collocation/
      nodes.go
      defect.go
    tfc/
      chebyshev.go
      constraints.go
      functional.go

  orbit/
    kepler/
      elements.go
      conversion.go
      anomaly.go
    periodic/
      lyapunov/
        seed.go
        correction.go
        continuation.go
        monodromy.go

  manifold/
    invariant/
      branch.go
      generate.go
      sample.go
      classify.go

  mission/
    patchedconics/
      departure.go
      arrival.go
      vinf.go
      c3.go
    cislunar/
      earth_moon_l1.go
    interplanetary/
      earth_mars.go
      launch_window.go
    gravityassist/
      flyby.go
      bplane.go

  backend/
    gonum/
      leastsq.go
    gomlx/
      residual.go
      autodiff.go

  validation/
    horizons/
      cases.go
    papers/
      almeida2026/
        constants.go
        reproduce_table4_test.go
        reproduce_transfer_test.go

  examples/
    earth_mars_lambert/
      main.go
    earth_moon_crtbp/
      main.go
    earth_moon_l1_tfc/
      main.go
```

This structure separates fundamental mechanics, propagation, solvers, mission-level workflows, and validation.

---

## Core abstractions

### State vector

```go
type Vec3 struct {
    X, Y, Z float64
}

type State struct {
    R Vec3 // position
    V Vec3 // velocity
}
```

Optional later extensions:

```go
type StateWithMass struct {
    R Vec3
    V Vec3
    M float64
}
```

For low-thrust trajectories, mass becomes part of the propagated state.

---

### Epoch

`astrodyn` should not initially reinvent all time-scale handling.

It should define a minimal epoch abstraction and delegate precision time-scale work to `astrogo` where possible.

```go
type Epoch struct {
    JD float64
    Scale TimeScale
}
```

Examples of scales:

```go
type TimeScale int

const (
    UTC TimeScale = iota
    TT
    TDB
)
```

Long term, this should integrate tightly with `astrogo`.

---

### Dynamics

```go
type Dynamics interface {
    Deriv(t float64, y State) State
}
```

For models requiring epochs:

```go
type EpochDynamics interface {
    Deriv(epoch Epoch, y State) (State, error)
}
```

The library may keep low-level propagators float-based for speed, while mission-level APIs use epochs.

---

### Propagator

```go
type Propagator interface {
    Propagate(dyn Dynamics, y0 State, t0, tf float64) (Trajectory, error)
}
```

Result:

```go
type Trajectory struct {
    Samples []Sample
}

type Sample struct {
    T float64
    Y State
}
```

Later:

* dense output
* event records
* integration statistics
* step rejection counts
* invariant drift metrics

---

### Event detection

Event detection is essential for real trajectory design.

Examples:

* altitude crossing
* sphere-of-influence crossing
* periapsis
* apoapsis
* lunar surface impact
* x-axis crossing in CRTBP
* Jacobi drift threshold
* target-plane crossing

```go
type Event interface {
    Value(t float64, y State) float64
    Direction() Direction
    Terminal() bool
}
```

---

### Maneuver

```go
type Maneuver struct {
    Epoch Epoch
    DV    Vec3
}

func (m Maneuver) Magnitude() float64
```

For impulsive designs, trajectory segments are connected by maneuvers.

For future finite-burn designs:

```go
type ThrustArc struct {
    Start Epoch
    End   Epoch
    Law   ThrustLaw
}
```

---

## Dynamics modules

### Two-body dynamics

The two-body module is the foundation for:

* Earth satellite propagation
* Keplerian orbit conversion
* Lambert transfer verification
* patched conics
* hyperbolic departure and arrival

Required capabilities:

* acceleration under central gravity
* specific orbital energy
* angular momentum
* eccentricity vector
* conversion between Cartesian state and Keplerian elements
* anomaly conversions
* circular velocity
* escape velocity
* hyperbolic excess velocity

Example:

```go
type TwoBody struct {
    Mu float64
}

func (tb TwoBody) Deriv(t float64, y State) State
func (tb TwoBody) Energy(y State) float64
func (tb TwoBody) Elements(y State) (Elements, error)
```

---

### CR3BP / CRTBP dynamics

The circular restricted three-body problem is required for:

* Earth–Moon cislunar analysis
* L1/L2/L3 dynamics
* Lyapunov orbits
* halo orbits later
* stable/unstable invariant manifolds
* low-energy transfer design

Required capabilities:

* canonical units
* rotating-frame equations
* inertial/rotating-frame conversion
* Jacobi constant
* Lagrange point computation
* variational equations
* state transition matrix integration
* planar and 3D support

Example:

```go
type System struct {
    Mu       float64 // mass parameter, canonical CR3BP
    LU       float64 // length unit in meters
    TU       float64 // time unit in seconds
    Primary1 BodyID
    Primary2 BodyID
}

func (s System) Deriv(t float64, y State) State
func (s System) Jacobi(y State) float64
func (s System) LagrangePoint(p Point) (Vec3, error)
```

The first target should be planar Earth–Moon CRTBP.

---

### N-body dynamics

N-body dynamics is required for high-fidelity mission analysis.

Initial model:

```go
type PointMassNBody struct {
    Bodies    []BodyID
    Ephemeris EphemerisProvider
    Frame     Frame
}
```

Acceleration:

```text
a = sum_i μ_i * (r_i - r_sc) / |r_i - r_sc|^3
```

Future perturbations:

* Earth J2
* Moon gravity harmonics
* solar radiation pressure
* atmospheric drag
* relativistic corrections
* thrust acceleration

The first N-body milestone should be Sun + Earth + Moon + Mars point-mass propagation using ephemerides from `astrogo`.

---

### Low-thrust dynamics

Low thrust should be a later milestone.

It requires state + mass propagation:

```text
r_dot = v
v_dot = gravity + thrust / mass
m_dot = -|thrust| / (Isp * g0)
```

Potential control laws:

* fixed inertial direction
* tangential thrust
* primer-vector-inspired direction
* parameterized thrust angles
* direct collocation control nodes

This should not be part of v0.

---

## Solver modules

### Lambert solver

The Lambert solver is the first major interplanetary capability.

It solves:

```text
Given r1, r2, time of flight, and central gravitational parameter,
find v1 and v2 such that a two-body trajectory connects r1 to r2.
```

This enables:

* Earth-to-Mars transfers
* Mars-to-Earth returns
* Venus transfers
* asteroid rendezvous first guesses
* launch-window maps
* porkchop plots
* patched-conic mission analysis

Example API:

```go
type LambertProblem struct {
    MuCentral   float64
    R1          Vec3
    R2          Vec3
    TimeOfFlight float64
    Revolutions int
    Direction   TransferDirection
}

type LambertSolution struct {
    V1 Vec3
    V2 Vec3
}

func Solve(problem LambertProblem, opts Options) ([]LambertSolution, error)
```

Important cases:

* short-way transfer
* long-way transfer
* zero-revolution transfer
* multi-revolution transfer later
* near-180-degree transfer edge case
* near-zero time-of-flight invalid case
* parabolic boundary cases

Recommended implementation path:

1. Implement robust zero-revolution Lambert solver.
2. Validate against known textbook cases.
3. Add multi-revolution support.
4. Add porkchop generation utilities.

---

### Patched conics

Patched conics translate heliocentric Lambert solutions into mission-level quantities:

* departure v∞
* arrival v∞
* C3
* parking-orbit injection ΔV
* capture ΔV
* hyperbolic periapsis velocity

Example:

```go
type Departure struct {
    PlanetMu     float64
    ParkingRadius float64
    VInf         Vec3
}

func InjectionDeltaV(d Departure) (float64, error)
```

For Earth departure:

```text
C3 = |v∞|²
ΔV = sqrt(v∞² + 2μ/rp) - sqrt(μ/rp)
```

For Mars capture:

```text
ΔV = v_periapsis_hyperbola - v_target_orbit_periapsis
```

This module is essential if the user wants meaningful Mars transfer numbers.

---

### Differential correction

Differential correction is required for:

* periodic orbit generation
* targeting boundary conditions
* correcting propagated arcs
* matching position/velocity constraints

Generic form:

```go
type CorrectionProblem interface {
    Residual(x []float64) ([]float64, error)
    Jacobian(x []float64) (*mat.Dense, error)
}
```

Use cases:

* Lyapunov orbit correction
* multiple shooting
* target-plane correction
* trajectory refinement under N-body dynamics

---

### Least squares

Many solvers reduce to nonlinear least squares:

```text
minimize ||F(x)||²
```

Backends:

* pure Go finite difference
* Gonum-based Levenberg–Marquardt
* GoMLX autodiff backend

Example:

```go
type LeastSquaresProblem interface {
    Residual(x []float64) ([]float64, error)
}

type LeastSquaresSolver interface {
    Solve(problem LeastSquaresProblem, x0 []float64, opts Options) (Result, error)
}
```

---

### Collocation

Collocation is needed for direct transcription methods.

Initial support:

* Chebyshev–Gauss–Lobatto nodes
* defect constraints
* polynomial basis evaluation
* derivative matrices

This directly supports TFC and later optimal-control methods.

---

### Theory of Functional Connections

The Theory of Functional Connections module should initially be narrow and benchmark-driven.

It should support constrained functionals used to embed boundary constraints analytically into candidate functions.

Initial target:

* Earth-to-stable-manifold transfer leg
* unstable-manifold-to-Moon transfer leg
* Chebyshev polynomial free functions
* nonlinear least-squares solution of collocation residuals

Do not begin with a fully generic TFC framework.

Begin by reproducing one paper-quality benchmark.

Then generalize.

---

## Cislunar paper benchmark module

The first advanced cislunar benchmark should reproduce the Earth–Moon L1 transfer via Lyapunov orbit and invariant manifolds.

This benchmark requires:

1. Earth–Moon CRTBP model
2. L1 location
3. planar Lyapunov orbit generation
4. continuation to target Jacobi constant
5. monodromy matrix
6. stable and unstable eigenvectors
7. manifold generation
8. TFC Earth-leg transfer
9. TFC Moon-leg transfer
10. ΔV computation
11. total transfer reconstruction

Reference target values:

```text
Jacobi constant:
C = 3.0945041790

Earth leg:
P1 ΔV = 3142.09 m/s
P2 ΔV = 200.87 m/s
Total ΔVE = 3342.96 m/s
TOF = 3.69 days

Moon leg:
P3 ΔV = 0.81 m/s
P4 ΔV = 647.83 m/s
Total ΔVM = 648.64 m/s
TOF = 4.7 days

Complete transfer:
Total ΔV = 3991.60 m/s
```

Initial tolerances should be realistic:

```text
Phase 1 tolerance: ±50 m/s
Phase 2 tolerance: ±10 m/s
Phase 3 tolerance: ±1 m/s
Phase 4 tolerance: reproduce trajectory points and costs within paper precision
```

Do not demand perfect reproduction before validating the lower layers.

Validation order:

1. Jacobi conservation
2. L1/L2/L3 location
3. planar Lyapunov periodicity
4. monodromy eigenstructure
5. manifold geometry
6. transfer costs

---

## Mars trajectory milestone

The first interplanetary milestone should be Earth-to-Mars Lambert transfer.

Goal:

```text
Given departure and arrival epochs, compute:
- Earth heliocentric state
- Mars heliocentric state
- Lambert transfer velocities
- departure v∞
- arrival v∞
- C3
- approximate Earth parking-orbit injection ΔV
- approximate Mars capture ΔV
```

Mission-level workflow:

```go
earth, _ := eph.State(bodies.Earth, departure, frame.ICRF)
mars, _ := eph.State(bodies.Mars, arrival, frame.ICRF)

sols, _ := lambert.Solve(lambert.Problem{
    MuCentral: bodies.SunGM,
    R1: earth.R,
    R2: mars.R,
    TimeOfFlight: arrival.Sub(departure).Seconds(),
})

vinfDep := sols[0].V1.Sub(earth.V)
vinfArr := sols[0].V2.Sub(mars.V)

c3 := vinfDep.Norm2()
```

Then patched conics:

```go
dvEarth, _ := patchedconics.DepartureDeltaV(patchedconics.Departure{
    PlanetMu:      bodies.EarthGM,
    ParkingRadius: bodies.EarthRadius + 200e3,
    VInf:          vinfDep,
})

dvMars, _ := patchedconics.CaptureDeltaV(patchedconics.Capture{
    PlanetMu:      bodies.MarsGM,
    PeriapsisRadius: bodies.MarsRadius + 300e3,
    VInf:          vinfArr,
    TargetOrbit:   patchedconics.CircularOrbit,
})
```

This should become the first public demo because it clearly shows the value of `astrogo + astrodyn`.

---

## Relationship with astrogo

`astrodyn` should not duplicate `astrogo` unless necessary.

Use `astrogo` for:

* JPL ephemerides
* time scales
* Julian dates
* constants
* body identifiers
* reference-frame transformations
* validation against Horizons

Use `astrodyn` for:

* force models
* propagation
* trajectory segments
* maneuvers
* Lambert solving
* patched conics
* CR3BP
* manifolds
* TFC
* optimization
* mission-design workflows

Possible adapter:

```go
type AstrogoEphemeris struct {
    Provider astrogo.EphemerisProvider
}

func (e AstrogoEphemeris) State(body BodyID, epoch Epoch, frame Frame) (State, error) {
    // Convert astrodyn epoch/frame/body into astrogo request.
    // Convert astrogo result into astrodyn State.
}
```

Keep the boundary clean.

---

## GoMLX strategy

GoMLX can be useful for:

* TFC residual evaluation
* automatic differentiation
* batch least-squares residuals
* collocation defect computation
* fixed-shape optimization kernels
* large candidate sweeps

GoMLX should not be mandatory for basic use.

Recommended backend design:

```go
type BackendKind int

const (
    BackendPureGo BackendKind = iota
    BackendGonum
    BackendGoMLX
)
```

Public APIs should accept normal Go values:

```go
type TFCOptions struct {
    Order   int
    Nodes   int
    Backend BackendKind
}
```

Internal backend-specific code may use GoMLX tensors, but those types should not leak into the top-level API.

This allows:

* simple builds for ordinary users
* accelerated builds for heavy users
* clean testing without GPU/XLA setup
* future backend expansion

---

## Numerical validation strategy

### Invariants

Two-body:

* specific orbital energy
* angular momentum
* eccentricity vector

CRTBP:

* Jacobi constant

N-body:

* comparison against reference ephemeris propagation
* step-size convergence
* energy checks only when model permits

### Solver validation

Lambert:

* known textbook cases
* cross-check with poliastro/PyKEP/Orekit where possible
* reverse-transfer consistency
* time-of-flight edge cases

Lyapunov:

* periodicity error after one period
* x-axis crossing symmetry
* Jacobi consistency
* monodromy eigenvalue reciprocal pairs

Manifolds:

* stable branch converges forward to periodic orbit
* unstable branch converges backward to periodic orbit
* branch geometry sanity plots

TFC:

* boundary constraints exactly satisfied
* residual norm decreases
* transfer endpoint velocities match required manifold/circular-orbit states
* ΔV values reproduce benchmark tables

---

## Concurrency and batch search

Go is especially strong for massive trajectory candidate evaluation.

Candidate search should be designed as a first-class concept.

Example:

```go
type Candidate struct {
    Params any
}

type Evaluation struct {
    Candidate Candidate
    Cost      float64
    Solution  any
    Err       error
}

func Search(ctx context.Context, candidates <-chan Candidate, workers int, eval Evaluator) <-chan Evaluation
```

For cislunar manifold search:

```go
type CislunarCandidate struct {
    Jacobi        float64
    ManifoldIndex int
    TimeOfFlight  float64
    Branch        invariant.Branch
}
```

For Mars launch-window search:

```go
type LaunchWindowCandidate struct {
    Departure Epoch
    Arrival   Epoch
}
```

Search results should support:

* deterministic ordering
* CSV output
* Parquet output later
* checkpointing
* resume
* top-K filtering
* Pareto-front filtering

---

## Public examples

### Example 1: two-body propagation

```go
func ExampleTwoBodyPropagation() {
    earth := twobody.TwoBody{Mu: bodies.EarthGM}

    y0 := state.State{
        R: state.Vec3{X: bodies.EarthRadius + 400e3, Y: 0, Z: 0},
        V: state.Vec3{X: 0, Y: 7670, Z: 0},
    }

    traj, err := propagate.Propagate(earth, y0, 0, 5400, propagate.DefaultOptions())
    if err != nil {
        panic(err)
    }

    _ = traj
}
```

### Example 2: Earth-to-Mars Lambert transfer

```go
func ExampleEarthMarsLambert() {
    eph := ephemeris.NewAstrogoAdapter(...)

    dep := time.MustParseEpoch("2031-09-01T00:00:00", time.TDB)
    arr := time.MustParseEpoch("2032-05-01T00:00:00", time.TDB)

    earth, _ := eph.State(bodies.Earth, dep, frame.ICRF)
    mars, _ := eph.State(bodies.Mars, arr, frame.ICRF)

    sols, _ := lambert.Solve(lambert.Problem{
        MuCentral:    bodies.SunGM,
        R1:           earth.R,
        R2:           mars.R,
        TimeOfFlight: arr.Sub(dep).Seconds(),
    }, lambert.DefaultOptions())

    vinfDep := sols[0].V1.Sub(earth.V)
    vinfArr := sols[0].V2.Sub(mars.V)

    _ = vinfDep
    _ = vinfArr
}
```

### Example 3: Earth–Moon CRTBP Jacobi constant

```go
func ExampleCRTBPJacobi() {
    sys := crtbp.EarthMoonCanonical()

    y := state.State{
        R: state.Vec3{X: 0.8, Y: 0, Z: 0},
        V: state.Vec3{X: 0, Y: 0.2, Z: 0},
    }

    c := sys.Jacobi(y)
    _ = c
}
```

---

## Version roadmap

### v0.1 — Foundation

* state vectors
* vector operations
* constants
* two-body dynamics
* adaptive RK integrator
* event detection skeleton
* basic Keplerian elements
* initial tests

Deliverable:

```text
Propagate a satellite around Earth and validate orbital invariants.
```

---

### v0.2 — Lambert and Mars first guess

* Lambert solver
* Earth/Mars ephemeris adapter to `astrogo`
* departure v∞
* arrival v∞
* C3
* basic launch-window grid

Deliverable:

```text
Compute Earth-to-Mars Lambert transfers from real ephemerides.
```

---

### v0.3 — Patched conics

* parking-orbit departure ΔV
* arrival capture ΔV
* hyperbolic periapsis velocity
* sphere-of-influence utilities
* mission summary reports

Deliverable:

```text
Produce practical Earth-to-Mars porkchop-style mission metrics.
```

---

### v0.4 — CRTBP core

* canonical units
* Earth–Moon CRTBP
* Jacobi constant
* Lagrange points
* rotating/inertial conversions
* variational equations

Deliverable:

```text
Propagate Earth–Moon CRTBP trajectories with Jacobi conservation.
```

---

### v0.5 — Periodic orbits and manifolds

* planar Lyapunov orbit seed
* differential correction
* continuation
* monodromy matrix
* stable/unstable eigenvectors
* invariant manifold generation

Deliverable:

```text
Generate L1 Lyapunov orbit and stable/unstable manifolds.
```

---

### v0.6 — TFC transfer benchmark

* Chebyshev basis
* Chebyshev–Gauss–Lobatto nodes
* constrained functionals
* Earth-leg TFC transfer
* Moon-leg TFC transfer
* nonlinear least-squares solve
* benchmark reproduction

Deliverable:

```text
Reproduce Earth–Moon L1 TFC transfer benchmark within useful tolerance.
```

---

### v0.7 — High-fidelity refinement

* N-body point-mass dynamics
* Sun/Earth/Moon/Mars ephemeris forcing
* trajectory correction from CRTBP initial guess
* epoch-dependent cislunar refinement

Deliverable:

```text
Use CRTBP/TFC solution as initial guess for ephemeris-driven refinement.
```

---

### v0.8+ — Advanced mission design

Potential future modules:

* multiple shooting
* gravity-assist targeting
* B-plane analysis
* halo orbits
* NRHO support
* low-thrust collocation
* uncertainty propagation
* covariance analysis
* Monte Carlo dispersions
* station-keeping estimation

---

## Quality bar

This project should be held to a high numerical and API standard.

Minimum expectations:

* deterministic tests
* documented assumptions
* explicit units
* benchmark references
* reproducible examples
* no hidden global state
* no silent frame conversions
* no silent unit conversions
* stable public APIs
* clear separation between model, solver, and mission workflow
* profiling before optimization
* correctness before speed

---

## Naming conventions

Preferred terms:

* `State`, not `Vector6`
* `Dynamics`, not `ModelFunc`
* `Propagator`, not `Integrator` when returning trajectories
* `Integrator`, when referring only to numerical stepping
* `Maneuver`, not `Burn` in generic APIs
* `Impulse`, when specifically instantaneous ΔV
* `Trajectory`, for sampled propagated paths
* `Segment`, for one continuous arc
* `Mission`, for connected segments and maneuvers
* `Frame`, for reference frame identity
* `Epoch`, for time with scale
* `BodyID`, for astronomical bodies
* `CR3BP` or `CRTBP`, but choose one consistently in package names

Recommended package name:

```text
crtbp
```

Even though `CR3BP` is common in English, `crtbp` matches "circular restricted three-body problem" and avoids package names starting with numbers.

---

## Initial implementation priorities

Do not begin with TFC.

Do not begin with GoMLX.

Do not begin with a generic optimizer.

Begin with this exact order:

1. `state.Vec3`
2. `state.State`
3. vector math
4. `dynamics.TwoBody`
5. adaptive RK integrator
6. two-body invariant tests
7. Keplerian element conversion
8. Lambert solver
9. Earth-to-Mars ephemeris adapter
10. patched-conic ΔV
11. CRTBP
12. Lagrange points
13. Jacobi constant
14. Lyapunov orbit
15. manifolds
16. TFC benchmark

This avoids the trap of building a sophisticated optimizer on top of an unvalidated dynamics engine.

---

## First concrete milestone

The first useful public demo should be:

```text
Earth-to-Mars Lambert transfer using astrogo ephemerides.
```

Why this first?

* easier to explain
* easier to validate
* useful to many users
* demonstrates `astrogo` integration
* establishes interplanetary capability
* does not require invariant manifolds or TFC

Expected output:

```text
Departure epoch
Arrival epoch
Time of flight
Departure v∞
Arrival v∞
C3
Earth parking orbit injection ΔV
Mars capture ΔV
```

---

## Second concrete milestone

The second major demo should be:

```text
Earth–Moon CRTBP L1 Lyapunov orbit and invariant manifolds.
```

Expected output:

* L1 location
* Lyapunov orbit period
* Jacobi constant
* monodromy eigenvalues
* stable manifold samples
* unstable manifold samples
* Jacobi drift statistics

---

## Third concrete milestone

The third major demo should be:

```text
Earth–Moon L1 TFC transfer benchmark.
```

Expected output:

* Earth-leg transfer ΔV
* Moon-leg transfer ΔV
* total ΔV
* trajectory points P1/P2/P3/P4
* comparison with reference paper values

---

## Final architectural position

`astrodyn` should be a general astrodynamics and trajectory-design engine in Go.

It should start with practical, validated capabilities:

1. two-body propagation
2. Lambert transfers
3. patched conics
4. Earth-to-Mars launch-window analysis
5. CRTBP cislunar dynamics
6. Lyapunov orbits and invariant manifolds
7. TFC-based constrained transfer design

The project should grow from validated physical models and benchmarked solvers, not from abstract optimization ambitions.

The long-term goal is ambitious:

```text
A Go-native mission-design engine that can move from preliminary trajectory search
all the way to high-fidelity ephemeris refinement, while keeping APIs clean,
units explicit, and numerical assumptions visible.
```

That is the standard this project should target.
