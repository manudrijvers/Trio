# Removed-Pod experiment build

This change targets `manudrijvers/Trio` branch `trio-o5-dex-direct`, preserving
its existing G7 implementation. The OmnipodKit pin combines that feature with
the bench runner on `manudrijvers/OmnipodKit` branch `o5-bench-dry-run`.

The implementation and validation handoff are in
[OmnipodKit/Research/O5Bench](../OmnipodKit/Research/O5Bench/README.md).

The owner's updated workflow uses **one active Pod**: remove it physically, keep
it active in Trio for the experiment, then replace it normally. No second
controller or active-context handoff is required.

This build provides a persistent offline runner, an Omnipod settings dry-run
screen, strict response validation and verified synthetic/redacted exports.
Physical experiments remain disabled pending provisioned signing-policy
integration, physical batch validation and an exclusive writer gate on the
existing active controller. The dry-run screen does not engage a physical bench
mode or change ordinary therapy behavior.

The original untracked `TRIO_NEXT_EXPERIMENT_BUILD.md` is preserved. Its
requirement for therapy continuing on a separate Pod/controller is superseded by
the owner's single-Pod clarification; its protocol and uncertainty requirements
remain the implementation reference.
