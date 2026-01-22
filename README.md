# SYSTEM ARCHITECTURE ANALYSIS REPORT

## SYSTEM CLASSIFICATION
AI-driven desktop automation system implementing a two-agent hierarchical command structure for autonomous UI interaction via vision-language model (Qwen3-VL-4b).

## CORE ARCHITECTURE

### Execution Model
**Type:** Stateless API communication with internal state management  
**Communication:** HTTP REST via LM Studio OpenAI-compatible endpoint  
**Agent Pattern:** Commander-Operator dual-agent architecture  
**Memory Model:** Hybrid - compressed semantic memory + rolling active history

### Agent Hierarchy

#### 1. TEAM LEADER (Commander Role)
**Function:** Strategic planning, objective decomposition, mission oversight  
**Mode:** Text-only evaluation (no screenshot required)  
**Decision Points:**
- Turn 1: Mission analysis and objective breakdown
- Every `team_review_interval` (default: 5) turns: Progress review
- On memory threshold breach: Compression execution
- On operator blocker escalation: Strategy adaptation

**Tools:**
- `spawn_operator`: Issue/update operator instructions (triggers mode switch to EXECUTION)
- `compress_memory`: Semantic history compression with loop analysis

**Temperature:** 0.35 (deterministic strategic thinking)  
**Max Tokens:** 1200

#### 2. OPERATOR SPECIALIST (Execution Role)
**Function:** Tactical UI manipulation, single-action execution  
**Mode:** Vision-language processing with screenshot input  
**Discipline:** ONE tool call per turn

**Tools (10 total):**
- UI Interaction: `click_element`, `double_click_element`, `right_click_element`, `drag_element`
- Input: `type_text`, `press_key`
- Navigation: `scroll_down`, `scroll_up`
- Reporting: `report_progress`, `report_completion`

**Temperature:** 0.5  
**Max Tokens:** 1024  
**Coordinate System:** Normalized 0-1000 scale (x,y), mapped to physical screen resolution

### State Management

#### AgentState (Primary State Container)
```
- task: str                    # Mission objective
- screenshot: bytes            # Current PNG capture
- screen_dims: (int, int)      # Physical resolution
- turn: int                    # Iteration counter
- mode: str                    # "EVALUATION" | "EXECUTION"
- memory: AgentMemory          # History manager
- plan: str                    # Commander's strategic plan
- operator_brief: str          # Current tactical instructions
- needs_review: bool           # Review flag for mode switch
```

#### AgentMemory (History Management)
```
- raw_history: list[ActionRecord]           # Full action log
- compressed_snapshots: list[MemorySnapshot] # Archived semantic summaries
- last_recovery_turn: int                    # Loop recovery tracking
```

**Compression Trigger:** Active history >= `memory_compression_threshold` (default: 8)  
**Compression Mechanism:** Commander archives turns into semantic snapshot, marks actions as archived, retains only active history in context

**Context Window Strategy:**
- Compressed snapshots: Summary (180 chars) + loop pattern (120 chars)
- Active history: Last 8 actions with tool/result/sitrep
- Total context assembled per turn for LLM input

### Mode Switching Logic

```
EVALUATION → EXECUTION: spawn_operator called
EXECUTION → EVALUATION: 
  - Every team_review_interval turns
  - needs_review flag set (objective DONE/BLOCKED)
  - memory.needs_compression == True
```

**Critical Design:** Commander never sees screenshots, only textual history/reports. Operator always processes visual input.

## TECHNICAL SUBSYSTEMS

### 1. Screen Capture Pipeline
**Technology:** Windows GDI direct bitmap capture  
**Process:**
1. GetDC → Desktop device context
2. CreateCompatibleDC → Memory DC
3. CreateDIBSection → 32-bit BGRA bitmap
4. StretchBlt (HALFTONE) → Downscale to target resolution (1536x864)
5. Cursor overlay via DrawIconEx with hotspot calculation
6. BGRA→RGB conversion, PNG encoding (zlib compression level 6)

