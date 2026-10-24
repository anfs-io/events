# Changelog

## 1.1.0

A minor version: types added to anfs's catalog (REGISTRY.md), with examples. Every v1.0.0-valid event stays valid; the schema is unchanged.

- Landing (agents/04a): `run.auth-required`; the approvals `approval.requested`, `.given`, `.refused`, `.expired`, `.done`, `.failed`; `inbox.dismissed`. New data keys on runs: `run.working` `step`, `run.completed` `landed` and `skipped`, `run.canceled` `reason`, and `run.failed`'s landing reasons. `source.updated` may have actor `anfsd`. Examples: `run-auth-required`, `approval-requested`, `approval-given`.

## 1.0.0

The first version of the spec, `anfsv` 1, moved out of anfs (anfs-io/docs `units/agents/01-events/plan.md`, "The envelope") and opened to every layer. Every event anfs has written so far stays valid.

Tightened from 01's profile:
- `time`, `datacontenttype` and `data` are required (anfs always wrote them); `time` has exactly milliseconds and equals the id's timestamp.
- No `null` for any attribute; no control characters in attribute strings; unique member names; `subject` not empty.
- `actor` has a pattern (`<kind>[:<name>]`, at most 128 bytes); `actorspace` only with `actorrole`.
- Extension names and values follow CloudEvents' rules as MUSTs.

Opened:
- Types in any registered namespace: `io.anfs`, `io.cnfs`, `io.lgat` (`REGISTRY.md`).
- Sources per layer: `/<layer>/spaces/<id>[/…]` and `/<layer>/host[/…]`; the space-id rule written down.
- `data` is any JSON object within 4096 bytes; flat strings stay the convention for `io.anfs` (`anfs.data.flat`).
- Extensions `actorclaimed`, `actorrun`, `traceparent`, `tracestate`; lgat's `commitment`.

Added:
- NATS subjects `anfs.<scope>.<namespace>.<rest>` (the type without `io.`, one shape for every layer), structured mode, `Nats-Msg-Id`, one stream.
- Direct publishing as a second way in beside the spool, with the publish contract, the publisher user per layer and the recorder.
- Delivery semantics, derived ids, actor attestation, versioning rules.
- The JSON Schema, the conformance examples (runtime's `bash.jsonl` and lines from a development host's spool promoted) with their manifest, and `tools/check`.
