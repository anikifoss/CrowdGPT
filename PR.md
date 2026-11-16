# Use the coordinator's batch size and report the number of optimizer steps actually run

## Summary

The coordinator merges the weight changes volunteers upload. To compare them fairly it needs two
facts from each volunteer: the batch it trained with, and how many optimizer steps it actually
completed. This PR makes the client take the first from the task it was given and report the second.

### What changes

- The client uses the batch size the coordinator sends (`targetAccumSteps`) exactly as given. If the
  coordinator sends none, it falls back to the original formula
  `round(131072 / (batch_size * seq_len))`.
- It reports `optimizerSteps` and `accum` in the submit request. The coordinator uses them to scale
  a delta by steps-taken vs steps-asked-for. A client that does not send them is treated as a legacy
  client and its delta is applied unscaled, so old and new coordinators keep working with this
  client.
- Nothing else in the payload is required: unknown fields are ignored, so this client runs unchanged
  against an older coordinator, and it adds no new command-line flags.

### Why the client no longer derives the batch itself

- An intermediate version of this work derived the batch from a step budget and re-derived the step
  count from it, which turned a 500-step round into two enormous steps and cost about a quarter of
  the throughput. That path is gone: the coordinator decides the batch, the client uses it and
  reports what happened.

### Verification

- 245 client tests pass (16 of them cover this change); one pre-existing failure in the ROCm
  memory-defaults test is unrelated to this branch.
- Run end-to-end against the coordinator on four AMD GPUs for three rounds: submissions were
  accepted, the cross-check passed, and the merged model improved on held-out data each round
  (cross-entropy 12.20 -> 6.40 nats/token).