**Output:** Base64-encoded PNG for API transmission  
**DPI Handling:** SetProcessDpiAwarenessContext(PER_MONITOR_AWARE_V2)

### 2. Input Simulation
**Technology:** Windows SendInput API  
**Input Types:**
- Mouse: Absolute positioning (SetCursorPos), button events, wheel scroll (delta ±120)
- Keyboard: Virtual key codes + Unicode injection (KEYEVENTF_UNICODE)

**Key Mapping:** 
- Alphanumeric: 0-9, a-z
- Function keys: F1-F12
- Modifiers: ctrl, alt, shift, windows
- Navigation: arrows, home, end, pageup/down
- Special: enter, tab, escape, backspace, delete, space

**Timing:**
- `ui_settle_delay`: 0.3s (post-action UI stabilization)
- `turn_delay`: 1.5s (inter-turn cooldown)
- `char_input_delay`: 0.01s (character typing interval)

**Drag Implementation:** 15-step linear interpolation with settle delays

### 3. API Communication Layer
**Protocol:** HTTP POST to LM Studio /v1/chat/completions  
**Timeout:** 240s  
**Payload Structure:**
```json
{
  "model": "qwen3-vl-2b-instruct",
  "messages": [
    {"role": "system", "content": "<agent_prompt>"},
    {"role": "user", "content": [
      {"type": "text", "text": "<context>"},
      {"type": "image_url", "image_url": {"url": "data:image/png;base64,..."}}
    ]}
  ],
  "tools": [...],
  "tool_choice": "auto",
  "temperature": <float>,
  "max_tokens": <int>
}
```

**Response Parsing:**
- Extract `choices[0].message.content` (reasoning/sitrep)
- Extract `choices[0].message.tool_calls` (structured actions)
- Tool arguments: JSON string or dict, parsed to dataclass instances

### 4. Logging & Observability
**ExecutionLogger:**
- `execution_log.txt`: Timestamped event log with API request/response redaction
- `screenshots/`: PNG archive per turn (turn_NNNN.png)
- **Redaction:** Base64 image data replaced with length metadata in logs

**Logged Events:**
- Section markers (turns, mode switches)
- API calls (request payload + response, redacted)
- Tool executions (tool name, args, result)
- State updates (turn, memory size)
- Screenshot saves

### 5. Loop Recovery Mechanism
**Trigger Conditions:**
- `enable_loop_recovery`: True
- `(current_turn - last_recovery_turn) > loop_recovery_cooldown` (default: 3)

**Recovery Actions:**
1. Press ESC key
2. Move mouse to center screen
3. Move mouse to bottom-left corner (20, screen_height-20)
4. Click left button
5. Press CTRL+ESC (Start menu)

**Purpose:** Break UI modal traps, dialog deadlocks, stuck states

## DATA FLOW

### Turn Execution (EXECUTION Mode)
```
1. Capture screenshot (PNG) → state.screenshot
2. Assemble context (memory + plan + operator_brief)
3. POST to LLM with screenshot
4. Parse response → (tool_call, sitrep)
5. Execute tool → result
6. Log action + screenshot
7. Add ActionRecord to memory
8. Check mode switch conditions
9. Sleep(turn_delay)
```

### Turn Execution (EVALUATION Mode)
```
1. Assemble context (memory + plan)
2. POST to LLM (no screenshot)
3. Parse response → (operator_brief?, compression?)
4. Apply compression if present
5. Update operator_brief if present
6. Switch to EXECUTION if briefed
7. Sleep(turn_delay)
```

## PROMPT ENGINEERING

### Team Leader Prompt Structure
```
Identity: SOF desktop automation unit leader
Mission: <user_task>
Plan Section: <current_plan (500 chars)>
Role: Coordinate with Operator specialist
Mode Switching Rules: EVALUATION vs EXECUTION
Tools: spawn_operator, compress_memory
Team Dynamics: Trust specialist, adapt on blockers, compress on growth, review every N turns
Critical Instructions: When to brief, when to compress
```

