# Removed-Pod experiment build

The physical runner follows the single-active-Pod workflow: remove the Pod from
the body, leave it active in Trio, run experiments, then deactivate and replace it
normally. Open **Removed Pod Experiment** in Omnipod 5 settings. Logging is
automatic; the screen provides redacted and private raw exports.

Turn automatic dosing off before starting and leave it off until the removed Pod
is replaced. The experiment verifies that setting. During the session it pauses
background status polling and alert maintenance; stopping or leaving the screen
releases that in-memory pause. It uses the existing BLE connection and serialized
command queue. Ordinary send, deactivation and discard paths have no experiment
lock. Experiment signing and recording live in dedicated transport methods.

A real BLE adapter, certificate-policy selection and separate experiment actions
are implemented. Full simulator builds run in the **O5 experiment simulator build**
GitHub Actions workflow; check its result for the exact source revision. Physical
Pod acceptance has not been verified. Portable tests use a deterministic radio
double and do not establish hardware behavior.

See [the implementation and validation notes](../OmnipodKit/Research/O5Bench/README.md)
for operation, safeguards, signing assumptions and remaining validation.

**Pod Experiment Dry Run** remains available for simulation. Its exports are
synthetic. The original `TRIO_NEXT_EXPERIMENT_BUILD.md` remains the protocol
reference; the owner's single-active-Pod clarification supersedes its earlier
separate-controller requirement.
