# Xiangqi Reinforcement Learning Project Plan

## Purpose

This document parks a future research-and-engineering project so it can be resumed when neural-network, reinforcement-learning, algorithms, and software-engineering foundations are stronger.

The project idea predates the current implementation stage: build a **Chinese Chess (Xiangqi) environment and intelligent agent that learns to play increasingly well**, eventually becoming something a human user — especially my father — can play against.

The immediate goal is **not** to build now. The goal is to preserve the motivation, scope, architectural direction, learning opportunities, and de-scoping rules so the idea does not need to be rediscovered later.

## North Star

Build a complete Xiangqi system in which:

* a human can play Xiangqi through a usable interface
* the game engine correctly represents all rules and legal actions
* multiple agents can play within the same environment
* increasingly sophisticated agents can be introduced over time
* a neural agent can learn a policy and/or value function
* reinforcement learning and self-play can improve playing strength
* agent versions can be evaluated objectively against prior versions and fixed opponents
* the final system is understandable end-to-end rather than being a wrapper around a black-box chess library

The personal end state is simple:

**My father can sit down and play Xiangqi against an agent that I designed, implemented, trained, and improved myself.**

## Why This Project Matters

This is not merely a game project.

It can eventually integrate several areas of study into one coherent system:

* programming and software design
* data structures
* algorithms and search
* state-space modeling
* testing and correctness
* neural networks
* reinforcement learning
* experiment design
* model evaluation
* training infrastructure
* inference and serving
* frontend / human interaction
* potentially production deployment

It is therefore best treated as a **long-horizon capstone project**, not a small side project.

## Core Invariants To Preserve

### Invariant 1 — Correct environment before intelligent agent

The game itself must be correct before learning begins.

The first serious subsystem is therefore:

**state + rules + legal actions + transitions + terminal conditions**

A sophisticated agent trained against a subtly incorrect environment learns the wrong game.

### Invariant 2 — Separate environment from agent

The Xiangqi engine must not depend on any particular AI technique.

Conceptually:

```text
state
  ↓
legal_actions(state)
  ↓
agent chooses action
  ↓
transition(state, action)
  ↓
next_state + reward + terminal
```

A human, random policy, search algorithm, or neural policy should all interact through the same environment contract.

### Invariant 3 — Intelligence should progress in stages

Do not begin with deep reinforcement learning.

A useful progression is:

```text
Human
  ↓
Random policy
  ↓
Minimax
  ↓
Alpha-beta search
  ↓
Handcrafted evaluation
  ↓
Neural value / policy model
  ↓
Self-play reinforcement learning
```

Each stage should establish a baseline that makes the next stage measurable.

### Invariant 4 — Reward must remain conceptually clean

The ultimate objective is winning the game.

A first terminal reward can remain simple:

```text
win   = +1
draw  =  0
loss  = -1
```

Reward shaping should only be introduced deliberately and with awareness that poorly chosen intermediate rewards can teach unintended behavior.

### Invariant 5 — State and action representation are fundamental design decisions

Before serious learning begins, explicitly determine:

* how the board is encoded
* how piece identity and ownership are represented
* how legal moves are represented
* whether actions use source/destination coordinates or another encoding
* how invalid actions are excluded
* how perspective is normalized between players
* how repetition/history-dependent rules are represented if needed

Representation quality will strongly affect both search and learning.

### Invariant 6 — Evaluation must be relative and reproducible

"Training loss decreased" does not mean the agent became better at Xiangqi.

Playing strength should be evaluated through controlled matches against:

* random agent
* fixed search agents
* previous model checkpoints
* potentially human players
* later, external Xiangqi engines if useful

Keep fixed evaluation conditions and enough games to reduce noise.

### Invariant 7 — Self-play requires protection against forgetting and false progress

A new agent beating its immediately previous version does not automatically mean global improvement.

Preserve historical checkpoints and periodically evaluate against a pool of older opponents and fixed baselines.

### Invariant 8 — Learning infrastructure is part of the project

Eventually the project is not only a model.

It may include:

