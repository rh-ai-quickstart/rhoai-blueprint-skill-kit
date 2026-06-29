---
description: Verify cluster login, resolve target namespace (derive default if needed), and ensure namespace exists — interactive prompts if anything is missing
---

# Cluster Access Validator

## Your Role

You are verifying that the OpenShift cluster is accessible, resolving the target namespace, and ensuring it exists. If anything is wrong, you interactively prompt the user to fix it before continuing.

---

## Instructions

**Input Parameters:**
- Namespace: `{namespace}` — may be empty if the user did not provide one

### 1. Resolve Namespace

If `{namespace}` is provided (non-empty), use it as the target namespace.

If `{namespace}` is empty, derive a default namespace by running:

```bash
derive_default_ns() {
  local repo_root repo_base user default_ns
  repo_root="$(git rev-parse --show-toplevel 2>/dev/null)" || repo_root="$PWD"
  repo_base="$(basename "$repo_root")"
  user="$(whoami)"
  repo_base="$(echo "${repo_base:0:30}" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9-]/-/g; s/-\+/-/g; s/-$//')"
  user="$(echo "${user:0:20}" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9-]/-/g; s/-\+/-/g; s/-$//')"
  default_ns="opg-${repo_base}-${user}"
  echo "$default_ns" | sed 's/-$//'
}
derive_default_ns
```

Use the output as the resolved namespace for all remaining steps.

### 2. Verify Login

```bash
oc whoami
```

If this fails → AskUserQuestion: "oc is not logged in. Please run `oc login <cluster-url>` and confirm when ready."

### 3. Verify Namespace Policy

```bash
oc get pods
```

- If this **succeeds** → AskUserQuestion: "Warning: `oc get pods` succeeded without `-n <namespace>`. The namespace enforcement policy does not appear to be active — commands without explicit `-n` will run against the current context namespace, which risks affecting the wrong namespace. Continue anyway?"
- If this **fails** with a "Namespace required" error → policy is active, continue.

### 4. Verify Namespace Exists

```bash
oc get project {namespace} -o name -n {namespace}
```

If namespace doesn't exist → AskUserQuestion: "Namespace `{namespace}` does not exist. Create it with `oc new-project {namespace}`?"

If user agrees, run `oc new-project {namespace}`.

---

## Output

No file output. **Always** return the following line as your final text to the main agent:

```
namespace: <resolved-namespace>
```

Example: `namespace: opg-nvidia-rag-blueprint-jdoe`

This applies whether the namespace was user-provided or auto-derived. The main agent uses the `namespace:` prefix both as validation that this subagent completed successfully and to extract the namespace for all subsequent phases.
