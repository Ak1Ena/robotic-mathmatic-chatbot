# Gemini Task Brief — Phase 1 MVP Implementation

> Read `plan.md` for the full architecture. This file is your task list.
> Use `@backend` for backend tasks, `@frontend` for frontend tasks, `@tester` for tests.
> Claude will review all output.

---

## Your Mission

Implement Phase 1 MVP of the RobothicMathmatic project — an AI-powered robotics math education platform.

**Key constraint:** The frontend is a PURE RENDERER. It only interprets JSON from the backend. No physics logic on the frontend.

---

## Task 1 — Simulation JSON Schema (do this FIRST)

**Agent: `@backend`**

Create `backend/app/schemas/simulation.py` with full Pydantic v2 models matching this contract:

```python
class SimulationJSON(BaseModel):
    version: str = "1.0"
    scene: SceneConfig          # type="2D_robot_arm", links list
    state: ArmState             # theta1, theta2, joint positions, end_effector
    timeline: list[TimelineEvent]
    gates: list[DiscoveryGate]
    vectors: list[VectorDef]
    frames: list[CoordinateFrameDef]
    metadata: dict = {}
```

Also create `backend/app/schemas/requests.py` with:
- `FKRequest`: theta1, theta2, l1, l2
- `IKRequest`: target_x, target_y, l1, l2, elbow_up=True
- `ChatRequest`: message, context (robot state dict), history list

---

## Task 2 — Backend Kinematics Engine

**Agent: `@backend`**

File: `backend/app/kinematics/forward.py`
```python
def compute_fk(theta1_deg, theta2_deg, l1, l2) -> dict:
    # 2D homogeneous transforms via NumPy
    # Returns: joint1_pos, joint2_pos, end_effector (x,y), T1, T2, T_ee matrices
```

File: `backend/app/kinematics/inverse.py`
```python
def compute_ik(target_x, target_y, l1, l2, elbow_up=True) -> dict:
    # Geometric IK: cos(θ2) = (x²+y²-L1²-L2²)/(2·L1·L2)
    # Raises WorkspaceError if target unreachable
    # Returns same shape as compute_fk
```

Also create `backend/requirements.txt`:
```
fastapi
uvicorn[standard]
numpy
sympy
scipy
httpx
pydantic>=2.0
pydantic-settings
python-dotenv
pytest
pytest-asyncio
```

---

## Task 3 — Simulation Builder

**Agent: `@backend`**

File: `backend/app/simulation/builder.py`
Class `SimulationBuilder` with methods:
- `build_fk_scene(fk_result, l1, l2, theta1, theta2) -> SimulationJSON`
- `build_ik_scene(ik_result, start_theta1, start_theta2, target_x, target_y, l1, l2) -> SimulationJSON`
- `build_explore_scene(fk_result, l1, l2) -> SimulationJSON`

IK scene must include:
- A `DiscoveryGate` at `time=0` asking "Where do you predict the arm will reach?"
- A `ROTATE_JOINT` timeline event for each joint
- A yellow `VectorDef` (target marker) at `(target_x, target_y)`

---

## Task 4 — FastAPI Routes

**Agent: `@backend`**

Files:
- `backend/app/main.py` — FastAPI app with CORS allowing `http://localhost:5173`
- `backend/app/config.py` — reads `.env`: `OLLAMA_URL`, `OLLAMA_MODEL`, `CORS_ORIGINS`
- `backend/app/routers/kinematics.py` — POST `/api/kinematics/fk` and `/api/kinematics/ik`
- `backend/app/routers/simulation.py` — POST `/api/simulation/scene`, GET `/api/simulation/health`
- `backend/.env.example`:
  ```
  OLLAMA_URL=http://localhost:11434
  OLLAMA_MODEL=llama3.2:3b
  ```

---

## Task 5 — Ollama AI Mentor

**Agent: `@backend`**

File: `backend/app/ai/ollama_client.py`
```python
class OllamaHTTPClient:
    async def stream_chat(self, messages: list[dict]) -> AsyncGenerator[str, None]:
        # POST http://localhost:11434/api/chat with stream=True
        # Yields token strings
    async def check_health(self) -> bool: ...
```

File: `backend/app/ai/mentor.py`
System prompt must enforce Socratic style:
- NEVER give direct answers
- ONLY ask guided questions and give hints
- Keep responses under 80 words
- Always reference current robot state in questions

File: `backend/app/routers/chat.py`
- POST `/api/chat/mentor` → `StreamingResponse` with `text/event-stream`
- Each SSE event: `data: <token>\n\n`
- Final: `data: [DONE]\n\n`

---

## Task 6 — Frontend Scaffolding

**Agent: `@frontend`**