```text
game engine
    ↓
self-play workers
    ↓
trajectory / experience storage
    ↓
training pipeline
    ↓
candidate model
    ↓
evaluation
    ↓
model promotion
    ↓
serving / human play
```

These components should be introduced only when the simpler system justifies them.

### Invariant 9 — Xiangqi is a feature, not a portfolio weakness

Do not switch to Western chess merely because it is more immediately recognizable.

The technical story is understandable without knowing Xiangqi rules:

**complete game environment + search + neural policy/value estimation + reinforcement learning + self-play + evaluation + deployment**

Xiangqi also gives the project authentic motivation and domain knowledge rather than making it another generic chess-engine project.

### Invariant 10 — Preserve inspectability

Especially while learning, prefer systems whose behavior can be traced and understood.

Do not hide the important RL mechanics behind a high-level library before understanding:

* state
* action
* transition
* reward
* return
* policy
* value
* exploration
* credit assignment
* optimization

Libraries can be introduced later without surrendering conceptual ownership.

## Recommended Language

### Primary language — Python

Python is the natural first choice because the central problem is **intelligent-agent learning**, not manual memory management.

It provides:

* rapid iteration
* NumPy
* PyTorch
* mature ML/RL tooling
* easy visualization
* straightforward experiment scripting

### Possible future native components

C/C++ or another systems language may later be useful for performance-critical components such as:

* move generation
* board evaluation
* search
* simulation throughput

Only introduce native code after profiling demonstrates a real bottleneck.

Do not make low-level implementation complexity part of v1 unless it is itself the learning objective.

## Project Architecture

A future repository might evolve toward:

```text
xiangqi-ai/

engine/
    board.py
    pieces.py
    moves.py
    rules.py
    state.py

agents/
    random_agent.py
    minimax_agent.py
    alpha_beta_agent.py
    neural_agent.py

models/
    policy.py
    value.py

training/
    self_play.py
    replay_buffer.py
    train.py
    evaluate.py

ui/
    ...

tests/
    ...

experiments/
    ...

docs/
    ...
```

This is a direction, not a required initial structure.

## Phase 0 — Park Now

Current action:

* preserve this project plan
* do not turn it into an active major commitment
* continue neural-network foundations
* continue CS/software-engineering foundations
* allow future learning to naturally clarify the architecture

The project should be started when there is enough technical and time capacity for it to become a primary learning project without disrupting higher-priority work.

## Phase 1 — Correct Xiangqi Environment

### Objective

Build a trustworthy Xiangqi game engine.

Required capabilities:

* board representation
* all piece types
* legal movement
* captures
* turn management
* check detection
* illegal self-check prevention
* checkmate / terminal-state detection
* draw handling as appropriate
* game reset
* move history if required

### Environment interface

Aim toward an explicit contract resembling:

```python
state = env.reset()

actions = env.legal_actions(state)

next_state, reward, terminated, info = env.step(action)
```

The exact API can change later.

### Success criterion

Two humans can play a complete legal game using the engine without AI.

## Phase 2 — Baseline Agents

### Agent 1 — Random policy

Choose uniformly or simply among legal actions.

Purpose:

* validate the agent/environment interface
* create the weakest measurable baseline
* exercise large numbers of game transitions

### Agent 2 — Minimax

Implement classical adversarial search.

Learn:

* game trees
* branching factor
* search depth
* terminal evaluation
* computational explosion

### Agent 3 — Alpha-beta pruning

Improve search efficiency while preserving minimax behavior.

Study:

* move ordering
* pruning
* depth limits
* search-performance measurement

### Agent 4 — Handcrafted evaluation

When full-depth search is impossible, evaluate intermediate board positions.

Potential concepts:

* material
* mobility
* king/general safety
* positional structure
* threats
* piece activity

The purpose is not to perfect Xiangqi heuristics. It is to experience the limitation that eventually motivates learned evaluation.

## Phase 3 — Neural Position Evaluation

Replace or supplement handcrafted evaluation with a neural network.

Possible first task:

