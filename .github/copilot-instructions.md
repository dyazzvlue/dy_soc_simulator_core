# Project Development Rules

Project-local guidance for the simulator. The approved behavior and open decisions are recorded in [design_plan](../design_plan/README.md).

## Contents

- [Scope](#scope)
- [Engineering Rules](#engineering-rules)

## Scope

- This project is currently in the planning phase. Do not generate simulator source, build files or test implementations until the user explicitly authorizes implementation.
- Follow the confirmed decisions in [the decision log](../design_plan/05_decisions_and_open_questions.md). Decide reversible internal details without repeated approval; confirm changes to model behavior, input contracts, failure policy or compatibility.
- Preserve existing user edits and staged changes. Do not stage or commit without authorization.

## Engineering Rules

- Use portable relative paths in project documentation, examples, source, build declarations and persisted configuration. Do not hard-code a developer's home directory or checkout location. Specify the base used to resolve each relative path; runtime path resolution must not become a persisted machine-specific identity.
- Use C++20 and CMake with `CMakeLists.txt`. Compiler versions, test libraries and serialization libraries are not yet selected.
- Preserve the existing `include/`, `src/`, `tests/ut/`, `tools/` and `design_plan/` layout. Do not introduce a parallel source or unit-test tree.
- Design and implement unit tests alongside each module under `tests/ut/`; cover new behavior, boundaries and failure paths. Run the affected tests and build before declaring code complete. Documentation checks are not simulator test results.
- Use fixed-width standard integer types such as `std::uint32_t` and `std::uint64_t` from `<cstdint>` for IDs and related fields. Record each field's width and range before relying on them; do not silently truncate or wrap. Keep the approved non-reuse and overflow-failure rules.
- Keep configurable parameters behind a format-independent configuration interface. Command-line versus JSON input is intentionally undecided; do not add a parser dependency or hard-code a production default to bypass that decision. Tests may supply explicit configuration directly.
- Keep object lifetime, transaction completion and simulated resource occupancy separate. Preserve the approved stable-ID, controlled `shared_ptr` and checkpoint contracts.
- Update directly affected design contracts and validation plans when a confirmed decision changes.
