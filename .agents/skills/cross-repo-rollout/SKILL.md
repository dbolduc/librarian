---
name: cross-repo-rollout
description:
  Plans and executes multi-PR configuration and code generation rollouts across
  librarian and downstream client library repositories (such as
  google-cloud-rust). Use when adding, renaming, deprecating, promoting, or
  removing librarian.yaml configuration fields or Sidekick codec options, or
  when rolling out generator changes that depend on downstream runtime or
  handwritten code.
---

# Cross-Repository Config and Codegen Rollout

Use this skill when modifying `librarian.yaml` configuration schemas, Sidekick
codec options, or code generation templates in ways that interact with
downstream language repositories (especially `googleapis/google-cloud-rust`).
It determines whether a change requires a 2-PR or 3-PR rollout so neither
`librarian` postsubmit CI nor downstream presubmit CI breaks.

## CI and Parsing Constraints

Cross-repository changes are governed by three technical invariants:

1.  **Asymmetric CI Pinning**:
    -   **`librarian` postsubmit CI**
        ([`.github/workflows/sidekick.yaml`](/.github/workflows/sidekick.yaml),
        plus `integration-test` jobs in other language workflows): On every push
        to `googleapis/librarian` `main`, the `integration` job builds
        `librarian` at `HEAD`, checks out `googleapis/google-cloud-rust` at
        `HEAD`, runs `librarian install -v`, `librarian generate --all`, and
        `cargo check -p google-cloud-showcase-v1beta1`, and opens a GitHub issue
        on failure.
    -   **Downstream presubmit CI** (`google-cloud-rust` `.gcb/format.yaml`):
        Pins `librarian` to the `version:` field in
        `google-cloud-rust/librarian.yaml` and runs `librarian tidy` followed by
        `git diff --exit-code`.
    -   **Invariant**: Every commit merged to `librarian` `main` must parse
        `google-cloud-rust@HEAD`'s `librarian.yaml`, generate all libraries
        without error, and emit code that compiles against
        `google-cloud-rust@HEAD`. Conversely, `google-cloud-rust` cannot add new
        `librarian.yaml` keys until `version:` is bumped to a `librarian` commit
        that recognizes them (otherwise pinned `librarian tidy` strips them).

2.  **YAML Unmarshaling vs. Codec Map Validation**:
    -   **Unknown YAML keys are silently ignored**:
        [`internal/yaml/yaml.go`](/internal/yaml/yaml.go) calls
        `go.yaml.in/yaml/v3`'s `yaml.Unmarshal` without `KnownFields(true)`.
        Removed YAML keys still present in `google-cloud-rust/librarian.yaml` at
        `HEAD` are ignored during `librarian generate --all`, and are
        automatically stripped when `librarian tidy`
        ([`internal/librarian/tidy.go`](/internal/librarian/tidy.go)) rewrites
        `librarian.yaml`.
    -   **Unknown Sidekick codec keys fail loudly**:
        [`internal/sidekick/rust/codec.go`](/internal/sidekick/rust/codec.go)
        (`newCodec`) rejects unrecognized map keys with
        `unknown Rust codec option %q`. Always keep
        [`internal/librarian/rust/codec.go`](/internal/librarian/rust/codec.go)
        (`buildCodec`, `newLibraryCodec`, `buildModuleCodec`) and
        [`internal/sidekick/rust/codec.go`](/internal/sidekick/rust/codec.go)
        in sync within the same `librarian` commit.

3.  **Generated Schema Docs and Config Merging**:
    -   [`internal/config/config.go`](/internal/config/config.go) has a
        `//go:generate` directive that regenerates
        [`doc/config-schema.md`](/doc/config-schema.md), enforced in CI by
        `TestGoGenerate` in [`all_test.go`](/all_test.go). Run
        `go generate ./...` whenever `internal/config/` changes.
    -   Update `fill<Lang>` and `merge<Lang>` in
        [`internal/librarian/library.go`](/internal/librarian/library.go)
        whenever adding, renaming, or removing fields on language config
        structs.

## Choosing Between a 2-PR and 3-PR Rollout