```text
board state
    ↓
neural network
    ↓
estimated probability/value of eventual victory
```

Important questions:

* How should the board be encoded?
* Should the network output a scalar value?
* How should player perspective be represented?
* What training targets are available?
* Can positions generated by search/self-play create training data?
* How should calibration and playing strength be evaluated?

This phase creates the bridge between classical game AI and learned intelligence.

## Phase 4 — Policy Learning

Introduce a model that estimates promising actions:

```text
state
  ↓
policy network
  ↓
probability / preference over legal actions
```

The legal-action mask must prevent invalid actions from being selected.

Possible future combination:

```text
state
  ├──> policy head → action preferences
  └──> value head  → expected outcome
```

Do not commit to a specific architecture until the relevant neural-network and RL foundations have been studied.

## Phase 5 — Reinforcement Learning and Self-Play

### Core loop

Conceptually:

```text
current policy
    ↓
self-play games
    ↓
(state, action, reward/outcome) experience
    ↓
training
    ↓
candidate policy
    ↓
evaluation
    ↓
promote if stronger
```

This phase should be designed only after the simpler agents and neural baselines are working.

### Central learning problem

The final reward may arrive many moves after an important action.

The project should therefore become a practical environment for studying:

* delayed reward
* credit assignment
* value functions
* temporal-difference learning
* policy optimization
* exploration vs exploitation
* self-play
* bootstrapping
* stability
* catastrophic forgetting

## Phase 6 — Stronger Search + Learned Models

A future sophisticated agent may combine learned policy/value estimates with tree search.

Possible research direction:

```text
policy/value network
        +
guided tree search
        +
self-play
        +
iterative training
```

This is intentionally architecture-neutral for now.

Do not prematurely decide that the final system must reproduce AlphaZero or any other specific algorithm.

Understand the problem first; choose techniques later.

## Phase 7 — Human Play System

Build a usable interface where a human can:

* start a game
* choose a side
* move pieces visually
* see legal moves
* play against a selected agent/checkpoint
* restart or resign
* potentially choose difficulty

The agent should run behind the same environment interface used during training and evaluation.

### Personal milestone

**Dad can play a complete game against the trained agent.**

## Phase 8 — Persistent Improvement System

Only if the project eventually warrants production-like complexity, explore collecting games and using them to improve future models.

Potential architecture:

```text
human/self-play games
        ↓
validated trajectories
        ↓
experience dataset
        ↓
offline training
        ↓
candidate model
        ↓
evaluation gate
        ↓
promoted model
```

Do **not** automatically train directly on every user game in production.

Model updates should be controlled, evaluated, reproducible, and reversible.

## Testing Strategy

The engine requires unusually strong testing because incorrect rules contaminate everything above them.

### Unit tests

Test every piece and important boundary condition.

Examples:

* legal movement
* illegal movement
* captures
* blocked paths
* palace restrictions
* river-related movement constraints
* cannon capture behavior
* check
* moves exposing own general
* terminal states

### Invariant/property tests

Useful invariants may include:

* every returned legal action is executable
* executing a legal move changes exactly the intended state
* a player cannot legally leave their own general in check
* captured pieces disappear exactly once
* turns alternate correctly
* deterministic state/action inputs produce deterministic transitions

### Regression tests

Every discovered rule bug should receive a permanent regression test.

## Experiment Requirements

Once learning begins, every meaningful experiment should preserve:

* code/version identifier
* environment version
* state representation version
* action representation version
* model configuration
* random seeds where practical
* training configuration
* checkpoint
* evaluation opponents
* win/draw/loss statistics
* training curves
* short interpretation notes

Without this discipline, apparent RL progress can be extremely misleading.

## Evaluation Framework

### Fixed baselines

Maintain stable versions of:

1. random agent
2. shallow minimax
3. stronger alpha-beta/search agent
4. selected historical neural checkpoints

### Head-to-head evaluation

For each candidate model:

* play both sides
* use a sufficiently large match set
* track win/draw/loss
* compare against multiple opponents
* retain results over time

### Human evaluation

