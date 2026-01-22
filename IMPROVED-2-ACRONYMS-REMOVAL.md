# LINGUISTIC ANALYSIS & OPTIMIZATION FOR 2B MODEL

## YOUR CONCERNS ARE VALID - CRITICAL ISSUES IDENTIFIED

### 1. THREE-LETTER ACRONYMS (DOD, OBS, ACT, OBJ)

**Problem:**
- 2B models have limited vocabulary association
- Acronyms create ambiguity: DOD could be "Department of Defense", "Definition of Done", "Dead on Delivery"
- Model must decode acronym → resolve meaning → generate response (extra cognitive load)
- Acronyms are not semantically weighted - model treats "OBJ:" same as "OBJECTIVE:" but latter has more contextual tokens

**Evidence from Current Prompts:**
```
OBJ: <current objective or 'AWAITING_BRIEF'>
DOD: <definition-of-done>
OBS: <UI state observed>
ACT: <tool chosen and why>
```

**Why This Fails:**
- "DOD" appears nowhere in training data with this specific meaning
- "OBS" could be "observation", "obsolete", "obscure" - model must guess from context
- "ACT" could be "action", "activate", "actual", "act as in theater"
- Model wastes tokens resolving ambiguity instead of generating useful output

**Proposed Fix:**
```
CURRENT_GOAL: <what you are trying to achieve now>
SUCCESS_CRITERIA: <how you know it worked>
SCREEN_SHOWS: <what you see in the screenshot>
NEXT_STEP: <tool you will use and why>
```

**Why This Works:**
- Each label is semantically unambiguous
- "SCREEN_SHOWS" directly maps to vision-language task
- "SUCCESS_CRITERIA" appears in instruction-tuning datasets
- No mental decoding required - model sees "CURRENT_GOAL" and immediately knows to output goal statement

---

### 2. SEPARATE TOOL SCHEMAS (COMMANDER vs OPERATOR)

**Problem:**
- Current architecture: `COMMANDER_TOOLS` (2 tools) vs `OPERATOR_TOOLS` (10 tools)
- Model receives different tool sets depending on mode
- 2B model has limited ability to track "which tools are available in which mode"
- Tool availability switching = cognitive load + potential hallucination of unavailable tools

**Evidence from Current Code:**
```python
COMMANDER_TOOLS = [spawn_operator, compress_memory]
OPERATOR_TOOLS = [report_completion, report_progress, click_element, ...]
```

**Why This Fails:**
- Model must remember: "When I am Commander, I can only spawn_operator or compress_memory"
- When switching modes, model must "forget" previous tool set and "load" new one
- 2B models are TERRIBLE at conditional tool availability
- Result: Commander tries to call `click_element` (not available) → parse error → wasted turn

**Proposed Fix: UNIFIED TOOL SCHEMA**

**Why Unified Works:**
- All tools visible to model always
- Prompt explicitly states: "As Commander, use ONLY spawn_operator and compress_memory"
- Model makes decision based on PROMPT (which it's good at) not TOOL AVAILABILITY (which it's bad at tracking)
- Reduces parser complexity - one schema, always present
- If Commander accidentally calls `click_element`, we can catch it in code: "Error: Commander cannot execute UI tools" instead of schema validation failure

**Better Alternative: IMPLICIT TOOL ROUTING**
- Single unified role: "Computer Control Specialist"
- Mode encoded in prompt context, not tool availability
- Tools have semantic names that self-document who should use them:
  - `update_team_strategy` (clearly strategic, not tactical)
  - `compress_memory` (clearly strategic)
  - `click_screen_element` (clearly tactical)
  - `report_objective_progress` (clearly tactical)
- Model naturally chooses correct tool based on current task in prompt

---

### 3. PROMPT SEMANTIC WEIGHT DISTRIBUTION

**Problem: Uneven Instruction Weight**

Current Operator prompt:
```
YOUR ROLE: Execute Team Leader's objective using UI tools. ONE action per turn. Validate previous action, then execute next.
```

**Issues:**
- "ONE action per turn" appears once
- "Validate previous action" appears once
- Model assigns equal weight to all instructions
- BUT: validation is CRITICAL, one action per turn is CRITICAL
- If model ignores "ONE action per turn", system breaks

**Linguistic Analysis:**

**Current Weight Distribution:**
```
Execute objective: ████ (appears 1x, generic verb)
ONE action: ██ (appears 1x, modifier)
Validate: ██ (appears 1x, buried in task list)
```

