# Trio implementation brief for the next removed Pod experiments

**Handoff message, 2026-10-15.** Please build a step-by-step research runner in Trio/OmnipodKit for an active Pod that has been physically removed and is no longer supplying therapy. The first question is whether enablement alone triggers the fault. If it does not, the second question is whether the recovered active-Pod G7 operation produces sustained glucose reception while the Pod remains in Manual mode.

Use the existing [bench plan](BENCH_EXPERIMENTS.md) as the experiment protocol. This brief specifies the next build, not an implemented feature or a proven pairing procedure. Keep normal activation and treatment behavior unchanged. The deliverable is a reviewable implementation, offline tests, a verified export, and an explicit statement of whether controller isolation is actually supported.

## First resolve how the bench Pod remains reachable

Inspect the actual installed Trio/OmnipodKit revisions and replacement/deactivation path before choosing the architecture. Explain how the removed Pod can remain active and retain its valid communication context while therapy continues independently. Do not assume that “Previous Pod” can communicate, that Trio supports two Pods, or that a deactivated Pod can be reused.

Use a separate bench context with its own target allowlist, persistence and command queue. Match private lot/sequence identity, BLE association and secure-session context; the displayed application address alone is insufficient. Require an explicit off-body confirmation before enabling bench writes. Enforce the target check at the send boundary, not only in the screen. Do not copy experimental glucose, delivery/history or settings into live therapy state, HealthKit or treatment calculations. Prevent unrelated/background writers from targeting the bench Pod.

If that separation cannot be demonstrated, deliver the offline runner and capture improvements with physical bench writes disabled. Report the concrete missing handoff capability; do not bypass ordinary therapy lifecycle requirements to obtain a test target.

## Build these separate actions

Default to a dry run with a fake transport. Physical writes require a deliberately armed bench session and a fresh baseline. Each action saves its result before another action can start. There is no “run all” control.

| Action | Required behavior |
|---|---|
| Start session and export baseline | Verify the target and export, then capture 10 minutes of status at ordinary app-like cadence. Save firmware/BLE versions, Pod age/remaining service time, faults, delivery mode, readiness, driver and sensor state. Firmware 9.0.4 / BLE 5.0.2 is the current research target; a different version requires a reviewed plan. |
| Reconnect without programming | Reconnect to the same bench Pod and repeat baseline reads. Assert that reconnect/startup sends no setup, identity or enablement writes. |
| Read target profile | Use the recovered group: GET `3.1`, AID status `3.12`, legacy status page 7. Preserve all 48 profile values. Optional if its diagnostic integration is not ready. |
| Read DIA | Use the recovered group: GET `3.9`, AID status `3.12`, legacy status page 7. Preserve the raw half-hour count and decoded duration. Optional if its diagnostic integration is not ready. |
| Enable only | The first experimental mutation: one control-only `4.2` operation. Then read back and observe for 15 minutes. No sensor identity or desired-type write in this action. |
| Provision G7 | Only after the enable-only observation succeeds, or an already-enabled baseline is documented: send the recovered active-Pod G7 batch, collect each member's result, then its separate status-read group. No activation-style sensor-type write. |
| Observe reception | Observe for 30 minutes, recording distinct sensor timestamps, validity, pairing state, Manual mode and faults. Preserve phone/watch feed observations separately. |
| Reconnect after reception | Reconnect and read the persisted state without repeating either write. Check that fresh glucose reporting resumes. |
| Mark alarm and export session | Timestamp an audible alarm/operator observation independently of software fault detection. Export all completed and interrupted steps at any point. |

The observation durations are experiment choices, not manufacturer guarantees. Record actual elapsed time and interruptions. Hard expiry, low reservoir and other end-of-life conditions can confound a result.

### Baseline and read handling

Read `4.2`, G7 `4.1`, unified `3.12` and established legacy status. Use the supported `3.6` read path if needed to resolve delivery mode. Require newly received, parsed Manual-mode and active/nonfaulted status before each mutation; missing or stale state is not permission to proceed. A “ready” bit alone does not prove equivalent official initialization.

