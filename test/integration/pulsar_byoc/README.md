# BYOC integration suite

End-to-end tests for the **Bring Your Own Compute** Pulsar feature
(`enable_pulsar_byoc: true`). They drive Galaxy's `PulsarByocManager` against
a real pulsar-relay subprocess, which is in turn backed by a real Keycloak
container.

## Prerequisites

- Docker (and either `docker compose` v2 or the legacy `docker-compose`).
- A local checkout of [`pulsar-relay`](https://github.com/galaxyproject/pulsar-relay)
  reachable via one of:
  - `PULSAR_RELAY_REPO=/path/to/pulsar-relay`
  - `~/src/pulsar-relay` (default)

  The relay is run from source so the suite exercises the in-tree code —
  this lets BYOC-side changes to the relay (pair-issuance, topic ACLs,
  chain-scoped revocation) be verified before they're cut into a release.

If either prerequisite is missing the suite skips cleanly. See `RUNBOOK.md`
for the full prerequisite checklist — a skip is easy to mistake for a pass.

## Running

The suite is gated on the `e2e` marker:

```
./run_tests.sh -unit "test/integration/pulsar_byoc -m e2e"
```

Or directly:

```
pytest test/integration/pulsar_byoc -m e2e -v
```

Typical wall clock: ~60 s (Keycloak boot + relay startup + device-flow +
tool execution).

## What's exercised

`test_complete_registration_against_real_relay`
- RFC 8628 device flow against Keycloak with `pair=true`.
- `HttpRelayClient.create_or_verify_topic` (from `pulsar-relay-client`) for each of the three BYOC topic names, against the live relay.
- Topic ownership matches the BYOC user.
- Re-running the create-or-verify loop is idempotent.
- The primary and secondary refresh tokens rotate on independent chains —
  replaying the rotated primary kills only the primary's chain, the
  secondary keeps refreshing cleanly.

`test_byoc_topic_pinning_against_real_relay`
- After a BYOC user pins its topics, the bootstrap admin creating the same
  bare names gets distinct topics under its own owner; the BYOC user's
  records stay owned by them (per-user topic namespacing).

`test_framework_tool_runs_via_byoc`
- A framework tool submitted through Galaxy is routed by TPV to the
  `pulsar_byoc` destination, dispatched to a real Pulsar subprocess via the
  relay, and returns `state=ok`.
- The job's `destination_params` carry the resource id and manager name
  that TPV injected.

## Files

- `docker-compose.yml` — Keycloak only. The relay is a subprocess.
- `conftest.py` — `keycloak` (session-scoped) and `relay_against_keycloak`
  (per-test) fixtures.
- `test_byoc_e2e.py` — registration and topic-ownership tests.
- `test_byoc_tool_execution.py` — full tool-execution stack: a Pulsar daemon
  configured against the relay plus in-process Galaxy, asserting a tool runs
  end-to-end through the multi-tenant BYOC runner.
- `RUNBOOK.md` — how to run the suite and how to triage failures.