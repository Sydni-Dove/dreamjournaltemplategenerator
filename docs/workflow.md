# **🛠️ DOVE EXPRESSIONS GENERATOR WORKFLOW (WORKFLOW.md)**

# **1. PURPOSE**

This document defines the exact execution flow for all generators.

It ensures:

- consistent parsing
- correct pagination
- no duplication
- stable rendering
- proper print output

🚨 This workflow must be followed strictly.

Do NOT skip steps or merge phases.

# **2. FULL PIPELINE**

Every generator MUST follow this sequence:

1. INPUT

2. PARSE

3. NORMALIZE

4. SECTION BUILD

5. PAGE PLANNING

6. RENDER (PREVIEW)

7. RENDER (PRINT)

# **3. STEP-BY-STEP BREAKDOWN**

# **3.1 INPUT**

**Sources:**

- pasted text
- uploaded image (OCR)
- manual fields

**Rules:**

- Do NOT assume structure
- Do NOT modify content yet
- Preserve raw input

# **3.2 PARSE**

**Goal:**

Extract meaningful parts from raw input.

**Example Output:**

{

title: "...",

date: "...",

rawSections: {

narrative: "...",

brainstorm: "...",

interpretation: "...",

application: "..."

}

}

**Rules:**

- Be flexible with messy input
- Detect headings if present
- If missing, infer structure cautiously

🚨 DO NOT format text here

🚨 DO NOT split into pages here

# **3.3 NORMALIZE**

**Goal:**

Convert parsed data into a strict internal format

**Example:**

{

sections: [

{ type: "narrative", content: "..." },

{ type: "brainstorm", content: "..." },

{ type: "interpretation", content: "..." },

{ type: "application", content: "..." }

]

}

**Rules:**

- Ensure ALL required sections exist
- Fill missing sections with empty strings
- Clean spacing only (no rewriting)

🚨 No pagination

🚨 No rendering logic

# **3.4 SECTION BUILD**

**Goal:**

Prepare sections for layout

**Tasks:**

- split content into paragraphs
- tag section types
- preserve order

**Example:**

{

type: "narrative",

blocks: ["paragraph 1", "paragraph 2"]

}

**Rules:**

- Do NOT measure height yet
- Do NOT split across pages yet

# **3.5 PAGE PLANNING (CRITICAL STEP)**

This is where most of your bugs are coming from.

**Goal:**

Create a complete page plan BEFORE rendering

# **3.5.1 Page Descriptor Model**

Each page must be defined like:

{

pageNumber: 1,

sections: [

{

type: "narrative",

content: "...",

isContinuation: false

}

]

}

# **3.5.2 Measurement Rules**

- Use a hidden measurement container
- Measure REAL content height
- Use actual page dimensions (no scaling)

AVAILABLE_HEIGHT = PAGE_HEIGHT - HEADER - FOOTER - MARGINS

# **3.5.3 Content Fitting Algorithm**

For each section:

1. Start new page
2. Add content progressively
3. Measure height after each addition
4. If overflow:
    - remove last addition
    - commit page
    - continue on next page
5. 

# **3.5.4 Continuation Handling**

When splitting:

{

type: "narrative",

content: "...continued...",

isContinuation: true

}

**Rules:**

- NEVER duplicate full content
- ONLY carry remaining content forward
- Maintain section identity

# **3.5.5 Section Integrity Rules**

- Narrative must stay narrative
- Brainstorm must not merge with interpretation
- Sections must not blend

🚨 NO CROSS-SECTION BLEED

# **3.5.6 Common Bugs to PREVENT**

**❌ Duplicate pages**

Cause: re-adding full content instead of remainder

**❌ Cut-off text**

Cause: not re-measuring after removal

**❌ Empty pages between pages**

Cause: incorrect page commit timing

**❌ Narrative restarting on next page**

Cause: re-rendering full section instead of remainder

# **3.6 RENDER (PREVIEW)**

**Goal:**

Display pages on screen

**Rules:**

- Use page descriptors ONLY
- DO NOT recalculate layout
- Allow scaling (zoom)

transform: scale(...)

# **3.7 RENDER (PRINT)**

**Goal:**

Generate accurate printable output

**Rules:**

- NO scaling
- TRUE dimensions only
- Use same page descriptors

# **4. THEME WORKFLOW**

Themes apply ONLY during rendering:

**Applied To:**

- page backgrounds
- borders
- typography

**NOT Applied To:**

- generator UI
- layout structure

# **5. STORAGE WORKFLOW**

When saving:

SAVE({

rawInput,

parsed,

normalized,

finalSections

})

**Rules:**

- Must be compatible with Hearing God Journal
- Avoid storing page descriptors (rebuildable)

# **6. MOBILE WORKFLOW**

**Rules:**

- Same data + page plan
- Different UI layout only

🚨 DO NOT change pagination for mobile

# **7. DEBUGGING WORKFLOW**

When something breaks:

**Step 1:**

Check NORMALIZED data

**Step 2:**

Check SECTION BUILD output

**Step 3:**

Inspect PAGE PLAN (most issues here)

**Step 4:**

Check RENDER (only if above are correct)

# **8. DEVELOPMENT COMMANDS (FOR AI)**

When modifying code, follow:

**Allowed:**

- “Refactor pagination logic only”
- “Fix continuation logic without touching parsing”
- “Separate preview from print rendering”

**Not Allowed:**

- rewriting entire file
- mixing phases
- changing data structures without reason

# **9. GOLDEN RULE**

# Rendering should be dumb.

# Planning should be smart.

If rendering is making decisions, the system is broken.