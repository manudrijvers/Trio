# Handover: Omnipod 5 pod-direct Dexcom G7 pairing (proof of concept)

This note is for whoever picks up the branch next, a person or Claude Code on a Mac. Delete it, and revert
`.gitmodules`, before anything goes upstream.

## Goal

An Omnipod 5 pod can talk to a Dexcom G7 sensor itself: the phone only gives the pod the sensor's
identity and later reads the glucose values back from the pod. This proof of concept does only one thing:
**connect a G7 sensor to an already-active O5 pod**, and read back what the pod reports. It does not use
the pod's glucose values in Trio yet. Keep it small; it is a proof of concept.

The protocol details come from static analysis of the official O5 app in the private repo
`manudrijvers/re-omnipod` (see `OVERVIEW.md`, `analysis/cgm-provisioning/*.md` and
`analysis/cgm-readback/android.md`). The byte layouts below have **never been checked against a real pod**.

## Branches and commits

| Repo | Branch | Commit | Remote |
|---|---|---|---|
| Trio | `trio-o5-dex-direct` | `98f63fe37` and the commit that adds this note | `origin` = `manudrijvers/Trio` |
| OmnipodKit (submodule) | `o5-dex-direct` (from `043b91d`) | `363f75a` | `manudrijvers/OmnipodKit` |

The Trio commit points `.gitmodules` for OmnipodKit at `https://github.com/manudrijvers/OmnipodKit`,
branch `o5-dex-direct`, and bumps the submodule pointer to `363f75a`.

Getting it on a Mac (once both branches are pushed):

```sh
git fetch origin && git checkout trio-o5-dex-direct
git submodule sync && git submodule update --init --recursive
git -C OmnipodKit log --oneline -1   # expect 363f75a "Add proof-of-concept Dexcom G7 pairing for O5 pods"
```

## What was changed (all in OmnipodKit, about 300 lines)

- `OmnipodKit/Bluetooth/O5AidCommands.swift`
  - `CgmInterfaceCommand` (feature 4, attribute 2): `payload()` gives `SE4.2=` + `00 04 04 00 01 07`
    (key 4 control = 0 enable, key 1 desired type = 7 G7), expected reply prefix `ES4.2=`.
    `getPayload()` gives `G4.2`, reply prefix `4.2=`.
  - `G7SensorInfoCommand` (feature 4, attribute 1): `payload(pairingCode:serialNumber:)` gives `SE4.1=` +
    `[u16 BE length][02][4 ASCII code][03][12 ASCII serial]`, reply prefix `ES4.1=`. `getPayload()` gives `G4.1`.
  - `extendedSetResultDescription(_:)` turns an `ES` reply (`0`, `-5`, `-14`, `-19`, …) into text.
  - `O5AidCgmStatus`: decodes the CGM part of a `G3.12` reply (EGV, EGV age, loop state, CGM type, and for
    G7 the session/sensor times, algorithm state, transmitter status and trend).
- `OmnipodKit/PumpManager/PodCommsSession.swift`: `sendO5AidCommand(_:)` so an ordinary pod session (not
  only activation) can send O5 AID commands. Only for O5 pods over `BlePodMessageTransport`.
- `OmnipodKit/PumpManager/OmniPumpManager.swift`: `connectDexcomG7(pairingCode:serialNumber:)` sends `SE4.2`,
  and only if that returns `0` sends `SE4.1`. `readDexcomG7Status()` reads `G4.2`, `G4.1` and `G3.12` and
  returns the raw hex plus the decoded status.
- `OmnipodKit/PumpManagerUI/Views/PodDiagnosticsView.swift`: two O5-only rows in Pod Diagnostics,
  **Connect Dexcom G7** (new `ConnectDexcomG7View`: 4-digit code, 12-digit serial, Connect button,
  result text) and **Read Dexcom G7 Status** (reuses `ReadPodInfoView`). The `DiagnosticCommands`
  protocol gained `podType`, `connectDexcomG7` and `readDexcomG7Status`; `OmniPumpManager` is its only
  conformer.
- `OmniTests/O5/O5AidCommandsTests.swift`: 5 new tests for the payload bytes, result text and `3.12` decoding.

Nothing in the Trio app target itself was changed.

## What has been checked, and what hasn't

- Checked: `O5AidCommands.swift` and its tests compiled and passed (15/15) with Swift 5.10 on Linux, in a
  small SwiftPM harness containing only that file, `Data.swift` and the tests.
- **Not checked:** the session, pump manager and SwiftUI code have never been compiled (there was no Xcode).
  First job on the Mac: build the Trio workspace and run the OmniTests `O5AidCommandsTests`, fix any
  compile errors, and commit the fixes on the OmnipodKit `o5-dex-direct` branch (then bump the
  submodule pointer in Trio).

## Testing on a real pod

1. Start an O5 pod as normal. Then Omnipod settings → Pod Diagnostics.
2. **Read Dexcom G7 Status** first, to see the pod's starting state.
3. **Connect Dexcom G7**, enter the code and serial from the sensor applicator. Tap Connect **once**.
   Both steps should show `0 (success)`.
4. **Read Dexcom G7 Status** again over the next minutes. Expect CGM type 7 in `4.2`/`3.12`, the code and
   serial in `4.1`, and, after warm-up, a glucose value and a G7 algorithm state.

The raw replies are logged as `O5 AID RawResp` in the pod comms log.

## Known risks and fallbacks

- **Signing.** The commands are sent unsigned (type 1), like the activation AID commands that already work.
  The official app decides between signed and unsigned from a cloud certificate (IDs 1124 for `4.1`, 1125 for
  `4.2`), and that policy is unknown. If the pod rejects the commands, signing is the first suspect.
- **`4.2` field order.** Sending both fields together is a guess at the official app's order. If the pod
  answers `-5`, send them separately: `00 02 01 07` (type G7), then `00 02 04 00` (enable).
- **Re-sending the same identity** may restart pairing. Connect does not read first; tap it once per sensor.
- **Unexpected replies.** `BlePodMessageTransport.sendO5AidCommand` throws on a reply-prefix mismatch
  before it ACKs the reply. That is existing behaviour; later commands in the same session may misbehave.
- **Older pods.** `3.12` needs pod firmware major version 7 or later; older O5 pods use `3.11`, so the status
  read will fail on them (the `4.x` reads still show up in the log).

## Out of scope for now

Pairing during pod activation, remembering the sensor across pods, reading first and only sending what
differs (the RE notes describe the `G4.1`/`G4.2` reply formats), removing a sensor, G6, and using the
pod's glucose in Trio as a CGM source or for automation.

## Before an upstream PR

- Revert the OmnipodKit URL/branch in `.gitmodules` and land the OmnipodKit change upstream first.
- Delete this file.
