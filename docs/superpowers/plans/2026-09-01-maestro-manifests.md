# Maestro Manifests Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Rust library and `maestro` CLI that discover, compose, validate, inspect, and configure the governed Maestro manifest catalog.

**Architecture:** Typed YAML resource documents are recursively discovered from a root catalog and composed with checksum-pinned local or HTTPS imports. The library returns an immutable resolved catalog; the CLI validates it, emits schemas or resolved JSON, and conducts an initialization interview that atomically writes `.maestro/config/settings.json`.

**Tech Stack:** Stable Rust 2024 edition, serde, serde_yaml, serde_json, schemars, clap, reqwest blocking client with rustls, sha2, thiserror, and tempfile for tests.

**Spec:** `docs/superpowers/specs/2026-09-01-maestro-manifests-design.md`

## Global Constraints

- The root entry point is `maestro.yaml`.
- Local discovery recursively loads `.yaml` and `.yml` files in lexical path order.
- Local sources must remain beneath the root manifest directory after canonicalization.
- HTTPS imports require an exact lowercase 64-character SHA-256 checksum.
- HTTP URLs and HTTPS-to-HTTP redirects are rejected.
- Duplicate `(kind, name)` identities and import cycles are errors.
- Every resource is visible; `selectable`, `default`, and `mandatory` control initialization state.
- Mandatory resources cannot be disabled, and local overrides are denied unless explicitly listed in `customizable`.
- Settings are written to `.maestro/config/settings.json` by default using temporary-file-plus-rename.
- LLM event enforcement and plugin execution remain outside this crate.
- Run `cargo fmt --check`, `cargo clippy --all-targets --all-features -- -D warnings`, and `cargo test --all-features` before completion.

## File map

- `Cargo.toml`: package, binary target, dependencies, and lint policy.
- `src/lib.rs`: public API and exports.
- `src/error.rs`: source-aware error variants.
- `src/model.rs`: root manifest, resource envelopes, typed resource specs, and identities.
- `src/resolver.rs`: local discovery, import graph traversal, HTTPS fetching, checksum verification, and resolved catalog.
- `src/validate.rs`: kind-specific invariants and cross-reference validation.
- `src/settings.rs`: settings model, selection/override resolution, and atomic JSON writes.
- `src/interview.rs`: terminal-independent initialization interview over `BufRead`/`Write`.
- `src/main.rs`: `check`, `resolve`, `init`, and `schema` CLI commands.
- `tests/model.rs`: resource parsing tests.
- `tests/resolver.rs`: discovery, imports, checksum, cycle, and duplicate tests.
- `tests/validation.rs`: references, activation, dependencies, and override tests.
- `tests/interview.rs`: deterministic interview and settings-write tests.
- `tests/cli.rs`: binary smoke tests.
- `tests/fixtures/`: complete valid and invalid catalogs.
- `maestro.yaml`: repository catalog entry point.
- `manifests/**`: one example file per supported resource kind.
- `examples/settings.json`: generated settings example.
- `.gitignore`: Rust build output, visual-companion state, and runtime-local Maestro artifacts.
- `.github/workflows/ci.yml`: formatting, Clippy, and test gate.
- `README.md`: contributor and consumer quick start.

---

### Task 1: Initialize the crate and typed document model

**Files:**

- Create: `Cargo.toml`
- Create: `src/lib.rs`
- Create: `src/error.rs`
- Create: `src/model.rs`
- Create: `tests/model.rs`
- Create: `.gitignore`

**Interfaces:**

- Consumes: Approved design spec only.
- Produces: `RootManifest`, `ResourceDocument`, `ResourceBody`, `ResourceKind`, `ResourceId`, `Activation`, `Import`, and `Error`.

- [ ] **Step 1: Initialize Git and write package metadata**

Run:

```bash
git init
```

Create `Cargo.toml` with:

```toml
[package]
name = "maestro-manifests"
version = "0.1.0"
edition = "2024"
license = "Apache-2.0"
description = "Governed manifest catalog and configuration CLI for Maestro"

[lib]
name = "maestro_manifests"
path = "src/lib.rs"

[[bin]]
name = "maestro"
path = "src/main.rs"

[dependencies]
clap = { version = "4", features = ["derive"] }
reqwest = { version = "0.12", default-features = false, features = ["blocking", "rustls-tls"] }
schemars = "1"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
serde_yaml = "0.9"
sha2 = "0.10"
thiserror = "2"

[dev-dependencies]
tempfile = "3"

[lints.rust]
unsafe_code = "forbid"

[lints.clippy]
all = "warn"
pedantic = "warn"
```

Create `.gitignore` with:

```gitignore
/target/
/.superpowers/
/.maestro/cache/
/.maestro/state/
/.maestro/logs/
```

- [ ] **Step 2: Write failing parsing tests**

Create `tests/model.rs`:

