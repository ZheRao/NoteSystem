# Music Intelligence Project Plan

## Purpose

This document parks a future research-and-engineering project so it can be resumed cleanly after current focus on `primitive-neural-networks` and autograd.

The project has two initial tracks:

1. **Piano music generation**
2. **Song generation**

The immediate goal is **not** to build now. The goal is to preserve the right invariants, scope boundaries, architecture choices, and production pipeline decisions so future work starts from clarity instead of re-discovery.

## North Star

Build a music-generation system that can:

* generate continuations from a prior musical prefix
* generate from no start (unconditional generation)
* eventually support conditioning from text / description / mood
* serve as an early stepping stone toward broader **music-intelligence-signal research**

This project should be treated as:

* a **representation-learning project**
* a **sequence modeling project**
* a **systems/pipeline project**
* a future bridge between symbolic structure and richer semantic/audio intelligence

## Core Invariants To Preserve

### Invariant 1 — Representation is the first bottleneck

Poor representation causes the model to learn surface sequencing instead of music structure.

### Invariant 2 — Symbolic and audio music are different problem classes

* **Piano / composition** is best treated first as a **symbolic event-sequence problem**.
* **Songs / produced audio** are not just note sequences; they involve timbre, texture, vocals, mixing, and acoustics.

### Invariant 3 — Start with symbolic music before raw audio

For early serious work, use **MIDI/event tokens** first. Do not begin with MP3/WAV tokenization unless the project explicitly targets acoustic generation.

### Invariant 4 — Tokenization must respect musical structure

Useful units include:

* bar
* position
* pitch
* duration
* velocity
* tempo
* track / instrument identity when relevant

### Invariant 5 — Music is hierarchical, not just sequential

Notes form motifs, motifs form phrases, phrases form sections. Flat token streams are useful, but only as a practical approximation.

### Invariant 6 — First solve composition, then solve sound

Generation of good musical structure and generation of realistic audio are distinct milestones.

### Invariant 7 — Production pipeline matters as much as the model

A weak tokenization/data/rendering/generation pipeline can make a good model look bad.

### Invariant 8 — Evaluation must include listening, not just loss

A low training loss is not enough. Music must be judged by:

* structure
* rhythm stability
* phrase coherence
* repetition vs novelty balance
* actual listening quality

### Invariant 9 — Continuation and unconditional generation should be separate benchmarks

A model that continues a prefix well may still be poor at free generation.

### Invariant 10 — Keep the first version small, inspectable, and reproducible

Future versions may become ambitious, but the first proper version must be understandable end-to-end.

## Project Split

## Track A — Piano Music Generation

### Objective

Generate symbolic piano music:

* from an optional starting prefix
* or from scratch
* with controllable sequence length
* with outputs renderable to MIDI and audio

### Recommended source format

**Primary source:** MIDI

Why:

* directly captures note events
* avoids unnecessary acoustic complexity
* supports exact symbolic manipulation
* ideal for studying tokenization and sequence modeling

### Recommended tokenization

**Primary choice:** REMI-style event tokens

Suggested event vocabulary:

* `Bar`
* `Position`
* `Pitch`
* `Duration`
* `Velocity`
* optional `Tempo`
* optional `TimeSignature`

#### Why REMI first

* structurally meaningful
* interpretable
* proven useful for symbolic music generation
* easier to debug than compressed compound formats

### Alternative tokenizations to consider later

1. **MIDI-like** (`Note-On`, `Note-Off`, `Time-Shift`)

   * useful baseline
   * more primitive and flexible
   * weaker explicit musical structure

2. **Compound-word / grouped event tokens**

   * may reduce sequence length
   * useful later if long-sequence efficiency becomes the main bottleneck

3. **Piano-roll / timestep matrix**

   * good educational baseline only
   * not preferred for serious structured generation

### First architecture recommendation

**Decoder-only autoregressive model** over event tokens.

Good first options:

