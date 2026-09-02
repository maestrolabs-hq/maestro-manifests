# Workspace Settings Boundary Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `maestro-manifests` the definitive producer and validator of portable workspace settings while reserving the installed `maestro` executable and the user-local workspace trust store for `maestro-core`.

**Architecture:** Extend the existing typed settings model with a validated workspace identity, a lexically ordered set of normalized direct-child repository names, and the fixed one-GiB per-operation artifact budget. Represent the manifest source as a validated relative path and resolve it only against an explicit workspace root. Rename the contributor binary to `maestro-manifests`. Centralize settings creation, parsing, schema generation, and durable replacement in the library so the contributor CLI and the future pinned `maestro-core` consumer share one contract. Use a reviewed atomic-write dependency for platform replacement primitives, sync the replacement before promotion, sync the containing directory on Unix, and refuse replacement when the platform primitive cannot provide the contract. Do not add trust records or trust commands here.

**Tech Stack:** Stable Rust 2024, existing serde/serde_json/schemars/clap stack, `atomicwrites = 0.4.4` for platform-specific atomic replacement, tempfile for filesystem tests, and GitHub Actions native `ubuntu-latest`, `macos-latest`, and `windows-latest` runners.

**Spec:**

- `docs/superpowers/specs/2026-09-01-maestro-manifests-design.md`
- `../maestro-core/docs/superpowers/specs/2026-09-01-workspace-intelligence-facades.md`

## Global Constraints

- Execute this plan on the existing `feature/maestro-v1` implementation branch (or its reviewed successor after merge); the current `main` branch contains the design but not the listed Rust files. Verify `Cargo.toml`, `src/settings.rs`, and `tests/validation.rs` exist before Task 1. Preserve the clean implementation worktree and use logical repository paths in commits.
- `maestro-core` owns the only installed executable named `maestro`; this repository's contributor binary is exactly `maestro-manifests`.
- `.maestro/config/settings.json` remains the single workspace marker and the sole local repository-membership list.
- Settings contain `workspace.id`, `workspace.repositories`, and `workspace.max_artifact_bytes_per_operation` with the exact value `1_073_741_824`.
- Repository names are unique normalized direct-child names. Reject empty names, absolute paths, separators, `.` and `..`, percent-encoded spellings, Windows prefixes, and any non-normalized component. Serialize them in lexical order.
- `manifest.source` is relative-only, is serialized with `/` separators, and always resolves from the explicit workspace root. It never resolves from the current process directory or `.maestro/config/`.
- Settings parsing, construction, interview output, and `resolve_settings` all run the same semantic validation. Deserialization alone must not create an accepted invalid `Settings` value.
- Existing resource selection, activation, dependency, and override behavior remains unchanged.
- Atomic replacement stages complete bytes in a same-parent temporary directory, flushes and syncs them, and then uses the platform overwrite primitive. Unix syncs the containing directory after promotion. Windows requests replace-existing and write-through semantics. Validation, serialization, write, flush, and temporary-file sync failures reported before publication preserve the previous destination bytes. The dependency reports its internal staging, publication, and Unix parent-sync failures through one opaque error path, so an `Error::Io` from that primitive or from the subsequent explicit parent sync is an indeterminate durability state: old or new destination bytes may be visible or durable. Callers must reread and validate the destination before retrying; do not promise rollback after publication.
- The trust store remains entirely core-owned. Do not add a trust-store model, platform data-directory lookup, settings digest, `workspace trust`, `workspace status`, or `workspace revoke` command to this repository.
- Use logical repository paths in changes and commits; do not refer to `.worktrees/maestro-v1` in implementation work.
- Every production step starts with a focused test that is observed RED for the stated missing behavior, then ends with the focused GREEN command and a conventional commit.
- Run the complete formatting, Clippy, test, catalog, schema, and native CI checks before completion. Every external action is pinned to a reviewed full commit SHA; do not use a tag, branch, or mutable toolchain action reference. Do not weaken `unsafe_code = "forbid"` or another gate.

## File Map

- `Cargo.toml`: rename the contributor binary and add the atomic replacement dependency.
- `Cargo.lock`: lock the reviewed atomic replacement dependency.
- `src/main.rs`: expose the `maestro-manifests` contributor command and make `init` workspace-rooted.
- `src/settings.rs`: own validated workspace settings, relative manifest source, shared validation, resolution, and durable JSON replacement.
- `src/interview.rs`: collect workspace identity and repository membership and construct settings through the validated library interface.
- `src/lib.rs`: continue re-exporting the stable settings interface used by `maestro-core`.
- `tests/cli.rs`: assert the definitive binary name and workspace-rooted initialization behavior.
- `tests/interview.rs`: assert exact interview-produced workspace settings.
- `tests/validation.rs`: assert settings semantics, relative source resolution, ordering, and serialization.
- `tests/atomic_settings.rs`: exercise create, overwrite, interrupted-write preservation, cleanup, and platform durability behavior.
- `examples/settings.json`: document the approved complete settings shape.
- `README.md`: document contributor naming, workspace initialization, and the library boundary with core-owned trust.
- `.github/workflows/ci.yml`: run the full gate on native Linux, macOS, and Windows runners.

