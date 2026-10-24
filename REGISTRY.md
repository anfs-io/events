# Registry

What makes names unique across layers: the type namespaces, the extensions, and anfs's own type catalog. A layer adds or changes its entry by a pull request to this file; that is a minor version of the spec (SPEC.md §10, §11). Validators check the grammar, not this file, so a new entry needs no validator release.

## Namespaces

A type is `io.<namespace>.…` (SPEC.md §2). Only the owner defines types in its namespace and documents them.

| Namespace | Owner | Shape | Catalog |
| --- | --- | --- | --- |
| `io.anfs` | anfs-io (the anfs-dev space) | `<noun>.<verb>` | below |
| `io.cnfs` | cnfs (the cnfs space) | `<engine>.<noun>.<verb>`, e.g. `io.cnfs.partitia.report.filed` | in cnfs's events gem; linked here when it is published |
| `io.lgat` | lgat (the lgat space) | `<program>.<event>`, e.g. `io.lgat.escrow.payment.settled` | in lgat's indexer; linked here when it is published |
| `io.rws` | RWS | reserved | |
| `io.roteoh` | Roteoh | reserved | |

- A namespace is `[a-z][a-z0-9-]*`. A tool that is not a layer (a tracker, Jev) registers one the same way when it publishes types of its own.
- No `io.anfs` type's first segment may be a registered namespace (`cnfs`, `lgat`, `rws`, `roteoh`), so the short type stays unambiguous (SPEC.md §2).
- A layer's catalog lists, per type: the scope it is written in, `subject`, `actor`, the `data` keys with their JSON types, and for a re-derived type its source key (SPEC.md §9).

## Extensions

