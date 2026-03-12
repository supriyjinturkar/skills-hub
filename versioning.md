# Versioning Standards Guide

This guide defines the general versioning and branch procedure for shared component repositories.

It applies across:
- skills
- tools
- plugins
- workflow

## Purpose

These repositories keep multiple components together, but each component still needs its own controlled lifecycle.

This guide uses a branch-based model for two cases:
- creating a brand-new component
- releasing a new version of an existing component

## Scope

This process applies to a shared repository that contains multiple components of the same type.

Examples:
- the `skills` repo contains many skills
- the `plugins` repo contains many plugins
- the `tools` repo contains many tools
- the `workflow` repo contains many workflow integrations/templates

## Branch Model

The repository should use these branch roles:

- `develop`
  - Main working branch.
  - Contains the latest approved state of all components in that repository.
  - All approved component work eventually lands here.

- `empty`
  - A completely empty baseline branch.
  - Used only as the clean starting point when creating a brand-new component from scratch.
  - Contains no existing component implementation.

- `<component-name>`
  - The main branch for one component.
  - Examples:
    - `pdf-processing`
    - `telecom-analyzer`
    - `kubernetes-tooling`
    - `approval-workflow`
  - Holds the ongoing history of that component.

- `<component-name>-<version>`
  - Used when releasing a new version of an existing component.
  - Examples:
    - `pdf-processing-1.1.0`
    - `telecom-analyzer-2.0.0`

## Recommended Naming

Use stable component names in lowercase kebab-case.

Recommended:
- component branch: `pdf-processing`
- version branch: `pdf-processing-1.1.0`

Avoid plain version-only branch names such as:
- `1.1.0`
- `2.0.1`

Those become ambiguous when multiple components are versioned in the same repository.

## Version Source Of Truth

Git branch flow controls review and merge behavior.
Actual version numbers must still live in the component's own source of truth.

Examples:
- skills: `metadata.version` inside `SKILL.md`
- plugins: package/plugin manifest version
- tools: package/module version definition
- workflow: package/workflow integration version definition

Rules:
- branch flow tracks change lifecycle
- component version fields track the actual released version
- maintainer-created tags mark the approved released version
- branch names, component version fields, and release tags must stay aligned

## Flow 1: Create A Brand-New Component

Use this when the component does not already exist.

### Steps

1. Start from the `empty` branch.
2. Create a new branch using the component name.
   - Example: `pdf-processing`
3. Add the new component data in the correct repository location.
4. Set the initial version in the component's version source of truth.
   - Example: `1.0.0`
5. Open a PR from the component branch into `develop`.
6. After approval, merge into `develop`.
7. Once it is approved and merged, the maintainer creates the release tag for the same component release with the corresponding version.
   - Example: `pdf-processing-1.0.0`

### Result

- the component now exists in its own component branch
- the same approved state is also present in `develop`
- the maintainer-created tag marks the approved released version
- future work on that component should build from the component branch, not from `empty`

## Flow 2: Release A New Version Of An Existing Component

Use this when the component already exists and a new version must be released.

### Steps

1. Start from the existing component branch.
   - Example: `pdf-processing`
2. Create a version branch from it.
   - Example: `pdf-processing-1.1.0`
3. Update the component content.
4. Update the component's version source of truth.
   - Example: change `1.0.0` to `1.1.0`
5. Open a PR from the version branch into the main component branch.
6. Merge the version branch into the component branch after approval.
7. On merge, the CI workflow creates the release tag and updates the registry.
   - Example: `pdf-processing-1.1.0`
8. A separate CI workflow detects the new release and syncs the updated component into `develop`.

### Result

- the component branch remains the long-term branch for that component
- the version branch is a temporary release-preparation branch
- the release tag is created when the version branch merges into the component branch
- `develop` receives the latest released version automatically via a CI workflow
- the release tag marks the approved released version

## Merge Strategy Summary

### New component

```text
empty
  -> <component-name>
  -> PR to develop
  -> merge to develop
  -> maintainer creates release tag
```

### Existing component new version

```text
<component-name>
  -> <component-name>-<version>
  -> PR to <component-name>
  -> merge to <component-name>
  -> CI creates release tag
  -> CI syncs to develop
```

## Required Rules

- Never create a new component directly from `develop` if the process requires a clean start.
- Always create a new component from `empty`.
- Always create a new version branch from the existing component branch.
- Always bump the component version field where that component type defines one.
- Always merge approved component changes back to `develop` (automated via CI workflow on release).
- Release tags are created automatically by CI when a version branch merges into the component branch.

## Component-Specific Notes

### Skills
- Version source of truth: `metadata.version` in `SKILL.md`
- Example:

```yaml
metadata:
  author: example-org
  version: "1.0.0"
```

### Plugins
- Version source of truth should be the plugin package or plugin manifest version.
- If plugins expose runtime metadata, that metadata should match the released component version.

### Tools
- Version source of truth should be the tools package or module release version.
- If one repo contains many tool groups, each independently versioned tool should still have a clear version mapping policy.

### Workflow
- Version source of truth should be the workflow package/integration release version.
- If a workflow component also carries internal template/schema versioning, document how it maps to release versions.

## Why This Model Works

- keeps each component isolated
- gives every component its own history branch
- allows version-specific review before updating the main component branch
- keeps `develop` as the latest approved integrated state
- uses maintainer-created tags as the approved release checkpoint

## Future Release Hardening

If the team later automates package publishing, use Git tags in addition to this branch model.

Recommended future pattern:
- branch flow for review and change control
- component-local version field for content/package version
- Git tag for release checkpoint
- package release built from the approved tagged state
