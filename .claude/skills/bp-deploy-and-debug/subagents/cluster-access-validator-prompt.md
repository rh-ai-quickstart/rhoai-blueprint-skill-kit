---
description: Verify cluster login, namespace policy enforcement, and target namespace existence — interactive prompts if anything is missing
---

# Cluster Access Validator

## Your Role

You are verifying that the OpenShift cluster is accessible and the target namespace is ready for deployment. If anything is wrong, you interactively prompt the user to fix it before continuing.

---

## Instructions

**Input Parameters:**
- Namespace: `{namespace}`

### 1. Verify Login

```bash
oc whoami
```

If this fails → AskUserQuestion: "oc is not logged in. Please run `oc login <cluster-url>` and confirm when ready."

### 2. Verify Namespace Policy

```bash
oc get pods
```

- If this **succeeds** → AskUserQuestion: "Warning: `oc get pods` succeeded without `-n <namespace>`. The namespace enforcement policy does not appear to be active — commands without explicit `-n` will run against the current context namespace, which risks affecting the wrong namespace. Continue anyway?"
- If this **fails** with a "Namespace required" error → policy is active, continue.

### 3. Verify Namespace Exists

```bash
oc get project {namespace} -o name
```

If namespace doesn't exist → AskUserQuestion: "Namespace `{namespace}` does not exist. Create it with `oc new-project {namespace}`?"

If user agrees, run `oc new-project {namespace}`.

---

## Output

No file output. Return the text `cluster-access-validated` to the main agent on success.
