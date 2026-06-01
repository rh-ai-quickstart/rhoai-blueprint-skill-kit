# RHOAI Conversion Subagents

This directory contains specialized subagent prompts that handle focused tasks during blueprint conversion. The main conversion skill ([../SKILL.md](../SKILL.md)) orchestrates these subagents to keep context lean while maintaining conversion quality.

## Architecture

**Main Agent** (SKILL.md):
- Orchestrates workflow phases
- Performs core reasoning (Phase 3-5: Dynamic reasoning, user decisions, pattern application)
- Loads knowledge base files (only top-ranked ones from scorer)
- Reviews subagent outputs
- Maintains conversion quality

**Subagents** (this directory):
- Execute focused, self-contained tasks
- Return structured data (JSON schemas enforced)
- Don't need full context - only specific inputs from main agent
- Keep main agent context low (~60-70% reduction)

## Subagent Prompts

### 1. blueprint-analyzer-prompt.md
**Purpose**: Analyze blueprint structure and extract feature vector

**Input**:
- Blueprint directory path

**Output**:
```json
{
  "components": ["redis", "triton", "milvus"],
  "architecture": "rag-pipeline",
  "deployment_types": ["helm", "docker-compose"],
  "resource_types_needed": ["gpu", "storage", "networking"]
}
```

**When used**: Phase 1 (Blueprint Analysis)

**Why subagent**: Pure file scanning/parsing, no decisions, self-contained

---

### 2. knowledge-scorer-prompt.md
**Purpose**: Score and rank knowledge base files by relevance

**Input**:
- Blueprint features (from analyzer)
- Knowledge base directory path

**Output**:
```json
{
  "ranked_files": [
    {
      "path": "components/redis-on-rhoai.md",
      "category": "component",
      "score": 18,
      "matched_on": ["redis", "helm", "storage"]
    }
  ]
}
```

**When used**: Phase 2.1 (Knowledge Retrieval)

**Why subagent**: Mechanical tag matching, no interpretation needed

---

### 3. documentation-generator-prompt.md
**Purpose**: Generate TEST-PLAN.md, RHOAI-CONVERSION.md, update README.md

**Input**:
- Blueprint directory
- Deployment method
- RHOAI mode toggle
- Components
- Patterns applied
- User decisions
- Modified/created files
- Knowledge sources

**Output**:
```json
{
  "files_created": ["TEST-PLAN.md", "RHOAI-CONVERSION.md"],
  "files_modified": ["README.md"],
  "test_plan_sections": [...],
  "conversion_doc_sections": [...]
}
```

**When used**: Phase 6 (Generate Documentation)

**Why subagent**: Template filling from structured data, reads [output-templates.md](../output-templates.md) internally

---

### 4. validation-prompt.md
**Purpose**: Validate generated conversion outputs

**Input**:
- Blueprint directory
- Deployment method
- RHOAI mode toggle
- Components list

**Output**: Validation report with errors/warnings (markdown format)

**When used**: Phase 6.5 (Automated Validation)

**Why subagent**: Self-contained validation checks, iterates up to 3 times

---

## Important Notes

### For Main Agent
**DO NOT read these subagent prompt files directly.** They're passed to subagents via the Agent tool's prompt parameter:

```python
# Good - main agent never loads subagent instructions
Agent(
    description="Analyze blueprint structure",
    prompt=f"""
Read and follow instructions from:
.claude/skills/bp-convert-to-rhoai/subagents/blueprint-analyzer-prompt.md

Blueprint directory: {blueprint_dir}
...
"""
)

# Bad - loads unnecessary context into main agent
instructions = Read("subagents/blueprint-analyzer-prompt.md")
Agent(prompt=f"{instructions}\n\n{data}")
```

### For Subagents
Each subagent prompt is **self-contained** with:
1. **Background** (2-3 paragraphs): Role, why it matters, what output is used for
2. **Instructions** (exact copy from original SKILL.md phases): Step-by-step task execution
3. **Output specification**: JSON schema or markdown format

### Quality Preservation
Subagent prompts contain **identical instructions** from the original SKILL.md - this is a **pure refactoring** for context management, not a behavior change.

## Token Impact

**Before** (monolithic SKILL.md):
- Main agent context: ~2000+ tokens (all phases loaded)

**After** (with subagents):
- Main agent context: ~600-800 tokens (orchestration only)
- Each subagent context: ~200-300 tokens (focused task)
- **Context reduction: ~60-70% for main agent**

The main agent only loads:
- SKILL.md (trimmed to orchestration)
- reasoning-guardrails.md (Phase 3)
- output-templates.md (Phase 7 summary)
- Top 5-10 scored knowledge files (Phase 2.2)

Everything else is delegated to subagents with clean interfaces.