1. small autoregressive Transformer
2. small recurrent baseline (LSTM/GRU)
3. future hybrid experiments

#### Recommended initial baseline stack

* **Baseline 1:** LSTM over REMI
* **Baseline 2:** Decoder-only Transformer over REMI

Why both:

* LSTM gives streaming intuition
* Transformer gives stronger long-range pattern modeling
* useful for comparing architectural tradeoffs cleanly

### Generation modes

1. **Continuation generation**

   * input: prefix sequence
   * output: continuation events

2. **Unconditional generation**

   * input: start token / seed / random short primer
   * output: full generated sequence

### Sampling controls

Must support at generation time:

* temperature
* top-k
* top-p (optional)
* max length
* early stop conditions
* validity constraints (e.g. event grammar)

### Important piano milestones

#### Milestone P1 — Data pipeline works

* MIDI ingestion
* tokenization
* detokenization
* rendering to MIDI
* rendering MIDI to audio for listening

#### Milestone P2 — Baseline model learns local structure

* train loss decreases
* generated output is syntactically valid
* no catastrophic event corruption

#### Milestone P3 — Continuations sound musical

* phrase continuation is coherent
* rhythm mostly stable
* not pure memorization

#### Milestone P4 — Unconditional generation works

* from seed only
* sustained structure for at least short pieces

#### Milestone P5 — Optional text conditioning

* mood / style / tempo tags
* simple symbolic conditioning before free-form descriptions

## Track B — Song Generation

### Objective

Generate songs with increasing realism across stages.

This must be split into two distinct subproblems.

### Subproblem B1 — Symbolic song structure

Generate:

* melody
* harmony
* chord progression
* accompaniment
* optional multitrack arrangement

### Subproblem B2 — Acoustic song rendering

Generate:

* timbre
* vocals
* texture
* production quality audio

### Crucial decision

Do **not** treat full song generation as one monolithic first project.
Break it into layers.

### Recommended roadmap for songs

#### Stage S1 — Symbolic song generation

Use MIDI or event-based multitrack symbolic representations.

Potential token components:

* bar
* position
* track id
* instrument id
* pitch
* duration
* velocity
* tempo
* section markers (optional)

Goal:

* produce musically structured multitrack symbolic songs
* optionally include a vocal melody track symbolically

#### Stage S2 — Condition symbolic generation

Condition on:

* tags: mood / genre / tempo / energy
* simple text descriptions
* maybe chord progressions

#### Stage S3 — Acoustic realization

Convert symbolic output into audio through:

* high-quality synths / soundfonts / VST rendering
* or later learned audio generation stack

#### Stage S4 — True audio-native song generation

Only at this stage consider:

* waveform-level models
* neural codec tokens
* vocal/acoustic generation

### Source format recommendations for songs

#### For structure/composition

* MIDI / multitrack MIDI
* optional lyric metadata if available

#### For acoustic generation

* WAV internally
* MP3 only as input storage if unavoidable
* learned discrete codec tokens if doing true audio modeling

### Architecture recommendation for songs

#### First proper song architecture

A **hierarchical symbolic model**:

* section/phrase planner (future)
* event-level generator
* separate rendering stage

#### Earliest practical version

* single autoregressive model over multitrack symbolic event tokens
* condition on simple metadata
* render with deterministic external audio tools

### Song milestones

#### Milestone S1 — Multitrack tokenization works

* parse tracks
* normalize timing
* merge or preserve tracks in a consistent ontology
* render back without corruption

#### Milestone S2 — Symbolic multitrack generation works

* melody and accompaniment are coherent enough to listen to

#### Milestone S3 — Conditioning works

* mood / genre / prompt has visible influence

#### Milestone S4 — Audio production pipeline works

* symbolic output can be rendered to convincing demos

#### Milestone S5 — Audio-native research begins

* codec token experiments
* semantic-to-audio generation

## Recommended Final Scope For First Future Project

To avoid over-scoping, the first future implementation should be:

