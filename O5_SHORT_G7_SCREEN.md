# Build and run the short G7 screening experiment

The OmnipodKit submodule pinned by this branch contains protocol
`short-g7-screen-v1`. Its complete scope, time budget, exact phone steps and
limits are in [SHORT_G7_SCREEN.md](OmnipodKit/Research/O5Bench/SHORT_G7_SCREEN.md).

On the Mac, retain any local work, then fetch and check out branch
`o5-short-g7-screen` and update the pinned submodules:

```sh
git fetch origin
git switch o5-short-g7-screen
git submodule update --init --recursive
```

Build/install Trio using your existing Xcode signing setup. Confirm the
**Removed Pod Experiment** screen contains **Short G7 screen · about 20 minutes**.
Use its two numbered actions, not the earlier enable-only protocol. The screen
requires the removed Pod to remain Trio's active Pod, delivery already suspended,
automatic dosing off, and the other recorded prerequisites. It does not change
the delivery program or add a second controller.

Budget two minutes of baseline plus overhead and 15 minutes after the single
batch send. A fault or uncertain outcome ends the run early. No new glucose
within that window is an observation, not proof that pairing cannot work.
Export immediately, then use normal deactivation/replacement. Do not delay a
needed replacement for this experiment.

Fresh-Pod activation code and settings are unchanged. Retain the normal new-Pod
activation log separately (`O5 AID Send`, `O5 AID RawResp`, `O5 AID Recv`) if
available. Never publish raw identifiers or credentials.

Android app work can resume independently; the research repository handoff is
`analysis/android-research-build/NEXT_STEPS.md`. Probe 0.3 successfully serialized
target/DIA GETs according to the owner; it has not communicated with a Pod.
