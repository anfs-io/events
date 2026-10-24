# events

The ANFS event spec: one CloudEvents 1.0 profile for every layer built on ANFS (anfs, cnfs, lgat, and those to come). It is the contract, and nothing else: no implementation code lives here. A tag of this repository is a version of the spec.

```
SPEC.md                 the profile: the event, types and namespaces, space ids and sources, actors,
                        NATS subjects, the spool and direct publishing, delivery, derived ids,
                        extensions, versioning, conformance
REGISTRY.md             namespaces, extensions, and anfs's type catalog
schema/event.v1.json    JSON Schema (draft 2020-12) for one event
examples/valid/         one valid event per file
examples/invalid/       one invalid event per file, each breaking one rule
examples/index.json     the manifest: per file valid, rule, schema, subject, key
tools/check             checks the examples against the manifest, the line rules and the schema
tools/requirements.txt  check-jsonschema, pinned with hashes
CHANGELOG.md            what each version changed
```

## Using it

Implementations vendor a tag's `schema/` and `examples/` into their own tests, with a `VERSION` file naming the tag, and assert the manifest: every valid example passes their validator and every invalid one fails it, their NATS subject for each valid example is the manifest's `subject`, and their id derivation gives each example's id from its `key`. Updating is copying a newer tag's files. Any language validates the schema with its own library (Go `santhosh-tekuri/jsonschema`, Ruby `json_schemer`, Python `jsonschema`) and adds the line rules SPEC.md §12 lists.

The schema's `$id` is `https://raw.githubusercontent.com/anfs-io/events/v1.0.0/schema/event.v1.json`.

## Checking this repository

```bash
python3 -m venv .venv
.venv/bin/pip install --require-hashes --only-binary=:all: -r tools/requirements.txt
PATH="$PWD/.venv/bin:$PATH" tools/check     # manifest, line rules, subjects, derived ids, schema
tools/check --no-schema                     # without check-jsonschema: Python 3 alone
tools/check --derive-id 2026-10-24T09:40:17.000Z 'io.lgat.escrow.payment.settled|<signature>|2|0|finalized'
```

CI (`.github/workflows/ci.yml`) runs the same on every push and pull request.

## Versions

Semantic versions; every `v1.y.z` is `anfsv` 1. A new namespace or extension is a minor version, a fix to the text or an example a patch, and anything that makes a v1-valid event invalid a new major version with a new `anfsv` (SPEC.md §11). anfs-io's maintainer drafts; the person approves and tags.