---

### Task 1: Rename the contributor binary definitively

**Files:**

- Modify: `Cargo.toml`
- Modify: `src/main.rs`
- Modify: `tests/cli.rs`
- Modify: `README.md`

**Interfaces:**

- Consumes: the approved one-installed-CLI ownership rule.
- Produces: only `CARGO_BIN_EXE_maestro-manifests` from this package and contributor help headed `maestro-manifests`.

- [ ] **Step 1: Write the failing binary-ownership test**

At the top of `tests/cli.rs`, replace the helper and every direct old binary lookup with:

```rust
fn maestro_manifests() -> Command {
    Command::new(env!("CARGO_BIN_EXE_maestro-manifests"))
}
```

Add:

```rust
#[test]
fn contributor_binary_has_the_reserved_name() {
    let output = maestro_manifests().arg("--help").output().unwrap();
    assert!(output.status.success());
    let stdout = String::from_utf8(output.stdout).unwrap();
    assert!(stdout.starts_with("Governed manifest catalog and configuration CLI for Maestro\n"));
    assert!(stdout.contains("Usage: maestro-manifests <COMMAND>"));
}
```

- [ ] **Step 2: Run RED**

Run:

```bash
cargo test --test cli contributor_binary_has_the_reserved_name
```

Expected: compilation fails because Cargo does not define `CARGO_BIN_EXE_maestro-manifests` while the target is still named `maestro`.

- [ ] **Step 3: Rename the Cargo target and Clap command**

Change the binary stanza in `Cargo.toml` to:

```toml
[[bin]]
name = "maestro-manifests"
path = "src/main.rs"
```

Change the parser declaration in `src/main.rs` to:

```rust
#[derive(Parser)]
#[command(name = "maestro-manifests", version, about)]
struct Cli {
    #[command(subcommand)]
    command: Command,
}
```

Rename all `maestro()` calls in `tests/cli.rs` to `maestro_manifests()` and replace every `env!("CARGO_BIN_EXE_maestro")` with `env!("CARGO_BIN_EXE_maestro-manifests")`.

Update contributor commands in `README.md` to use:

```bash
cargo run --bin maestro-manifests -- check maestro.yaml
cargo run --bin maestro-manifests -- init --workspace .
```

Add this ownership statement immediately before the quick start:

```markdown
The contributor executable is `maestro-manifests`. The installed `maestro`
executable and all workspace trust commands are owned by `maestro-core`.
```

- [ ] **Step 4: Run GREEN and prove the old target is absent**

Run:

```bash
cargo test --test cli contributor_binary_has_the_reserved_name
cargo metadata --no-deps --format-version 1
```

Expected: the test passes, and the package target list contains a binary named `maestro-manifests` and no binary named `maestro`.

- [ ] **Step 5: Commit**

```bash
git add Cargo.toml src/main.rs tests/cli.rs README.md
git commit -m "fix: reserve maestro name for core cli"
```

---

### Task 2: Add the validated workspace settings contract

**Files:**

- Modify: `src/settings.rs`
- Modify: `tests/validation.rs`

**Interfaces:**

- Consumes: version-1 settings JSON.
- Produces:
  - `MAX_ARTIFACT_BYTES_PER_OPERATION: u64 = 1_073_741_824`.
  - `WorkspaceSettings { id, repositories, max_artifact_bytes_per_operation }`.
  - `ManifestSource` proving a normalized relative path.
  - `Settings::new(manifest_source, workspace)` and `Settings::validate()`.
  - `ManifestSource::resolve_from(workspace_root)` with no current-directory dependency.

- [ ] **Step 1: Write failing model and rejection tests**

Append to `tests/validation.rs`:

```rust
use std::path::Path;
use maestro_manifests::{
    ManifestSource, MAX_ARTIFACT_BYTES_PER_OPERATION, Settings, WorkspaceSettings,
};

fn approved_workspace() -> WorkspaceSettings {
    WorkspaceSettings::new(
        "maestro-labs",
        ["maestro-manifests", "dot-github", "maestro-core"],
        MAX_ARTIFACT_BYTES_PER_OPERATION,
    )
    .unwrap()
}

#[test]
fn workspace_settings_are_normalized_and_lexically_ordered() {
    let settings = Settings::new(
        ManifestSource::new("catalogs/maestro.yaml").unwrap(),
        approved_workspace(),
    );
    assert_eq!(settings.workspace.id(), "maestro-labs");
    assert_eq!(
        settings.workspace.repositories(),
        ["dot-github", "maestro-core", "maestro-manifests"]
    );
    assert_eq!(
        settings.workspace.max_artifact_bytes_per_operation(),
        1_073_741_824
    );
    assert_eq!(
        settings.manifest.source.resolve_from(Path::new("workspace")),
        Path::new("workspace/catalogs/maestro.yaml")
    );
}

#[test]
fn manifest_source_rejects_every_non_relative_or_non_normalized_form() {
    let drive_slash = ["C", ":/maestro.yaml"].concat();
    let drive_backslash = ["C", ":\\maestro.yaml"].concat();
    for source in [
        "", ".", "..", "../maestro.yaml", "/maestro.yaml",
        "catalogs/../maestro.yaml", "catalogs//maestro.yaml",
        "catalogs\\maestro.yaml", drive_slash.as_str(), drive_backslash.as_str(),
    ] {
        assert!(ManifestSource::new(source).is_err(), "accepted {source:?}");
    }
}

#[test]
fn workspace_rejects_invalid_identity_repository_and_budget() {
    for id in ["", ".", "..", "Maestro Labs", "maestro/labs", "maestro%2flabs"] {
        assert!(WorkspaceSettings::new(id, ["maestro-core"], 1_073_741_824).is_err());
    }
    let drive_relative = ["C", ":maestro-core"].concat();
    for repository in [
        "", ".", "..", "../maestro-core", "/maestro-core", "team/maestro-core",
        "team\\maestro-core", drive_relative.as_str(), "maestro%2fcore",
    ] {
        assert!(WorkspaceSettings::new("maestro-labs", [repository], 1_073_741_824).is_err());
    }
    assert!(WorkspaceSettings::new("maestro-labs", ["core", "core"], 1_073_741_824).is_err());
    assert!(WorkspaceSettings::new("maestro-labs", ["core"], 1_073_741_823).is_err());
}

#[test]
fn load_settings_runs_semantic_validation() {
    let fixture = tempfile::tempdir().unwrap();
    let path = fixture.path().join("settings.json");
    fs::write(
        &path,
        br#"{"version":1,"manifest":{"source":"../maestro.yaml"},"workspace":{"id":"maestro-labs","repositories":["core"],"max_artifact_bytes_per_operation":1073741824}}"#,
    )
    .unwrap();
    assert!(load_settings(path).is_err());
}
```

- [ ] **Step 2: Run RED**

Run:

```bash
cargo test --test validation workspace_settings_are_normalized_and_lexically_ordered
cargo test --test validation manifest_source_rejects_every_non_relative_or_non_normalized_form
cargo test --test validation workspace_rejects_invalid_identity_repository_and_budget
cargo test --test validation load_settings_runs_semantic_validation
```

Expected: compilation fails because `ManifestSource`, `WorkspaceSettings`, and the artifact-budget constant do not exist.

- [ ] **Step 3: Implement parsed types that make invalid states unavailable**

In `src/settings.rs`, add these public contracts:

```rust
pub const MAX_ARTIFACT_BYTES_PER_OPERATION: u64 = 1_073_741_824;

#[derive(Debug, Clone, PartialEq, Eq, Serialize, JsonSchema)]
#[serde(transparent)]
pub struct ManifestSource(String);

impl ManifestSource {
    pub fn new(source: impl Into<String>) -> Result<Self, Error> {
        let source = source.into();
        validate_relative_manifest_source(&source)?;
        Ok(Self(source))
    }

    pub fn as_str(&self) -> &str { &self.0 }

    pub fn resolve_from(&self, workspace_root: &Path) -> PathBuf {
        workspace_root.join(self.0.split('/').collect::<PathBuf>())
    }
}

impl<'de> Deserialize<'de> for ManifestSource {
    fn deserialize<D: Deserializer<'de>>(deserializer: D) -> Result<Self, D::Error> {
        let source = String::deserialize(deserializer)?;
        Self::new(source).map_err(serde::de::Error::custom)
    }
}

#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
#[serde(deny_unknown_fields)]
pub struct WorkspaceSettings {
    id: String,
    repositories: BTreeSet<String>,
    max_artifact_bytes_per_operation: u64,
}
```

Implement `WorkspaceSettings::new`, accessors, and a private `validate`. Workspace IDs and repository names use `^[a-z0-9][a-z0-9._-]*$`; repository validation additionally rejects `%`, `/`, `\\`, `:`, duplicates, `.` and `..`. Require at least one repository and the exact budget constant. Implement custom `Deserialize` through a private wire struct so every deserialized value passes `WorkspaceSettings::new`.

Change the settings fields to:

```rust
pub struct Settings {
    #[schemars(range(min = 1, max = 1))]
    pub version: u32,
    pub manifest: ManifestSelection,
    pub workspace: WorkspaceSettings,
    #[serde(default)]
    pub preferences: Preferences,
    #[serde(default, with = "enabled_resources")]
    #[schemars(with = "EnabledResources")]
    pub enabled: BTreeMap<ResourceKind, BTreeSet<String>>,
    #[serde(default)]
    pub overrides: BTreeMap<ResourceKind, BTreeMap<String, BTreeMap<String, Value>>>,
}

pub struct ManifestSelection {
    pub source: ManifestSource,
}
```

Replace `Settings::default()` with an explicit constructor:

```rust
impl Settings {
    pub fn new(source: ManifestSource, workspace: WorkspaceSettings) -> Self {
        Self {
            version: 1,
            manifest: ManifestSelection { source },
            workspace,
            preferences: Preferences::default(),
            enabled: BTreeMap::new(),
            overrides: BTreeMap::new(),
        }
    }

    pub fn validate(&self) -> Result<(), Error> {
        if self.version != 1 {
            return Err(Error::UnsupportedVersion(self.version));
        }
        self.workspace.validate()
    }
}
```

Make `load_settings` deserialize and then call `validate`. Call `settings.validate()` first in `resolve_settings` and `write_settings_atomic`. Update existing tests to construct settings with `Settings::new(ManifestSource::new("maestro.yaml").unwrap(), approved_workspace())`; do not restore a context-free default.

- [ ] **Step 4: Run GREEN**

Run:

```bash
cargo test --test validation
cargo clippy --lib -- -D warnings
```

Expected: all settings, selection, override, path, ordering, and rejection tests pass; Clippy reports no warnings.

- [ ] **Step 5: Commit**

```bash
git add src/settings.rs tests/validation.rs
git commit -m "feat: add governed workspace settings"
```

---

### Task 3: Root initialization and manifest loading at the workspace

**Files:**

- Modify: `src/settings.rs`
- Modify: `src/interview.rs`
- Modify: `src/main.rs`
- Modify: `tests/interview.rs`
- Modify: `tests/cli.rs`

**Interfaces:**

- Consumes: an explicit workspace root, relative manifest source, workspace ID, and comma-separated direct-child repository names.
- Produces:
  - `load_workspace_settings(workspace_root)` for core consumers.
  - `load_workspace_manifest(workspace_root, source)` for contributor initialization.
  - settings written only to `<workspace>/.maestro/config/settings.json`.

- [ ] **Step 1: Write failing workspace-root and interview tests**

Add to `tests/interview.rs`:

```rust
#[test]
fn interview_records_workspace_authority_and_exact_budget() {
    let (_fixture, manifest) = interview_fixture(&[]);
    let input = b"maestro-labs\nmaestro-core, dot-github\n\n\n";
    let mut output = Vec::new();
    let settings = maestro_manifests::run_interview(
        &manifest,
        maestro_manifests::ManifestSource::new("maestro.yaml").unwrap(),
        &input[..],
        &mut output,
    )
    .unwrap();
    assert_eq!(settings.workspace.id(), "maestro-labs");
    assert_eq!(settings.workspace.repositories(), ["dot-github", "maestro-core"]);
    assert_eq!(settings.workspace.max_artifact_bytes_per_operation(), 1_073_741_824);
}
```

Add to `tests/cli.rs`:

```rust
#[test]
fn init_resolves_manifest_from_workspace_and_writes_the_single_marker() {
    let fixture = tempdir().unwrap();
    fs::create_dir(fixture.path().join("catalog/manifests")).unwrap();
    fs::write(
        fixture.path().join("catalog/maestro.yaml"),
        "version: 1\nmetadata:\n  name: test\ndiscover:\n  root: manifests\n",
    )
    .unwrap();
    let mut child = maestro_manifests()
        .args(["init", "--workspace"])
        .arg(fixture.path())
        .args(["--manifest", "catalog/maestro.yaml"])
        .current_dir(fixture.path().parent().unwrap())
        .stdin(Stdio::piped())
        .stdout(Stdio::piped())
        .spawn()
        .unwrap();
    child.stdin.take().unwrap().write_all(
        b"maestro-labs\nmaestro-core\nrust\nenglish\ny\n",
    ).unwrap();
    let output = child.wait_with_output().unwrap();
    assert!(output.status.success(), "{}", String::from_utf8_lossy(&output.stderr));
    let marker = fixture.path().join(".maestro/config/settings.json");
    let settings = load_settings(marker).unwrap();
    assert_eq!(settings.manifest.source.as_str(), "catalog/maestro.yaml");
    assert_eq!(settings.workspace.repositories(), ["maestro-core"]);
}

#[test]
fn init_rejects_absolute_manifest_before_reading_it() {
    let fixture = tempdir().unwrap();
    let output = maestro_manifests()
        .args(["init", "--workspace"])
        .arg(fixture.path())
        .arg("--manifest")
        .arg(fixture.path().join("maestro.yaml"))
        .output()
        .unwrap();
    assert!(!output.status.success());
    assert!(String::from_utf8_lossy(&output.stderr).contains("relative to the workspace root"));
}
```

