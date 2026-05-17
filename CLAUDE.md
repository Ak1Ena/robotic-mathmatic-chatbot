# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Status

This project is in **pre-implementation / documentation phase**. No source code exists yet. The full specification is in `project.md`, `CONCEPT.md`, `requirement.md`, and `roadmap.md`.

## What This Project Is

An AI-powered robotics mathematics education platform. The core idea: instead of teaching formulas, the system makes math physically intuitive through cinematic visualization and guided Socratic discovery.

Teaching flow: `Visualization → Interaction → Prediction → Explanation → Mathematics`

## Architecture: JSON-Driven Simulation Pipeline

This is the central architectural contract — the AI never directly renders graphics. Instead:

1. **Backend** (FastAPI + NumPy/SymPy) computes kinematics/physics and an AI agent determines how to visualize the result
2. **Simulation JSON** is generated — a structured payload with scene, timeline, and interactive gates
3. **Frontend** (React + Three.js/R3F) renders deterministically from the JSON

Example payload shape:
```json
{
  "scene": { "type": "2D_robot_arm" },
  "timeline": [
    { "time": 0, "action": "show_coordinate_frame" },
    { "time": 2, "action": "rotate_joint", "joint": "theta1" }
  ]
}
```

The frontend must be a **pure renderer** — no physics logic, no AI reasoning. This separation is a hard architectural requirement from `requirement.md §6.2`.

## Tech Stack (Planned)

| Layer | Technology |
|-------|-----------|
| Frontend | React, Three.js, React Three Fiber, TailwindCSS |
| Backend | Python, FastAPI |
| Math | NumPy, SymPy, SciPy |
| AI | OpenAI API (Socratic mentor), LangGraph (optional orchestration) |
| Database | PostgreSQL, Redis (optional cache) |

## Phase 1 MVP Scope

Build only these features first (2D Two-Link Arm):

- FastAPI backend with FK/IK kinematics engine (NumPy)
- Simulation JSON Schema definition
- React + R3F 2D canvas renderer interpreting that schema
- Forward Kinematics (end-effector position + coordinate frame updates)
- Inverse Kinematics solver with animated joint movement
- Interactive sliders for joint angles and link lengths
- Coordinate frame + vector visualization
- Cinematic timeline sequencing with smooth transitions
- Discovery Gates: simulation pauses, user predicts outcome, simulation resumes
- Basic OpenAI integration for Socratic hints

## Visual Language Convention

These color assignments are part of the instructional design and must be consistent throughout the codebase:

| Color | Meaning |
|-------|---------|
| Cyan | Velocity and motion vectors |
| Magenta | Force and torque applications |
| Yellow | Control setpoints and error margins |
| Amber | Discovery Gates (interactive prediction pauses) |

## Key Design Decisions

- **Separation of concerns**: AI/physics backend vs. deterministic frontend renderer. The JSON schema is the API contract between them.
- **Discovery Gates** are first-class citizens — not optional. The simulation must support pause/predict/resume flow at arbitrary timeline points.
- **Intuition before abstraction**: visualizations precede equation rendering. LaTeX display is Phase 2, not MVP.
- **Real-time slider updates**: simulation state must recompute instantly on input change (no debounce that causes perceptible lag).