```rust
use maestro_manifests::{Activation, ResourceBody, ResourceKind};

#[test]
fn parses_agent_resource_envelope() {
    let yaml = r#"
version: 1
kind: agent
name: reviewer
description: Reviews changes
activation: selectable
customizable: [description]
spec:
  instructions: [reviewer-system]
  skills: [code-review]
  capabilities:
    write: false
"#;

    let resource = maestro_manifests::parse_resource(yaml).unwrap();
    assert_eq!(resource.name, "reviewer");
    assert_eq!(resource.activation, Activation::Selectable);
    assert_eq!(resource.kind(), ResourceKind::Agent);
    assert!(matches!(resource.body, ResourceBody::Agent(_)));
}

#[test]
fn rejects_unknown_resource_kind() {
    let yaml = "version: 1\nkind: mystery\nname: nope\ndescription: nope\nspec: {}\n";
    assert!(maestro_manifests::parse_resource(yaml).is_err());
}
```

- [ ] **Step 3: Run the model tests and confirm failure**

Run:

```bash
cargo test --test model
```

Expected: compilation fails because the public model and `parse_resource` do not exist.

- [ ] **Step 4: Implement the typed model and parse function**

Create `src/error.rs` with a non-exhaustive `Error` enum containing concrete variants for `Io`, `Yaml`, `Json`, `UnsupportedVersion`, `UnsafePath`, `Http`, `InvalidUrl`, `ChecksumMismatch`, `ImportCycle`, `DuplicateResource`, `MissingReference`, `InvalidResource`, `ForbiddenOverride`, and `InvalidSelection`. Each source-related variant carries a `source: String`.

Create `src/model.rs` around this shape:

```rust
use std::{collections::BTreeMap, path::PathBuf};
use schemars::JsonSchema;
use serde::{Deserialize, Serialize};
use serde_json::Value;

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Serialize, Deserialize, JsonSchema)]
#[serde(rename_all = "kebab-case")]
pub enum ResourceKind {
    Agent,
    Skill,
    Instruction,
    Reference,
    Handoff,
    Workflow,
    Addon,
    Extension,
    Injection,
    BlockCommand,
    Policy,
}

#[derive(Debug, Clone, Copy, Default, PartialEq, Eq, Serialize, Deserialize, JsonSchema)]
#[serde(rename_all = "kebab-case")]
pub enum Activation {
    #[default]
    Selectable,
    Default,
    Mandatory,
}

#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord, Serialize, Deserialize, JsonSchema)]
pub struct ResourceId {
    pub kind: ResourceKind,
    pub name: String,
}

#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct RootManifest {
    pub version: u32,
    pub metadata: Metadata,
    #[serde(default)]
    pub discover: Option<Discover>,
    #[serde(default)]
    pub imports: Vec<Import>,
}

#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct Metadata {
    pub name: String,
    #[serde(default)]
    pub description: String,
}

#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct Discover {
    pub root: PathBuf,
    #[serde(default)]
    pub recursive: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
#[serde(untagged)]
pub enum Import {
    Local { path: PathBuf },
    Https { url: String, sha256: String },
}

#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct ResourceDocument {
    pub version: u32,
    pub name: String,
    pub description: String,
    #[serde(default)]
    pub tags: Vec<String>,
    #[serde(default)]
    pub activation: Activation,
    #[serde(default)]
    pub customizable: Vec<String>,
    #[serde(flatten)]
    pub body: ResourceBody,
}

#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
#[serde(tag = "kind", content = "spec", rename_all = "kebab-case")]
pub enum ResourceBody {
    Agent(AgentSpec),
    Skill(SkillSpec),
    Instruction(InstructionSpec),
    Reference(ReferenceSpec),
    Handoff(HandoffSpec),
    Workflow(WorkflowSpec),
    Addon(ExecutableSpec),
    Extension(ExtensionSpec),
    Injection(InjectionSpec),
    BlockCommand(BlockCommandSpec),
    Policy(PolicySpec),
}

#[derive(Debug, Clone, Default, Serialize, Deserialize, JsonSchema)]
pub struct AgentSpec {
    #[serde(default)] pub instructions: Vec<String>,
    #[serde(default)] pub skills: Vec<String>,
    #[serde(default)] pub extensions: Vec<String>,
    #[serde(default)] pub addons: Vec<String>,
    #[serde(default)] pub capabilities: BTreeMap<String, bool>,
}

#[derive(Debug, Clone, Default, Serialize, Deserialize, JsonSchema)]
pub struct SkillSpec {
    #[serde(default)] pub instructions: Vec<String>,
    #[serde(default)] pub references: Vec<String>,
    #[serde(default)] pub skills: Vec<String>,
}

#[derive(Debug, Clone, Default, Serialize, Deserialize, JsonSchema)]
pub struct InstructionSpec { pub text: Option<String>, pub path: Option<PathBuf> }
#[derive(Debug, Clone, Default, Serialize, Deserialize, JsonSchema)]
pub struct ReferenceSpec { pub path: Option<PathBuf>, pub url: Option<String>, pub sha256: Option<String> }
#[derive(Debug, Clone, Default, Serialize, Deserialize, JsonSchema)]
pub struct HandoffSpec { #[serde(default)] pub from: Vec<String>, pub to: String, #[serde(default)] pub instructions: Vec<String> }
#[derive(Debug, Clone, Default, Serialize, Deserialize, JsonSchema)]
pub struct WorkflowSpec { #[serde(default)] pub steps: Vec<WorkflowStep> }
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct WorkflowStep { pub name: String, pub agent: Option<String>, pub skill: Option<String>, pub handoff: Option<String>, pub workflow: Option<String> }
#[derive(Debug, Clone, Default, Serialize, Deserialize, JsonSchema)]
pub struct ExecutableSpec { pub entrypoint: String, #[serde(default)] pub config: Value }
#[derive(Debug, Clone, Default, Serialize, Deserialize, JsonSchema)]
pub struct ExtensionSpec { pub entrypoint: String, #[serde(default)] pub events: Vec<String>, #[serde(default)] pub config: Value }
#[derive(Debug, Clone, Default, Serialize, Deserialize, JsonSchema)]
pub struct InjectionSpec { pub event: String, pub instruction: String, #[serde(default)] pub condition: Value }
#[derive(Debug, Clone, Default, Serialize, Deserialize, JsonSchema)]
pub struct BlockCommandSpec { #[serde(default)] pub patterns: Vec<String>, pub reason: String }
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct PolicySpec { #[serde(default)] pub events: Vec<String>, pub effect: PolicyEffect, #[serde(default)] pub rules: Value }
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
#[serde(rename_all = "kebab-case")]
pub enum PolicyEffect { Allow, Deny, Inject, Audit }
```