Implement the known keyed reply schemas and exact framing checks from [the parser findings](analysis/cgm-provisioning/reply-parsers.md). Distinguish absent fields, unknown enum values, malformed replies and recognized empty replies. The incident's `4.1` binary body `00 00` needs its own “empty/unconfigured shape observed” classification; it is not a decoded identity and does not itself establish permission to write. Other parse failures block dependent mutations. Never interpret a parsing failure as “different sensor, send SET.”

For a first-add session, an existing different identity or indeterminate nonempty identity blocks provisioning. An identity matching the requested code and serial skips the write and enters observation; record that provisioning was not exercised. Do not force a repeated identical SET. Compare only present, validated identity fields, using the documented case-insensitive comparison.

The target/DIA groups have official recovery-flow precedent, with a local setup-completion prerequisite. Their diagnostic use here is a reconstruction. Do not fabricate official setup-complete state or claim their acceptance proves all algorithm inputs are initialized. [Readback contracts](analysis/pod-fault-e2/configuration-readback.md).

### Exact mutation scope

The following are **application command bodies**, not complete BLE/TW packets. Use the reviewed transport, framing and policy selection; do not send these strings directly to a BLE characteristic.

**Enable only:** extended SET `SE4.2=` followed by `00 02 04 00`. This sets control key 4 to ENABLE, with no desired sensor type. Inspect the baseline first: if already enabled, skip this mutation and label it “not exercised.” For the initial incident-comparison branch, arm it only for the recognized DEFAULT (`255`) control state with no configured identity and otherwise valid baseline. Capture other control states for review instead of treating all nonzero values alike. This narrower first-build gate is a research choice, not a firmware rule.

Require canonical success for the SET, then matching `4.2` enable readback, continued Manual status and no fault through the observation window before allowing the next action. ACK alone is insufficient. A rejection, uncertain send or missing/mismatching readback stops the mutation chain; do not change inputs or signing and retry.

**Provision G7:** reproduce the recovered application batch in this order:

```text
batch: SE4.2=<00 02 04 00>, SE4.1=<G7 identity payload>
then:  AID status, legacy status page 7
```

The G7 identity payload contains the big-endian length `00 12`, key `02` plus the actual four-byte pairing code, then key `03` plus the actual twelve-byte serial. Validate encoded byte lengths, preserve leading zeroes, and use the user's real sensor information only in private session storage. Do not append address, timing, interval or other optional identity fields. [Codec and sequence](analysis/cgm-provisioning/g7-enable-sequence.md), [identity encoding](analysis/cgm-provisioning/android.md#dexcom-g7).

Validate **both** SET results; a successful envelope or the first successful member is insufficient. The enable within this batch is intentional recovered behavior, not a retry of the separate enable-only experiment. If OmnipodKit cannot reproduce the batch, leave the action disabled and report that gap; do not silently substitute two independent sends.

After the recovered status group, add labeled bench diagnostic reads of `4.2`/`4.1` and compare the reported identity. Keep these additional observations distinct from the recovered official command group. Distinguish accepted command, confirmed configuration, reported pairing, first fresh valid glucose and sustained glucose. Preserve negative/special glucose status values as states, not numeric glucose. A new history record or phone receive time does not prove a new sensor sample.

Do **not** include `4.2` desired-type key 1 in either experimental action. In particular, do not prepend the activation-style two-field enable/select-G7 command from the earlier incident. The active-Pod batch has no such field. Its behavior on a Trio-initialized Pod remains unverified. [Supported sequences and limits](analysis/pod-fault-e2/supported-sequences.md).

## Persist uncertainty and capture useful evidence

Before sending, durably save an operation ID, experiment ID, target context reference, exact application request, expected values, selected signing policy/mode and timestamps. Record send-start, transport completion, received reply and validation separately; a transport completion is not an application ACK. Keep an interrupted operation pending across app restart. If persistence fails before sending, do not send; if capture fails afterward, stop further mutations and preserve the uncertain outcome.

