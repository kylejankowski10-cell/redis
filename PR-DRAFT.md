# Draft upstream contribution (NOT submitted; submission is Kyle's call)

## Proposed title

Propagate consumer creation from zero-effect XCLAIM and XAUTOCLAIM

## Proposed body

### Summary

`XCLAIM` and `XAUTOCLAIM` create the claiming consumer on the primary
before processing, suppress their own propagation, and emit only
per-entry synthesized effects. When an execution transfers no entry to
the claiming consumer, no propagated form names that consumer, so the
consumer exists on the primary but never on the replica: a permanent
consumer-registry divergence visible via `XINFO CONSUMERS` on a fully
converged replica. The same history also diverges across recovery
channels: the phantom consumer survives an RDB restart but disappears
on AOF replay.

Minimal reproducer (no pending entries required). Setup: a replica is
already attached to the primary and fully synchronized before the
sequence begins (REPLICAOF issued, replication link up, initial sync
complete):

```
# on the replica, beforehand: REPLICAOF <primary-host> <primary-port>
# then, on the primary:
XGROUP CREATE s g 0 MKSTREAM
WAIT 1 5000           # group creation converged on the replica
XCLAIM s g cNew 0 1-1
WAIT 1 5000           # primary and replica fully converged
XINFO CONSUMERS s g   # primary: cNew; replica: (empty)
```

The gap also fires when the call does emit an effect that does not
name the new consumer: a purge-only claim (synthesized XCLAIM naming
the previous owner, or XACK) and a LASTID-only claim (XGROUP SETID)
both converge every other aspect of state while still losing the
consumer. The trigger is therefore not "no effect was propagated" but
"no propagated operation establishes the new consumer."

### Relation to prior fixes

This is the claim-path analogue of the inconsistency repaired for
`XREADGROUP`: issue #7140 and PR #7526 introduced
`XGROUP CREATECONSUMER` propagation for the NOACK case, and PR #14963
extended it to the empty-read case, propagating creation whenever no
synthesized XCLAIM would implicitly create the consumer. The claim
commands never received the equivalent guard.

### Changes

`src/t_stream.c`, in `xclaimCommand` and `xautoclaimCommand`: track
whether the command created the claiming consumer and whether any
propagated operation establishes it (set only where a synthesized
XCLAIM names the claiming consumer as the transfer target). When the
consumer was created but not established, propagate
`XGROUP CREATECONSUMER` via the existing
`streamPropagateConsumerCreation` helper, mirroring `XREADGROUP`.
No redundant effect is emitted when a transfer already establishes
the consumer.

### Tests

Two replication tests appended to `tests/unit/type/stream-cgroups.tcl`:

- "Zero-transfer claims propagate new consumers": XCLAIM against a
  nonexistent ID and an XAUTOCLAIM empty scan by new consumers; both
  servers must list {cAuto cNew} with an empty PEL.
- "Purge-only claims propagate new consumers": tombstone purges by new
  consumers via XCLAIM and XAUTOCLAIM; registries must match on both
  servers after each step.

Both tests fail on the unpatched build at the replica assertion and
pass with this change. The full `unit/type/stream-cgroups` file
passes unchanged on the patched build (185/185), so no existing
expectation about consumer counts or creation timing is affected.

## Framing rules (internal, do not submit)

- Describe the externally observable consistency failure only. PEL
  behavior was correct throughout; do not speculate about impact on
  pending-message processing.
- Credit the #7140/#7526/#14963 lineage explicitly; the argument is
  that an established repair principle was not carried to two sibling
  commands, not that a new consistency requirement is proposed.