Implement `ResourceDocument::kind`, `ResourceDocument::id`, and `parse_resource` with `serde_yaml::from_str`, then reject versions other than `1`.

Create `src/lib.rs`:

```rust
mod error;
mod model;

pub use error::Error;
pub use model::*;

pub fn parse_resource(input: &str) -> Result<ResourceDocument, Error> {
    let resource: ResourceDocument = serde_yaml::from_str(input)?;
    if resource.version != 1 {
        return Err(Error::UnsupportedVersion(resource.version));
    }
    Ok(resource)
}
```

- [ ] **Step 5: Run and format the model implementation**

Run:

```bash
cargo fmt --all
cargo test --test model
cargo clippy --lib -- -D warnings
```

Expected: both tests pass and Clippy reports no warnings.

- [ ] **Step 6: Commit the typed model**

```bash
git add Cargo.toml .gitignore src/lib.rs src/error.rs src/model.rs tests/model.rs
git commit -m "feat: add typed manifest resource model"
```

---

### Task 2: Resolve local discovery and import graphs

**Files:**

- Create: `src/resolver.rs`
- Modify: `src/lib.rs`
- Create: `tests/resolver.rs`
- Create: `tests/fixtures/valid/maestro.yaml`
- Create: `tests/fixtures/valid/manifests/agents/reviewer.yaml`

**Interfaces:**

- Consumes: `RootManifest`, `Import`, `ResourceDocument`, `ResourceId`, and `Error` from Task 1.
- Produces: `ResolvedManifest`, `ResolvedResource`, `ResolverLimits`, and `load_manifest(path: impl AsRef<Path>) -> Result<ResolvedManifest, Error>`.

- [ ] **Step 1: Write failing local-resolution tests**

Create fixtures where `tests/fixtures/valid/maestro.yaml` discovers `manifests` recursively and the reviewer resource contains the valid agent YAML from Task 1.

Create tests that assert:

```rust
#[test]
fn discovers_nested_yaml_resources() {
    let resolved = maestro_manifests::load_manifest("tests/fixtures/valid/maestro.yaml").unwrap();
    assert!(resolved.get(ResourceKind::Agent, "reviewer").is_some());
}

#[test]
fn rejects_duplicate_resource_ids() {
    let fixture = duplicate_fixture_with_two_agents_named("reviewer");
    let error = maestro_manifests::load_manifest(fixture.path().join("maestro.yaml")).unwrap_err();
    assert!(matches!(error, maestro_manifests::Error::DuplicateResource { .. }));
}

#[test]
fn rejects_local_import_cycles() {
    let fixture = cycle_fixture("a.yaml", "b.yaml");
    let error = maestro_manifests::load_manifest(fixture.path().join("a.yaml")).unwrap_err();
    assert!(matches!(error, maestro_manifests::Error::ImportCycle { .. }));
}
```

Fixture helpers create files with `tempfile::TempDir` and `std::fs::write`; they return the `TempDir` so files remain alive for the assertion.

- [ ] **Step 2: Run the resolver tests and confirm failure**

Run:

```bash
cargo test --test resolver
```

Expected: compilation fails because `load_manifest`, `ResolvedManifest`, and resolver error variants are unavailable.

- [ ] **Step 3: Implement bounded local graph resolution**

Create `src/resolver.rs` with:

```rust
pub const DEFAULT_MAX_SOURCES: usize = 256;
pub const DEFAULT_MAX_DEPTH: usize = 32;
pub const DEFAULT_MAX_DOCUMENT_BYTES: u64 = 1_048_576;

#[derive(Debug, Clone)]
pub struct ResolverLimits {
    pub max_sources: usize,
    pub max_depth: usize,
    pub max_document_bytes: u64,
}

#[derive(Debug, Clone, Serialize)]
pub struct ResolvedResource {
    pub id: ResourceId,
    pub source: String,
    pub document: ResourceDocument,
}

#[derive(Debug, Clone, Serialize)]
pub struct ResolvedManifest {
    pub metadata: Metadata,
    pub resources: Vec<ResolvedResource>,
    #[serde(skip)]
    index: BTreeMap<ResourceId, usize>,
}

impl ResolvedManifest {
    pub fn get(&self, kind: ResourceKind, name: &str) -> Option<&ResolvedResource>;
    pub fn iter(&self) -> impl Iterator<Item = &ResolvedResource>;
}

pub fn load_manifest(path: impl AsRef<Path>) -> Result<ResolvedManifest, Error> {
    Resolver::default().load(path.as_ref())
}
```

Implement a private resolver state with `root_dir`, `visiting`, `visited`, `resources`, and a source counter. For each local source:

1. Canonicalize it.
2. Reject it unless `canonical.starts_with(root_dir)`.
3. Reject repeated entries in `visiting` as a cycle; skip entries already in `visited`.
4. Reject metadata lengths over `max_document_bytes` before reading.
5. Parse a YAML mapping containing `kind` as `ResourceDocument`; otherwise parse `RootManifest`.
6. For a catalog, resolve local imports and then deterministic discovery.
7. For discovery, walk directories with `std::fs::read_dir`, sort each collected path, recurse only when requested, and include only `.yaml` or `.yml` files.
8. Insert resources with `BTreeMap::entry`; occupied entries return `DuplicateResource` with both source names.
9. Convert the sorted map into `ResolvedManifest.resources` and build its skipped lookup index before returning.

Do not add a directory-walking dependency.

- [ ] **Step 4: Export resolver interfaces and run tests**

Add `mod resolver;` and public re-exports to `src/lib.rs`.

Run:

```bash
cargo fmt --all
cargo test --test resolver
cargo clippy --lib -- -D warnings
```

Expected: discovery passes; duplicate, cycle, source-count, depth, file-size, and root-escape cases return their expected errors.

- [ ] **Step 5: Commit local resolution**

```bash
git add src/lib.rs src/resolver.rs tests/resolver.rs tests/fixtures/valid
git commit -m "feat: resolve local manifest graphs"
```

---

### Task 3: Add checksum-pinned HTTPS imports

**Files:**

- Modify: `src/resolver.rs`
- Modify: `tests/resolver.rs`

**Interfaces:**

- Consumes: Task 2 resolver state and `Import::Https`.
- Produces: HTTPS source loading through the existing `load_manifest` API; no new public fetch abstraction.

- [ ] **Step 1: Write failing HTTPS resolver unit tests with an injected byte fetcher**

Inside `src/resolver.rs`, add a `#[cfg(test)] mod remote_tests` so tests can use a private fetch seam without exposing it from the crate:

```rust
#[test]
fn loads_https_import_with_matching_sha256() {
    let body = br#"version: 1
kind: agent
name: remote-reviewer
description: Remote reviewer
spec: {}
"#.to_vec();
    let digest = sha256_hex(&body);
    let fixture = tempfile::tempdir().unwrap();
    let root = fixture.path().join("maestro.yaml");
    std::fs::write(&root, format!(r#"version: 1
metadata:
  name: test
imports:
  - url: https://example.test/reviewer.yaml
    sha256: {digest}
"#)).unwrap();
    let fetched = body.clone();
    let resolver = Resolver::for_test(Box::new(move |url| {
        assert_eq!(url.as_str(), "https://example.test/reviewer.yaml");
        Ok(fetched.clone())
    }));

    let resolved = resolver.load(&root).unwrap();
    assert!(resolved.get(ResourceKind::Agent, "remote-reviewer").is_some());
}

#[test]
fn rejects_https_checksum_mismatch() {
    let fixture = tempfile::tempdir().unwrap();
    let root = fixture.path().join("maestro.yaml");
    std::fs::write(&root, "version: 1\nmetadata:\n  name: test\nimports:\n  - url: https://example.test/a.yaml\n    sha256: 0000000000000000000000000000000000000000000000000000000000000000\n").unwrap();
    let resolver = Resolver::for_test(Box::new(|_| Ok(b"version: 1\n".to_vec())));

    assert!(matches!(resolver.load(&root), Err(Error::ChecksumMismatch { .. })));
}

#[test]
fn rejects_http_imports_before_fetching() {
    let fixture = tempfile::tempdir().unwrap();
    let root = fixture.path().join("maestro.yaml");
    std::fs::write(&root, "version: 1\nmetadata:\n  name: test\nimports:\n  - url: http://example.test/a.yaml\n    sha256: 0000000000000000000000000000000000000000000000000000000000000000\n").unwrap();
    let resolver = Resolver::for_test(Box::new(|_| panic!("HTTP import must not fetch")));

    assert!(matches!(resolver.load(&root), Err(Error::InvalidUrl { .. })));
}
```