| Name | Owner | Type | Values | On |
| --- | --- | --- | --- | --- |
| `actor` | anfs-io | String | `<kind>[:<name>]`, at most 128 bytes; required | every event |
| `actorclaimed` | anfs-io | String | `<actor>[ as [<space id>]/<role>]`: the claim an attesting publisher replaced | every namespace |
| `actorspace` | anfs-io | String | a space id; only with `actorrole` | every namespace |
| `actorrole` | anfs-io | String | `^[a-z0-9][a-z0-9._-]*$` | every namespace |
| `actorrun` | anfs-io | String | the run's id, a UUIDv7, set by the publisher that started the run | every namespace |
| `spacepath` | anfs-io | String | the space's path when written; informational | every namespace |
| `traceparent` | CloudEvents (distributed tracing) | String | W3C Trace Context `traceparent` | every namespace |
| `tracestate` | CloudEvents (distributed tracing) | String | W3C Trace Context `tracestate`, at most 512 characters; only with `traceparent` | every namespace |
| `anfsv` | anfs-io | Integer | `1`; required | every event |
| `commitment` | lgat | String | `confirmed` or `finalized` (Solana's commitment levels; `processed` is not published) | `io.lgat.*` only |

To register an extension: its name (`^[a-z0-9]{1,20}$`, not a name CloudEvents' documented extensions define), owner, CloudEvents type (String, Integer or Boolean), values, and the namespaces that carry it; add it to `schema/event.v1.json` (`properties`) when its values can be checked, with a valid and an invalid example.

## anfs's catalog (`io.anfs`)

Every `io.anfs` type's data is a flat object of string values (SPEC.md §1.2); a key is left out when it has no value. The scope is the space id of the space the event concerns, or `host`. The types are core's `ANFS_EVENT_TYPES` (`lib/anfs/events.sh`).

### Reports (core: `anfs report`)

| Type | Scope | Subject | Actor | Data |
| --- | --- | --- | --- | --- |
| `report.filed` | the space of the report file; `host` for a GitHub issue | the report id (`r-20261023-1`) | who ran `anfs report` | `package` (when given), `title`, `to` (`file` or `github`), `repo` (for `github`) |
| `report.closed` | as `report.filed` | the report id | who closed it | `commit` (the fixing commit), `source` (the source alias it landed in) |

### Sources (core: `anfs src update`)

| Type | Scope | Subject | Actor | Data |
| --- | --- | --- | --- | --- |
| `source.updated` | `host` | the source's alias (`core`) | who ran the update, or `anfsd` when a merge it performed moved the source | `from`, `to` (the HEAD commit before and after) |

### Runs (anfsd: the supervisor)

A run is one agent started by a dispatch rule. Scope: the rule's space. Subject: the run id (a UUIDv7). Actor: `anfsd`. `spacepath` is set. The states are A2A's task states.

| Type | Data |
| --- | --- |
| `run.submitted` | `rule`, `role`, `harness`, `event` (the id of the event that matched), `job` (when the rule makes one) |
| `run.rejected` | `rule`, `reason` (`stale`, `rate`, `queue`, `loop`), `event` |
| `run.working` | `rule`, `harness`, `attempt`, `job` and `branch` (when the rule makes a job); after the harness ended, `step` (`verify`, `merge`, `install`, `push`: the landing step under way) |
| `run.auth-required` | `rule`, `job`, `step` (the landing step waiting), `approval` (the approval id): the run waits on the person (A2A's `auth-required`); not written when the approval is answered `auto` |
| `run.completed` | `rule`, `job`, `commits` (`<repo>:<count>` per repo, space-separated; `?` when unknown), `summary`, `landed` and `skipped` (the landing steps done and skipped, comma-separated), and the run usage below |
| `run.failed` | `rule`, `job`, `reason` (`exit`, `timeout`, `spawn`, `orphaned`, a preparation failure, or a landing step's: `verify`, `merge`, `install`, `push`, `expired`), `exit` (the exit status, when there is one), `detail`, `attempt`, `retry` (`<n> of <m>` when it will be retried), and the run usage when known |
| `run.canceled` | `rule`, `job`, `by`, `reason` (`refused`: its approval was refused) |

**Run usage** on `run.completed`, and on `run.failed` when known; each key is left out when the harness does not report it:

| Key | Meaning | OpenTelemetry generative-AI convention |
| --- | --- | --- |
| `model` | the model the harness reports it used, else the rule's | `gen_ai.response.model` |
| `input_tokens` | every prompt token the model processed, cached or not (decimal) | `gen_ai.usage.input_tokens` |
| `output_tokens` | output tokens (decimal) | `gen_ai.usage.output_tokens` |
| `cost_usd` | the harness's own cost figure in US dollars, an estimate (decimal, `1.8423`) | |

### Approvals (anfsd: the lander)

An approval is a step of landing a run's work (or a job worked by hand) that waits on who answers it. Scope: the run's space, `host` for a request with no run. Subject: the approval id (a UUIDv7). Actor: `anfsd`, or the answerer for `approval.given` and `approval.refused` (with `actorspace` and `actorrole` when it acts as a role). Every type has `step` (`merge`, `install`, `push`, `policy`), and `run`, `rule`, `job` when it comes from a run.

| Type | Data, besides the above |
| --- | --- |
| `approval.requested` | `to` (`person`), `policy`, `trigger` (the id of the event that started the run), `report`; per step: `repos`, `bases`, `heads`, `commits`, `files` (merge: `<repo>=<value>`, comma-separated), `packages` (install), `repos`, `remote` (push), `was`, `now` (policy) |
| `approval.given` | `via` (`cli`, `ui`, `auto`), `through` (`install` or `push`: the later steps given with it), `from` (the approval it was given through) |
| `approval.refused` | `via`, `reason` |
| `approval.expired` | `reason` (`moved`, `diverged`, `gone`, `canceled`, `superseded`), `detail` |
| `approval.done` | `result` |
| `approval.failed` | `reason` (the step, or `interrupted`), `detail` |

### Inbox (anfsd: unhandled work)

| Type | Scope | Subject | Actor | Data |
| --- | --- | --- | --- | --- |
| `inbox.dismissed` | the rule's space | the run id | the answerer | `run`, `rule`, `trigger`, `reason`, `via` |

### Containers and pods (anfsd: the podman follower)

Scope: `host`. Actor: `podman`. Subject: the container's or pod's name.

| Type | Data |
| --- | --- |
| `container.created`, `container.started`, `container.stopped`, `container.removed`, `container.paused`, `container.unpaused` | `id` (12 hex characters), `image`, `pod` (the pod's id, 12 hex characters, when in one), `pcm` (the pcm service, from the `io.pcm.name` label) |
| `container.died` | as above, and `exit` (the exit status) |
| `container.health` | as above, and `health` (`healthy`, `unhealthy`, `starting`); written only when the container's health changes |
| `pod.created`, `pod.started`, `pod.stopped`, `pod.removed` | `id` (12 hex characters) |
| `podman.connected` | none (`{}`); no subject |
| `podman.disconnected` | `error`; no subject |