Select the rollout sequence based on whether `librarian@HEAD` can safely ignore
the existing downstream `librarian.yaml` and compile against downstream `HEAD`:

-   **Use the 2-PR Flow when**:
    -   Promoting an opt-in config or codec flag to always-on and removing it,
        provided unconditional generation already compiles cleanly against
        `google-cloud-rust@HEAD` (for example, `#7578` / `b613abf4`, or
        promoting `extensible_builders`, `extend_grpc_transport`,
        `detailed_tracing_attributes`, or `lro_stub_options` once downstream
        support is in place).
    -   Removing an unused, obsolete, or already-defaulted config option (for
        example, `#7786`).
    -   Adding a new opt-in config option or backwards-compatible generator
        change (for example, `#7782`, `#7784`).

-   **Use the 3-PR Flow when**:
    -   Renaming a config option whose configured values are still needed on
        `google-cloud-rust@HEAD` (for example, renaming
        `allow_streaming_any_types` to `allow_grpc_any_fields` across `#7504`,
        downstream migration, and `#7587`). Deleting the old field immediately
        would cause `yaml.Unmarshal` to silently drop active overrides on
        `google-cloud-rust@HEAD`.
    -   Removing an option that requires migrating `google-cloud-rust`
        configuration or handwritten code first (for example, replacing a
        bespoke flag with `skipped_ids` or updating veneers before dropping an
        override).
    -   Introducing generated code that references new runtime types, traits, or
        helpers not yet on `google-cloud-rust@HEAD` (for example, `#5942`, where
        `google_cloud_lro::PollerOptions` had to exist in `google-cloud-rust`
        before generated stubs could emit it unconditionally).

## Pattern 1: 2-PR Flow

### Promoting an Opt-In Flag to Always-On (or Removing an Unused Option)

Because `yaml.Unmarshal` ignores unknown keys in `google-cloud-rust@HEAD` and
the unconditional output compiles cleanly on `google-cloud-rust@HEAD`, you can
remove the option in `librarian` first:

1.  **PR 1 (`googleapis/librarian`)**:
    -   Make the template, annotation, or generator behavior unconditional (or
        remove dead code for unused options).
    -   Remove the struct fields from
        [`internal/config/language.go`](/internal/config/language.go) (such as
        `RustDefault`, `RustCrate`, and `RustModule`).
    -   Remove defaulting and merging logic from
        [`internal/librarian/library.go`](/internal/librarian/library.go) and
        option plumbing from `internal/librarian/<lang>/` and
        `internal/sidekick/<lang>/` in the same commit.
    -   Run `go generate ./...` to update
        [`doc/config-schema.md`](/doc/config-schema.md) and update unit tests.
    -   Merge PR 1 and wait for `main` CI to pass.

2.  **PR 2 (`googleapis/google-cloud-rust`, after PR 1 merges)**:
    -   Bump `version:` in `librarian.yaml` to the merged `librarian` commit,
        strip the removed YAML keys via `librarian tidy`, and regenerate:

        ```bash
        V=$(GOPROXY=direct go list -m -f '{{.Version}}' github.com/googleapis/librarian@main)
        go run github.com/googleapis/librarian/cmd/librarian@${V} config set version ${V}
        go run github.com/googleapis/librarian/cmd/librarian@${V} tidy
        go run github.com/googleapis/librarian/cmd/librarian@${V} generate --all
        ```

    -   If handwritten code or tests in `google-cloud-rust` are also updated,
        keep automated `librarian` output (`librarian.yaml`, `src/generated/`,
        `Cargo.toml`, `Cargo.lock`) in a separate commit from handwritten edits.