From project root, run:
```bash
pnpm create vite frontend --template react-ts
cd frontend
pnpm add three @react-three/fiber @react-three/drei zustand tailwindcss @tailwindcss/vite
```

Create:
- `frontend/src/types/simulation.ts` — TypeScript mirror of SimulationJSON Pydantic schema
- `frontend/src/store/simulationStore.ts` — Zustand store for SimulationJSON + playback state
- `frontend/src/store/controlsStore.ts` — Zustand store for θ1, θ2, L1, L2
- `frontend/src/store/gateStore.ts` — Zustand store for Discovery Gate state
- `frontend/src/api/client.ts` — typed fetch wrappers for all backend endpoints

---

## Task 7 — Canvas + Robot Rendering

**Agent: `@frontend`**

- `SimulationCanvas.tsx` — R3F `<Canvas>` with orthographic camera, `zoom: 100` (1 unit = 1 meter)
- `RobotArm2D.tsx` — renders links as thick lines, joints as circles. Reads `controlsStore` for zero-lag immediate rendering. Also reads `simulationStore` for coordinate frames.
- `CoordinateFrame.tsx` — X axis in red, Y axis in green, at given origin and rotation
- `VectorArrow.tsx` — colored arrow. Colors: cyan=`#00FFFF`, magenta=`#FF00FF`, yellow=`#FFD700`
- `EndEffectorMarker.tsx` — small yellow circle at end-effector

---

## Task 8 — Controls + Real-Time FK Loop

**Agent: `@frontend`**

- `SliderPanel.tsx` — four range sliders: θ1 (-180° to 180°), θ2 (-180° to 180°), L1 (0.3 to 2.0m), L2 (0.3 to 2.0m)
- `hooks/useFK.ts`:
  ```typescript
  // Subscribes to controlsStore
  // Debounces 16ms (one frame)
  // POSTs to /api/kinematics/fk
  // Updates simulationStore.setSimulation()
  ```
- Wire together in `App.tsx`: canvas left, controls panel right

---

## Task 9 — Timeline Player

**Agent: `@frontend`**

- `hooks/useSimulationPlayer.ts` — uses `requestAnimationFrame`, advances `playbackTime`
- `TimelineController.tsx` — processes `SimulationJSON.timeline` events by time:
  - `ROTATE_JOINT`: interpolate angle with smoothstep easing over `event.duration`
  - `SHOW_COORDINATE_FRAME` / `HIDE_COORDINATE_FRAME`: toggle frame visibility
  - When `playbackTime` reaches a gate's `at_time`: pause, call `gateStore.activateGate()`

---

## Task 10 — Discovery Gates

**Agent: `@frontend`**

- `DiscoveryGate.tsx` — full amber (`#FFA500`) overlay when `gateStore.activeGate !== null`
  - Shows `gate.prompt`
  - Has `PredictionInput` + submit button
  - On submit: `gateStore.submitAndReveal()` → timeline resumes
- `PredictionInput.tsx` — text input for user's prediction

---

## Task 11 — Chat Panel

**Agent: `@frontend`**

- `ChatWindow.tsx` — scrollable message list, auto-scroll to bottom
- `ChatMessage.tsx` — user bubble (right, blue) and mentor bubble (left, dark)
- `ChatInput.tsx` — text input + send button
- Connect to SSE stream from `/api/chat/mentor`
- Pass current robot state from `controlsStore` + `simulationStore` as context
- Wire into `SidePanel.tsx` below the sliders

---

## Task 12 — Tests

**Agent: `@tester`**

Write pytest tests in `backend/tests/`:

```python
# test_forward_kinematics.py
# compute_fk(0, 0, 1.0, 0.8) → EE ≈ (1.8, 0.0)
# compute_fk(90, 0, 1.0, 0.8) → EE ≈ (0.0, 1.8)
# compute_fk(45, -45, 1.0, 1.0) → EE ≈ (1.414, 0.0)

# test_inverse_kinematics.py
# Round-trip FK→IK→FK: EE position matches within 1e-4
# Unreachable target raises WorkspaceError

# test_simulation_builder.py
# builder.build_fk_scene() → validates as SimulationJSON without error
# builder.build_ik_scene() → has len(gates) > 0 and len(timeline) > 0
```

---

## Done Criteria

After all tasks complete, verify:
1. `cd backend && pytest tests/ -v` — all pass
2. `uvicorn app.main:app --reload` — starts without error
3. `curl -X POST http://localhost:8000/api/kinematics/fk -H "Content-Type: application/json" -d '{"theta1":45,"theta2":30,"l1":1.0,"l2":0.8}'` — returns valid JSON
4. `cd frontend && pnpm dev` — starts without error
5. Open browser at `http://localhost:5173` — see robot arm
6. Move sliders — arm updates instantly
7. Chat with mentor — Socratic response streams in