### Project 1 — Symbolic Piano Generator

**Goal:** A proper, reproducible symbolic piano generation system.

Includes:

* MIDI ingestion
* REMI tokenization
* train/val/test split
* decoder-only autoregressive model
* continuation + unconditional generation
* MIDI + audio rendering
* experiment tracking

Why this first:

* small enough to finish
* deeply aligned with representation learning
* direct bridge from previous 2022 work
* useful as a foundation for all later music research

### Project 2 — Symbolic Multitrack Song Generator

Only after Project 1 is solid.

### Project 3 — Text-conditioned symbolic music

After Project 1 or 2.

### Project 4 — Audio-native song generation

Park for much later.

## Production Pipeline Blueprint

## Pipeline A — Piano symbolic pipeline

1. Collect MIDI data
2. Validate / normalize files
3. Tokenize to REMI
4. Build vocabulary and token metadata
5. Create train/val/test datasets
6. Train autoregressive model
7. Generate token sequences
8. Validate token grammar
9. Detokenize to MIDI
10. Render MIDI to audio
11. Store artifacts + metrics + listening samples

## Pipeline B — Symbolic song pipeline

1. Collect multitrack MIDI
2. Normalize timing / meter / tempo handling
3. Define multitrack token ontology
4. Tokenize with track/instrument identity
5. Train autoregressive symbolic model
6. Generate symbolic output
7. Detokenize to MIDI
8. Render via soundfonts / synths / DAW-compatible export
9. Store outputs and evaluation samples

## Pipeline C — Audio-native pipeline (future)

1. Collect audio dataset
2. Convert to waveform
3. Encode with learned codec into discrete tokens
4. Train sequence model over audio tokens or semantic+audio hierarchy
5. Decode to waveform
6. Evaluate acoustics + musicality

## System Design Requirements

### Repository structure should separate these concerns

* `data/` raw and processed metadata
* `ingest/` source parsers
* `tokenizers/` symbolic + future audio tokenizers
* `schemas/` token vocab and event definitions
* `datasets/` loaders and sequence builders
* `models/` LSTM / Transformer / future hybrids
* `train/` training loops and configs
* `generate/` inference and sampling
* `render/` MIDI/audio export
* `eval/` metrics and listening set generation
* `experiments/` configs and logs
* `docs/` invariants, decisions, notes

### Configuration principles

* no hard-coded experiment parameters
* all tokenization and model choices externalized in config
* deterministic seeds where possible
* version every dataset + tokenizer + vocabulary

### Artifact requirements

Every experiment should save:

* model config
* tokenizer version
* dataset version
* training curves
* generated samples
* MIDI outputs
* audio renders
* short evaluation notes

## Tokenizer Design Requirements

### Symbolic tokenizer must support

* reversible encode/decode
* grammar validation
* sequence statistics
* unknown/invalid event reporting
* configurable quantization
* optional track-aware mode

### Questions to settle when implementation begins

1. What time quantization is used per bar?
2. Are note durations explicit tokens or inferred from note-off?
3. How are tempo changes handled?
4. How are sustain pedal and expressive controls handled?
5. Are tracks merged or preserved separately?
6. What constraints prevent invalid event orderings?

### Strong recommendation

Write tokenizer as a real subsystem, not notebook code.

It should expose:

* `encode(midi) -> token sequence`
* `decode(tokens) -> midi`
* `validate(tokens) -> report`
* `summarize(tokens) -> stats`

## Model Design Requirements

### For first piano model

Use **causal autoregressive prediction** only.

Training objective:

* predict next event token
* sequence shifted right
* causal masking for Transformer

### Do not repeat previous architecture mistake

For Transformer generation, use:

* **decoder-only / causal self-attention**
  not bidirectional encoder-only attention

### Baseline architecture progression

1. simple LSTM baseline
2. small decoder-only Transformer
3. improved Transformer with better context and sampling
4. optional hierarchical model later

### Constraints for first future model

