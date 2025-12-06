---
name: skill-creator-enhanced
description: |
  Enhanced guide for creating effective skills with automatic packaging workflow.
  This skill should be used when users want to create a new skill (or update an
  existing skill) that extends Claude's capabilities with specialized knowledge,
  workflows, or tool integrations. Automatically offers to package and provide
  download links for immediate installation.
license: Complete terms in LICENSE.txt
---

# Skill Creator (Enhanced)

This skill provides guidance for creating effective skills with streamlined save/distribution workflow.

## About Skills

Skills are modular, self-contained packages that extend Claude's capabilities by providing
specialized knowledge, workflows, and tools. Think of them as "onboarding guides" for specific
domains or tasks—they transform Claude from a general-purpose agent into a specialized agent
equipped with procedural knowledge that no model can fully possess.

### What Skills Provide

1. Specialized workflows - Multi-step procedures for specific domains
2. Tool integrations - Instructions for working with specific file formats or APIs
3. Domain expertise - Company-specific knowledge, schemas, business logic
4. Bundled resources - Scripts, references, and assets for complex and repetitive tasks

### Anatomy of a Skill

Every skill consists of a required SKILL.md file and optional bundled resources:

```
skill-name/
├── SKILL.md (required)
│   ├── YAML frontmatter metadata (required)
│   │   ├── name: (required)
│   │   └── description: (required)
│   └── Markdown instructions (required)
└── Bundled Resources (optional)
    ├── scripts/          - Executable code (Python/Bash/etc.)
    ├── references/       - Documentation intended to be loaded into context as needed
    └── assets/           - Files used in output (templates, icons, fonts, etc.)
```

#### SKILL.md (required)

**Metadata Quality:** The `name` and `description` in YAML frontmatter determine when Claude will use the skill. Be specific about what the skill does and when to use it. Use the third-person (e.g. "This skill should be used when..." instead of "Use this skill when...").

#### Bundled Resources (optional)

##### Scripts (`scripts/`)

Executable code (Python/Bash/etc.) for tasks that require deterministic reliability or are repeatedly rewritten.

- **When to include**: When the same code is being rewritten repeatedly or deterministic reliability is needed
- **Example**: `scripts/rotate_pdf.py` for PDF rotation tasks
- **Benefits**: Token efficient, deterministic, may be executed without loading into context

##### References (`references/`)

Documentation and reference material intended to be loaded as needed into context.

- **When to include**: For documentation that Claude should reference while working
- **Examples**: `references/finance.md` for financial schemas, `references/api_docs.md` for API specifications
- **Best practice**: If files are large (>10k words), include grep search patterns in SKILL.md

##### Assets (`assets/`)

Files not intended to be loaded into context, but rather used within the output Claude produces.

- **When to include**: When the skill needs files that will be used in the final output
- **Examples**: `assets/logo.png` for brand assets, `assets/slides.pptx` for PowerPoint templates

### Progressive Disclosure Design Principle

Skills use a three-level loading system to manage context efficiently:

1. **Metadata (name + description)** - Always in context (~100 words)
2. **SKILL.md body** - When skill triggers (<5k words)
3. **Bundled resources** - As needed by Claude (Unlimited*)

## Skill Creation Process

To create a skill, follow the "Skill Creation Process" in order, skipping steps only if there is a clear reason why they are not applicable.

### Step 1: Understanding the Skill with Concrete Examples

To create an effective skill, clearly understand concrete examples of how the skill will be used. Ask:

- "What functionality should the skill support?"
- "Can you give some examples of how this skill would be used?"
- "What would a user say that should trigger this skill?"

Conclude this step when there is a clear sense of the functionality the skill should support.

### Step 2: Planning the Reusable Skill Contents

Analyze each example by:

1. Considering how to execute on the example from scratch
2. Identifying what scripts, references, and assets would be helpful when executing these workflows repeatedly

### Step 3: Initializing the Skill

When creating a new skill from scratch, run the `init_skill.py` script:

```bash
scripts/init_skill.py <skill-name> --path <output-directory>
```

The script creates the skill directory with SKILL.md template and example resource directories.

### Step 4: Edit the Skill

**Writing Style:** Write using **imperative/infinitive form** (verb-first instructions), not second person.

To complete SKILL.md, answer:

1. What is the purpose of the skill?
2. When should the skill be used?
3. How should Claude use the skill?

### Step 5: Packaging a Skill

Once ready, package the skill:

```bash
scripts/package_skill.py <path/to/skill-folder>
```

The script validates and creates a distributable zip file.

### Step 6: Save Skill to Destinations

⚠️ **CRITICAL: Always offer these save options after creating a skill.**

After creating or packaging a skill, ALWAYS present this table to the user:

---

## Save Skill to:

| Destination | Path/Method | Action |
|-------------|-------------|--------|
| ☐ **Claude Desktop** | Native "Save Skill" button | User clicks button in UI |
| ☐ **Claude Code** | `~/.claude/skills/<skill-name>/` | Copy files to directory |
| ☐ **Organized Codebase** | `[iCloud]/Organized AI/Windsurf/skills/<skill-name>/` | Copy files to iCloud path |
| ☐ **GitHub** | `organized-ai` or `jhillbht` repo | Git add, commit, push |

**Which destinations would you like?** (e.g., "all", "Claude Code + GitHub", or specific ones)

---

#### Save Actions by Destination

##### Claude Desktop
- Inform user to use the "Save Skill" button in the Claude Desktop UI
- The skill content should be displayed in a format the button can capture

##### Claude Code
```bash
# Create skill directory
mkdir -p ~/.claude/skills/<skill-name>

# Copy all skill files
cp -r <skill-folder>/* ~/.claude/skills/<skill-name>/

# Verify
ls -la ~/.claude/skills/<skill-name>/
```

##### Organized Codebase
```bash
# Full path
OCB_SKILLS="/Users/supabowl/Library/Mobile Documents/com~apple~CloudDocs/BHT Promo iCloud/Organized AI/Windsurf/skills"

# Create and copy
mkdir -p "$OCB_SKILLS/<skill-name>"
cp -r <skill-folder>/* "$OCB_SKILLS/<skill-name>/"
```

##### GitHub
```bash
# Navigate to Organized Codebase
cd "/Users/supabowl/Library/Mobile Documents/com~apple~CloudDocs/BHT Promo iCloud/Organized AI/Windsurf"

# Add skill files
git add skills/<skill-name>/

# Commit with descriptive message
git commit -m "feat: Add <skill-name> skill

<brief description of what the skill does>"

# Push - ASK USER which repo:
# - Personal: https://github.com/jhillbht
# - Organization: https://github.com/organized-ai
git push origin main
```

### Step 7: Iterate

After testing the skill, users may request improvements. Often this happens right after using the skill, with fresh context of how the skill performed.

**Iteration workflow:**
1. Use the skill on real tasks
2. Notice struggles or inefficiencies
3. Identify how SKILL.md or bundled resources should be updated
4. Implement changes and test again
5. **Re-run Step 6** to save updated skill to all destinations