`Resolver::for_test` accepts a boxed private closure `Fn(&reqwest::Url) -> Result<Vec<u8>, Error>`. Production construction supplies the reqwest implementation; the seam is unavailable outside this module.

- [ ] **Step 2: Run checksum tests and confirm failure**

Run:

```bash
cargo test --test resolver checksum https http_imports
```

Expected: new tests fail because remote imports are not implemented.

- [ ] **Step 3: Implement remote loading and digest verification**

Add a private boxed fetch closure to `Resolver`; `Resolver::default` creates a blocking `reqwest::Client` configured with a 15-second timeout, at most 5 redirects, and a redirect policy rejecting any destination whose scheme is not `https`. `Resolver::for_test` substitutes deterministic bytes. Before fetching, validate:

```rust
fn valid_sha256(value: &str) -> bool {
    value.len() == 64 && value.bytes().all(|byte| byte.is_ascii_digit() || (b'a'..=b'f').contains(&byte))
}
```

Read at most `max_document_bytes + 1` bytes, reject oversized bodies, calculate `Sha256::digest(&bytes)`, format bytes with `write!(&mut digest, "{byte:02x}")`, compare using exact string equality, and only then parse YAML. Use normalized `reqwest::Url::to_string()` as the cycle identity.

Imported remote catalog documents may contain further HTTPS imports. Reject `discover` and local-path imports when their parent source is remote because they have no trusted local root.

- [ ] **Step 4: Run resolver verification**

Run:

```bash
cargo fmt --all
cargo test --test resolver
cargo clippy --all-targets -- -D warnings
```

Expected: valid pinned content loads; checksum mismatch, malformed checksum, HTTP URL, downgrade redirect, oversized response, and remote-local import cases fail.

- [ ] **Step 5: Commit remote resolution**

```bash
git add src/resolver.rs tests/resolver.rs
git commit -m "feat: resolve pinned HTTPS manifest imports"
```

---

### Task 4: Validate resources, selections, and overrides

**Files:**

- Create: `src/validate.rs`
- Create: `src/settings.rs`
- Modify: `src/lib.rs`
- Create: `tests/validation.rs`

**Interfaces:**

- Consumes: `ResolvedManifest` and typed resource specs.
- Produces: `validate_manifest(&ResolvedManifest)`, `Settings`, `ResolvedConfiguration`, `load_settings`, `resolve_settings`, and `write_settings_atomic`.

- [ ] **Step 1: Write failing semantic-validation tests**

Create tests covering exact invariants:

```rust
#[test]
fn rejects_agent_with_missing_skill() {
    let manifest = resolved_fixture_with_agent_skill("missing");
    let error = maestro_manifests::validate_manifest(&manifest).unwrap_err();
    assert!(matches!(error, Error::MissingReference { .. }));
}

#[test]
fn rejects_instruction_with_both_text_and_path() {
    let manifest = resolved_instruction(Some("inline"), Some("prompt.md"));
    assert!(matches!(validate_manifest(&manifest), Err(Error::InvalidResource { .. })));
}

#[test]
fn mandatory_resources_are_added_and_cannot_be_removed() {
    let manifest = manifest_with_mandatory_policy("protect-secrets");
    let settings = Settings::default();
    let resolved = resolve_settings(&manifest, &settings).unwrap();
    assert!(resolved.enabled.contains(&ResourceId::new(ResourceKind::Policy, "protect-secrets")));
}

#[test]
fn rejects_override_not_declared_customizable() {
    let manifest = manifest_with_agent_customizable("description");
    let settings = settings_overriding("agent", "reviewer", "activation", json!("selectable"));
    assert!(matches!(resolve_settings(&manifest, &settings), Err(Error::ForbiddenOverride { .. })));
}
```

Also test one valid and one broken reference path for every kind that contains references.

- [ ] **Step 2: Run validation tests and confirm failure**

Run:

```bash
cargo test --test validation
```

Expected: compilation fails because validation and settings APIs are missing.

- [ ] **Step 3: Implement manifest semantic validation**

In `src/validate.rs`, implement:

```rust
pub fn validate_manifest(manifest: &ResolvedManifest) -> Result<(), Error>;
```

Use one private helper:

```rust
fn require(
    manifest: &ResolvedManifest,
    owner: &ResourceId,
    kind: ResourceKind,
    name: &str,
) -> Result<(), Error>;
```

Validate:

