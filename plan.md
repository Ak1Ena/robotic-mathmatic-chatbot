# Phase 1 MVP — Implementation Plan

> **Workflow:** Gemini agents code → Claude reviews output.
> AI layer: **Ollama** (model set via `OLLAMA_MODEL` in `.env`, default `llama3.2:3b`).

---

## Directory Structure

```
RobothicMathmatic_ChatWithAnimation/
├── backend/
│   ├── app/
│   │   ├── main.py                    # FastAPI app, CORS
│   │   ├── config.py                  # Settings from .env
│   │   ├── schemas/
│   │   │   ├── simulation.py          # SimulationJSON Pydantic contract ← WRITE FIRST
│   │   │   └── requests.py            # FKRequest, IKRequest, ChatRequest
│   │   ├── kinematics/
│   │   │   ├── forward.py             # compute_fk()
│   │   │   └── inverse.py             # compute_ik()
│   │   ├── simulation/
│   │   │   ├── builder.py             # SimulationBuilder
│   │   │   └── timeline.py            # TimelineFactory
│   │   ├── ai/
│   │   │   ├── ollama_client.py       # httpx async streaming client
│   │   │   └── mentor.py              # SocraticMentor, Socratic prompt
│   │   └── routers/
│   │       ├── kinematics.py          # /api/kinematics/fk, /ik
│   │       ├── simulation.py          # /api/simulation/scene, /health
│   │       └── chat.py                # /api/chat/mentor (SSE stream)
│   ├── tests/
│   │   ├── test_forward_kinematics.py
│   │   ├── test_inverse_kinematics.py
│   │   └── test_simulation_builder.py
│   ├── requirements.txt
│   └── .env.example
│
└── frontend/
    ├── src/
    │   ├── types/simulation.ts        # TypeScript mirror of SimulationJSON
    │   ├── store/
    │   │   ├── simulationStore.ts     # Holds SimulationJSON + playback state
    │   │   ├── controlsStore.ts       # θ1, θ2, L1, L2 slider values
    │   │   └── gateStore.ts           # Discovery Gate active state
    │   ├── api/client.ts              # Typed fetch wrappers
    │   ├── hooks/
    │   │   ├── useFK.ts               # Debounced (16ms) FK API call
    │   │   └── useIK.ts               # IK API call
    │   └── components/
    │       ├── canvas/
    │       │   ├── SimulationCanvas.tsx   # R3F <Canvas> orthographic
    │       │   ├── RobotArm2D.tsx         # Links + joints renderer
    │       │   ├── CoordinateFrame.tsx    # World/local axis arrows
    │       │   ├── VectorArrow.tsx        # Colored vectors
    │       │   └── EndEffectorMarker.tsx
    │       ├── controls/
    │       │   └── SliderPanel.tsx        # θ1, θ2, L1, L2 sliders
    │       ├── timeline/
    │       │   └── TimelineController.tsx # RAF-based playback engine
    │       ├── gates/
    │       │   ├── DiscoveryGate.tsx      # Amber overlay, pause/predict/resume
    │       │   └── PredictionInput.tsx
    │       └── chat/
    │           ├── ChatWindow.tsx
    │           ├── ChatMessage.tsx
    │           └── ChatInput.tsx
    ├── package.json
    ├── vite.config.ts
    └── tailwind.config.ts
```

---

## Simulation JSON Schema (the API contract between backend and frontend)

```python
# backend/app/schemas/simulation.py

class SimulationJSON(BaseModel):
    version: str = "1.0"
    scene: SceneConfig          # type="2D_robot_arm", links=[{id, length}]
    state: ArmState             # theta1, theta2, joint positions, end_effector {x,y}
    timeline: list[TimelineEvent]  # [{time, action, duration, joint, to}]
    gates: list[DiscoveryGate]  # [{id, at_time, type, prompt, hint}]
    vectors: list[VectorDef]    # [{id, type, color, origin, direction, magnitude}]
    frames: list[CoordinateFrameDef]  # [{id, origin, rotation_deg, label}]
    metadata: dict = {}
```

