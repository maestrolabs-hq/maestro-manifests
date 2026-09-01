# Maestro Manifests Design

## Purpose

This repository is an inner-source catalog for everything Maestro can expose to Pi configurations: agents, skills, instructions, references, handoffs, workflows, addons, extensions, injections, command blocks, and policies.

The Rust project loads this catalog, validates it, presents it through `maestro init`, and writes the user's exact selections to `.maestro/config/settings.json`. Maestro's separate event runtime consumes the resolved configuration and enforces it over LLM events.

## Goals

- Keep one governed entry point at `maestro.yaml`.
- Store each contributed resource in its own YAML file.
- Discover all local resource files automatically.
- Compose local and HTTPS manifests.
- Make remote imports reproducible with required SHA-256 checksums.
- Reject ambiguous or unsafe composition.
- Let users see the full catalog and select the resources they want.
- Keep mandatory governance controls non-disableable.
- Provide a typed Rust library, CLI, JSON Schema, examples, and tests.

## Non-goals for v1

- Enforcing policies directly over LLM events.
- Git imports or package registries.
- A network service or database.
- A plugin runtime for executing addons or extensions.
- Silent conflict resolution.
- A graphical configuration interface.

## Repository layout

```text
maestro.yaml
Cargo.toml
src/
  lib.rs
  main.rs
  model.rs
  resolver.rs
  validate.rs
  settings.rs
  init.rs
manifests/
  agents/
    reviewer.yaml
  skills/
    code-review.yaml
  instructions/
    reviewer-system.yaml
  references/
    rust-style.yaml
  handoffs/
    implementation-to-review.yaml
  workflows/
    safe-development.yaml
  addons/
  extensions/
  injections/
  block-commands/
    destructive-shell.yaml
  policies/
    protect-secrets.yaml
examples/
  settings.json
tests/
  fixtures/
docs/
```

The directories communicate intent to contributors, but the `kind` field inside each file determines its resource type.

## Root manifest

`maestro.yaml` is the catalog entry point rather than a list of every resource:

```yaml
version: 1

metadata:
  name: maestro-community
  description: Shared Maestro resources and governance

discover:
  root: manifests
  recursive: true

imports:
  - url: https://example.org/maestro/security.yaml
    sha256: 0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
```

Local discovery loads `.yaml` and `.yml` files in deterministic lexical path order. Adding a resource requires adding its file; contributors do not also edit a central index.

## Resource documents

Every resource file has a common envelope:

```yaml
version: 1
kind: agent
name: reviewer
description: Review changes without editing files
tags: [review, read-only]
activation: selectable
customizable: [description]

spec:
  instructions: [reviewer-system]
  skills: [code-review]
  capabilities:
    write: false
    shell: false
```

Common fields:

- `version`: schema version, initially `1`.
- `kind`: one of `agent`, `skill`, `instruction`, `reference`, `handoff`, `workflow`, `addon`, `extension`, `injection`, `block-command`, or `policy`.
- `name`: stable identifier unique across resources of the same kind.
- `description`: contributor-facing explanation shown during initialization.
- `tags`: optional discovery metadata.
- `activation`: `selectable`, `default`, or `mandatory`.
- `customizable`: fields a local configuration may override; absent means none.
- `spec`: typed data for the selected kind.

All resources are visible during initialization:

- `selectable` resources are offered but initially unselected.
- `default` resources are initially selected and may be deselected.
- `mandatory` resources are selected and locked.

This preserves user choice without allowing local configuration to bypass centrally required governance.

## Initial resource specifications

The first schema validates the relationships Maestro needs while leaving execution to Maestro:

- **Agent:** instruction, skill, extension, and addon references plus capability flags.
- **Skill:** instruction and reference dependencies.
- **Instruction:** inline text or one repository-relative content path.
- **Reference:** local path or HTTPS URL; HTTPS requires SHA-256.
- **Handoff:** source agents, destination agent, and referenced instructions.
- **Workflow:** ordered named steps referencing agents, skills, handoffs, or other workflows.
- **Addon:** entry point and JSON-compatible configuration.
- **Extension:** entry point, events, and JSON-compatible configuration.
- **Injection:** target LLM event, referenced instruction, and optional condition metadata.
- **Block command:** command patterns and a required reason.
- **Policy:** target LLM events, effect, and JSON-compatible rule configuration.

Kind-specific `config` objects remain JSON-compatible where Maestro owns the final semantics. The manifest crate validates their envelope and references without duplicating Maestro's event engine.

## Import and discovery resolution

A source may be:

- the root local manifest;
- a discovered local resource document;
- an explicitly imported local manifest; or
- an explicitly imported HTTPS manifest or resource.

HTTPS imports require a lowercase 64-character SHA-256 checksum. The resolver verifies bytes before parsing YAML.

