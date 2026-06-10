---
description: Template for conversion specification YAML file structure
---

# Conversion Specification Template

This template defines the structure of `/tmp/conversion-spec.yaml` - a single YAML file containing conversion specifications for all components in a blueprint.

## File Structure

```yaml
# Conversion Specification for <Blueprint-Name>
blueprint_name: <name>
deployment_method: helm|oc-apply|notebook

# Component conversion plans
components:
  <component-name>:  # e.g., redis, triton-notebook, frontend-route, milvus
    type: helm-service|notebook|route|pvc|deployment|other
    
    # PRIMARY FOCUS: Conversion approach and KB sources
    conversion_approach:
      strategy: "Brief description of how OpenShift support is added"
      kb_pattern_references:
        - "path/to/kb-file.md#section-anchor"
        - "path/to/another-kb-file.md#section"  # Can reference multiple KB files
      kb_summary: "Why these patterns apply to this specific scenario"
      openshift_additions:
        - "List of OpenShift-specific changes being made"
        - "e.g., Add openshiftMode flag with conditional securityContext"
        - "e.g., Override init container to UBI-based image"
    
    # Implementation details
    implementation:
      files_to_modify:
        - path: relative/path/to/file.yaml
          changes:
            - "Line X: Add/modify specific content"
            - "Section Y: Change configuration Z"
          kb_reference: "path/to/kb-file.md#implementation-example"
        
      files_to_create:
        - path: relative/path/to/new-file.yaml
          content_description: "Brief description of what this file contains"
          kb_reference: "path/to/kb-file.md#template-section"
      
      configuration:
        helm_chart: chart-name:version  # if using Helm
        container_images:
          - registry.io/image:tag
          - registry.io/init-image:tag
        dependencies:
          - component: other-component-name
            reason: "Why this dependency exists"
    
    # SECONDARY FOCUS: Context for validators
    validation_context:
      blueprint_specifics: "Relevant details about this component in the blueprint (scale, architecture, etc.)"
      user_cluster_constraints: "Cluster capabilities that affect this component (storage, GPU, SCC, etc.)"
      deployment_decisions: "User choices that impact validation (model deployment strategy, etc.)"
```

## Field Descriptions

### Blueprint-Level Fields

- **blueprint_name**: Name of the NVIDIA Blueprint being converted
- **deployment_method**: Primary deployment approach (helm, oc-apply, or notebook)

### Component-Level Fields

#### conversion_approach (PRIMARY - what validators verify)
- **strategy**: High-level description of the conversion approach
- **kb_pattern_references**: Array of paths to KB files + section anchors (e.g., `components/redis-on-rhoai.md#approach-a-helm`). Can reference multiple KB files when relevant.
- **kb_summary**: Why these KB patterns apply to this blueprint's scenario
- **openshift_additions**: List of OpenShift-specific changes being introduced

#### implementation (HOW to apply the conversion)
- **files_to_modify**: Existing files that need changes
  - **path**: Relative path from blueprint root
  - **changes**: Specific modifications (line numbers helpful)
  - **kb_reference**: KB source for this implementation
- **files_to_create**: New files needed for RHOAI support
  - **path**: Where to create the file
  - **content_description**: What the file contains
  - **kb_reference**: KB template or example
- **configuration**: Technical details
  - **helm_chart**: Chart name and version (if applicable)
  - **container_images**: All images used by this component
  - **dependencies**: Other components this depends on (documents relationships between components)

#### validation_context (SECONDARY - context for validators)
- **blueprint_specifics**: Component details from blueprint (single/multi-replica, current deployment method, etc.)
- **user_cluster_constraints**: Cluster capabilities from Phase 3 user decisions (storage class, GPU availability, SCC policies)
- **deployment_decisions**: User choices from Phase 3 (model deployment strategy, ingress type, etc.)

## Example: Redis Component

```yaml
blueprint_name: nvidia-rag-pipeline
deployment_method: helm

components:
  redis:
    type: helm-service
    
    conversion_approach:
      strategy: "Add OpenShift support via Helm openshiftMode flag with conditional securityContext"
      kb_pattern_references:
        - "components/redis-on-rhoai.md#approach-a-helm"
      kb_summary: "Bitnami Redis chart supports nullable securityContext for OpenShift SCC compatibility"
      openshift_additions:
        - "Add openshiftMode: false flag to values.yaml (default preserves original)"
        - "Add conditional securityContext override in templates/redis.yaml"
        - "Override init container to registry.access.redhat.com/ubi9/ubi-minimal"
    
    implementation:
      files_to_modify:
        - path: deploy/helm/values.yaml
          changes:
            - "Line 1: Add openshiftMode: false"
            - "Add redis.podSecurityContext.enabled: true"
            - "Add redis.containerSecurityContext.enabled: true"
          kb_reference: "components/redis-on-rhoai.md#values-yaml-example"
        
        - path: deploy/helm/templates/redis.yaml
          changes:
            - "Wrap securityContext with {{- if .Values.openshiftMode }} conditional"
            - "Override volumePermissions.image to ubi9/ubi-minimal"
          kb_reference: "components/redis-on-rhoai.md#template-conditional-example"
      
      files_to_create: []
      
      configuration:
        helm_chart: bitnami/redis:19.0.2
        container_images:
          - docker.io/bitnami/redis:7.2.4-debian-12-r9
          - docker.io/bitnami/redis-exporter:1.58.0
          - registry.access.redhat.com/ubi9/ubi-minimal
        dependencies:
          - component: redis-pvc
            reason: "PVC must exist before Redis deployment"
    
    validation_context:
      blueprint_specifics: "Single-replica Redis from docker-compose, migrating to Helm chart"
      user_cluster_constraints: "RWO storage available, restricted-v2 SCC enforced, no GPU needed"
      deployment_decisions: "Using Helm deployment method with openshiftMode toggle"
```

## Usage

### For Main Agent (Phase 4.6)
Generate this spec after completing reasoning (Phase 4). Use knowledge loaded in Phase 2 to populate `kb_pattern_references` and `kb_summary` fields.

### For Validator Subagents (Phase 4.7)
Read component sections, load referenced KB patterns first, validate approach against KB + external resources.

### For Implementer Subagent (Phase 5)
Read refined spec, apply changes precisely as specified, perform basic self-testing.

## Notes

- **Single file**: All components in one spec file for easier management
- **KB traceability**: Implementation decisions trace back to KB sources when KB coverage exists
- **validation_context**: Provides context, doesn't dictate validation steps (validators know from their prompt)
- **Refined spec**: After validation, main agent writes `/tmp/conversion-spec-refined.yaml` with blocker fixes applied