### Operator Specialist Prompt Structure
```
Identity: SOF Operator specialist
Current Brief: <operator_instructions>
Execution Discipline: ONE tool per turn, milestone reporting, blocker escalation after 2 attempts
SITREP Format: OBJ, DOD, OBS, ACT (structured reporting)
Coordinate System: Normalized 0-1000
Field Intel: Early blocker reporting, trust UI judgment, escalate ambiguity
```

**Key Design for 4b Model:**
- Structured role definitions with military metaphor (reduces ambiguity)
- Explicit tool usage conditions
- Mandatory SITREP format (forces structured output)
- Numeric thresholds stated clearly
- Single-action discipline (prevents multi-tool hallucination)

## COORDINATE SYSTEM

### Normalization
**Input Space:** [0, 1000] × [0, 1000]  
**Physical Mapping:**
```python
x_pixel = round((x_norm / 1000.0) * screen_width)
y_pixel = round((y_norm / 1000.0) * screen_height)
```
**Clamping:** Coordinates clamped to [0, 1000] before conversion, pixels clamped to screen bounds

**Rationale:** Resolution-independent coordinate system for model, simplifies training/prompting

## CONFIGURATION PARAMETERS

```python
lmstudio_endpoint: "http://localhost:1234/v1/chat/completions"
lmstudio_model: "qwen3-vl-2b-instruct"  # Note: Code default differs from stated 4b
lmstudio_timeout: 240
lmstudio_temperature: 0.5  # Operator
lmstudio_max_tokens: 1024  # Operator

screen_capture_w: 1536
screen_capture_h: 864

max_steps: 30
team_review_interval: 5
memory_compression_threshold: 8

ui_settle_delay: 0.3
turn_delay: 1.5
char_input_delay: 0.01

enable_loop_recovery: True
loop_recovery_cooldown: 3
```

## TERMINATION CONDITIONS

### Success
`report_completion` tool called with evidence >= 100 characters

### Failure
- Max steps reached (30 turns)
- No operator brief available in EXECUTION mode
- Critical exception (screen capture, API timeout, input simulation)

### Implicit
- User CTRL+C interrupt
- LM Studio connection failure

## ERROR HANDLING

**Tool Parsing Errors:** Logged, return (None, sitrep), skip tool execution  
**Missing Tool Arguments:** Return error result string to memory  
**Unknown Keys:** Validate against VK_MAP, return error if not found  
**API Failures:** Unhandled - will raise exception and terminate  
**Screen Capture Failures:** Raise RuntimeError, terminate execution

## MEMORY COMPLEXITY

**Active History Growth:** O(n) per turn until compression  
**Compression Frequency:** Every ~8 actions (configurable)  
**Context Size:** O(k) where k = snapshots + last 8 actions  
**Worst Case:** 30 turns × no compression = 30 ActionRecords in context (unlikely due to threshold)

## AGENT COORDINATION PATTERNS

### Blocker Escalation
```
Operator: report_progress(status="BLOCKED", evidence="...")
→ state.needs_review = True
→ Next turn: Mode switch to EVALUATION
→ Commander: Analyze blocker, spawn_operator with new strategy
```

### Objective Completion
```
Operator: report_progress(status="DONE", evidence="...")
→ state.needs_review = True
→ Next turn: Mode switch to EVALUATION
→ Commander: Verify completion, spawn_operator with next objective
```

### Loop Detection
```
Commander in EVALUATION: Analyze compressed_snapshots.loop_pattern
→ compress_memory with loop_analysis
→ Archived turns removed from context
→ spawn_operator with loop-breaking strategy
```

## DEPENDENCIES

**Standard Library Only:**
- ctypes (Windows API bindings)
- json, base64, zlib, struct
- urllib.request (HTTP client)
- time, os, sys
- dataclasses, typing, functools

**External:**
- LM Studio (external process, OpenAI-compatible API)
- Windows OS (GDI32, User32 DLLs)

## CRITICAL DESIGN CONSTRAINTS

### Model-Specific (Qwen3-VL-4b)
1. **Stateless API:** No conversation memory retention by model
2. **Single Instruction Weight:** Repeated instructions may be over-prioritized
3. **Small Model Capacity:** 4b parameters → simplified prompts, structured output, strict discipline
4. **Vision Capability:** Required for Operator, unused by Commander