- names are nonempty ASCII lowercase identifiers containing only `a-z`, `0-9`, `-`, `_`, or `.`;
- instruction has exactly one of `text` and `path`;
- reference has exactly one of `path` and `url`, and URL references follow HTTPS/checksum rules;
- workflow steps have unique nonempty names and exactly one target field;
- executable entry points, injection events/instructions, block patterns/reasons, and policy events are nonempty;
- every typed reference points to the required kind;
- skill-to-skill and workflow-to-workflow dependency graphs are acyclic.

Use iterative depth-first traversal with `BTreeSet` for deterministic cycle diagnostics.

- [ ] **Step 4: Implement settings and resolved configuration**

In `src/settings.rs`, define:

```rust
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct Settings {
    pub version: u32,
    pub manifest: ManifestSelection,
    #[serde(default)] pub preferences: Preferences,
    #[serde(default)] pub enabled: BTreeMap<ResourceKind, BTreeSet<String>>,
    #[serde(default)] pub overrides: BTreeMap<ResourceKind, BTreeMap<String, BTreeMap<String, Value>>>,
}

impl Default for Settings {
    fn default() -> Self {
        Self {
            version: 1,
            manifest: ManifestSelection { source: "./maestro.yaml".into() },
            preferences: Preferences::default(),
            enabled: BTreeMap::new(),
            overrides: BTreeMap::new(),
        }
    }
}

#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct ManifestSelection { pub source: String }

#[derive(Debug, Clone, Default, Serialize, Deserialize, JsonSchema)]
pub struct Preferences {
    #[serde(default)] pub programming_languages: Vec<String>,
    #[serde(default)] pub response_language: Option<String>,
}

#[derive(Debug, Clone, Serialize)]
pub struct ResolvedConfiguration {
    pub manifest: ResolvedManifest,
    pub settings: Settings,
    pub enabled: BTreeSet<ResourceId>,
}

pub fn load_settings(path: impl AsRef<Path>) -> Result<Settings, Error>;
pub fn resolve_settings(manifest: &ResolvedManifest, settings: &Settings) -> Result<ResolvedConfiguration, Error>;
pub fn write_settings_atomic(path: impl AsRef<Path>, settings: &Settings) -> Result<(), Error>;
```

`resolve_settings` validates version `1`, starts from explicit selections plus every `default` and `mandatory` resource, rejects absent names and missing required dependencies, and checks every override key against `customizable`. Identity and activation are never overrideable even if mistakenly listed.

`write_settings_atomic` creates the parent directory, writes pretty JSON plus a trailing newline to a uniquely named sibling using `create_new(true)`, calls `sync_all`, and renames it over the destination. Remove the temporary file after write failures.

- [ ] **Step 5: Export and verify validation/settings**

Update `src/lib.rs` to export the modules and call `validate_manifest` at the end of `load_manifest`.

Run:

```bash
cargo fmt --all
cargo test --test validation
cargo clippy --all-targets -- -D warnings
```

Expected: all semantic, activation, dependency, override, and atomic-write tests pass.

- [ ] **Step 6: Commit semantic validation and settings**

```bash
git add src/lib.rs src/validate.rs src/settings.rs tests/validation.rs
git commit -m "feat: validate manifests and user settings"
```

---

### Task 5: Build the initialization interview

**Files:**

- Create: `src/interview.rs`
- Modify: `src/lib.rs`
- Create: `tests/interview.rs`

**Interfaces:**

- Consumes: `ResolvedManifest`, `Settings`, `Activation`, and `write_settings_atomic`.
- Produces: `run_interview<R: BufRead, W: Write>(manifest, source, input, output) -> Result<Settings, Error>`.

- [ ] **Step 1: Write a failing transcript-driven interview test**

Create `tests/interview.rs`:

```rust
#[test]
fn interview_lists_every_resource_and_records_exact_choices() {
    let manifest = interview_fixture();
    let input = b"rust\nenglish\ny\nn\ny\n";
    let mut output = Vec::new();

    let settings = maestro_manifests::run_interview(
        &manifest,
        "./maestro.yaml",
        &input[..],
        &mut output,
    ).unwrap();

    let transcript = String::from_utf8(output).unwrap();
    assert!(transcript.contains("reviewer — Reviews changes"));
    assert!(transcript.contains("protect-secrets [mandatory]"));
    assert!(settings.enabled[&ResourceKind::Agent].contains("reviewer"));
    assert!(settings.enabled[&ResourceKind::Policy].contains("protect-secrets"));
}
```

Add a second test where selecting an agent requires an unselected skill, the transcript asks to include it, and answering `n` leaves both resources unselected.

- [ ] **Step 2: Run interview tests and confirm failure**

Run:

```bash
cargo test --test interview
```

Expected: compilation fails because `run_interview` is missing.

- [ ] **Step 3: Implement the minimal line-oriented interview**

Implement:

```rust
pub fn run_interview<R: BufRead, W: Write>(
    manifest: &ResolvedManifest,
    source: &str,
    mut input: R,
    mut output: W,
) -> Result<Settings, Error>;
```

Prompt for comma-separated programming languages and one response language. Iterate resources in `BTreeMap` order, always print identity, description, tags, and activation. Use `[y/N]` for selectable, `[Y/n]` for default, and `[mandatory]` without a prompt for mandatory.