* must fit on modest hardware
* must be trainable end-to-end in a few hours to a day
* sequence lengths chosen based on representation statistics, not guesswork

## Evaluation Framework

## Quantitative evaluation

Use cautiously. Metrics should support, not replace, listening.

Possible measurements:

* training / validation loss
* token validity rate
* note density distribution
* pitch-class distribution
* repetition statistics
* rhythm distribution
* length of valid generated continuation

## Qualitative evaluation

Create a fixed listening set:

* 10 continuation generations from fixed seeds
* 10 unconditional generations
* compare across checkpoints

Judge on:

* rhythm stability
* phrase coherence
* melodic shape
* unwanted collapse/repetition
* memorization suspicion
* novelty vs musicality balance

## Overfitting checks

* compare generations against nearest training pieces
* use holdout composers / pieces if dataset size allows
* inspect whether continuations merely copy long spans

## Minimal Future Roadmap

## Phase 0 — Park now

Output of this document:

* preserve invariants
* do not implement yet
* keep focus on `primitive-neural-networks`

## Phase 1 — Re-entry prep

When ready:

* review this document
* choose Track A only
* implement tokenizer subsystem first
* establish reversible MIDI round-trip before any modeling

## Phase 2 — Piano baseline

* REMI tokenizer
* dataset builder
* LSTM baseline
* decoder-only Transformer baseline
* continuation generation

## Phase 3 — Proper piano project release

* add unconditional generation
* add evaluation suite
* add rendering pipeline
* write real documentation / examples / reproducible commands

## Phase 4 — Extension paths

Choose one:

1. text-conditioned symbolic piano
2. multitrack symbolic song generation
3. hierarchical phrase-level planning

## Phase 5 — Long-horizon research

* semantic music planning
* learned latent abstractions over phrases/sections
* audio token generation
* description-to-music or idea-to-music systems

## Suggested Success Criteria For The First Real Future Project

A successful first future music project is **not**:

* state-of-the-art audio quality
* commercial music generation
* full vocal song synthesis

A successful first future music project **is**:

* clean symbolic representation
* correct autoregressive architecture
* reproducible training
* good continuation generation
* decent unconditional symbolic generation
* strong tokenizer/pipeline subsystem
* clear documentation of what worked and failed

## Explicit De-scoping Rules

To avoid losing years to ambition drift, do **not** include these in v1:

* raw MP3 generation
* singing voice synthesis
* diffusion audio models
* giant datasets requiring distributed training
* end-to-end text-to-produced-song systems
* DAW-grade mixing/mastering automation

These are future expansions, not first-project requirements.

## Relationship To Current Trajectory

This project is parked because current high-priority sequencing is correct:

1. build foundations through autograd and primitive neural networks
2. continue strengthening engineering/system-building through real production work
3. return later with stronger architecture understanding and better pipeline discipline

This means the music project should later become stronger in three dimensions:

* better model understanding
* better tokenizer/system design
* better production/research discipline

## Final Decision

### Park — high future value

This project is worth revisiting, but only after the current foundation-building stage is stronger.

When resumed, the correct first implementation is:

**A symbolic piano generation project using MIDI + REMI + causal autoregressive modeling + proper render/evaluation pipeline.**

Everything else grows from that.

## What I think is the real tokenizer bottleneck

The bottleneck is not only “what discrete units should I use?” It is:

**What should be discrete at all, and at what level?**

For Northern Light language, you already dislike flattening everything into one token stream. Music makes that flaw even more obvious. The strongest design principle here is:
- keep **continuous reality continuous** as long as possible
- discretize only where there is a meaningful symbolic boundary
- use **hierarchy** instead of one flat token stream

That suggests this progression:
1. Piano: note/event tokens
2. Song semantics: phrase/section/conditioning tokens
3. Final audio: learned codec tokens
4. Eventually: a model that plans in semantic music space and renders to sound later

That is extremely aligned with your existing perception → abstraction → expression split.