Corresponding TypeScript in `frontend/src/types/simulation.ts` **must stay in sync**.

---

## API Routes

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/kinematics/fk` | Body: `{theta1, theta2, l1, l2}` → SimulationJSON |
| POST | `/api/kinematics/ik` | Body: `{target_x, target_y, l1, l2, elbow_up}` → SimulationJSON with gate |
| POST | `/api/simulation/scene` | Body: `{scene_type, l1, l2, theta1, theta2}` → SimulationJSON |
| GET  | `/api/simulation/health` | → `{status, ollama}` |
| POST | `/api/chat/mentor` | Body: `{message, context, history}` → SSE stream |

---

## Key Kinematics Math

**Forward Kinematics (2D homogeneous transforms):**
```
T1 = R(θ1) @ Trans(L1, 0)
T2 = T1 @ R(θ2) @ Trans(L2, 0)
joint2_pos = T1[:2, 2]
ee_pos     = T2[:2, 2]
```

**Inverse Kinematics (geometric):**
```
cos(θ2) = (x² + y² − L1² − L2²) / (2·L1·L2)
θ2 = arccos(cos_theta2)   [elbow_up: +, elbow_down: -]
θ1 = arctan2(y, x) − arctan2(L2·sin(θ2), L1 + L2·cos(θ2))
```

---

## Ollama Integration

- Endpoint: `http://localhost:11434/api/chat` (stream=True)
- Model: from `.env` → `OLLAMA_MODEL=llama3.2:3b`
- System prompt enforces Socratic style: **never gives direct answers, only questions + hints**
- Chat endpoint streams SSE tokens; final token sends `data: [DONE]`

---

## Color Convention (enforce everywhere)

| Color | Hex | Meaning |
|-------|-----|---------|
| Cyan | `#00FFFF` | Velocity / motion vectors |
| Magenta | `#FF00FF` | Force / torque |
| Yellow | `#FFD700` | Control setpoints / targets |
| Amber | `#FFA500` | Discovery Gates |

---

## Architecture Rules (do not violate)

1. **Frontend is a pure renderer** — no physics, no AI logic whatsoever
2. **Sliders → `controlsStore` only** — `useFK` hook handles API sync asynchronously
3. `RobotArm2D` may do trivial local trig (4 ops) for zero-lag visual feedback only
4. **Discovery Gates are mandatory** — simulation must support pause/predict/resume at any timeline point
5. **Socratic mentor never gives direct answers** — only guided questions and hints
6. CORS: allow `http://localhost:5173` from the backend

---

## Build Order

1. `backend/app/schemas/simulation.py` — Pydantic models (contract)
2. `backend/app/kinematics/` + tests — FK/IK math
3. `backend/app/simulation/` — builder + timeline
4. `backend/app/routers/` + `main.py` — HTTP routes
5. `backend/app/ai/` — Ollama client + Socratic mentor
6. Frontend scaffolding (`pnpm create vite`)
7. `frontend/src/types/` + stores + API client
8. Canvas + robot rendering
9. Sliders + `useFK` real-time loop
10. Timeline player + Discovery Gates
11. Chat panel (SSE streaming)

---

## Verification Checklist

- [ ] `pytest backend/tests/` — all pass (FK round-trip within 1e-4)
- [ ] `GET /api/simulation/health` → `{"status":"ok","ollama":"ok"}`
- [ ] `POST /api/kinematics/fk` → valid SimulationJSON with coordinate frames
- [ ] `POST /api/kinematics/ik` → SimulationJSON with discovery gate + animation timeline
- [ ] Browser: slider moves → arm updates with zero perceptible lag
- [ ] Browser: IK scene → arm animates, amber gate overlay appears
- [ ] Browser: chat → Socratic response streams in from Ollama
