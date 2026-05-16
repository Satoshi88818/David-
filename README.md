David v6.0 — Mathematically Grounded Autonomous Humanoid Entity

v6.0 fuses the fluid generative motion of v5.0 with the rigorous formal safety and game-theoretic intelligence of the FSDQuant autonomous driving stack. Every heuristic scaling factor from prior versions has been replaced with a provably correct mathematical primitive.

Table of Contents

Overview

What's New in v6.0

Architecture

Directory Structure

Module Reference 

schemas.py

world_model.py

diagnostic_reasoner.py

social_grid.py

vla_adapter.py

dreamer.py

planner_node.py

behavior_tree.py

auditory_cortex.py

hardware_interface.py

FSDQuant Integration Details

Data Flow

Dependencies & Installation

Key Design Decisions

Versioning & Upgrade Notes

Overview

David is a full-stack software architecture for an autonomous humanoid robot. It integrates:

A Vision-Language-Action (VLA) model based on diffusion sampling for generative motor control

A World Model providing real-time semantic and physical state

A Social Grid for human-aware navigation

A Dreamer loop for trajectory safety filtering

A Diagnostic Reasoner for dynamic payload and friction estimation

A Behavior Tree for high-level task sequencing

A Planner Node (ROS 2) orchestrating all subsystems at 10 Hz

v6.0 elevates the system from heuristic-based control to mathematically guaranteed safety and liveness through four key FSDQuant modules.

What's New in v6.0

IDNameModuleSummaryFSD-6HUKF Parameter Observerdiagnostic_reasoner.pyDynamically estimates payload mass and floor friction using joint torque residuals via an Unscented Kalman FilterFSD-7ALevel-K Reasoningsocial_grid.pyReplaces naive constant-velocity human predictions with a 2-step game-theoretic loop; eliminates the "hallway dance" (frozen robot problem)FSD-7BValue-Aware FPS (V-FPS)vla_adapter.pySamples N diffusion trajectories, then selects the K most strategically diverse ones using a blended spatial + value distance metricFSD-7C + FSD-2BCBF + CLF Safety Filterdreamer.pyReplaces heuristic q * 0.7 scaling with a QP that enforces ZMP safety (Control Barrier Function) and goal convergence (Control Lyapunov Function)FIX-KRCU Lock-Free Readsworld_model.pyAtomic pointer swap (Read-Copy-Update pattern) enables O(1), zero-latency state reads from the VLA and Behavior Tree with no mutex contention 

Architecture

┌──────────────────────────────────────────────────────────────────┐ │ planner_node.py (10 Hz) │ │ │ │ ┌────────────┐ ┌─────────────────────────────────────────┐ │ │ │ UKF │ │ WorldModel (FIX-K) │ │ │ │ Payload │───▶│ _state_snapshot (atomic pointer swap) │ │ │ │ Observer │ │ │ │ │ │ (FSD-6H) │ │ ┌───────────────┐ ┌───────────────┐ │ │ │ └────────────┘ │ │ SocialGrid │ │ DreamerLoop │ │ │ │ │ │ (FSD-7A) │ │ (FSD-7C/2B) │ │ │ │ ┌────────────┐ │ └───────────────┘ └───────────────┘ │ │ │ │ Hardware │ └─────────────────────────────────────────┘ │ │ │ Torque │ │ │ │ │ Sensors │ Lock-Free Read │ │ └────────────┘ │ │ │ ▼ │ │ ┌───────────────────────────┐ │ │ │ DiffusionVLA │ │ │ │ (FSD-7B V-FPS + FSD-7C) │ │ │ └───────────────────────────┘ │ │ │ │ │ ▼ │ │ ┌───────────────────────────┐ │ │ │ BehaviorTree / Execution │ │ │ └───────────────────────────┘ │ │ │ │ │ ▼ │ │ ┌───────────────────────────┐ │ │ │ hardware_interface.py │ │ │ └───────────────────────────┘ │ └──────────────────────────────────────────────────────────────────┘ 

Directory Structure

