# **DOVE EXPRESSIONS SYSTEM (SYSTEM.md)**

# **1. PURPOSE OF THIS SYSTEM**

Dove Expressions is a prophetic discipleship and creation ecosystem designed to help users:

- Draw near to God
- Hear God clearly
- Interpret what they receive
- Execute what God reveals

This system is not a generic app.

It is a spiritual workflow translated into software tools.

All development must support this core mission:

# Revelation → Understanding → Application → Execution

# **2. CORE SYSTEM MODULES**

The platform consists of four primary modules:

**2.1 HEARING GOD JOURNAL (App)**

- Ongoing journaling system
- Stores entries (dreams, prophetic words, notes)
- User-facing “home base”

**2.2 DREAM JOURNAL GENERATOR**

- Converts raw dream input into structured journal pages
- Handles:
    - narrative
    - brainstorm
    - interpretation
    - application
- 

**2.3 PROPHETIC WORD GENERATOR**

- Converts prophetic words into structured printable layouts
- Handles:
    - word content
    - interpretation/reflection
    - application
- 

**2.4 FUTURE: DOVE OS**

- Unified system connecting all tools
- Shared storage, templates, and outputs

# **3. GLOBAL ARCHITECTURE RULES**

These rules apply to ALL modules.

**3.1 SEPARATION OF CONCERNS (MANDATORY)**

Each generator MUST follow this pipeline:

INPUT → PARSE → NORMALIZE → PAGE PLAN → RENDER

**Definitions:**

- INPUT
Raw user text, OCR, or pasted content
- PARSE
Extract fields (title, date, sections, etc.)
- NORMALIZE
Convert into a consistent internal data structure
- PAGE PLAN
Determine how content is split across pages
- RENDER
Display pages (preview + print)

🚨 Never combine these steps in one function.

**3.2 PREVIEW ≠ PRINT**

Preview and Print must be treated as separate systems.

**Preview:**

- May use scaling (zoom, transform)
- Optimized for screen

**Print:**

- Must use REAL dimensions (8.5x11, A4, etc.)
- No scaling hacks
- True layout only

🚨 Print layout must NEVER depend on preview zoom.

**3.3 THEMES (STRICT RULE)**

Themes apply ONLY to output pages.

Themes must NOT affect:

- generator UI
- buttons
- inputs
- layout structure

Themes may affect:

- page backgrounds
- borders
- typography styles
- decorative elements

**3.4 PAGE SYSTEM**

All generators must use a page descriptor model.

Each page is defined BEFORE rendering.

Example:

{

type: "content",

section: "dream_narrative",

content: "...",

isContinuation: false

}

Rendering must only display what is already planned.

🚨 Rendering must NOT decide pagination.

**3.5 CONTINUATION RULES**

When content overflows:

- Create a new page
- Continue the SAME section
- Do NOT duplicate previous content

Example:

- Page 1: Dream Narrative (start)
- Page 2: Dream Narrative (continuation)

🚨 Never restart or duplicate content on continuation pages.

**3.6 NO DUPLICATION POLICY**

The system must NEVER:

- repeat sections unintentionally
- duplicate narrative pages
- confuse sections (dream vs brainstorm vs interpretation)

Each section must remain clearly separated.

**3.7 MOBILE BEHAVIOR**

Mobile is NOT a scaled desktop.

Mobile rules:

- simplify layout
- reduce visual clutter
- maintain full functionality

BUT:

- page output logic must remain identical across devices

# **4. GENERATOR-SPECIFIC RULES**

# **4.1 DREAM GENERATOR RULES**

**Structure:**

1. Dream Narrative
2. Brainstorm
3. Interpretation
4. Application

**Critical Rules:**

**A. Brainstorm vs Interpretation**

- Brainstorm = thinking process, symbol exploration
- Interpretation = cohesive message

🚨 Interpretation must NOT:

- list symbols
- define symbols directly
- feel analytical

It must read like:

- a flowing revelation
- written in second person

**B. Narrative Handling**

- Narrative must fully fill pages before continuing
- Must not cut early
- Must not duplicate

**C. Section Boundaries**

Each section must:

- start clean
- not bleed into others
- maintain identity

# **4.2 PROPHETIC GENERATOR RULES**

**Structure:**

1. Prophetic Word
2. Reflection / Interpretation
3. Application

**Rules:**

- Word must remain intact (no modification)
- Reflection builds understanding
- Application drives action

# **5. STORAGE RULES**

- Entries saved in generator must be compatible with Hearing God Journal
- Data structure must be consistent across modules
- No “generator-only” formats

# **6. UI RULES**

**Generator UI (Stable Shell)**

- Must remain consistent across tools
- Must not be theme-dependent

**Output Pages**

- Fully themeable
- Printable
- Structured

# **7. DEVELOPMENT RULES (FOR AI + DEV)**

When modifying code:

**ALWAYS:**

- preserve existing functionality
- make minimal, targeted changes
- follow system architecture

**NEVER:**

- redesign entire app without instruction
- merge unrelated logic
- break separation of concerns

# **8. PRIORITY ORDER FOR FIXES**

When bugs occur, prioritize:

1. Data integrity (no duplication, no corruption)
2. Pagination accuracy
3. Section correctness
4. Print accuracy
5. UI behavior

# **9. FUTURE EXPANSION**

System must support:

- additional generators
- shared templates
- unified storage
- discipleship workflows

# **10. CORE PRINCIPLE**

Every feature must answer:

# Does this help the user move from revelation to execution?

If not, it does not belong.

# **🔥 What you do next**

Drop this into your repo as SYSTEM.md.

Then when you prompt Copilot or Codex, say:

# Follow SYSTEM.md. Do not break architecture.