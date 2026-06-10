---
description: Generate TEST-PLAN.md, RHOAI-CONVERSION.md, and README updates from conversion data
---

# Conversion Documentation Generator

## Your Role

You are generating the final documentation for a completed RHOAI conversion. This documentation helps users deploy the converted blueprint and understand what changed.

Your output will be used to:
1. **Enable testing** - TEST-PLAN.md provides step-by-step deployment and validation
2. **Explain changes** - RHOAI-CONVERSION.md documents what was modified and why
3. **Update README** - Add RHOAI deployment section to existing README

## Documentation Purpose

The generated docs should:
- Be **actionable** - Users can follow TEST-PLAN.md to deploy successfully
- Be **transparent** - RHOAI-CONVERSION.md shows all modifications made
- Be **complete** - Cover prerequisites, deployment steps, validation, and troubleshooting

---

## Instructions

**IMPORTANT: Before starting, read the output templates:**
```bash
cat .claude/skills/bp-convert-to-rhoai/output-templates.md
```

This file contains the exact templates you must follow for:
- TEST-PLAN.md structure
- RHOAI-CONVERSION.md structure
- Conversion Summary Report format

**Input Parameters:**
- Blueprint directory: {blueprint_dir}
- Deployment method: {deployment_method}
- Components: {components}
- Patterns applied: {patterns_applied}
- User decisions: {user_decisions}
- Modified files: {modified_files}
- Created files: {created_files}
- Knowledge sources: {knowledge_sources}

### 6.1 Create TEST-PLAN.md

Use TEST-PLAN.md template from `output-templates.md`, customizing:
- Deployment steps (Helm vs oc apply)
- RHOAI mode toggle method (openshiftMode vs OPENSHIFT_MODE)
- Component-specific tests based on blueprint
- Prerequisites specific to this blueprint

**Structure:**
1. Prerequisites (cluster, tools, resources)
2. Deployment steps (how to enable RHOAI mode)
3. Validation steps (per component)
4. Troubleshooting (common issues)

**Write to:** `{blueprint_dir}/TEST-PLAN.md`

### 6.2 Create RHOAI-CONVERSION.md

Use RHOAI-CONVERSION.md template from `output-templates.md`, documenting:
- What changed and why
- Which knowledge sources were applied
- User decisions made during conversion
- How to toggle RHOAI mode for this deployment method
- Files modified and created

**Structure:**
1. Conversion Overview
2. Changes Made
3. Knowledge Sources Applied
4. User Decisions
5. RHOAI Mode Toggle Usage
6. Files Modified/Created

**Write to:** `{blueprint_dir}/RHOAI-CONVERSION.md`

### 6.3 Update README.md

Add RHOAI deployment section to existing README showing:
- How to enable RHOAI mode (openshiftMode=true or OPENSHIFT_MODE=true)
- Prerequisites for RHOAI deployment
- Link to TEST-PLAN.md for detailed steps

**Important:** 
- Edit existing README.md (use Edit tool)
- Add new section after existing deployment instructions
- Don't remove or modify existing content
- Use heading: `## Deployment on Red Hat OpenShift AI`

**Edit file:** `{blueprint_dir}/README.md`

---

## Output

Return a JSON summary of generated documentation:

```json
{
  "files_created": [
    "TEST-PLAN.md",
    "RHOAI-CONVERSION.md"
  ],
  "files_modified": [
    "README.md"
  ],
  "test_plan_sections": [
    "Prerequisites",
    "Deployment",
    "Validation",
    "Troubleshooting"
  ],
  "conversion_doc_sections": [
    "Overview",
    "Changes",
    "Knowledge Sources",
    "User Decisions",
    "RHOAI Toggle",
    "Files"
  ]
}
```

**Important:** 
- Actually create/modify the files (use Write/Edit tools)
- Follow templates from output-templates.md exactly
- Return the JSON summary after files are written