david_v6.0/ ├── requirements.txt ├── cognitive_core/ │ ├── schemas.py # ← UPDATED: Physics params, UKF state, CBF metrics │ ├── world_model.py # ← UPDATED: FIX-K (RCU lock-free concurrency) │ ├── dreamer.py # ← UPDATED: FSD-7C (CBF + CLF trajectory QP) │ ├── social_grid.py # ← UPDATED: FSD-7A (Level-K Reasoning) │ ├── diagnostic_reasoner.py # ← UPDATED: FSD-6H (UKF Payload Observer) │ ├── behavior_tree.py # Level-K Yield integration │ └── auditory_cortex.py # Unchanged from v5.0 AEC ├── vla/ │ └── vla_adapter.py # ← UPDATED: FSD-7B (Value-Aware FPS) ├── nodes/ │ └── planner_node.py # ← UPDATED: Wires UKF to VLA and Dreamer └── low_level/ └── hardware_interface.py 

Module Reference

1. cognitive_core/schemas.py — UPDATED

Purpose: Pydantic data models for all inter-module communication.

v6.0 additions:

SchemaDescriptionPhysicsParamsHolds UKF-estimated payload_mass_kg, com_offset_xyz, and contact_friction_mu. Updated at 10 Hz by the UKFPayloadObserver.IntentPredictionExtended with level_k_depth (1 = naive CV, 2 = reactive to robot path).WorldStateSnapshotImmutable, frozen snapshot of all world state used for lock-free RCU reads (FIX-K).ActionChunkExtended with cbf_clf_applied: bool flag indicating whether the CBF/CLF safety filter modified the trajectory. 

Key constants:

ACTION_CHUNK_HORIZON = 16 # Waypoints per chunk ACTION_CHUNK_HZ = 10 # Control frequency LEVEL_K_ITERATIONS = 2 # FSD-7A reasoning depth 

2. cognitive_core/world_model.py — UPDATED

Purpose: Central state authority for all robot knowledge.

FIX-K — RCU Lock-Free Concurrency

In prior versions, the VLA and Behavior Tree acquired a mutex to read world state, causing jitter in the motor control thread. v6.0 adopts the Read-Copy-Update (RCU) pattern:

Writers (update_physics, update_intent_predictions, update_agents) acquire _write_lock, deep-copy the current snapshot, mutate the copy, then atomically swap the pointer.

Readers (get_state) perform a single Python attribute read — which is atomic in CPython's GIL model — achieving O(1) wait-free access.

def get_state(self) -> WorldStateSnapshot: return self._state_snapshot # Atomic pointer read — no lock needed 

Key methods:

MethodDescriptionget_state()Lock-free read; returns the current immutable snapshotupdate_physics(params)Writer path; triggers after each UKF tickupdate_intent_predictions(predictions)Writer path; triggered by SocialGridupdate_agents(agents, robot_proposed_path)Triggers FSD-7A Level-K prediction pipeline 

3. cognitive_core/diagnostic_reasoner.py — UPDATED

Purpose: Real-time estimation of dynamic physical parameters.

FSD-6H — UKF Parameter Observer

Instead of assuming a fixed robot mass, David recursively estimates two latent parameters using an Unscented Kalman Filter:

State VariableDescriptionClamp Rangepayload_mass_kgMass of object currently held0 – 30 kgcontact_friction_muDynamic floor friction coefficient0.1 – 1.2 

The UKF is driven by the residual between Pinocchio's predicted joint torque and the real hardware torque reading.

State-space model:

Process model (fx): Random walk — parameters assumed piecewise constant.

Measurement model (hx): expected_torque_offset = payload_mass × g × lever_arm (approx. 35 cm).

Update rate: 10 Hz, synchronized with the main control tick.

The resulting PhysicsParams object is pushed into the WorldModel each tick, making it immediately available to the DreamerLoop's CBF/CLF QP solver.

4. cognitive_core/social_grid.py — UPDATED

Purpose: Human trajectory prediction and collision avoidance planning.

FSD-7A — Level-K Reasoning

Naive constant-velocity (CV) prediction causes the "hallway dance": David predicts a human will continue straight and steers around them; the human sees David steer and mirrors the move; both end up blocking each other indefinitely.

Level-K reasoning breaks this by simulating the human's reaction to David's action:

Level 0 (L0): Human moves at constant velocity, ignoring David. Level 1 (L1): Human detects David's path and rationally deviates (modelled as a repulsion vector away from the robot's trajectory). Nash check: Is the resulting L1 future position still an intersection risk? 