Resolution is deterministic:

1. Load the root document.
2. Canonicalize and sort discovered local files.
3. Resolve explicit imports depth-first in declaration order.
4. Verify remote checksums.
5. Detect cycles by canonical local path or normalized URL.
6. Reject duplicate `(kind, name)` identities.
7. Validate resource schemas and cross-references.
8. Return an immutable `ResolvedManifest`.

Local discovery and imports may not escape the root manifest directory after canonicalization. HTTPS redirects may not downgrade to HTTP. Resolver limits bound document size, source count, and graph depth to avoid unbounded input.

## User configuration

`maestro init` interviews the user against the entire resolved catalog and writes:

```text
.maestro/
  config/
    settings.json
```

Example:

```json
{
  "version": 1,
  "manifest": {
    "source": "./maestro.yaml"
  },
  "preferences": {
    "programming_languages": ["rust"],
    "response_language": "english"
  },
  "enabled": {
    "agents": ["reviewer", "implementer"],
    "skills": ["rust", "code-review"],
    "instructions": ["reviewer-system"],
    "references": [],
    "handoffs": [],
    "workflows": ["safe-development"],
    "addons": [],
    "extensions": [],
    "injections": [],
    "block_commands": ["destructive-shell"],
    "policies": ["protect-secrets"]
  },
  "overrides": {}
}
```

The settings file stores stable names and preferences, not copied resource definitions. Maestro resolves it against the selected manifest each time it validates or runs.

Initialization flow:

1. Select or confirm the manifest source.
2. Resolve and validate the complete catalog.
3. Ask project and response language preferences.
4. Present every resource grouped by kind and tags; dependencies such as instructions and references remain visible even when selected through a parent resource.
5. Show required dependencies for each selection.
6. Start `default` resources selected and `mandatory` resources selected and locked.
7. Validate the final selection and permitted overrides.
8. Preview the settings.
9. Create `.maestro/config/` and atomically write `settings.json`.

The initializer does not silently add dependencies. It explains them and asks for confirmation so the resulting settings remain an exact record of the user's choices. Refusing a required dependency prevents selecting the dependent resource.

## Local customization

Settings may override only fields listed in a resource's `customizable` array. Resolution rejects:

- overrides of unlisted fields;
- changes to resource identity or activation;
- disabling mandatory resources;
- selected resources with missing dependencies; and
- references to resources absent from the resolved catalog.

## CLI

The binary is named `maestro` and initially exposes:

```text
maestro check [maestro.yaml]
maestro resolve [maestro.yaml] [--settings PATH] [--output PATH]
maestro init [maestro.yaml] [--output .maestro/config/settings.json]
maestro schema [manifest|resource|settings]
```

- `check` validates the import graph and catalog.
- `resolve` emits deterministic resolved JSON, optionally applying settings.
- `init` runs the interactive configuration interview.
- `schema` emits JSON Schema for editor and CI integration.

Commands return nonzero status on failure and print concise source-aware diagnostics.

## Library API

The crate exposes the minimum stable surface Maestro needs:

```rust
pub fn load_manifest(source: impl AsRef<Path>) -> Result<ResolvedManifest, Error>;
pub fn load_settings(path: impl AsRef<Path>) -> Result<Settings, Error>;
pub fn resolve_settings(
    manifest: &ResolvedManifest,
    settings: &Settings,
) -> Result<ResolvedConfiguration, Error>;
```

Concrete model types and schema generation are also public. Network fetching remains behind the resolver rather than a general plugin interface.

## Errors and safety

Diagnostics include the source path or URL, resource identity when known, and the failing field. Distinct errors cover malformed YAML, unsupported versions, unsafe paths, failed downloads, checksum mismatches, import cycles, duplicate identities, missing references, forbidden overrides, and invalid selections.

Settings writes use a temporary sibling file followed by rename so interruption cannot truncate an existing configuration.

## Testing

The project includes focused tests for:

- loading one valid resource per kind;
- deterministic recursive discovery;
- local and HTTPS import resolution;
- checksum mismatch rejection;
- cycle and duplicate detection;
- broken cross-reference rejection;
- mandatory/default/selectable activation;
- forbidden overrides and missing dependencies;
- `maestro init` settings generation through separated interview I/O; and
- atomic settings output.

CLI smoke tests cover successful checks and useful failure output. Formatting, Clippy with warnings denied, and tests form the initial CI quality gate. Repository peer review remains the contribution gate.

## Future artifacts

The `.maestro/` root intentionally leaves room for runtime-owned artifacts such as caches, state, logs, and handoffs. They are not created until a concrete runtime feature needs them.

Possible future work includes Git imports, a lock file for mutable manifest sources, richer policy schemas, organization-provided initialization presets, and a graphical interview. None are required by the v1 contract.