### System-Specific
1. **Windows 11 Pro 25H2:** GDI APIs, DPI awareness, SendInput
2. **1080p @ 125% Scaling:** DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2 required
3. **Single Display:** No multi-monitor support

## EXECUTION FLOW DIAGRAM

```
main()
  ├─ init_dpi()
  ├─ Input mission
  ├─ Initial screenshot
  ├─ AgentState(mode="EVALUATION")
  └─ run_agent()
      └─ for turn in range(max_steps):
          ├─ state.increment_turn()
          │
          ├─ if mode == "EVALUATION":
          │   ├─ invoke_team_leader_evaluation()
          │   │   ├─ POST /chat/completions (text-only)
          │   │   ├─ Parse spawn_operator → operator_brief
          │   │   ├─ Parse compress_memory → apply
          │   │   └─ Switch to EXECUTION if briefed
          │   └─ continue
          │
          ├─ capture_png() → screenshot
          ├─ Check review conditions → switch to EVALUATION?
          │
          ├─ invoke_operator()
          │   ├─ POST /chat/completions (with image)
          │   ├─ Parse tool_call + sitrep
          │   └─ return
          │
          ├─ if report_completion → TERMINATE
          │
          ├─ execute_operator_tool()
          │   ├─ Coordinate conversion
          │   ├─ Windows API calls
          │   └─ return result
          │
          ├─ memory.add_action()
          ├─ Check progress status → set needs_review
          └─ sleep(turn_delay)
```

## SECURITY & SAFETY CONSIDERATIONS

**Unrestricted System Access:**
- Full keyboard/mouse control
- No sandboxing of actions
- No confirmation prompts
- Administrator privileges not required but may be needed for some targets

**Attack Surface:**
- LM Studio endpoint (localhost assumed, no authentication)
- Unbounded tool execution based on model output
- No input validation on mission string

**Failure Modes:**
- Infinite loops (mitigated by max_steps, loop_recovery)
- Destructive actions (file deletion, data entry in critical apps)
- Unintended navigation (mitigated by evidence requirements)

## FILE STRUCTURE

```
<working_directory>/
├─ <script>.py              # Single-file system
└─ dumps/                   # Created at runtime
   ├─ execution_log.txt     # Event log
   └─ screenshots/          # turn_NNNN.png files
```

## TOOL ARGUMENT DATACLASSES

```
ClickArgs: justification, label, position(Coordinate)
DragArgs: justification, label, start(Coordinate), end(Coordinate)
TypeTextArgs: justification, text
PressKeyArgs: justification, key
ScrollArgs: justification
ReportCompletionArgs: evidence
ReportProgressArgs: objective_id, status, evidence
CompressMemoryArgs: compression_summary, loop_analysis, archived_turns
SpawnOperatorArgs: operator_instructions, rationale
```

All immutable (frozen=True for Coordinate, mutable for others).

## PERFORMANCE CHARACTERISTICS

**Turn Duration:** ~2-5 seconds (API latency + turn_delay + action execution)  
**Screenshot Encoding:** <100ms for 1536×864 PNG  
**Action Execution:** <500ms (UI settle delays dominant)  
**API Latency:** 5-30s typical for 4b model on consumer hardware  
**Total Mission Duration:** 30 turns × ~10s avg = ~5 minutes max

## OBSERVABILITY OUTPUTS

**Execution Log Sections:**
- EXECUTION LOG START
- INITIALIZATION (config dump)
- TURN N - EVALUATION (text-only reasoning)
- TURN N - EXECUTION (vision-based action)
- API REQUEST/RESPONSE (redacted)
- TOOL execution (args + result)
- STATE UPDATE (turn, memory size)
- MISSION COMPLETE / DEBRIEF

**Screenshot Archive:** Sequential PNG files with turn number, enables post-hoc analysis

---

**SYSTEM CLASSIFICATION:** Research prototype - autonomous desktop automation agent with hierarchical planning and adaptive memory management for Windows environments.