The depth used (level_k_depth = 1 or 2) is stored in IntentPrediction and passed downstream to influence the Behavior Tree's yield decisions.

Collision threshold: 0.8 m (L0 proximity) → triggers repulsion. 0.6 m (post-repulsion) → flags will_intersect_robot_path = True.

5. vla/vla_adapter.py — UPDATED

Purpose: Vision-Language-Action model interface and trajectory selection.

FSD-7B — Value-Aware Farthest Point Sampling (V-FPS)

The v5.0 VLA generated a single trajectory. v6.0 generates N=16 candidate trajectories and selects the K=3 most strategically distinct ones before choosing the best:

V-FPS Algorithm:

Score all N trajectories using a cost-to-go energy heuristic.

Seed the selection set with the lowest-cost trajectory.

For each subsequent pick, find the candidate maximising a blended distance:

blend_dist = (1 - value_weight) × spatial_L2_dist + value_weight × score_dist 

value_weight = 0.5 by default, equally weighting spatial diversity and strategic diversity.

This prevents selecting two trajectories that go around the same side of an obstacle — guaranteeing genuinely different tactical options.

Post-selection: The best of the K diverse trajectories is passed through the DreamerLoop's CBF/CLF filter (FSD-7C) before execution.

Confidence reporting:

cbf_clf_applied = False → confidence = 0.9

cbf_clf_applied = True → confidence = 0.7 (trajectory was modified for safety)

6. cognitive_core/dreamer.py — UPDATED

Purpose: Trajectory safety filtering and deadlock prevention.

FSD-7C + FSD-2B — Control Barrier Function & Control Lyapunov Function QP

v5.0 used an ad-hoc q * 0.7 velocity scaling whenever balance risk was detected. v6.0 replaces this with a Quadratic Program (QP) that simultaneously satisfies:

CBF (Safety — Zero Moment Point):

h_dot ≥ -γ × h ⟺ J_ZMP · dq ≥ -γ × zmp_margin 

Guarantees the ZMP never leaves the support polygon (no tipping).

CLF (Liveness — Goal Convergence):

V_dot ≤ -λ × V ⟺ -2 × (target - hand)ᵀ × J_hand · dq ≤ -λ × ||target - hand||² 

Prevents the robot from stalling at saddle points by pulling the end-effector toward the goal.

QP formulation (per waypoint):