**Proposed Weight Distribution:**
```
PRIMARY_TASK: Validate then execute ONE action
CRITICAL_CONSTRAINT: Exactly one tool call per turn
VALIDATION_REQUIRED: Check previous result before proceeding
```

**Why This Works:**
- Repetition of key concepts: "one", "validate", "single"
- Semantic reinforcement: "PRIMARY", "CRITICAL", "REQUIRED" are high-weight tokens
- Position: Critical instructions at START of prompt (models have recency bias to beginning of context)

---

### 4. COMMANDER/OPERATOR ROLE CONFUSION

**Problem:**
- "Team Leader" vs "Operator Specialist"
- Military metaphor adds cognitive load
- Model must maintain two distinct personalities
- 2B model struggles with role consistency across turns

**Evidence:**
```
TEAM_LEADER = "You are Team Leader of a SOF desktop automation unit."
OPERATOR_SPECIALIST = "You are a SOF Operator specialist in desktop automation."
```

**Issues:**
- "SOF" (Special Operations Forces) - military jargon, not in training data
- "Team Leader" vs "Operator" - model must remember which it is
- Role switching creates identity confusion

**Proposed Fix: FUNCTIONAL ROLES**

**Option A: Explicit Function Names**
```
STRATEGIC_PLANNER = "You plan and coordinate computer automation tasks."
TACTICAL_EXECUTOR = "You execute individual computer control actions."
```

**Option B: Unified Role with Context Switching**
```
COMPUTER_CONTROLLER = "You control a computer to complete tasks.

CURRENT_MODE: {mode}

If PLANNING mode:
- Break tasks into steps
- Decide next objective
- Use: update_plan, compress_history

If EXECUTION mode:
- Look at screen
- Validate previous action
- Execute one action
- Use: click, type, press_key, report_status
```

**Why Option B is Superior:**
- Single identity maintained across turns
- Mode is explicit context, not role confusion
- Model sees "CURRENT_MODE: EXECUTION" and knows exactly what to do
- No role-switching mental overhead

---

### 5. TOOL NAME SEMANTIC CLARITY

**Current Tool Names:**
```
spawn_operator ← WAT? "spawn" is process management term, not task planning
compress_memory ← Ambiguous: compress how? why? when?
click_element ← "element" is web dev term, not universal
report_progress ← "progress" is vague
report_completion ← Why two report tools?
```

**Proposed Semantic Renaming:**

**Strategic Tools:**
```
update_task_plan → Clear: updating the plan for the task
archive_old_actions → Clear: removing old history
```

**Tactical Tools:**
```
click_screen_position → Clear: clicking screen at coordinates
type_text_input → Clear: typing text
press_keyboard_key → Clear: pressing key
move_mouse_cursor → Explicit action
scroll_screen_down/up → Clear direction

report_objective_complete → Clear: this objective is done
report_objective_blocked → Clear: this objective is stuck
report_objective_status → Clear: update on current objective
```

**Why This Matters:**
- Tool names self-document purpose
- No ambiguity: "click_screen_position" can only mean one thing
- Model sees tool list and immediately understands available actions
- Reduces justification verbosity - tool name already explains intent

---

### 6. JUSTIFICATION FIELD REDUNDANCY

**Current Schema:**
```json
{
  "justification": "string (30-60 words)",
  "label": "string",
  "position": [x, y]
}
```

**Problem:**
- "justification" forces model to generate 30-60 words explaining obvious action
- 2B model burns tokens on justification instead of validation
- Justification is for HUMANS debugging, not for system operation
- Model might hallucinate justification to meet word count

**Evidence:**
```
Justification: "The Start button is located at the bottom-left corner of the screen at normalized coordinates [20, 980]. Clicking this button will open the Start menu which provides access to the application search functionality required to locate and launch MS Paint as per the current objective."
```
**Token waste:** 47 tokens for obvious action

**Proposed Fix: OPTIONAL JUSTIFICATION**

**Option A: Remove entirely**
- System prompt instructs model to explain reasoning in SITREP, not in tool args
- Tool args are pure data: `{"label": "Start Button", "position": [20, 980]}`
- Explanation goes in SITREP: "Clicking Start to access app search"

**Option B: Make optional**
- Schema allows justification but doesn't require it
- Model includes justification only when action is non-obvious
- Reduces token waste on obvious actions

---

### 7. VALIDATION PROMPT STRUCTURE