- [ ] **Step 2: Run RED**

Run:

```bash
cargo test --test interview interview_records_workspace_authority_and_exact_budget
cargo test --test cli init_resolves_manifest_from_workspace_and_writes_the_single_marker
cargo test --test cli init_rejects_absolute_manifest_before_reading_it
```

Expected: compilation fails because the interview still accepts a string source and `init` has neither `--workspace` nor `--manifest`.

- [ ] **Step 3: Implement the workspace-rooted library and CLI flow**

Add to `src/settings.rs`:

```rust
pub const SETTINGS_RELATIVE_PATH: &str = ".maestro/config/settings.json";

pub fn load_workspace_settings(workspace_root: impl AsRef<Path>) -> Result<Settings, Error> {
    load_settings(workspace_root.as_ref().join(SETTINGS_RELATIVE_PATH))
}

pub fn load_workspace_manifest(
    workspace_root: impl AsRef<Path>,
    source: &ManifestSource,
) -> Result<ResolvedManifest, Error> {
    load_manifest(source.resolve_from(workspace_root.as_ref()))
}
```

Change the init command in `src/main.rs` to:

```rust
Init {
    #[arg(long, default_value = ".")]
    workspace: PathBuf,
    #[arg(long, default_value = "maestro.yaml")]
    manifest: String,
},
```

In its handler, parse `ManifestSource::new(manifest)` before filesystem access, load with `load_workspace_manifest(&workspace, &source)`, pass the typed source to `run_interview`, and write only `workspace.join(SETTINGS_RELATIVE_PATH)`. Keep the preview and confirmation. Remove the arbitrary `--output` path from `init`; `resolve --output` remains unchanged.

Change `run_interview` to accept `source: ManifestSource`. Before preference questions, prompt exactly:

```rust
write!(output, "Workspace ID: ").map_err(io_error)?;
let workspace_id = read_line(&mut input)?;
write!(output, "Repositories (comma-separated direct-child names): ").map_err(io_error)?;
let repositories = read_line(&mut input)?
    .split(',')
    .map(str::trim)
    .filter(|value| !value.is_empty())
    .map(str::to_owned)
    .collect::<Vec<_>>();
let workspace = WorkspaceSettings::new(
    workspace_id.trim(),
    repositories,
    MAX_ARTIFACT_BYTES_PER_OPERATION,
)?;
let mut settings = Settings::new(source, workspace);
```

Keep resource questions and exact-choice behavior unchanged. Invalid workspace input returns a source-aware validation error and writes nothing.

- [ ] **Step 4: Run GREEN**

Run:

```bash
cargo test --test interview
cargo test --test cli
cargo clippy --all-targets --all-features -- -D warnings
```

Expected: interview tests preserve resource behavior, CLI tests prove workspace-rooted source resolution, and absolute sources fail before access.

- [ ] **Step 5: Commit**

```bash
git add src/settings.rs src/interview.rs src/main.rs tests/interview.rs tests/cli.rs
git commit -m "feat: root initialization at workspace settings"
```

---

### Task 4: Make settings overwrite durable on each supported platform

**Files:**

- Modify: `Cargo.toml`
- Modify: `Cargo.lock`
- Modify: `src/settings.rs`
- Create: `tests/atomic_settings.rs`

**Interfaces:**

- Consumes: validated serializable settings and the fixed workspace marker path.
- Produces: create-or-replace semantics that never expose partial JSON and do not silently weaken overwrite guarantees.

- [ ] **Step 1: Write failing create, overwrite, interruption, and cleanup tests**

Create `tests/atomic_settings.rs`:

```rust
use std::fs;
use maestro_manifests::{
    ManifestSource, Settings, WorkspaceSettings, MAX_ARTIFACT_BYTES_PER_OPERATION,
    write_settings_atomic,
};

fn settings(id: &str) -> Settings {
    Settings::new(
        ManifestSource::new("maestro.yaml").unwrap(),
        WorkspaceSettings::new(id, ["maestro-core"], MAX_ARTIFACT_BYTES_PER_OPERATION).unwrap(),
    )
}

#[test]
fn atomic_write_creates_and_replaces_complete_settings() {
    let fixture = tempfile::tempdir().unwrap();
    let path = fixture.path().join(".maestro/config/settings.json");
    write_settings_atomic(&path, &settings("first")).unwrap();
    write_settings_atomic(&path, &settings("second")).unwrap();
    let bytes = fs::read(&path).unwrap();
    assert_eq!(bytes.last(), Some(&b'\n'));
    let value: serde_json::Value = serde_json::from_slice(&bytes).unwrap();
    assert_eq!(value["workspace"]["id"], "second");
    assert_eq!(fs::read_dir(path.parent().unwrap()).unwrap().count(), 1);
}

#[test]
fn serialization_failure_preserves_existing_bytes() {
    let fixture = tempfile::tempdir().unwrap();
    let path = fixture.path().join("settings.json");
    fs::write(&path, b"previous\n").unwrap();
    let result = maestro_manifests::write_json_atomic(&path, &FailingSerialize);
    assert!(result.is_err());
    assert_eq!(fs::read(&path).unwrap(), b"previous\n");
    assert_eq!(fs::read_dir(fixture.path()).unwrap().count(), 1);
}

struct FailingSerialize;

impl serde::Serialize for FailingSerialize {
    fn serialize<S>(&self, _serializer: S) -> Result<S::Ok, S::Error>
    where S: serde::Serializer {
        Err(serde::ser::Error::custom("injected serialization failure"))
    }
}
```

