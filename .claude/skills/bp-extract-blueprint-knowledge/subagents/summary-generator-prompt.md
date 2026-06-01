---
description: Generate high-density summary for a knowledge base file using Chain of Density technique
---

# Knowledge Base Summary Generation

You are an expert at creating high-density technical summaries using Anthropic's Chain of Density technique.

## Your Task

Generate a 4-sentence summary for the provided knowledge base file following this EXACT format:

**Sentence 1:** What this pattern solves (1 sentence, focus on the problem/use case)
**Sentence 2:** When to use it vs alternatives (1 sentence with decision criteria)
**Sentence 3:** Critical YAML/config changes (1 sentence with minimal code snippet if needed)
**Sentence 4:** Common gotchas/failure modes (1 sentence)

## Input

You will receive the file path to summarize via the prompt context.

## CRITICAL INSTRUCTIONS

### 1. Front-load decision criteria (the "WHY" before the "HOW")

- ❌ Bad: "Use fsGroup: 1000"
- ✅ Good: "fsGroup: 1000 required because restricted SCC blocks runtime chown, but fsGroup applies ownership at mount time"

### 2. If the file contains multiple approaches (Approach A, Approach B)

- Mention ALL approaches in the summary
- Include brief decision criteria for choosing between them
- Example: "Use Approach A (init container) when restricted SCC blocks runtime chown; use Approach B (fsGroup) when SCC allows it"

**IMPORTANT:** If this is an existing file with a summary that was just updated to add a new approach, you MUST regenerate the entire summary to cover all approaches. The old summary only covered the original approach - replace it completely.

### 3. Compress code examples to minimal diffs

- Show BEFORE/AFTER pattern when possible
- Use compact syntax: `{{- if .Values.openshiftMode }} securityContext: runAsNonRoot {{- end }}`

### 4. Apply Chain of Density iterations

- **Draft 1:** Extract all key points (be comprehensive)
- **Draft 2:** Remove redundancy, compress prose
- **Draft 3:** Verify decision criteria intact - test: "Can I choose correct approach from summary alone?"

## Instructions

### Step 1: Read the knowledge base file

Read the file provided in the context.

### Step 2: Generate the 4-sentence summary

Follow the Chain of Density technique to create a high-density summary that covers ALL approaches if multiple exist.

### Step 3: Update the frontmatter using Edit tool

Use the Edit tool to add or update the summary field in the YAML frontmatter:

1. If `summary:` field does not exist: Add it after the `description:` field
2. If `summary:` field already exists: Replace it with the new summary (regenerated to cover all approaches)
3. Escape double quotes in summary with backslash: `\"`
4. Preserve all other frontmatter fields and file content exactly

**Use Edit tool, NOT Write tool** - only modify the summary line, don't rewrite the entire file.

### Example frontmatter result:

```yaml
---
name: redis-on-rhoai
description: Redis deployment on RHOAI with OpenShift-compatible security contexts
summary: "Solves Redis deployment on OpenShift with restricted SCC compliance while maintaining standard Kubernetes compatibility via conditional toggles or standalone deployments. Use Approach A (conditional {{- if .Values.openshift.enabled }} in existing Helm chart) when original uses Helm and needs unified deployment; use Approach B (standalone openshift/ directory with hardcoded security contexts) when original uses docker-compose and separation from NVIDIA's method is preferred. Critical changes: pod-level runAsNonRoot: true + container allowPrivilegeEscalation: false + capabilities drop ALL; OpenShift init uses UBI minimal without chmod because restricted SCC blocks runtime permission changes, standard K8s uses busybox with chmod 755. Common gotchas: chmod commands fail in restricted-v2 SCC (rely on OpenShift's automatic namespace UID/GID assignment instead); emptyDir volumes lose data on pod restart which is acceptable for ephemeral Celery task queues but requires PersistentVolumeClaim for production persistence."
metadata:
  type: component
---
```

## Return JSON

```json
{
  "file": "components/redis-on-rhoai.md",
  "success": true
}
```

If generation or update fails:
```json
{
  "file": "components/redis-on-rhoai.md",
  "success": false,
  "error": "Description of what went wrong"
}
```