Objective: min 0.5 × ||dq - dq_nominal||² (stay as close to the VLA's intent as possible)

Constraints: CBF inequality (hard), CLF inequality (soft / secondary)

Solver: scipy.optimize.minimize with SLSQP; production upgrade path to qpOASES or OSQP

Fallback: If QP fails to converge → dq = dq_nominal × 0.1 (extreme brake)

CBF/CLF is only triggered when zmp_margin < 0.05 m — safe waypoints pass through unmodified.

7. nodes/planner_node.py — UPDATED

Purpose: ROS 2 node orchestrating all subsystems at 10 Hz.

Wiring summary:

control_tick() [10 Hz]: 1. Get hardware torque residual 2. UKFPayloadObserver.update(residual) → PhysicsParams 3. world_model.update_physics(PhysicsParams) 4. Get tracked agents + current robot XY path 5. world_model.update_agents(agents, robot_path) → SocialGrid.predict_intents_level_k() → world_model.update_intent_predictions() execute_task(task_text): 1. world_model.get_state() ← Lock-free RCU read (FIX-K) 2. vla.generate_action_chunk(frames, task, world_state) → V-FPS (FSD-7B) + CBF/CLF filter (FSD-7C) 3. _tick_chunk(chunk, task) → hardware_interface 

8. cognitive_core/behavior_tree.py

Purpose: High-level task sequencing and reactive behavior selection. Integrates Level-K Yield logic from social_grid to pause navigation when will_intersect_robot_path = True and resume once the predicted intersection clears.

(Unchanged core from v5.0; Level-K Yield hook added in v6.0)

9. cognitive_core/auditory_cortex.py

Purpose: Acoustic echo cancellation (AEC) and speech-to-intent pipeline for voice-commanded task initiation.

(Unchanged from v5.0)

10. low_level/hardware_interface.py

Purpose: Low-level joint command dispatch and hardware telemetry (torque, encoder, IMU) ingestion.

(Unchanged from v5.0)

FSDQuant Integration Details

The FSDQuant modules originate in the autonomous vehicle domain and have been adapted for legged robotics as follows:

FSDQuant IDAV OriginDavid AdaptationFSD-6HVehicle mass / tire-slip estimationPayload mass + floor friction via joint torque residualsFSD-7APedestrian / vehicle intent predictionHuman hallway navigation predictionFSD-7BMulti-modal path sampling diversityDiffusion trajectory candidate diversityFSD-7CVehicle lateral stability barrierZMP support-polygon barrier (bipedal balance)FSD-2BLongitudinal CLF convergenceEnd-effector goal-convergence CLFFIX-KAutonomous vehicle sensor-fusion concurrencyRobotics world-model concurrent read safety 

Data Flow

Hardware Sensors │ torque_residual ▼ UKFPayloadObserver (FSD-6H) │ PhysicsParams {payload_mass, friction_mu} ▼ WorldModel.update_physics() Camera / Tracking │ │ agents[], robot_path │ ▼ │ SocialGrid.predict_intents_level_k() (FSD-7A) │ │ IntentPrediction[] │ ▼ │ WorldModel.update_intent_predictions() │ │ └──────────────────┬─────────────────┘ │ WorldStateSnapshot (lock-free) ▼ DiffusionVLA.generate_trajectory() │ N=16 raw trajectories ▼ V-FPS selection (FSD-7B) → K=3 diverse trajectories │ best trajectory ▼ DreamerLoop.constrain_cbf_clf() (FSD-7C/2B) │ safe waypoints + cbf_applied flag ▼ ActionChunk │ ▼ BehaviorTree / planner_node._tick_chunk() │ ▼ hardware_interface → Joint Commands 

Dependencies & Installation

pip install -r requirements.txt 

requirements.txt (v6.0 additions):

# Inherited from v5.0 torch diffusers pin3 (pinocchio python bindings) sounddevice # ... # NEW v6.0: Mathematical optimization and physics filtering scipy>=1.13.0 qpsolvers>=4.0.0 # CBF/CLF QP formulation numpy>=1.26.0 filterpy>=1.4.5 # UKF mathematical primitives 

ROS 2 (Humble or later) is required for planner_node.py.

Key Design Decisions

Why UKF over EKF for payload estimation? The relationship between payload mass and observed torque is mildly nonlinear (gravity × lever arm). The UKF's sigma-point propagation handles this without requiring an explicit Jacobian of the measurement model.

Why RCU over a standard RWLock? In a 10 Hz control loop, even a microsecond read-lock acquisition can accumulate jitter. RCU makes the reader path branch-free and lock-free — critical for the motor control thread that calls get_state() every tick.

Why V-FPS over plain top-K diffusion sampling? Top-K by score alone tends to cluster trajectories in the same local minimum (e.g., all 16 paths go left around an obstacle). V-FPS ensures the K selected paths represent genuinely different strategies, giving the downstream CBF/CLF solver meaningful alternatives to choose from.

Why SLSQP for the CBF QP in v6.0? scipy.optimize.minimize with SLSQP is dependency-minimal and readable for development. The architecture is designed for a drop-in swap to qpOASES or OSQP in production where sub-millisecond solve times are required.

Why does cbf_clf_applied reduce confidence to 0.7? When the CBF modifies a trajectory, the executed motion diverges from the VLA's learned intent. The reduced confidence signals to the Behavior Tree that the current action chunk may not achieve the semantic goal and a re-plan may be warranted sooner.

Versioning & Upgrade Notes

VersionKey Changev5.0Fluid diffusion-based generative motion; heuristic safety scaling (q * 0.7); constant-velocity human predictionv6.0FSDQuant stack: UKF payload observer, Level-K game-theoretic social prediction, CBF+CLF QP safety filter, V-FPS trajectory diversity, RCU lock-free world model 

Migration from v5.0:

DreamerLoop no longer exposes warp_trajectory(). Replace all call sites with constrain_cbf_clf().

SocialGrid.predict_intents() is replaced by predict_intents_level_k(agents, robot_path). The robot_path argument is required for Level-K reasoning; pass None to fall back to Level-0 (CV) behavior.

WorldModel no longer accepts direct mutation of _state_snapshot. Use the update_* writer methods exclusively.

David v6.0 — Built on provably correct foundations.