The first test is intentionally required on Windows, where the existing `std::fs::rename` overwrite fails. The second proves that a pre-publication serialization failure preserves the prior file and that ordinary RAII cleanup removes the same-parent temporary directory. Cleanup failure can leave unselected staging residue; that residue is non-authoritative and must never be treated as workspace settings.

- [ ] **Step 2: Run RED on native runners**

Run locally:

```bash
cargo test --test atomic_settings
```

Push the test-only commit to a draft branch and run the same command on `windows-latest` before implementation. Expected: the replacement test fails on Windows because `std::fs::rename` does not replace the existing destination. On Unix, record RED by first adding the injected serialization hook test against the new `AtomicSettingsWriter` seam; it fails to compile because that seam does not exist. Do not accept an all-green test-only change as RED evidence.

- [ ] **Step 3: Add the reviewed atomic writer without repository-owned unsafe code**

Add to `Cargo.toml`:

```toml
atomicwrites = "=0.4.4"
```

Replace the local PID/counter temporary-file implementation with a private `AtomicSettingsWriter` wrapping `atomicwrites::AtomicFile` configured with `OverwriteBehavior::AllowOverwrite`. `AtomicFile` stages `tmpfile.tmp` inside a randomized temporary directory beneath the destination parent, keeping publication on the same filesystem. In the `AtomicFile::write` closure, write pretty JSON and the trailing newline, flush, and call `sync_all` before publication. Map closure write, flush, and sync failures to destination-aware `Error::Io`, preserve JSON serialization errors as `Error::Json`, and map the dependency's opaque internal staging/publication error to destination-aware `Error::Io`. Because that internal error does not identify whether publication occurred, callers must treat it as an indeterminate old-or-new state and reread and validate the destination before retrying. Do not invent a more specific error variant without a primitive that reports the commit stage honestly. The reviewed `0.4.4` Windows source implements overwrite through `MoveFileExW` with `MOVEFILE_REPLACE_EXISTING | MOVEFILE_WRITE_THROUGH`; pin that exact version and test the behavior natively rather than adding repository-owned unsafe code.

After successful commit, sync the containing directory on Unix:

```rust
#[cfg(unix)]
fn sync_parent_directory(parent: &Path) -> Result<(), Error> {
    fs::File::open(parent)
        .and_then(|directory| directory.sync_all())
        .map_err(|error| io_error(parent, &error))
}

#[cfg(windows)]
fn sync_parent_directory(_parent: &Path) -> Result<(), Error> {
    Ok(())
}
```

On Windows, directory durability comes from the reviewed replacement implementation's write-through flag; do not claim directory `sync_all`. Do not fall back to remove-then-rename and do not relax `unsafe_code = "forbid"`.

Use `atomicwrites`' same-parent temporary directories and ordinary `TempDir` RAII cleanup after serialization, flush, sync, or publication failure. Test that ordinary failures leave only the destination. Do not claim cleanup cannot fail: any leftover `.atomicwrite*` staging directory is unselected and non-authoritative because only the fixed destination path is workspace settings.

- [ ] **Step 4: Run GREEN on Linux, macOS, and Windows**

Run on each native runner:

```bash
cargo test --test atomic_settings
cargo test --test validation atomic_write_replaces_existing_file
```

Expected on all three operating systems: create, overwrite, complete JSON, trailing newline, pre-publication prior-byte preservation, and ordinary temporary cleanup tests pass. On Unix, the success path reaches parent-directory sync. On Windows, overwrite succeeds without deleting the destination first. A publication or post-publication parent-sync error never triggers remove-then-rename or rollback; it requires rereading and validating the fixed destination before retry.

- [ ] **Step 5: Commit**

```bash
git add Cargo.toml Cargo.lock src/settings.rs tests/atomic_settings.rs
git commit -m "fix: make settings replacement platform durable"
```

---

### Task 5: Publish the schema, example, documentation, and library boundary

**Files:**

- Modify: `tests/cli.rs`
- Modify: `examples/settings.json`
- Modify: `README.md`
- Modify: `src/lib.rs`