A private `read_choice` accepts `y`, `yes`, `n`, `no`, or an empty line with the provided default and repeats after other input. Before accepting a selected resource, compute its direct typed dependencies, print them, and ask whether to include each missing dependency. If any required dependency is refused, print why and omit the parent selection. Run `resolve_settings` before returning.

Do not add a terminal UI dependency; `BufRead`/`Write` makes the interview deterministic and testable.

- [ ] **Step 4: Verify and commit the interview**

Run:

```bash
cargo fmt --all
cargo test --test interview
cargo clippy --all-targets -- -D warnings
```

Expected: transcript and dependency-confirmation tests pass.

Commit:

```bash
git add src/lib.rs src/interview.rs tests/interview.rs
git commit -m "feat: add manifest-driven initialization interview"
```

---

### Task 6: Add CLI commands and JSON Schema output

**Files:**

- Create: `src/main.rs`
- Create: `tests/cli.rs`

**Interfaces:**

- Consumes: all Task 1–5 library APIs.
- Produces: `maestro check`, `maestro resolve`, `maestro init`, and `maestro schema`.

- [ ] **Step 1: Write failing CLI smoke tests**

Create `tests/cli.rs` using `env!("CARGO_BIN_EXE_maestro")` and `std::process::Command`:

```rust
#[test]
fn check_accepts_valid_fixture() {
    let output = Command::new(env!("CARGO_BIN_EXE_maestro"))
        .args(["check", "tests/fixtures/valid/maestro.yaml"])
        .output().unwrap();
    assert!(output.status.success(), "{}", String::from_utf8_lossy(&output.stderr));
    assert!(String::from_utf8_lossy(&output.stdout).contains("valid"));
}

#[test]
fn schema_emits_valid_json() {
    let output = Command::new(env!("CARGO_BIN_EXE_maestro"))
        .args(["schema", "settings"])
        .output().unwrap();
    assert!(output.status.success());
    let _: serde_json::Value = serde_json::from_slice(&output.stdout).unwrap();
}
```

Add tests for nonzero invalid-manifest status, deterministic `resolve` JSON, and `init --output` with piped answers.

- [ ] **Step 2: Run CLI tests and confirm failure**

Run:

```bash
cargo test --test cli
```

Expected: binary compilation fails because `src/main.rs` is absent.

- [ ] **Step 3: Implement the Clap command surface**

Define:

```rust
#[derive(Parser)]
#[command(name = "maestro", version, about)]
struct Cli { #[command(subcommand)] command: Command }

#[derive(Subcommand)]
enum Command {
    Check { #[arg(default_value = "maestro.yaml")] manifest: PathBuf },
    Resolve {
        #[arg(default_value = "maestro.yaml")] manifest: PathBuf,
        #[arg(long)] settings: Option<PathBuf>,
        #[arg(long)] output: Option<PathBuf>,
    },
    Init {
        #[arg(default_value = "maestro.yaml")] manifest: PathBuf,
        #[arg(long, default_value = ".maestro/config/settings.json")] output: PathBuf,
    },
    Schema { target: SchemaTarget },
}

#[derive(Clone, ValueEnum)]
enum SchemaTarget { Manifest, Resource, Settings }
```

Behavior:

- `check`: load and print `valid: N resources`.
- `resolve`: load; optionally load and apply settings; serialize pretty JSON to stdout or atomically to `--output`.
- `init`: load, run interview over locked stdin/stdout, preview pretty settings JSON, ask `Write settings? [Y/n]`, then call `write_settings_atomic`.
- `schema`: call `schemars::schema_for!` for `RootManifest`, `ResourceDocument`, or `Settings` and print pretty JSON.
- `main`: print the complete source-aware error chain to stderr and exit `1` on failure.

- [ ] **Step 4: Verify CLI behavior**

Run:

```bash
cargo fmt --all
cargo test --test cli
cargo run -- check tests/fixtures/valid/maestro.yaml
cargo run -- schema resource | python3 -m json.tool >/dev/null
cargo clippy --all-targets -- -D warnings
```

Expected: CLI tests pass, check prints a valid resource count, schema parses as JSON, and Clippy is clean.

- [ ] **Step 5: Commit CLI and schemas**

```bash
git add src/main.rs tests/cli.rs
git commit -m "feat: add Maestro manifest CLI"
```

---

### Task 7: Populate the inner-source catalog and CI gate

**Files:**

- Create: `maestro.yaml`
- Create: one YAML file under `manifests/` for each supported resource kind
- Create: `examples/settings.json`
- Create: `README.md`
- Create: `.github/workflows/ci.yml`
- Modify: `tests/fixtures/valid/` to include the instruction and skill referenced by its reviewer agent

**Interfaces:**

- Consumes: public schemas and CLI from Tasks 1–6.
- Produces: contributor-ready repository, valid example catalog, and pull-request verification.

- [ ] **Step 1: Add a failing repository-catalog smoke test**

