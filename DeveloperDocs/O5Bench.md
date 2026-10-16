# Removed-Pod experiment build

The physical runner follows the single-active-Pod workflow: remove the Pod from
the body, leave it active in Trio, run experiments, then deactivate and replace it
normally. Open **Removed Pod Experiment** in Omnipod 5 settings. Logging is
automatic; the screen provides redacted and private raw exports.

A real BLE adapter, certificate-policy selection, persisted exclusive-access lock
and separate experiment actions are implemented. A full Xcode build and physical
Pod acceptance have not been verified in this Linux workspace. Portable tests and
iOS-source syntax checks do not establish that the app is ready to install.

See [the implementation and validation notes](../OmnipodKit/Research/O5Bench/README.md)
for operation, safeguards, signing assumptions and remaining validation.

**Pod Experiment Dry Run** remains available for simulation. Its exports are
synthetic. The original `TRIO_NEXT_EXPERIMENT_BUILD.md` remains the protocol
reference; the owner's single-active-Pod clarification supersedes its earlier
separate-controller requirement.
