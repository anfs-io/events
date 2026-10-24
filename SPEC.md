# anfs events, profile v1

Version 1.1.0 (`anfsv` 1).

This is the contract for events on the anfs bus: one CloudEvents 1.0 profile shared by every layer built on ANFS (anfs itself, cnfs's Rails apps, lgat's indexer, and whatever comes next). It says what an event is, who may name its types, how an event gets onto the bus, and what a consumer can rely on. It is language-neutral. anfs (`anfsd` and core's `lib/anfs/events.sh`) is the reference implementation on a development host; other implementations follow this text, `schema/event.v1.json` and the examples in `examples/`.

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are to be read as in RFC 2119.

## 1. The event

An event is one CloudEvents 1.0 event (spec 1.0.2) in the structured JSON format. This profile tightens CloudEvents and never loosens it: every event valid here is a valid CloudEvent.

| Attribute | Profile | Rule |
| --- | --- | --- |
| `specversion` | required | `"1.0"` |
| `id` | required | A UUIDv7 (RFC 9562), lower-case and hyphenated: `^[0-9a-f]{8}-[0-9a-f]{4}-7[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$`. Globally unique, so `id` alone (not `source` plus `id`) is the deduplication key. A fresh id has random bits; a derived id is allowed (§9). |
| `source` | required | `/<layer>/spaces/<space id>[/<segment>…]` or `/<layer>/host[/<segment>…]` (§3). |
| `type` | required | `io.<namespace>.<segment>(.<segment>)+`, each part `[a-z][a-z0-9-]*` (§2). |
| `subject` | optional | A non-empty string, in the type's own terms (a report id, a container name, a transaction signature). |
| `time` | required | UTC, RFC 3339, exactly milliseconds and `Z` (`2026-10-24T09:15:02.481Z`). Its milliseconds MUST equal the id's 48-bit timestamp. It is when the occurrence happened, as well as the producer knows it. |
| `datacontenttype` | required | `"application/json"`. |
| `data` | required | A JSON object (never an array, a scalar or `null`); `{}` when there is nothing to say. Top-level keys match `^[A-Za-z_][A-Za-z0-9_]*$`; values are any JSON. The type's owner defines it (§2). `io.anfs.*` data is a flat object of strings (§1.2). |
| `dataschema` | not used | Producers do not set it in v1; readers ignore it. |
| `actor` | extension, required | Who acted: `<kind>[:<name>]`, the kind `^[a-z][a-z0-9-]*$`, the name one or more characters with no space and no control character (U+0000–U+0020, U+007F–U+009F); at most 128 bytes. `claude:2e9431a0`, `shell:rob`, `podman`, `anfsd`, `user:u7f3a`. Self-asserted on a development host, attested by the publisher in production (§4). |
| `actorclaimed` | extension, optional | The identity a producer claimed when an attesting publisher set a different one (§4): `<actor>[ as [<space id>]/<role>]`, at most 256 bytes. |
| `actorspace` | extension, optional | The space id the actor acted as a member of (`^[a-z0-9]{10}$`). Only together with `actorrole`. |
| `actorrole` | extension, optional | The role the actor acted as, `^[a-z0-9][a-z0-9._-]*$`. It MAY appear without `actorspace` (a role in a space that has no id yet, as the bash emitter writes it). |
| `actorrun` | extension, optional | The run an attesting publisher started the actor in: the run's id, a UUIDv7 (§4). |
| `spacepath` | extension, optional | The space's path when the event was written (`anfs-dev`, `cnfs/apps`). Informational only: the space id in `source` is the identity. |
| `traceparent` | extension, optional | CloudEvents' distributed tracing extension: a W3C Trace Context `traceparent`, `^00-[0-9a-f]{32}-[0-9a-f]{16}-[0-9a-f]{2}$`, its trace id and parent id not all zeros (§10). |
| `tracestate` | extension, optional | A W3C Trace Context `tracestate`, printable ASCII, 1 to 512 characters. Only together with `traceparent`. |
| `anfsv` | extension, required | The integer `1`: this profile's version (§11). |

Layers register further extensions in `REGISTRY.md` (§10); lgat's `commitment` is the first.

### 1.1 Rules for every event

- **No `null` for any attribute or extension.** CloudEvents allows `null` for an unset attribute; this profile requires the attribute to be left out. Inside `data`, `null` is the type owner's choice (anfs's types never use it).
- **No control characters in string attributes.** Attribute and extension strings contain no U+0000–U+001F and no U+007F–U+009F (CloudEvents' String type). Strings inside `data` may contain them, escaped as JSON requires.
- **Member names are unique**, at the top level and inside `data` at every depth. JSON parsers disagree about which of two equal names wins.
- **Integers in `data` stay within ±(2^53 − 1).** A larger integer (a u64 amount, lamports) MUST be written as a string: JavaScript, `jq` and Go's `any` lose precision otherwise.
- **Extensions.** Every top-level member that is not a CloudEvents attribute is an extension. Its name matches `^[a-z0-9]{1,20}$`; its value is a string (no control characters), an integer between −2^31 and 2^31 − 1, or a boolean (CloudEvents' rules, made MUST).
- **At most 4096 bytes.** The event serialized as one line, plus its newline, is at most 4096 bytes: in the spool, on the bus (where the body is the event without the newline, so at most 4095 bytes), by either way in (§6, §7). An event says what changed; detail is read from the item it names. A larger payload is a reference in `data`.
- **One line.** A serialized event contains no CR or LF byte (JSON strings escape them; insignificant whitespace MUST NOT include them).
- **Readers are tolerant.** A reader ignores attributes, extensions and `data` keys it does not know. A reader that meets an `anfsv` it does not know skips the event and logs that once.

### 1.2 `io.anfs` data: flat strings

`data` is any JSON object for the envelope. Flat strings and thin events are anfs's convention for its own namespace, not a rule of the envelope: every `io.anfs.*` type has a flat object of string values (the bash emitter writes nothing else; dispatch rules and templates read top-level strings), and validators hold `io.anfs.*` to it (rule `anfs.data.flat`). Another layer's data is its own JSON object: cnfs maps a `Rails.event` payload, lgat an Anchor event, nested where that is natural. Thin is still the advice for every layer.

## 2. Types and namespaces

- A type is `io.<namespace>.<rest>`. Each namespace is owned by one layer and listed in `REGISTRY.md`: `anfs` (anfs-io), `cnfs`, `lgat`; `rws` and `roteoh` are reserved for theirs. A tool that is not a layer registers a namespace the same way when it publishes types of its own.
- Only a namespace's owner defines types in it and documents them. anfs's catalog is in `REGISTRY.md`; other layers keep theirs in their own repositories and link them from it.
- Inside its namespace the owner chooses the depth. anfs uses `<noun>.<verb>` (`io.anfs.report.filed`); cnfs `<engine>.<noun>.<verb>` (`io.cnfs.partitia.report.filed`); lgat `<program>.<event>` (`io.lgat.escrow.payment.settled`, mapped from Anchor's `PaymentSettled` by a table, not mechanically). Verbs are past tense: an event is a fact.
- **The short type** is what people and rules type: `--type` filters and dispatch rules' `on.type`. It drops `io.anfs.` from anfs's types and `io.` from every other namespace's: `report.filed`, `cnfs.partitia.report.filed`. So that it stays unambiguous, no `io.anfs` type's first segment may be a registered namespace (no `io.anfs.cnfs.…`). NATS subjects do not use the short type (§5).
- A type's data may grow: a new key is compatible. Removing a key or changing its meaning needs a new type.

## 3. Space ids and sources

### 3.1 Space ids

- A space id is 10 characters of `[a-z0-9]` (`^[a-z0-9]{10}$`); readers accept exactly that.
- It is random. anfs generates ids from `a-z2-7` (32^10, about 2^50: `anfs_space_id_new`, `wsm id`); another generator MUST be random over at least that alphabet and length, so ids stay unique across layers without coordination.
- It is made once, never changed and never reused. It survives renames and moves of the space. `host` is reserved and is never a space id.
- It is the scope everywhere: in `source`, in the spool's directory (§6), as the second token of a NATS subject (§5), in `actorspace`.
- A layer's tenant that is a space (an organization in a cnfs app) has a space id by this rule, generated once and stored beside the tenant's own primary key, which stays the app's. Events never carry the app's key as a scope.

### 3.2 Sources

`source` is `/<layer>/spaces/<space id>` or `/<layer>/host`, optionally followed by segments the layer owns:

```
^/[a-z][a-z0-9-]*/(host|spaces/[a-z0-9]{10})(/[A-Za-z0-9._~-]+)*$
```

- `<layer>` names the producing layer: `/cnfs/spaces/m2vq7cfr4k/partitia` is cnfs's partitia app in that space. It SHOULD be the type's namespace; it differs when one layer's producer emits another's type (anfs's `anfs report`, run in a cnfs space, still writes `/anfs/spaces/<id>`). Validators check the grammar, not the registry, so a new layer needs no validator release.
- **The scope** of an event is the space id in its source, or `host`, whatever the layer. Where an event happened is a space, which every layer shares.
- anfs writes exactly `/anfs/spaces/<id>` and `/anfs/host`.

## 4. Actors: asserted in development, attested in production

- `actor`, `actorspace` and `actorrole` say who acted and as what.
- **On a development host they are self-asserted.** Any spool writer or bus publisher sets them, and nothing checks them. That is acceptable only on one person's development host, and only there.
- **In production the publisher attests the actor.** The component that puts an event on a production bus (cnfs's outbox relay, lgat's indexer, a bridge) MUST set `actor`, `actorspace` and `actorrole` from its own authenticated context (the session or job that made the outbox row, the indexer itself), never from a caller's payload or claim. The bus authenticates the publisher (a NATS user per publisher, §7). Consumers trust `actor` only as far as they trust its publisher. Producers in production MUST NOT design around self-asserted identity.
- **`actorclaimed`.** An attesting publisher that receives a different claim keeps it in `actorclaimed`, so the record shows the attempt instead of erasing it. Its value is the claimed actor, followed by ` as <actorspace>/<actorrole>` when the claim named a role (`shell:roberto as tkj76y3nfx/person`; ` as /<role>` for a role with no space). It is present only when the claim differs from the attested identity written the same way (rule `actorclaimed.same`). Readers show `actor`; `actorclaimed` is evidence, never an identity.
- **`actorrun`.** A publisher that started the actor in a run (anfsd's supervisor) MAY name the run's id. A consumer that guards against an agent reacting to its own events uses `actorrun` when present.
- On the development bus, a layer's publisher user (§7.2) attests the layer, since it can publish only its layer's subjects; it does not attest the actor.
- Approvals never come from an event's actor: they come through anfs's inbox.

## 5. Subjects on NATS

- **The binding is CloudEvents' NATS protocol binding, structured mode only.** The message body is the event (the spool line without its newline); the header `Content-Type` is `application/cloudevents+json`. Binary mode (`ce-*` headers) is not published, and a consumer that receives it MAY refuse it.
- **Every publish carries the header `Nats-Msg-Id: <id>`** (JetStream's deduplication).
- **The subject is `anfs.<scope>.` followed by the type without `io.`**, one shape for every layer, anfs included:

| Event | Subject |
| --- | --- |
| `io.anfs.report.filed` in space `tkj76y3nfx` | `anfs.tkj76y3nfx.anfs.report.filed` |
| `io.anfs.container.started` on the host | `anfs.host.anfs.container.started` |
| `io.cnfs.partitia.report.filed` in space `m2vq7cfr4k` | `anfs.m2vq7cfr4k.cnfs.partitia.report.filed` |
| `io.lgat.escrow.payment.settled` in space `q7mwx4ne2r` | `anfs.q7mwx4ne2r.lgat.escrow.payment.settled` |

- The first token names the bus and its protocol, the second the scope, the third the type's namespace. The mapping is mechanical both ways: subject = `anfs.<scope>.` + type without `io.`; type = `io.` + the subject from its third token; scope = its second token. No registry is needed to read a subject.
- **Filters:** a space `anfs.<id>.>`; a namespace `anfs.*.<namespace>.>` (`anfs.*.cnfs.>`, `anfs.*.anfs.>`); a kind `anfs.*.anfs.report.>`; one type in one space `anfs.<id>.anfs.report.filed`.
- **One stream, `ANFS`, on `anfs.>`, holds every layer.** JetStream deduplicates by `Nats-Msg-Id` per stream, so one stream is what makes "deduplicate by id" hold across layers and across both ways in. A space's grant or filter stays one subject (`anfs.<id>.>`); a layer's publish permission is one subject (`anfs.*.<layer>.>`) that cannot match another layer's types.
- In production a layer runs its own NATS (or none); the same scheme carries over unchanged.

## 6. The first way in: the spool

On a host with anfs, a producer in any language appends to the spool and anfsd ships it to the bus. The producer needs no NATS client.

- **Path:** `${XDG_STATE_HOME:-$HOME/.local/state}/anfs/events/<scope>/<YYYY-MM>.jsonl`, `<scope>` the source's space id or `host`, the month the event's `time` (UTC). The directory is created when missing (mode 0755), the file with mode 0644. `<YYYY-MM>.bus.jsonl` beside it is the recorder's (§7.3): producers never write it.
- **The scope** is the top-level `id:` of the nearest `space.yml` at or above the producer's working directory (a Rails app's root); with none, or none with an id, the host scope.
- **One event per line**, UTF-8, ended by `\n`, in the **canonical form**: compact JSON (no whitespace outside strings), `data` the last member, `anfsv` written as `"anfsv":1`. The bash reader relies on it: it reads the attributes before the first `,"data":`. Attribute order before `data` is free; anfs writes `specversion`, `id`, `source`, `type`, `subject`, `time`, `datacontenttype`, `actor`, `actorclaimed`, `actorspace`, `actorrole`, `actorrun`, `spacepath`, `traceparent`, `tracestate`, other extensions, `anfsv`, `data`. Ruby's `JSON.generate`, Python's `json.dumps(event, separators=(",", ":"))` and Go's `encoding/json` produce it from a map built in that order.
- **Atomic append, no locks:** the whole line in one `write(2)` on a descriptor opened with `O_APPEND`, at most 4096 bytes, on a local filesystem (not NFS). Bash: `cat >> "$f" <<< "$line"` (`printf` splits the write on macOS). Go: `os.OpenFile(f, os.O_WRONLY|os.O_APPEND|os.O_CREATE, 0o644)` then one `Write`. Ruby: `File.open(f, File::WRONLY | File::APPEND | File::CREAT, 0o644) { _1.syswrite(line) }`. Python: `os.write(fd, line)` on `os.open(f, os.O_WRONLY | os.O_APPEND | os.O_CREAT, 0o644)`.
- **Never** rewrite, truncate or delete a line. The files are kept. An emit never fails its caller: a refused or failed write is a warning.
- A line that is a valid event but not canonical is still shipped; only the bash reader needs the canonical form, so validators report it as `noncanonical`.

## 7. The second way in: publishing to NATS

Publishing directly is first class: in production a cnfs app publishes to NATS (or its outbox relay does), so on a development host it can build against the same interface. A producer uses one way in per event; using both for the same id is harmless (deduplicated) and pointless.

### 7.1 The publish contract

The same on the development bus and on a production one:

- The subject is `anfs.<scope>.<type without io.>` (§5) and agrees with the event's `source` and `type`.
- The body is the event, valid, at most 4095 bytes; it SHOULD be the canonical form (§6).
- Headers `Content-Type: application/cloudevents+json` and `Nats-Msg-Id: <id>`.
- **A JetStream publish that awaits its acknowledgement**, retried with the same body until acknowledged. A core NATS publish with no acknowledgement is at most once and does not meet §8. An outbox row is marked sent only after the acknowledgement.

### 7.2 The publisher user per layer (development bus)

- core's `core/nats` makes one NATS user per layer that publishes directly, named after its namespace (`cnfs`, `lgat`), its password in `~/.local/state/nats/users/<layer>` (mode 0600), from a list of layers in nats's settings.
- Its permissions: publish `anfs.*.<layer>.>` only (never `anfs.*.anfs.>` or another layer's subjects); subscribe `anfs.>` and `_INBOX.>`; the JetStream API a consumer needs on `ANFS` (its own durable consumers, fetch, ack); never stream management.
- It is distinct from `anfsd` (ships the spool, manages the stream, publishes everything), `admin` (the person's) and `reader` (subscribe only).

### 7.3 The recorder (development host)

The recorder keeps "the files are the complete record" true for both ways in.

- It is anfsd's durable consumer `anfsd-recorder` on `ANFS`, filter `anfs.>`, delivering all messages when first made, explicit acknowledgement.
- For each message it validates the event against this profile and checks the subject against it. An invalid or mismatched message is logged once (rule and subject) and terminated: it stays in the stream, absent from the record.
- When the event's id is already in its month's files (`<scope>/<YYYY-MM>.jsonl` or `<scope>/<YYYY-MM>.bus.jsonl`, scope and month from the event), the message is acknowledged and nothing is written. This is how everything shipped from the spool is skipped, with no header to trust or forge.
- Otherwise the recorder appends the event's canonical form (the same members and values; whitespace removed and `data` moved last if the publisher did not) to `<scope>/<YYYY-MM>.bus.jsonl` in one `O_APPEND` write, then acknowledges.
- Recorded events go to their own file so the shipper never publishes them again: it reads only `<YYYY-MM>.jsonl`. Past the deduplication window a second publish would be a duplicate on the bus.
- Readers of the record read both files of a month. Each file is in append order; a reader that wants time order sorts by `id`.
- The record is complete within the stream's retention (90 days on a development host); a recorder that is down longer loses what the stream dropped meanwhile. In production the record is the layer's own store.

## 8. Delivery

- **At least once**, end to end, by either way in. Deduplication is by `id`: JetStream drops a repeat inside its window (24 hours on a development host), the recorder by its files, and every consumer MUST be idempotent by `id` beyond that (it records what it has handled).
- **Distinct occurrences have distinct ids.** The same fact sent again has the same id. A new fact about the same thing (lgat's `finalized` after `confirmed`) is a new event with a new id, correlated with the first by `subject` and `data`.
- **Ordering:** per scope, in append order, for the spool on one host; none across scopes or between the two ways in. `id` order is time order to the millisecond. A consumer that needs causal order uses data (a slot, a sequence).
- Consumers acknowledge after handling. The log is append-only: no event is retracted; a correction is a new event.

## 9. Derived ids

A producer that sends the same occurrence more than once MUST send the same id, or deduplication fails. There are two cases.

- **Sending a stored event again** (an outbox row, a retry): the event, id included, is stored with its record when the record is made (the id MAY be a fresh random UUIDv7 then), and every send uses the stored id. It is never generated again: a relay that rebuilds the event from the row reads the id from the row.
- **Deriving an event again from a source of truth that is not the event** (lgat's indexer reading the chain again after a crash or into a new database, a rebuild from an external log): the id MUST be derived from a stable source key, deterministically, as follows.

The derivation:

1. The key is a UTF-8 string of fields joined by `|`: the full type, then the fields that identify the occurrence in the source, then anything that makes it a distinct fact. For example `io.lgat.escrow.payment.settled|<signature>|<instruction index>|<event index>|finalized`. The type's owner documents the key for each type in its catalog.
2. `h` is the first 74 bits of SHA-256 of the key (the most significant bit of the digest's first byte first).
3. The id is the UUIDv7 whose 48-bit `unix_ts_ms` is the event's `time` in milliseconds, whose 12-bit `rand_a` is the first 12 bits of `h`, and whose 62-bit `rand_b` is the remaining 62 bits of `h`, with the version (`7`) and variant (`10`) bits set as RFC 9562 says.

The derivation departs from RFC 9562's SHOULD that those bits be random, knowingly: it keeps "the id is a UUIDv7, ordered by time" and makes a re-derivation idempotent. A producer MAY store a derived id (a cache), but only ever the derived value: a fresh id stored in the producer's own database does not meet this rule, since that database can be lost or rebuilt. The examples `lgat-payment-confirmed` and `lgat-payment-finalized` carry their keys in the manifest, so every implementation checks its derivation against the same bytes; `tools/check --derive-id TIME KEY` prints a derived id.

## 10. Extensions

- The profile's own extensions are part of v1: `actor`, `actorclaimed`, `actorspace`, `actorrole`, `actorrun`, `spacepath`, `traceparent`, `tracestate`, `anfsv`.
- A layer registers an extension in `REGISTRY.md` (name, owner, type, values, the namespaces that carry it) by a pull request to this repository. Registering one is a minor version of this spec (`v1.1.0`), never an `anfsv` change, because v1 readers ignore extensions they do not know.
- Names carry no prefix; registering them is what keeps them apart. A registered name MUST NOT be one that CloudEvents' documented extensions define (`authtype`, `authid`, `authclaims`, `sequence`, `dataref`, `partitionkey`, `recordedtime`, `expirytime`, …), except `traceparent` and `tracestate`, which this profile uses with their CloudEvents meaning; the list (References) is checked when registering.
- **Tracing.** `traceparent` and `tracestate` are CloudEvents' distributed tracing extension, with its meaning: the trace context of the code that produced the event, so a consumer's work joins the producer's trace. They travel in the event, which carries them across the spool, an outbox and a job boundary. A publisher MAY also copy `traceparent` into a NATS header for its own instrumentation; consumers read the attribute.
- **`commitment` (lgat)**, the worked example of a layer's extension: a string, `confirmed` or `finalized` (Solana's commitment levels; `processed` is not published), on `io.lgat.*` types only. Money moves only on `finalized`; a `confirmed` that never becomes `finalized` simply has no successor.

## 11. Versioning and compatibility

- `anfsv` is the envelope's version, `1`. This repository's tags are semantic versions: every `v1.y.z` is `anfsv` 1; a `v2.0.0` would be `anfsv` 2.
- anfs-io owns the versions, as the anfs-dev space's: the maintainer drafts, the person approves and tags. Layers own their namespaces' types and version them in their own repositories without a release here; they change this spec (a namespace, an extension) by a pull request to `REGISTRY.md`, which is a minor version.
- **No `anfsv` change** for what every v1 reader already tolerates: a new type, a new data key, a registered extension, a new namespace.
- **An `anfsv` change** for anything that makes a v1-valid event invalid or changes an attribute's meaning: `data` other than an object, a new source form, a larger size limit.
- **Deploy order on a change:** readers and shippers first, writers last. A writer writes the lowest `anfsv` that expresses the event.

## 12. Conformance

The examples in `examples/` are the conformance fixtures and the one source of truth. Each implementation vendors a tag's copy with a `VERSION` file naming the tag (runtime `internal/cloudevent/testdata/spec/`, core `tests/fixtures/events-spec/`, the cnfs gem's `spec/fixtures/`); updating is copying a newer tag's.

- **`examples/valid/*.jsonl` and `examples/invalid/*.jsonl`:** one serialized event per file, as one spool line ended by `\n` (the invalid `two-lines` example is the exception it tests). Every invalid example breaks one rule.
- **`examples/index.json`**, the manifest: `{"version", "examples": [...]}`, each entry `{file, valid, rule, schema, subject, key, about}`:
  - `rule`: for an invalid example, the code of the rule it breaks (the table below); for a valid one `null`, or `noncanonical` when it is valid but not in the spool's canonical form.
  - `schema`: for an invalid example, whether the JSON Schema alone catches it (`true`) or only the line rules do (`false`); `null` for a valid one.
  - `subject`: for a valid example, the NATS subject it is published on (§5), so the subject mapping is tested too; `null` for an invalid one.
  - `key`: for a valid example with a derived id, the source key it is derived from (§9); otherwise `null`.
  - `about`: what the example shows, for people.
- **`schema/event.v1.json`:** JSON Schema draft 2020-12. It checks every rule a schema can express. It cannot check the line rules: the size, one line, `time` equal to the id's milliseconds, integers within 2^53 − 1, unique member names, `actorclaimed` differing from the attested identity, the actor's byte length (it counts characters), and the canonical form. Its patterns are the subset of ECMA-262 that Go, Python and Ruby read alike; a Ruby implementation anchors them to the whole string (`\A`, `\z`), as `json_schemer` does.
- **A conforming validator** applies the schema and the line rules, and agrees with the manifest on every example: each valid example passes (reporting `noncanonical` where the manifest says so), each invalid one fails, and its subject and derived id match.
- **This repository's check:** `tools/check` (Python 3, standard library) checks the manifest against the files, the line rules, the subjects and the derived ids, and runs `check-jsonschema` (pinned with hashes in `tools/requirements.txt`) over every example: valid ones pass, invalid ones with `schema: true` fail, invalid ones with `schema: false` pass. CI runs it on every push.
- Implementations: runtime's `anfsd validate` (and core's `anfs events validate`, which runs it) is the runnable validator on a development host; runtime's Go tests, core's bats and the cnfs gem's RSpec run the vendored examples.

| Rule | What it requires | Schema |
| --- | --- | --- |
| `specversion.required`, `specversion.value` | `specversion` present, `"1.0"` | yes |
| `id.required`, `id.uuidv7` | `id` present, a lower-case hyphenated UUIDv7 | yes |
| `source.required`, `source.form` | `source` present, the grammar of §3.2 | yes |
| `type.required`, `type.form` | `type` present, `io.<namespace>.<segment>(.<segment>)+` | yes |
| `time.required`, `time.form` | `time` present, UTC with milliseconds and `Z` | yes |
| `time.id` | `time`'s milliseconds equal the id's timestamp | no |
| `datacontenttype.required`, `datacontenttype.value` | present, `"application/json"` | yes |
| `actor.required`, `actor.form` | `actor` present, `<kind>[:<name>]`, at most 128 bytes | yes (bytes: no) |
| `actorclaimed.form` | `<actor>[ as [<space id>]/<role>]` | yes |
| `actorclaimed.same` | `actorclaimed` differs from the attested identity | no |
| `actorspace.role` | `actorspace` only with `actorrole` | yes |
| `traceparent.form` | W3C `traceparent`, version 00, ids not all zeros | yes |
| `tracestate.traceparent` | `tracestate` only with `traceparent` | yes |
| `anfsv.required`, `anfsv.value` | `anfsv` present, the integer `1` | yes |
| `data.required`, `data.object` | `data` present, a JSON object | yes |
| `data.key` | top-level data keys `^[A-Za-z_][A-Za-z0-9_]*$` | yes |
| `data.int53` | integers in data within ±(2^53 − 1) | no |
| `anfs.data.flat` | `io.anfs.*` data values are strings | yes |
| `attr.null` | no attribute or extension is `null` | yes |
| `attr.control` | no control characters in attribute strings | yes |
| `subject.empty` | `subject`, when present, is not empty | yes |
| `ext.name` | extension names `^[a-z0-9]{1,20}$` | yes |
| `ext.value` | extension values a string, a 32-bit integer or a boolean | yes |
| `commitment.value`, `commitment.namespace` | `confirmed` or `finalized`, on `io.lgat.*` only | yes |
| `size.max` | at most 4096 bytes with the newline | no |
| `line.one` | no CR or LF in the serialized event | no |
| `json.unique` | member names unique | no |
| `noncanonical` (a flag, not an error) | compact, `data` last, `"anfsv":1` | no |

## References

- CloudEvents 1.0.2: https://github.com/cloudevents/spec/blob/ce%40v1.0.2/cloudevents/spec.md
- CloudEvents JSON format: https://github.com/cloudevents/spec/blob/main/cloudevents/formats/json-format.md
- CloudEvents NATS protocol binding: https://github.com/cloudevents/spec/blob/main/cloudevents/bindings/nats-protocol-binding.md
- CloudEvents documented extensions: https://github.com/cloudevents/spec/tree/main/cloudevents/extensions
- CloudEvents distributed tracing extension: https://github.com/cloudevents/spec/blob/main/cloudevents/extensions/distributed-tracing.md
- W3C Trace Context: https://www.w3.org/TR/trace-context/
- RFC 9562 (UUIDs, version 7): https://www.rfc-editor.org/rfc/rfc9562
- RFC 3339 (timestamps): https://www.rfc-editor.org/rfc/rfc3339
- RFC 2119 (key words): https://www.rfc-editor.org/rfc/rfc2119
- The plan this spec comes from: https://github.com/anfs-io/docs/blob/main/units/agents/05-spec/plan.md
