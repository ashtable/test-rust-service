# Plan: an Axum service with tested and proved business rules

Status: researched proposal, refreshed 2026-10-28. Implementation has not started. This file supersedes the earlier `IMPLEMENTATION_PLAN.md` draft.

## 1. Outcome and requirements

Build a small Rust/Axum service, a Docker Compose development setup, and a separate Lean proof container. Every business rule belongs in a pure Rust module with meaningful specifications and proofs over the model generated from that actual Rust source. Develop behavior in small test-first slices, and keep the user in control of each implementation diff.

The concrete demonstration is a POST command to **request to stand down**, carrying the version of the latest status update the caller has seen. The mandatory rule is:

> If the authoritative latest status version is greater than the submitted version, deny the request automatically. A denied command must not record or enact a stand-down request.

Required delivery mechanisms are a **pre-commit hook checking the staged source** and a **CI proof job required before merging to `main`**. Both must regenerate the Rust-derived Lean model and check the same business invariants. A passing run must not rely on stale generated files, missing proofs, or admitted assumptions.

This is a planning deliverable. The later Sol run will execute the slices below, explaining the change and its reason and obtaining approval before applying each diff. Planning approval does not approve every future implementation patch.

## 2. Research conclusions

### Rust-to-Lean path

Charon extracts rustc MIR into LLBC, and Aeneas translates LLBC into a functional model for Lean. Charon and Aeneas support different subsets: the current Charon manual excludes async, while Aeneas targets a narrower safe, sequential subset. Therefore, isolate business rules from Axum, Tokio, synchronization, serialization, and storage. Check each introduced Rust construct against the pinned translator. [Charon manual](https://aeneasverif.github.io/charon/manual/), [Aeneas projects](https://aeneasverif.github.io/projects/), [Aeneas supported subset](https://github.com/AeneasVerif/aeneas/blob/main/README.md#targeted-subset-and-current-limitations)

Upstream's basic pipeline is `charon cargo --preset=aeneas`, `aeneas -backend lean`, then a Lake project importing the generated code and the Aeneas library. Translation generates the model; it does not invent the business specification or complete its proofs. [Aeneas getting started](https://github.com/AeneasVerif/aeneas/blob/main/documentation/getting-started.md), [Lean's Aeneas case study](https://lean-lang.org/use-cases/aeneas/)

### Compatible pins and complete extraction

Select one immutable Aeneas commit, its required `charon-pin`, that Charon revision's Rust toolchain, and the corresponding Aeneas Lean backend/toolchain. Pin build dependencies too, including the OCaml/OPAM/Dune environment. Independently choosing the newest tool versions is not a compatibility strategy. No compatible set has yet been built in this repository. [Aeneas installation](https://github.com/AeneasVerif/aeneas/blob/main/README.md#installation--build)

An emitted LLBC file is not enough: Charon can represent missing bodies, and extraction options can exclude or make definitions opaque. Check diagnostics and required application bodies as well as exit codes. Current source includes `--error-on-warnings`; inspect the pinned binary before fixing the invocation. Never use permissive matching or skipped checking to make the gate green. [Charon paper, section 2.3](https://arxiv.org/pdf/2410.18042), [Charon options](https://github.com/AeneasVerif/charon/blob/main/charon/src/options.rs)

### Kernel checking and the trust boundary

Use three checks: build the Lean project, audit theorem dependencies for unwanted axioms, and replay compiled declarations through the kernel in a fresh process. Modern Lean includes `leanchecker`; the old separate `lean4checker` repository documents its migration into Lean 4.28 and later. Validate `lake env leanchecker --fresh <root-module>` against the selected version. [Lean proof validation](https://lean-lang.org/doc/reference/latest/ValidatingProofs/), [Checker migration](https://github.com/leanprover/lean4checker#using-the-built-in-leanchecker)

Initially permit only the standard logical foundations `propext`, `Classical.choice`, and `Quot.sound`. Reject transitive `sorryAx`, custom application axioms, and native-evaluation/compiler-trust axioms. A successful `lake build`, or a text search for `sorry`, is insufficient by itself. Inspect dependencies through Lean's environment. [Lean axioms](https://lean-lang.org/doc/reference/latest/Axioms/)

The 2022 Aeneas paper treats its implementation as trusted; the 2024 paper proves borrow-semantics results and leaves functional-translation correctness to future work. Our claim is that reviewed properties hold for freshly translated models, subject to the trusted translation/build path. This does not prove rustc, Charon, Aeneas, HTTP, the operating system, or synchronization correct. Kernel replay also does not establish that a theorem expresses the intended requirement. [2022 paper, section 5](https://doi.org/10.1145/3547647), [2024 paper, introduction](https://doi.org/10.1145/3674640)

## 3. Proposed behavior to approve before implementation

Only stale-version denial was explicitly specified. The following defaults make the demonstration concrete; Sol must present them for approval with the test boundaries before writing tests. If they change, revise the contracts and examples first.

| Topic | Proposed default and reason |
| --- | --- |
| API | `GET /status` returns the authoritative version and separate request metadata. `POST /commands` accepts JSON such as `{"command":"request_to_stand_down","version":1}`. `/healthz` provides a container smoke check. |
| Versions | Server-owned status-update sequence numbers are positive `u32` values, beginning at 1. Submitted values may be any `u32`, including 0, which is stale against every valid server version. Negative, fractional, and out-of-range JSON values are transport errors. This keeps the entire numeric range exact for common JSON clients and gives an explicit finite bound to prove. |
| Older submitted version | `409 Conflict`, structured `stale_version` decision, current version in the response, no state change. This is the mandatory behavior. |
| Equal submitted version | `200 OK`, record that the request was accepted at this status version. Acceptance records a request; it does not assert that an operational stand-down has occurred. |
| Future submitted version | Reject with `409 Conflict` and `future_version`, leaving state unchanged. This is a proposed conservative policy, not implied by the user's stale-version requirement. |
| Request effects | Track `last_request_version: Option<u32>`. Acceptance sets it to the current status version. Repeating the same request at that version returns acceptance with the same state. No queue, physical action, or exactly-once delivery claim is introduced. |
| Status advancement | Publishing a status update increases the status version by exactly one and preserves the historical request record. Accepting a command does not itself publish a new status update or increase that version. |
| Exhaustion | At `u32::MAX`, status publication fails explicitly and leaves state unchanged; never wrap to zero. |
| Initial storage | One process, one in-memory state protected by one lock. A fresh process starts a fresh test session. Version guarantees span one session; persistent versions, multiple replicas, and cross-restart client continuity require a later storage/epoch design. |

The pure state is conceptually `(status_version, last_request_version)`. A valid state has `status_version >= 1` and any recorded request version between 1 and `status_version`. There is no authorization or additional eligibility policy in this test. Adding one is new business logic and needs its own reviewed contracts and proofs.

Use a controlled application-level status publisher for tests: it invokes the same proved status-advance function under the same lock as commands. Do not expose a public administrative status-update endpoint merely to make tests convenient. A seeded in-memory store provides deterministic test setup; no sleeps or live external service are needed.

The acceptance rule is evaluated against the authoritative state **at the atomic decision/recording point**. A status update committed before that point must be observed. An update committed afterward does not retroactively invalidate a decision already recorded. This timing definition is part of the contract.

## 4. Architecture and coverage

```text
HTTP client -> Axum adapter -> application operation under one state lock
                                      |
                                      v
                              pure Rust domain transition
                                      |
                                same Rust source
                                      v
                 rustc -> Charon LLBC -> Aeneas Lean model
                                      + specifications/proofs
                                      v
                    build -> axiom audit -> fresh kernel replay
```

Proposed layout:

```text
crates/domain/                 State, decisions, command and status transitions
services/api/                  HTTP, parsing, state synchronization, process lifecycle
proofs/                        Handwritten specifications, proofs, contract inventory
verification/                  Toolchain lock, extraction configuration, gate metadata
docker/                        Web and proof image recipes
scripts/                       Preparation, working-tree proof, staged proof entry points
.githooks/                     Versioned hook and installation support
.github/workflows/             Required verification workflow
```

Keep the domain dependency-free initially, with `no_std` and unsafe code forbidden. These help maintain the boundary but do not mechanically establish purity. Business comparisons, eligibility, version advancement, request effects, and invariant maintenance belong here. Adapters perform transport and synchronization; they must not duplicate the policy comparison. Axum's extraction and response mechanisms provide the transport boundary. [Axum documentation](https://docs.rs/axum/latest/axum/)

Maintain a coverage inventory mapping **every application-owned domain function**, including private helpers, to a specification and a proved theorem or an explicitly identified compositional proof that covers it. Reconcile that inventory with discovered Rust/LLBC definitions and generated Lean declarations. A helper need not have a redundant standalone theorem when its behavior is covered by the enclosing proof, but its body must be extracted and accounted for. Reject unclassified business functions, missing bodies, and unexpected application opacity.

Every new business rule must arrive with its tests, statement, proof, and coverage entry in the same reviewed slice. No permanent 'proof later' list, hand-maintained copy of the Rust logic in Lean, or opaque application substitute. Review adapters for policy that escaped the domain boundary. Automation cannot decide whether a function is business logic or whether a specification was weakened; human review remains required for those decisions.

### Business proof obligations

These are mathematical specifications, not claims that the final Lean syntax has already been tested. Prove successful return/no panic as well as the result; an implication about a result that never exists is insufficient.

| Domain operation | Required proofs over the generated Rust model |
| --- | --- |
| Initialization | Produces version 1, no prior request, and a valid state. |
| Command transition: stale | For every valid state and submitted version below its status version, returns `DeniedStale` successfully and the exact input state. In particular, version 0 is stale against all valid states. |
| Command transition: equal | For every valid state, submitting its status version succeeds, returns acceptance, records that version, and preserves the status version. This rules out an always-deny implementation. |
| Command transition: future | Under the proposed policy, every higher submitted version returns `DeniedFuture` successfully and the exact input state. |
| Command transition: overall | Total/no panic on valid states for every `u32` input, preserves the state invariant, and repeated equal-version requests are idempotent in state. |
| Status publication | Below the maximum, advances the version exactly once, preserves request history, and preserves validity. At the maximum, returns exhaustion successfully and preserves the whole state. |
| Composition | A finite sequence of these successful domain calls preserves validity. After publication from version 1 to 2, a version-1 request is denied; the same property holds for arbitrary increasing versions. |

The composition result concerns sequential domain transitions. Thread scheduling, lock use, HTTP decoding, and persistence are integration-test/review responsibilities. For the initial short in-memory critical section, an ordinary mutex wrapped by synchronous application methods is appropriate; never hold it across an await. All reads and writers use the same state authority. [Tokio mutex guidance](https://docs.rs/tokio/latest/tokio/sync/struct.Mutex.html)

## 5. TDD and proof development protocol

Agree the test boundaries first, as required by the TDD workflow. Approval of the boundary list is distinct from approval of later code diffs.

| Public test boundary | What it checks | What it does not establish |
| --- | --- | --- |
| Domain operations and generated Lean contracts | Business outcomes, state changes, arithmetic limits; proofs generalize beyond example inputs. | HTTP and concurrency behavior. |
| Application operations: command, publish, read | Real shared-state coordination and observable transitions, including racing writers. Approve a test-only scheduling checkpoint immediately before entry to the command's decision/recording critical section. | Arbitrary thread schedules by formal proof or multi-process storage. |
| Axum router/HTTP interface | Actual JSON parsing, response codes, state visibility, and routing into the domain. | Container networking and deployment. |
| Proof runner and Git hook as processes | Correct source selection, failure propagation, complete proof checking, and isolation. | Mathematical adequacy of a deliberately weakened contract. |
| Compose service and GitHub required check | Runnable image/network behavior and actual enforcement before merge. | Whole-program formal correctness. |

For each behavior slice:

1. Write one public-behavior test with literal or independently specified expected results. Observe it fail for the intended reason.
2. State the corresponding Lean contract before implementing the business rule. Once extraction is available, observe the missing/incorrect implementation fail that contract; a missing tool is not meaningful evidence of a false invariant.
3. Add the minimum Rust implementation, regenerate Lean, and complete the proof. Check both the behavior test and the full established proof gate.
4. Review code/spec alignment and perform any justified refactor as a separate review step, keeping the same behavioral tests and proofs green.
5. Present the proposed patch with a short **what/why**, red/green evidence, proof names and outcome, and remaining limits. Wait for user approval before applying it to the main working tree.

Use an isolated proposal checkout to prepare and validate a patch before approval. Do not write all tests first and then bulk-implement the service. Each green slice retains prior tests and proofs; a temporary red state must never be merged. Do not mock the domain or verify private helper call counts.

The scheduling checkpoint controls execution only in tests; it must not supply a different business rule or state. Pause a version-1 command at that checkpoint, publish status version 2 through the real application operation, then resume the command and require stale denial with no request recorded. Deliberately move the version read before the checkpoint in a temporary mutant and require the same test to fail. Also test command-before-publication as an allowed history. Ordering whole calls or examining only the final state is insufficient: `(version 2, last_request_version 1)` can legitimately result when the command was accepted before publication. Assert operation results and the publisher's returned snapshot as well as final state. Use bounded waits only to fail a hung test, not to infer ordering.

## 6. Containers and reproducibility

`web` uses a multi-stage build and a small non-root runtime image. `proof` is an on-demand Compose service containing the matched compiler/extractor/translator, Lean/Lake/checker, and prebuilt library dependencies. `docker compose up --build web` starts the service; proof execution must not require a running web service or database. Put `proof` behind a Compose profile and invoke it explicitly. [Docker multi-stage builds](https://docs.docker.com/build/building/multi-stage/), [Compose profiles](https://docs.docker.com/compose/how-tos/profiles/)

Establish compatibility before adding real business behavior. Build and run a tiny extraction/checker smoke fixture in a temporary proposal checkout, select immutable tool hashes, then record the proven combination. Build the service's domain and extract it with the same Rust toolchain and equivalent semantic inputs: edition, feature resolution, target, cfg, and overflow behavior. Set release overflow checks explicitly; forbid verification-only alternate implementations. Cargo configuration/environment can override profile settings, so control and report effective inputs. [Cargo profiles](https://doc.rust-lang.org/cargo/reference/profiles.html)

Use locked Rust and Lean dependencies and immutable base-image references. Pin or snapshot tool build dependencies sufficiently to reproduce the environment, and record any remaining mutable package inputs. The proof image identity must include the Docker recipe, tool pins, dependency locks, and platform; a version string or mutable tag is not enough. Execute project scripts from the selected source snapshot rather than stale copies baked into an old image.

Download/build tools in an explicit preparation command. A normal verification run uses the prepared image with networking disabled, a read-only input mount, temporary writable output, and no host Docker socket inside the container. Regenerate application LLBC/Lean into an empty output directory and rebuild application proofs. Reuse prebuilt dependency artifacts; do not reuse a previous project's 'passed' marker or compiled application proofs as a substitute for this run.

Validate Linux ARM64 on the current Apple Silicon host and Linux AMD64 if that is the CI target. An explicitly selected AMD64 image under emulation is an acceptable initial fallback only after measuring it and aligning service/proof targets. Do not promise multi-platform support, small proof-image size, or a subsecond hook before testing. [Docker multi-platform builds](https://docs.docker.com/build/building/multi-platform/)

## 7. One verification gate, used locally and before merge

### Gate contents

The same runner must:

1. Identify the source snapshot, execution platform, matched toolchain, and expected image build inputs.
2. Reconcile the domain/function/contract inventory; reject missing, empty, or unclassified coverage.
3. Run strict Charon extraction and require all application bodies; translate them freshly with Aeneas. Fail on unsupported code, unexpected model templates, and unresolved application definitions.
4. Compile every project proof module, regardless of manually maintained root imports, and resolve every required theorem by its fully qualified name.
5. Audit all registered contracts and project proof declarations for prohibited transitive axioms; ensure generated business definitions are covered as well. Fail on admitted definitions even if no currently selected theorem happens to use them.
6. Run fresh kernel replay over the root importing the complete generated/project proof set. Treat replay, audit, or compilation failure as gate failure.
7. Report the source identity, discovered coverage, checked contract names, assumptions, and phase timings. Return success only after all phases finish successfully.

Named proof roots and inventory reconciliation prevent accidental omissions. Deleting both a theorem and its inventory entry, replacing its statement with `True`, or assuming the desired result as a precondition still requires specification review. Do not claim that coverage automation proves specification completeness.

### Pre-commit hook

- Export the exact staged index, including Rust, proof sources, locks, Cargo configuration, scripts, and container configuration. Git supports index export; account for attributes/filters, symlinks, and any future submodules rather than silently reading outside the snapshot. Fail on unsupported snapshot inputs. [Git index export](https://git-scm.com/docs/git-checkout-index)
- Run the snapshot's runner/configuration in the matching prepared environment. Do not stash, auto-stage, or rewrite the user's working tree. Check that the index has not changed while verification runs; abort if the verified snapshot no longer matches it.
- Initially run on every commit after hook installation. Optimize triggering only in a separately reviewed change with dependency-sensitive regression tests.
- Missing Docker access, missing or mismatched image inputs, failed extraction, failed proofs, and timeouts block the commit with an actionable reason. Tool/image preparation is explicit, outside the hook.
- Install a version-controlled hook without overwriting existing hook behavior. Test partial staging in both directions, including staged runner and toolchain changes.

### CI and `main` merge enforcement

- Run the same gate on every `pull_request` targeting `main`, with a unique stable job/check name such as `lean-invariants`. Run service tests too. Check the proposed merge result, not just an unrelated branch checkout; record the tested commit.
- Do not path-filter or conditionally skip the required proof job, and do not use `continue-on-error`. Keep environment preparation and proof checking in the same required job initially so a failed prerequisite cannot turn into a successful skipped dependent job. GitHub documents these skip behaviors. [Required-check troubleshooting](https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/troubleshooting-required-status-checks)
- Configure `main` branch protection/rulesets to require that check before merging. Require an up-to-date tested merge result; if a merge queue is enabled, include the `merge_group` trigger. Verify direct-push and bypass settings match the intended enforcement. [Protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches), [Workflow events](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows)
- Build or obtain the image matching the candidate's recipe/lock inputs; do not run a changed candidate against a stale trusted-branch image merely because its tag exists. Keep PR execution unprivileged and avoid secrets-dependent proof setup.
- Demonstrate a failing proof makes the required check fail and blocks merging, then demonstrate the repaired proof passes. If repository permissions or account features prevent required-check enforcement, report that exact unmet acceptance criterion; a YAML workflow alone is not completion.

Local Git hooks can be bypassed. The required CI check is the merge gate, while review of specifications, coverage, and workflow changes protects against intentionally weakening what is checked. [Git hooks](https://git-scm.com/docs/githooks#_pre_commit)

## 8. Ordered implementation slices for Sol

These are bounded review units, not permission to batch changes. Each row may split into smaller patches when necessary. Preserve red/green evidence in the review summary; do not commit deliberately broken mutations.

Keep all implementation slices on the implementation branch. No implementation merge to `main` is permitted until the proof CI job and required merge rule are active and their enforcement has been demonstrated. Approving a diff authorizes applying that patch; it does not waive this merge prerequisite. Review the concrete repository-rule change alongside the CI diff before applying it.

| Slice | Change and why | Red evidence, then acceptance evidence |
| --- | --- | --- |
| 0. Contract and test boundaries | Approve section 3 defaults and section 5 interfaces, including the test-only scheduling checkpoint. Establish what acceptance, version advancement, and atomicity mean. | Review the truth table and observable states. No tests before boundary approval. |
| 1. Toolchain feasibility and minimal proof environment | Pin the compatible toolchain, create a minimal dependency-free Rust/proof harness and proof image, test fresh replay. Resolve the highest integration risk first. | Missing pipeline initially fails; a tiny correctly extracted function passes; an incorrect result/admitted proof fails. Check the actual service dependencies compile with the chosen Rust version. |
| 2. Domain initialization and current-version acceptance | Introduce real state and the first meaningful command behavior, so the exercise cannot pass by denying everything. | Version 1 with no history + POST version 1 must be accepted and record 1 without changing the status version. Add initialization, positive-return, and invariant proofs. |
| 3. Stale and future denial | Implement the required stale rule and, if approved, future rejection. | State version 2 + submitted 1 must deny unchanged; submitted 3 follows the approved future policy. Prove both for all valid states, plus totality and repeat-request behavior. |
| 4. Status publication and arithmetic bounds | Make version progression a proved business operation and establish sequential composition. | 1 advances to 2; a subsequent version-1 command is denied; maximum version returns exhaustion unchanged. Prove increment, preservation, and composition. |
| 5. Axum and atomic application operations | Wire the proved core into real HTTP and one authoritative state boundary. | Router tests first fail, then exercise accepted/stale/future/malformed inputs and actual state visibility. Controlled status-update/command interleavings establish the specified atomic order. |
| 6. Runnable web container and Compose | Add the small runtime image and on-demand proof service integration. | HTTP/container smoke fails before wiring; both services then work independently; graceful shutdown, non-root execution, and configured port are checked. |
| 7. Complete proof gate and staged hook | Add completeness/coverage checks, image-input validation, source snapshotting, and installation. | Adversarial process fixtures fail for stale artifacts, missing contracts, bad axioms, and partial staging; correct staged snapshots pass offline with working tree/index preserved. |
| 8. Required CI and operator instructions | Apply the same gate to PRs and enforce it before `main` merges. Document setup, editing proofs, upgrades, and diagnostics. | An invalid invariant blocks the PR through the required check; repaired candidate passes; required-rule settings are inspected. Measure warm/cold runs and report actual figures. |

Slices 2-4 must already build and check every accumulated proof using the harness from slice 1. Slice 7 hardens orchestration and installs the hook; it does not postpone business proofs until the end. For any new business function added in later slices, extend the proof inventory and prove it in that same slice.

## 9. Adversarial acceptance matrix

Apply these attacks to temporary test copies and restore the valid candidate afterward.

| Attack or boundary | Required result |
| --- | --- |
| Replace stale comparison with the wrong direction or remove it | Required invariant and literal stale HTTP case fail. |
| Always deny every command | Positive acceptance theorem/test fail. |
| Use `>=` to classify equal versions as stale | Equal-version theorem/test fail. |
| Wrap version at the maximum | Exhaustion and invariant-preservation proofs/tests fail. |
| Cache the version before the test checkpoint, then use it inside the later admission operation | Pause at the checkpoint, publish version 2, resume command version 1: the mutant accepts and the test fails. The correct implementation denies unchanged. Also check command-before-update as a valid history; use the publisher's returned snapshot to distinguish histories, without timing sleeps. |
| Add a business helper without coverage, omit a body, or use an opaque substitute | Coverage/extraction checks fail before success can be reported. |
| Delete a proof import/file or point the root at an empty module | Full module discovery and contract resolution fail. |
| Insert `sorry`, an indirectly admitted lemma, a custom axiom, or native-evaluation trust | Axiom audit fails; malformed proof terms must also fail kernel replay. |
| Keep old generated Lean or old application `.olean` files after a Rust change | Clean generation/build uses current source; the altered invariant fails. |
| Stage a bad Rust change but leave a good fix unstaged | Commit hook fails on the staged version. The inverse case checks the good staged version successfully. |
| Stage a new Docker recipe/lock/runner while an old image is available | Image-input validation demands the matching prepared environment; snapshot runner changes are exercised. |
| Delete a theorem and its inventory entry, or weaken its requirement | Review must reject the weakened contract; automated coverage alone is explicitly insufficient. |
| Proof step fails, is cancelled, or a prerequisite fails in CI | Required job cannot report success and merging remains blocked. |

## 10. Handoff and definition of done

For each proposed implementation patch, Sol reports:

- What changes and why it is needed.
- The exact diff, the behavior/specification being introduced, and any change to assumptions.
- Evidence of the intended red result and the final tests/proofs/checker result; clearly label unrun checks.
- The decision requested: approve, revise, or deny this patch. Wait before applying it, then verify the approved result and proceed to the next slice.

Done means the concrete endpoint behaves as approved, all business logic is accounted for by proofs over freshly generated Rust models, meaningful negative cases fail, Docker/Compose runs the service and proof environment, the staged hook works, and GitHub actually requires the proof check before merging into `main`.

Remaining implementation evidence: a real matched-toolchain build, a concrete extracted proof passing the strict axiom policy and replay, platform support, offline runs and timing measurements, deterministic race tests, and hosted merge-rule enforcement. This report does not claim those experiments have already passed.

## 11. Adversarial review record

An independent adversarial review was completed on the fresh draft after both the staged pre-commit gate and required pre-merge CI gate were included. It identified two substantive gaps, now revised:

- **Race-test sensitivity:** whole-operation ordering could miss a cached-read bug. Sections 5, 8, and 9 now require an approved scheduling checkpoint, an intervening real status publication, and a failing cached-read mutant. The oracle observes operation results and publication snapshots, not merely a final state compatible with multiple histories.
- **Merge timing during bootstrap:** green early slices could otherwise reach `main` before its proof gate existed. Section 8 now explicitly keeps implementation on its branch until the required proof job and merge rule are active and demonstrated.

The review also prompted clarification that server versions start at 1, submitted 0 is stale, and malformed numeric inputs are transport errors. The reviewer independently checked the current `leanchecker --fresh` command and found the remaining material requirements consistent. A follow-up review confirmed both findings were resolved and found no remaining inconsistency in the revised sections. Implementation experiments remain outstanding as listed above; review of the plan is not evidence that those experiments have run.