A timeout or disconnect after sending means **outcome unknown**, not “nothing happened.” Do not automatically replay SETs, recreate the operation on reconnect, or clear it on a generic disconnect. Permit diagnostic readback and export while mutations remain blocked. The first build should not automatically advance an interrupted experiment even if a later read matches; preserve uncertainty about timing and intervening effects. Keep existing lower-level transport retransmission distinct from issuing a new logical SET, and log both. [Recovery evidence](analysis/pod-fault-e2/settings-recovery.md).

Capture plaintext application requests/replies in protected local research storage, decoded fields, validation errors, batch-member results, monotonic and wall-clock timestamps, Pod/sample/record timestamps, mode/fault changes, and operation provenance. Preserve exact bytes on parse failure. Save Trio/OmnipodKit/build revisions and activation provenance. Do not depend solely on OS log retention. Export a structured session manifest plus raw exchange records and checksums; provide a redacted sharing export that removes sensor/device identifiers and sensitive payload fields. Never export private signing keys, session keys or credentials. Include a synthetic sample export with the implementation.

Select signing **before** sending from the applicable provisioned context. There is no unsigned-first, signed-on-rejection experiment. Android uses 1124/1125 for the individual attributes and 65536 for the batch; the traced iOS wrapper uses its registration default through signing ID 0. These findings do not determine Trio's actual policy. Inspect and record Trio's chosen path; if it cannot be resolved, disable mutations. [Android policy](analysis/cgm-provisioning/signing-policy.md), [iOS distinction](analysis/cgm-provisioning/ios-signing-policy.md).

Stop further experimental writes on a fault, unexpected mode, unknown mutation outcome, target mismatch, logging failure or unexpected interruption of the existing phone/watch sensor feed. Do not add reset/shutdown as an automatic recovery. Normal bench-Pod deactivation/disposal is a separate, explicitly targeted closeout after evidence capture.

## Scope and acceptance before a physical session

Leave fresh-Pod activation, TDI/history inputs, target/DIA values, thresholds, UTC, `3.14`, Auto mode and delivery commands unchanged. This build is not an algorithm-input sweep. Optional history download may be added later through the established logger path with cursor changes logged; it is not required for this first experiment build. The active sensor's current phone/watch connections must be recorded and preserved; simultaneous connections are not a guarantee of no interference.

Before declaring the build ready, demonstrate with fake transport and persisted-state tests:

- Wrong target, on-body/unconfirmed bench status, stale/unknown mode and concurrent operations cannot dispatch a mutation.
- Enable-only serialization contains exactly control key 4; the identity batch preserves both members and has no desired-type key, extra sensor fields or hidden setup writes.
- A failed second batch member, malformed/empty/missing reply or readback mismatch cannot report success or unlock the next mutation.
- Timeouts and process restarts at send milestones retain the operation without replaying it. Reconnection itself emits no mutation.
- Matching identity skips provisioning. Missing identity fields do not compare equal or become a reason to overwrite.
- Missing/unknown keyed fields, truncation, duplicate keys, wrong masks, extended SET errors and negative glucose status values are handled explicitly without stale successful state leaking from a previous reply.
- Lost logging or an observed fault locks further mutations, while saved evidence remains exportable. Experimental data cannot reach live treatment stores.

Reuse and review the retained [33-test OmnipodKit validation patch](analysis/pod-fault-e2/incident-replay-and-validation.md) against the actual target revision; do not assume it is installed. The [32 original Android parser cases](analysis/original-app-runtime/README.md) provide additional fixtures and comparison evidence, not firmware acceptance. Full original command construction remains blocked by [native runtime dependencies](analysis/original-app-runtime/construction/README.md).

Finish the handoff with the exact build revisions, test results, a sample redacted export, the demonstrated controller-isolation method and any disabled actions. No physical experiment is required merely to call the software implementation complete; an unresolved isolation, framing or signing prerequisite must remain visible rather than be silently bypassed.