**Interfaces:**

- Consumes: the validated settings types from Tasks 2-4.
- Produces: one JSON Schema and example accepted by the library, plus a documented core integration that reads but does not implement trust.

- [ ] **Step 1: Write the failing schema and public-interface assertions**

Extend `emitted_schemas_validate_v1_fixtures` in `tests/cli.rs`:

```rust
assert_eq!(valid_settings["workspace"]["id"], "maestro-labs");
assert_eq!(
    valid_settings["workspace"]["max_artifact_bytes_per_operation"],
    1_073_741_824u64
);
for invalid in [
    json!({
        "version": 1,
        "manifest": { "source": "/maestro.yaml" },
        "workspace": {
            "id": "maestro-labs",
            "repositories": ["maestro-core"],
            "max_artifact_bytes_per_operation": 1_073_741_824u64
        }
    }),
    json!({
        "version": 1,
        "manifest": { "source": "maestro.yaml" },
        "workspace": {
            "id": "maestro-labs",
            "repositories": ["maestro-core", "maestro-core"],
            "max_artifact_bytes_per_operation": 1_073_741_824u64
        }
    }),
    json!({
        "version": 1,
        "manifest": { "source": "maestro.yaml" },
        "workspace": {
            "id": "maestro-labs",
            "repositories": ["maestro-core"],
            "max_artifact_bytes_per_operation": 1
        }
    }),
] {
    assert_schema_rejects(&settings_schema, &invalid);
}
```

Add a compile-time public API use in `tests/validation.rs`:

```rust
#[test]
fn core_facing_library_interface_loads_from_workspace_root() {
    let fixture = tempfile::tempdir().unwrap();
    let marker = fixture.path().join(maestro_manifests::SETTINGS_RELATIVE_PATH);
    write_settings_atomic(&marker, &Settings::new(
        ManifestSource::new("maestro.yaml").unwrap(),
        approved_workspace(),
    )).unwrap();
    let bytes = std::fs::read(&marker).unwrap();
    let parsed = maestro_manifests::parse_settings(&bytes).unwrap();
    let loaded = maestro_manifests::load_workspace_settings(fixture.path()).unwrap();
    assert_eq!(parsed, loaded);
    assert_eq!(loaded.workspace.id(), "maestro-labs");
}
```

- [ ] **Step 2: Run RED**

Run:

```bash
cargo test --test cli emitted_schemas_validate_v1_fixtures
cargo test --test validation core_facing_library_interface_loads_from_workspace_root
```

Expected: the old example has no `workspace` section, its schema does not enforce the approved invariants, or the core-facing workspace-root loader is not exported.

- [ ] **Step 3: Update the exact example and documentation**

Insert this object after `manifest` in `examples/settings.json`:

```json
"workspace": {
  "id": "maestro-labs",
  "repositories": [
    "dot-github",
    "maestro-core",
    "maestro-manifests"
  ],
  "max_artifact_bytes_per_operation": 1073741824
},
```

Ensure schemars emits:

- `manifest.source` as a string constrained to the normalized relative syntax;
- `workspace.id` with `^[a-z0-9][a-z0-9._-]*$`;
- at least one unique repository matching the direct-child-name syntax;
- `max_artifact_bytes_per_operation` with both minimum and maximum `1073741824`;
- `additionalProperties: false` for settings, manifest, and workspace objects.

In `README.md`, document:

```markdown
## Workspace settings and trust boundary

`maestro-manifests init --workspace . --manifest maestro.yaml` writes the single
workspace marker at `.maestro/config/settings.json`. Its manifest source is
relative to the workspace root, and `workspace.repositories` is the sole local
repository-membership list. The fixed v1 artifact budget is 1073741824 bytes per
operation.

The library exports `parse_settings`, `load_workspace_settings`,
`load_workspace_manifest`, `resolve_settings`, `write_settings_atomic`,
`ManifestSource`, `WorkspaceSettings`, and `SETTINGS_RELATIVE_PATH` for a pinned `maestro-core`
consumer. Workspace discovery, exact-byte digesting, user-local trust records,
and `maestro workspace trust|status|revoke` remain owned by `maestro-core`.
```

Add `parse_settings(bytes: &[u8]) -> Result<Settings, Error>` as the single semantic parser used by `load_settings` and `load_workspace_settings`, then keep `src/lib.rs` re-exports explicit enough that all named core-facing items are public. This lets core parse bytes obtained through its no-follow handle instead of reopening a path. Do not add a trust type or command.

- [ ] **Step 4: Run GREEN**

Run:

```bash
cargo test --test cli emitted_schemas_validate_v1_fixtures
cargo test --test validation core_facing_library_interface_loads_from_workspace_root
cargo run --bin maestro-manifests -- schema settings > target/settings.schema.json
cargo run --bin maestro-manifests -- resolve maestro.yaml > target/resolved.json
```