Append to `tests/cli.rs`:

```rust
#[test]
fn repository_manifest_is_valid() {
    let output = Command::new(env!("CARGO_BIN_EXE_maestro"))
        .args(["check", "maestro.yaml"])
        .output().unwrap();
    assert!(output.status.success(), "{}", String::from_utf8_lossy(&output.stderr));
}
```

Run:

```bash
cargo test --test cli repository_manifest_is_valid
```

Expected: failure because `maestro.yaml` does not exist.

- [ ] **Step 2: Create the root catalog and coherent example resources**

Create root `maestro.yaml`:

```yaml
version: 1
metadata:
  name: maestro-community
  description: Shared Maestro resources and governance
discover:
  root: manifests
  recursive: true
imports: []
```

Create one small resource per kind with references forming a valid chain:

- `instructions/reviewer-system.yaml`: inline reviewer instruction.
- `references/rust-style.yaml`: repository-relative `README.md` reference.
- `skills/code-review.yaml`: uses the instruction and reference.
- `agents/reviewer.yaml`: uses `reviewer-system` and `code-review`, activation `selectable`.
- `agents/implementer.yaml`: minimal write-capable selectable agent.
- `handoffs/implementation-to-review.yaml`: implementer to reviewer.
- `workflows/safe-development.yaml`: implementer step then handoff step then reviewer step.
- `addons/example-addon.yaml`: inert documented `maestro-addon` entry point.
- `extensions/audit-events.yaml`: inert documented entry point for `before-tool` and `after-tool`.
- `injections/review-context.yaml`: injects `reviewer-system` at `before-response`.
- `block-commands/destructive-shell.yaml`: mandatory patterns `rm -rf /` and `git push --force`, each represented as literal patterns, with a reason.
- `policies/protect-secrets.yaml`: mandatory deny policy targeting `before-tool` with `rules.categories: [credentials, private-keys]`.

The executable examples are declarations only; README must state this crate does not execute them.

- [ ] **Step 3: Add example settings and contributor documentation**

Generate `examples/settings.json` through the library shape, selecting reviewer, code-review, safe-development, and all mandatory resources.

Write `README.md` with these exact sections:

1. Purpose and v1 boundary.
2. `cargo run -- check maestro.yaml` quick start.
3. `cargo run -- init maestro.yaml` initialization.
4. Directory tree and one-resource-per-file convention.
5. Resource envelope example.
6. Activation semantics.
7. Local and checksum-pinned HTTPS imports.
8. Contribution workflow: add one file, run format/Clippy/tests/check, open peer-reviewed pull request.
9. Library API example using `load_manifest` and `resolve_settings`.

- [ ] **Step 4: Add pull-request CI**

Create `.github/workflows/ci.yml`:

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt, clippy
      - uses: Swatinem/rust-cache@v2
      - run: cargo fmt --check
      - run: cargo clippy --all-targets --all-features -- -D warnings
      - run: cargo test --all-features
      - run: cargo run -- check maestro.yaml
```

- [ ] **Step 5: Run complete verification**

Run:

```bash
cargo fmt --check
cargo clippy --all-targets --all-features -- -D warnings
cargo test --all-features
cargo run -- check maestro.yaml
cargo run -- resolve maestro.yaml --output /tmp/maestro-resolved.json
python3 -m json.tool /tmp/maestro-resolved.json >/dev/null
```

Expected: every command exits `0`; the repository manifest is valid and resolved output is valid JSON.

- [ ] **Step 6: Commit the contributor-ready scaffold**

```bash
git add maestro.yaml manifests examples README.md .github tests/fixtures/valid tests/cli.rs
git commit -m "docs: add inner-source manifest catalog"
```

---

### Task 8: Final diagnostics and acceptance review

**Files:**

- Modify: only files implicated by diagnostics or failed checks.

**Interfaces:**

- Consumes: complete repository from Tasks 1–7.
- Produces: verified v1 scaffold with no known blocking diagnostics.

- [ ] **Step 1: Run language-server diagnostics on all Rust sources**

Run Pi `lsp_diagnostics` on `src/` and `tests/` with severity `all`. Fix every error and warning attributable to the new project, then rerun until clean.

- [ ] **Step 2: Run cached session diagnostics**

Run Pi `lens_diagnostics` with `mode=all`. Resolve every blocking finding in edited files and rerun until no blocking errors remain.

- [ ] **Step 3: Run the acceptance command set from a clean shell**

```bash
cargo fmt --check && \
cargo clippy --all-targets --all-features -- -D warnings && \
cargo test --all-features && \
cargo run -- check maestro.yaml
```

Expected: all commands exit `0` without ignored failures.

- [ ] **Step 4: Inspect repository state and commit final corrections**

```bash
git status --short
git diff --check
```

Expected: `git diff --check` exits `0`. Commit any verification corrections:

```bash
git add -A
git commit -m "fix: satisfy Maestro manifest acceptance checks"
```

If no corrections were required, do not create an empty commit.