**Current Validation Section:**
```
VALIDATION PROTOCOL (CRITICAL):
Before executing new tool:
1. Look at current screenshot
2. Check if previous action succeeded:
   - Did expected UI element appear?
   - Did expected UI element disappear?
   - Is UI in expected state?
3. If FAILED: Include in SITREP: "VALIDATION: Previous action FAILED - [reason]"
4. If SUCCESS: Include in SITREP: "VALIDATION: Previous action SUCCESS - [evidence]"
```

**Problems:**
- Nested list structure (1, 2 with sub-bullets, 3, 4)
- 2B models struggle with nested instructions
- "If FAILED" / "If SUCCESS" requires conditional logic execution
- Ambiguous: "expected UI element" - expected by whom? model must infer from previous action context

**Proposed Restructure:**

```
STEP 1: VALIDATE PREVIOUS ACTION
Look at the current screenshot.
Compare to what the previous action was supposed to do.
Answer: Did the previous action work?

If the previous action worked:
Write: "LAST_ACTION_RESULT: Success - [what you see that proves it worked]"

If the previous action did not work:
Write: "LAST_ACTION_RESULT: Failed - [what you see that proves it failed]"
Then adjust your strategy.

STEP 2: CHOOSE NEXT ACTION
Based on what you see now, pick ONE tool to use next.
```

**Why This Works:**
- Linear structure (no nesting)
- Explicit answer format: model knows exactly what string to generate
- "what you see" grounds validation in visual evidence
- "adjust your strategy" is actionable instruction
- STEP 1 / STEP 2 creates clear cognitive phases

---

### 8. COORDINATE SYSTEM EXPLANATION

**Current:**
```
COORDINATES:
- [x,y] in 0-1000 normalized scale
- (0,0) = top-left, (1000,1000) = bottom-right
```

**Problems:**
- "normalized scale" is mathematical term, not intuitive
- Two formats shown: [x,y] and (0,0) - inconsistent notation
- No visual anchor

**Proposed:**
```
SCREEN_COORDINATES:
The screen is divided into a 1000x1000 grid.
Top-left corner: [0, 0]
Top-right corner: [1000, 0]
Bottom-left corner: [0, 1000]
Bottom-right corner: [1000, 1000]
Center of screen: [500, 500]

When you see an element at the left side of the screen, X is small (0-300).
When you see an element at the right side, X is large (700-1000).
When you see an element at the top, Y is small (0-300).
When you see an element at the bottom, Y is large (700-1000).
```

**Why This Works:**
- Concrete examples: model can pattern-match
- Spatial language: "left side" → "X is small"
- Anchoring: specific corner coordinates given
- Relational explanation: links visual position to numerical range

---

## SUMMARY OF REQUIRED CHANGES

### CRITICAL (System-Breaking if Not Fixed):
1. **Remove all acronyms** - Replace with full semantic labels
2. **Unify tool schemas** - Single tool list, mode-based filtering in prompt
3. **Restructure validation** - Linear steps, explicit output format

### HIGH PRIORITY (Significant Quality Impact):
4. **Rename tools** - Semantic, self-documenting names
5. **Simplify roles** - Single identity with mode context, drop military metaphor
6. **Weight critical instructions** - Repetition + position + semantic markers

### MEDIUM PRIORITY (Reduced Token Waste):
7. **Optional justification** - Move explanation to SITREP
8. **Clarify coordinates** - Concrete examples, spatial language

---

## RECOMMENDED PROMPT ARCHITECTURE

### Unified System Prompt Structure:
```
IDENTITY: Computer Control Agent

CURRENT_MODE: {PLANNING | EXECUTING}

TASK: {user_task}

[Mode-specific instructions here]

CRITICAL_RULES:
1. [Rule with highest priority]
2. [Next rule]
...

AVAILABLE_TOOLS:
[Unified tool list with semantic names]

OUTPUT_FORMAT:
[Exact template model must follow]
```

**Why This Works:**
- Clear sections (headings act as attention markers)
- Mode is explicit state variable
- Rules numbered by priority
- Output format is template (reduces generation ambiguity)

---

**QUESTION FOR YOU:**
Do you want me to implement these changes with:
1. **Conservative approach**: Fix critical issues only (acronyms, unified tools, validation structure)
2. **Comprehensive overhaul**: All 8 changes including role simplification and tool renaming
3. **Hybrid**: Critical + high priority, defer medium priority for testing

Which approach aligns with your goals?