Expected: the example validates, all invalid variants fail schema validation, the public workspace loader compiles and passes, and both emitted JSON files parse successfully.

- [ ] **Step 5: Commit**

```bash
git add src/lib.rs tests/cli.rs tests/validation.rs examples/settings.json README.md
git commit -m "docs: publish workspace settings contract"
```

---

### Task 6: Enforce native cross-platform validation and close the boundary

**Files:**

- Modify: `.github/workflows/ci.yml`
- Modify: `README.md`

**Interfaces:**

- Consumes: the complete contributor binary, settings, schema, atomic writer, and catalog.
- Produces: required native Linux, macOS, and Windows evidence without adding trust-store behavior.

- [ ] **Step 1: Change CI to a native operating-system matrix**

Replace the single test job with:

```yaml
jobs:
  test:
    name: fast / cross-platform (${{ matrix.os }})
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false
      - run: rustup component add rustfmt clippy
      - run: cargo fmt --check
      - run: cargo clippy --all-targets --all-features -- -D warnings
      - run: cargo test --all-features
      - run: cargo run --bin maestro-manifests -- check maestro.yaml
      - run: cargo run --bin maestro-manifests -- schema settings
```

- [ ] **Step 2: Run the complete local gate**

Run:

```bash
cargo fmt --all --check
cargo clippy --all-targets --all-features -- -D warnings
cargo test --all-features
cargo run --bin maestro-manifests -- check maestro.yaml
cargo run --bin maestro-manifests -- schema settings > target/settings.schema.json
cargo metadata --no-deps --format-version 1 > target/metadata.json
git grep -nE 'CARGO_BIN_EXE_maestro([^_-]|$)|\[\[bin\]\][[:space:]]*$' -- Cargo.toml src tests README.md
git grep -nE 'Trust(Store|Binding)|workspace (trust|status|revoke)' -- src tests
git diff --check
```

Expected:

- formatting, Clippy, tests, catalog validation, and schema emission pass;
- `target/settings.schema.json` and `target/metadata.json` are valid JSON;
- metadata exposes `maestro-manifests` and no `maestro` binary;
- the old-binary grep has no old `CARGO_BIN_EXE_maestro` references (inspect the `[[bin]]` hit only to confirm its name);
- the trust-ownership grep has no production or test implementation hits;
- `git diff --check` reports no whitespace errors.

- [ ] **Step 3: Run the required native CI gate**

Push the branch and require all three jobs:

```text
fast / cross-platform (ubuntu-latest)
fast / cross-platform (macos-latest)
fast / cross-platform (windows-latest)
```

Expected: all three pass the same test suite, including `tests/atomic_settings.rs`. Preserve the job name when wiring the repository ruleset so the required contexts match exactly.

- [ ] **Step 4: Request review against the facade boundary**

The reviewer must verify:

- no binary named `maestro` remains in `maestro-manifests`;
- manifest source resolution receives a workspace root explicitly;
- settings parsing and schemas reject invalid workspace membership and budgets;
- overwrite is atomic and durable by the correct native primitive;
- examples, docs, CLI output, and public library types agree; and
- trust storage and trust commands remain absent and core-owned.

Record any blocker with file and line, fix it through a new RED/GREEN cycle, rerun the complete gate, and obtain approval before merge.

- [ ] **Step 5: Commit CI and documentation adjustments**

```bash
git add .github/workflows/ci.yml README.md
git commit -m "ci: test settings durability on native platforms"
```

## Completion Evidence

Before opening the pull request, attach:

- the RED output for each task and the matching focused GREEN output;
- `cargo metadata` evidence showing only the `maestro-manifests` contributor binary;
- settings-schema acceptance and rejection output;
- native Linux, macOS, and Windows job URLs, including atomic overwrite tests;
- the complete `cargo fmt`, Clippy, test, catalog, and diff-check output;
- `git status --short` showing no staged or uncommitted implementation files; and
- reviewer approval confirming the trust store remains core-owned.

## Post-merge library handoff

Core Task 2 cannot begin until the reviewed library is remotely addressable. After the pull request is squash-merged:

```bash
gh repo view maestrolabs-hq/maestro-manifests
git fetch origin main
test "$(cargo metadata --no-deps --format-version 1 | python3 -c 'import json,sys; print(json.load(sys.stdin)["packages"][0]["version"])')" = "0.1.0"
git tag -a v0.1.0 origin/main -m "maestro-manifests v0.1.0"
git push origin v0.1.0
git ls-remote --exit-code origin refs/tags/v0.1.0
```

Expected: the repository exists at the reviewed organization URL, the merged package version is exactly `0.1.0`, and the immutable tag resolves remotely. If the repository is not yet published, that publication is a blocking governance action; do not substitute a sibling path dependency or an unreviewed branch.