Human games are useful but noisy.

Use them as qualitative evidence, not the sole benchmark of improvement.

## Portfolio Story

The project should be presented in universally understandable technical terms rather than assuming the audience understands Xiangqi.

A future description might emphasize:

> Built an end-to-end reinforcement-learning game system for Xiangqi, including a rules engine, legal-action generation, adversarial-search baselines, neural policy/value models, self-play training, checkpoint evaluation, and interactive human play.

The unusual game domain can make the project more memorable while the underlying engineering and ML concepts remain general.

Strong documentation should explain Xiangqi only to the degree necessary to understand the technical problem.

## Explicit De-scoping Rules

To prevent the project from becoming enormous immediately, v1 should **not** require:

* reinforcement learning
* neural networks
* distributed self-play
* cloud training
* multiplayer accounts
* continuous online learning
* sophisticated frontend design
* native C/C++ optimization
* mobile apps
* production-scale deployment
* state-of-the-art Xiangqi strength

The first real deliverable is simply:

**a correct, tested Xiangqi environment.**

## Entry Criteria

Do not attach an arbitrary calendar date such as "start in two years."

Start serious implementation when:

* neural-network foundations are substantially stronger
* basic RL concepts are understood well enough to reason about the eventual direction
* algorithms/search foundations are sufficient to implement classical baselines
* there is enough spare capacity for Xiangqi to become a primary personal learning project
* starting it would not materially undermine higher-priority production work or foundational study

It is acceptable to implement the environment earlier if it naturally fits current learning, but doing so should not silently turn the parked project into another major obligation.

## Relationship To Current Trajectory

Current work contributes directly to this future project:

```text
CS foundations
    ├── data structures
    ├── algorithms/search
    ├── systems
    └── software design

Neural-network foundations
    ├── autograd
    ├── optimization
    ├── architectures
    └── training intuition

Future RL study
    ├── rewards
    ├── value functions
    ├── policies
    ├── credit assignment
    └── self-play

Production engineering
    ├── testing
    ├── APIs
    ├── databases
    ├── deployment
    └── observability

                ↓

          XIANGQI AGENT
```

The project therefore does not need to compete with current learning.

Current learning is preparation for it.

## Suggested Success Criteria

A successful project is **not** necessarily:

* grandmaster-level Xiangqi
* a novel RL algorithm
* a commercial game platform
* a huge neural network
* a perfect UI

A successful project **is**:

* a correct game environment
* clean separation between environment and agents
* meaningful classical baselines
* neural models whose role is understood
* reinforcement learning implemented with conceptual clarity
* reproducible evidence that later agents outperform earlier agents
* an end-to-end system that can be explained technically
* a human-play interface
* a system my father can actually play against

## Long-Horizon Research Questions

Park these rather than answer them now:

1. What is the best state representation for Xiangqi?
2. What is the cleanest fixed action space?
3. How large is the practical branching factor?
4. Which classical search baseline provides the right benchmark?
5. How should board symmetries or player perspective be exploited?
6. Should policy and value share one network?
7. What architecture best captures Xiangqi spatial relationships?
8. How should self-play opponents be sampled?
9. How should historical checkpoints be retained for evaluation?
10. What RL algorithm is appropriate?
11. When does tree search become useful alongside a neural model?
12. How much compute is needed for meaningful improvement?
13. Can human games improve the system without destabilizing it?
14. How should model promotion be gated?
15. What level of strength is sufficient for the personal goal?

These questions are deliberately unresolved because future knowledge should determine the answers.

## Final Decision

### Park — very high future value

This project is worth doing, but it should not become a competing major commitment before the foundations and available time justify it.

The correct first implementation is:

**A clean, fully tested Xiangqi environment with a stable agent interface and a random baseline.**

From there:

```text
correct game
    ↓
search
    ↓
evaluation
    ↓
neural networks
    ↓
reinforcement learning
    ↓
self-play
    ↓
human-facing agent
```

The long-term destination is deliberately ambitious.

The next step, when the project is eventually activated, is deliberately small.
