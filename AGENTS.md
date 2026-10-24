# events: for agents

The ANFS event spec: `SPEC.md` (normative), `REGISTRY.md`, `schema/event.v1.json`, and `examples/` with `examples/index.json` (the conformance fixtures every implementation vendors). The plan it comes from is anfs-io/docs `units/agents/05-spec/plan.md`; its implementations are runtime's `internal/cloudevent` (anfsd), core's `lib/anfs/events.sh`, and cnfs's events gem.

## Rules

- **The contract only.** No implementation code here; `tools/check` is the repository's own check and stays Python 3 with the standard library (the schema part runs check-jsonschema).
- **Text, schema and examples change together.** A rule added or changed in SPEC.md has its schema change (when a schema can express it), a valid and an invalid example, and a line in the manifest, in the same commit. `tools/check` passes before every commit.
- **Examples are fixtures others vendor.** Never change a released example's bytes or its file name: add a new example instead. One event per file, one line ended by `\n`; an invalid example breaks exactly one rule, and its manifest entry names that rule and whether the schema alone catches it.
- **Rule codes are stable.** A code in SPEC.md §12's table is never renamed or reused.
- **Versions.** A new namespace or extension is a minor version, a fix a patch, anything that makes a v1-valid event invalid a major version and a new `anfsv` (SPEC.md §11). Every change gets a line in CHANGELOG.md. Never tag: the person tags after reading the change.
- **Schema patterns stay portable:** no lookaround, no backreferences, control characters as JSON `\u` escapes, so Go, Python and Ruby read them alike.
- **Pins:** GitHub Actions by commit, check-jsonschema and everything it pulls in by version and sha256 (`tools/requirements.txt`), each version at least two weeks old when chosen.
- Prose: plain and concrete, one line per paragraph or bullet.