> **Note on strict YAML validation**: If `librarian` enables strict unknown-key
> validation (`KnownFields(true)`, tracked in
> https://github.com/googleapis/librarian/issues/6464), removing a struct field
> while `google-cloud-rust@HEAD` still sets it will fail postsubmit CI. That
> case will shift to the 3-PR flow: hardcode the behavior while keeping the
> config field in PR 1, bump `version:` and remove YAML usages in PR 2, and
> delete the config field in PR 3.

### Adding a New Opt-In Option

1.  **PR 1 (`googleapis/librarian`)**: Add the struct field in
    [`internal/config/language.go`](/internal/config/language.go) (defaulting to
    existing behavior when unset), wire `fill<Lang>` / `merge<Lang>` in
    [`internal/librarian/library.go`](/internal/librarian/library.go) and codec
    plumbing in `internal/librarian/<lang>/` and `internal/sidekick/<lang>/`,
    and run `go generate ./...`.
2.  **PR 2 (`googleapis/google-cloud-rust`, after PR 1 merges)**: Bump
    `version:` in `librarian.yaml`, set the new option in `librarian.yaml`, and
    run `librarian tidy` and `librarian generate --all`.

## Pattern 2: 3-PR Flow

### Variant A: Renaming or Migrating a Config Option

1.  **PR 1 (`googleapis/librarian`)**:
    -   Add the new field to
        [`internal/config/language.go`](/internal/config/language.go) and keep
        the old field marked deprecated.
    -   Support both fields in
        [`internal/librarian/library.go`](/internal/librarian/library.go) and
        the language runner, falling back to the deprecated field when the new
        field is unset.
    -   Run `go generate ./...` and merge PR 1.

2.  **PR 2 (`googleapis/google-cloud-rust`, after PR 1 merges)**:
    -   Bump `version:` in `librarian.yaml` to include PR 1.
    -   Migrate `librarian.yaml` entries from the old key to the new key (along
        with any required handwritten code migrations).
    -   Run `librarian tidy` and `librarian generate --all`, then merge PR 2.

3.  **PR 3 (`googleapis/librarian`, after PR 2 merges)**:
    -   Remove the deprecated field and fallback logic from `internal/config/`,
        `internal/librarian/`, and `internal/sidekick/`.
    -   Run `go generate ./...` and merge PR 3.

### Variant B: Codegen Depending on New Downstream Runtime Symbols

Choose one of two staging approaches:

-   **Additive runtime API**:
    1.  **PR 1 (`google-cloud-rust`)**: Land the new runtime type, trait, or
        helper in a backwards-compatible way so existing generated crates still
        compile.
    2.  **PR 2 (`librarian`, after PR 1 merges)**: Update templates and
        annotations to emit references to the new runtime symbol. Postsubmit CI
        against `google-cloud-rust@HEAD` now compiles cleanly.
    3.  **PR 3 (`google-cloud-rust`, after PR 2 merges)**: Bump `version:` in
        `librarian.yaml`, run `librarian generate --all`, and remove any
        temporary runtime shims if needed.

-   **Mutually incompatible signature or staged crate-by-crate rollout**:
    1.  **PR 1 (`librarian`)**: Gate the new generated code behind a temporary
        opt-in config/codec flag (defaulting to off so `google-cloud-rust@HEAD`
        stays green).
    2.  **PR 2 (`google-cloud-rust`, after PR 1 merges)**: Land the runtime and
        handwritten changes, bump `version:` in `librarian.yaml`, enable the
        opt-in flag (under `default.rust` or per crate/module), and run
        `librarian generate --all`.
    3.  **Cleanup (2-PR Flow)**: Once enabled across `google-cloud-rust@HEAD`,
        use the 2-PR flow above to make the behavior unconditional in
        `librarian` and strip the temporary flag in `google-cloud-rust`.

## Local Pre-PR Verification

Before opening a `librarian` PR that touches config or codegen, verify both
`librarian` checks and downstream compatibility against a clean
`google-cloud-rust@HEAD` checkout (mirroring `.github/workflows/sidekick.yaml`):

```bash
# 1. In the librarian worktree: regenerate schema docs, format, lint, and test
go generate ./...
gofmt -s -w .
go tool goimports -w .
go tool golangci-lint run
go test -short ./...
go test -count=1 .

# 2. Against a disposable google-cloud-rust@HEAD worktree: simulate postsubmit CI
go -C <librarian-worktree>/cmd/librarian build -o /tmp/librarian-dev
(cd <rust-worktree> && /tmp/librarian-dev generate --all)
(cd <rust-worktree> && cargo check -p google-cloud-showcase-v1beta1)
```
