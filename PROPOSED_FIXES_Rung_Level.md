# Proposed rung-level corrections

> **Scope note.** Fixes A, B and D in this document have been superseded by the narrower corrections
> actually applied to the exports; see `KNEEJ_STAGGD_REVIEW.md`. Fix C and fixes E to I remain parked,
> because they are original vendor code rather than KneeJ or StaggD adjustments.

**Version 1.0.** Companion to `ANALYSIS_Hoist_DroppingPlate_Coordination.md`.
Program `P_Stacker`, controller `P2850_CHH_Trimming_Repairing`.

**Nothing in this document has been applied.** The `.txt` L5X exports in this repository are unchanged.
These are proposals for a controls engineer to enter in Studio 5000 and commission on the machine.

Rung text below is Studio 5000 neutral text, in the same form the exports use, so it can be pasted
straight into a rung. Line breaks in the "after" blocks are for reading only; enter each rung as one line.

## If you only do one thing

Do fix A and fix B. Between them they move the infeed release from roughly 400 mm below the receiving
position to 100 mm below it, and stop the release bit flickering with the drive's run command. That is
the root cause of both reported faults.

---

## New tags required

| Tag | Scope | Type | Initial | Used by |
|---|---|---|---|---|
| `_L25_AlmostUpPosition` | `P_Stacker` program | DINT | measured top minus 100 | Fix A |
| `_L21_PanelsOnPlates` | `P_Stacker` program | DINT | 0 | Fix C |
| `_L21_MaxPanelsOnPlates` | `P_Stacker` program | DINT | 2 | Fix C |
| `_L21_PlatesCanAcceptPanel` | `P_Stacker` program | BOOL | - | Fixes B, C |
| `_L21_Ons_4` | `P_Stacker` program | BOOL | - | Fix C |

`_L25_StackerHoistUp_Down.OUT_ActualPosition` is a DINT, so `_L25_AlmostUpPosition` must be a DINT for a
clean `GEQ`. `_L25_StackerHoistUp_Down` and `_L25_Platform_RdyInfeed` are program-scoped in `P_Stacker`,
so `L21` can address them directly; it already does for `_L25_Platform_RdyInfeed` in rung 11.

Set `_L25_AlmostUpPosition` from the measurement in step 1 of the verification list, not from a guess.
Make it an HMI setpoint if you want to tune it without a download.

---

## Fix A - move the "almost up" point and stop it flickering

**`L25_StackerHoist_UpDown` rung 37.** Addresses findings 1 and 2.

Two edits inside the existing rung. Do not insert a new rung; keeping it in rung 37 avoids renumbering
and avoids a one-scan lag on the position value.

1. Remove `OTE(_L25_Platform_AlmostUp)` from the 400-speed branch, leaving that branch as speed only.
2. Add a new parallel branch that drives `_L25_Platform_AlmostUp` from position alone, with no
   `XIC(...IN_RunFw)` qualifier, against the new tunable threshold.

Before:

```
XIC(_L25_Auto)[XIC(_L25_StackerHoistUp_Down.IN_RunFw) MOV(1800,_L25_StackerHoistUp_Down.IN_SpeedReferenceEU) ,XIC(OUT_StackerIsEmpty) XIC(_L25_StackerHoistUp_Down.IN_RunFw) MOV(2500,_L25_StackerHoistUp_Down.IN_SpeedReferenceEU) ,XIC(_L25_StackerHoistUp_Down.IN_RunFw) GEQ(_L25_StackerHoistUp_Down.OUT_ActualPosition,1700) [MOV(400,_L25_StackerHoistUp_Down.IN_SpeedReferenceEU) ,OTE(_L25_Platform_AlmostUp) ] ,XIC(_L25_StackerHoistUp_Down.IN_RunBw) [MOV(200.0,_L25_StackerHoistUp_Down.IN_SpeedReferenceEU) ,[XIC(I_PC_Stkr_MaxHeight) ,[XIC(StackerHoistIndexDownMaxTime.DN) ,XIO(I_PC_Stkr_MaxHeight_Left_Backup) ] LES(HMI_972CP_Set_Panel_Thickness,20) ] MOV(0,_L25_StackerHoistUp_Down.IN_SpeedReferenceEU) ] ,XIC(_L25_StackerHoistUp_Down.IN_RunBw) [XIC(OUT_Stkr_Is_Full) [TON(_L25StkrFull,?,?) XIC(_L25StkrFull.DN) ,XIC(OUT_PromiceFromDroppingPlates) XIC(Out_PromiceFwd_SideAlignment) ] MOV(2000,_L25_StackerHoistUp_Down.IN_SpeedReferenceEU) ,LEQ(_L25_StackerHoistUp_Down.OUT_ActualPosition,100) MOV(400,_L25_StackerHoistUp_Down.IN_SpeedReferenceEU) ] ];
```

After:

```
XIC(_L25_Auto)[XIC(_L25_StackerHoistUp_Down.IN_RunFw) MOV(1800,_L25_StackerHoistUp_Down.IN_SpeedReferenceEU) ,XIC(OUT_StackerIsEmpty) XIC(_L25_StackerHoistUp_Down.IN_RunFw) MOV(2500,_L25_StackerHoistUp_Down.IN_SpeedReferenceEU) ,XIC(_L25_StackerHoistUp_Down.IN_RunFw) GEQ(_L25_StackerHoistUp_Down.OUT_ActualPosition,1700) MOV(400,_L25_StackerHoistUp_Down.IN_SpeedReferenceEU) ,GEQ(_L25_StackerHoistUp_Down.OUT_ActualPosition,_L25_AlmostUpPosition) OTE(_L25_Platform_AlmostUp) ,XIC(_L25_StackerHoistUp_Down.IN_RunBw) [MOV(200.0,_L25_StackerHoistUp_Down.IN_SpeedReferenceEU) ,[XIC(I_PC_Stkr_MaxHeight) ,[XIC(StackerHoistIndexDownMaxTime.DN) ,XIO(I_PC_Stkr_MaxHeight_Left_Backup) ] LES(HMI_972CP_Set_Panel_Thickness,20) ] MOV(0,_L25_StackerHoistUp_Down.IN_SpeedReferenceEU) ] ,XIC(_L25_StackerHoistUp_Down.IN_RunBw) [XIC(OUT_Stkr_Is_Full) [TON(_L25StkrFull,?,?) XIC(_L25StkrFull.DN) ,XIC(OUT_PromiceFromDroppingPlates) XIC(Out_PromiceFwd_SideAlignment) ] MOV(2000,_L25_StackerHoistUp_Down.IN_SpeedReferenceEU) ,LEQ(_L25_StackerHoistUp_Down.OUT_ActualPosition,100) MOV(400,_L25_StackerHoistUp_Down.IN_SpeedReferenceEU) ] ];
```

Notes.

- The 1700 slow-down point is deliberately left alone. Deceleration distance and the release point are
  two different decisions and should stay separate. Once fix A is in, 1700 can be tuned for ride quality
  without moving the infeed release.
- Dropping `XIC(...IN_RunFw)` means `_L25_Platform_AlmostUp` stays true while the platform sits above the
  threshold, instead of collapsing the moment the drive stops or stalls. Check the two cases where that
  is a behaviour change: during normal stacking near the top, rung 47 branch 1 already grants the promise
  through `_L25_Platform_RdyInfeed`, so nothing changes; during the pack-change descent, rung 47's series
  `XIO(OUT_Stkr_Is_Full)` blocks branch 2, so nothing changes.
- Rung 36 slows manual jog above 2150 while rung 37 slows auto above 1700. Once the top position is
  measured, reconcile those two numbers.

## Fix B - tighten rung 47

**`L25_StackerHoist_UpDown` rung 47.** Addresses findings 3 and 10.

Before:

```
XIO(OUT_Stkr_Is_Full)[XIC(_L25_Platform_RdyInfeed) XIO(_L25_HoistAutoMode_Resumed) ,XIC(HMI_972_1120CP_Pb_DroppingPlatesSelect) XIC(_L25_Platform_AlmostUp) XIC(OUT_PromiceFromDroppingPlates) ][OTE(Out_PromiceFwd_1123M1) ,OTE(HMI_972CP_STatusStkrRdyInfeed) ];
```

After:

```
XIO(OUT_Stkr_Is_Full)XIO(_L25_HoistAutoMode_Resumed)[XIC(_L25_Platform_RdyInfeed) ,XIC(HMI_972_1120CP_Pb_DroppingPlatesSelect) XIC(_L25_Platform_AlmostUp) XIC(_L21_PlatesCanAcceptPanel) ][OTE(Out_PromiceFwd_1123M1) ,OTE(HMI_972CP_STatusStkrRdyInfeed) ];
```

Two changes. `XIO(_L25_HoistAutoMode_Resumed)` moves into the series section so it covers both branches
rather than only branch 1. `OUT_PromiceFromDroppingPlates` is replaced by the new
`_L21_PlatesCanAcceptPanel`, which is the "plates have room" condition rather than the "plates are in a
known position" condition. Fix C creates it.

Do not tighten `OUT_PromiceFromDroppingPlates` itself. It is overloaded: rungs 5, 32 and 37 of `L25` use
it to decide whether the hoist may move, and during a pack change the plates legitimately hold panels
while the hoist has to descend. Adding a capacity condition to it would prevent the hoist from ever
leaving. That is why fix C adds a separate bit instead of editing rung 22.

## Fix C - count the panels on the plates

**`L21_DroppingPlates`, append three rungs after rung 22, plus one edit to rung 13.**
Addresses finding 3.

Appending at the end avoids renumbering, so the change-log references to R10, R11 and so on stay valid.
`_L21_PlatesCanAcceptPanel` is consumed in other routines, so at worst it is one scan old there, which is
irrelevant for a conveyor permissive.

New rung 23, count a panel onto the plates:

```
XIC(HMI_972_1120CP_Pb_DroppingPlatesSelect)XIC(_L21_Auto_Down)XIC(MIS_StackingOnePanelPulse)ONS(_L21_Ons_4)ADD(1,_L21_PanelsOnPlates,_L21_PanelsOnPlates);
```

`MIS_StackingOnePanelPulse` comes from `L20` rung 42 and fires 500 ms after the trailing edge clears
`I_PC_Stkr_PanelUnderPressingRoll` while the rolls are running forward, which is "a panel has left the
rolls and entered the stacker". `XIC(_L21_Auto_Down)` restricts the count to panels that landed while the
plates were extended. `L20` rung 42 already one-shots the pulse; the `ONS` here is defensive and costs
one BOOL.

New rung 24, housekeeping:

```
[[XIC(_L21_AutoPulse) ,XIC(HMI_972CP_Pb_Reset_All) ,XIO(HMI_972_1120CP_Pb_DroppingPlatesSelect) ,XIC(_L21_Manual) ] CLR(_L21_PanelsOnPlates) ,[LES(_L21_MaxPanelsOnPlates,1) ,GRT(_L21_MaxPanelsOnPlates,2) ] MOV(2,_L21_MaxPanelsOnPlates) ];
```

The second branch is a guard. If `_L21_MaxPanelsOnPlates` is ever left at 0 the line stops dead, so clamp
it into the range 1 to 2.

New rung 25, the permission:

```
[XIC(HMI_972_1120CP_Pb_DroppingPlatesSelect) XIC(_L21_Sensors_Sylinders_Out) LES(_L21_PanelsOnPlates,_L21_MaxPanelsOnPlates) ,XIO(HMI_972_1120CP_Pb_DroppingPlatesSelect) XIC(OUT_PromiceFromDroppingPlates) ,XIC(_L21_Manual) ]OTE(_L21_PlatesCanAcceptPanel);
```

With the plates deselected this falls through to the existing `OUT_PromiceFromDroppingPlates`, so the
non-plate behaviour is unchanged.

Rung 13, clear the count when the plates finish releasing. Before:

```
XIC(_L21_Auto_Up)[XIO(I_PC_Stkr_StackOnPlatform) ,XIC(HMI_972_1120CP_Pb_DroppingPlatesSelect) ]MUL(HMI_972_1120CPSettingDroppingPlatesDown,1000,_L21_ToDownDelay.PRE)TON(_L21_ToDownDelay,?,?)XIC(_L21_ToDownDelay.DN)[OTU(_L21_Auto_Up) ,OTU(_L21_PanelComing) ];
```

After:

```
XIC(_L21_Auto_Up)[XIO(I_PC_Stkr_StackOnPlatform) ,XIC(HMI_972_1120CP_Pb_DroppingPlatesSelect) ]MUL(HMI_972_1120CPSettingDroppingPlatesDown,1000,_L21_ToDownDelay.PRE)TON(_L21_ToDownDelay,?,?)XIC(_L21_ToDownDelay.DN)[OTU(_L21_Auto_Up) ,OTU(_L21_PanelComing) ,CLR(_L21_PanelsOnPlates) ];
```

Then in **`L20_PressureRoll` rung 4**, swap the plate permissive so the rolls also respect the capacity.
Before:

```
[XIC(Out_PromiceFwd_1122M1) XIC(Out_PromiceFwd_1123M1) XIC(Out_PromiceFwd_SideAlignmentWagon) XIC(Out_Promice_PressRollsFromSideAlignment) XIC(OUT_PromiceFromDroppingPlates) ,XIO(I_PC_Stkr_PanelUnderPressingRoll) ]XIO(OUT_Stkr_Is_Full)OTE(_L20_PromiceFwd);
```

After:

```
[XIC(Out_PromiceFwd_1122M1) XIC(Out_PromiceFwd_1123M1) XIC(Out_PromiceFwd_SideAlignmentWagon) XIC(Out_Promice_PressRollsFromSideAlignment) XIC(_L21_PlatesCanAcceptPanel) ,XIO(I_PC_Stkr_PanelUnderPressingRoll) ]XIO(OUT_Stkr_Is_Full)OTE(_L20_PromiceFwd);
```

The `XIO(I_PC_Stkr_PanelUnderPressingRoll)` branch stays. It is intentional: with no panel at the
photocell the rolls are free to run, and the next panel is drawn up to the photocell and then held there
by the interlock chain. That is the buffer position and it is where the third panel should stop.

## Fix D - give the plates a real settle time, measured from the right event

**`L21_DroppingPlates` rungs 10 and 11.** Addresses finding 4.

Rung 11, move `Out_PromiceFwd_SideAlignment` and `_L25_Platform_RdyInfeed` ahead of the timer so the
delay is timed from "panel landed, squared, platform in position", and add a check that the platform has
actually stopped.

Before:

```
XIC(_L21_Auto)XIC(Out_PromiceFwd_1123M1)[XIC(_L21_PanelComing) MUL(HMI_972_1120CPSettingDroppingPlatesUp,1000,_L21_ToUpDelay.PRE) TON(_L21_ToUpDelay,?,?) XIC(_L21_ToUpDelay.DN) XIC(Out_PromiceFwd_SideAlignment) XIC(_L25_Platform_RdyInfeed) ONS(_L21_Ons_2) ,XIO(HMI_972_1120CP_Pb_DroppingPlatesSelect) XIC(I_PC_Stkr_StackOnPlatform) ]OTL(_L21_Auto_Up);
```

After:

```
XIC(_L21_Auto)XIC(Out_PromiceFwd_1123M1)[XIC(_L21_PanelComing) XIC(Out_PromiceFwd_SideAlignment) XIC(_L25_Platform_RdyInfeed) XIO(_L25_StackerHoistUp_Down.OUT_RunningFw) MUL(HMI_972_1120CPSettingDroppingPlatesUp,1000,_L21_ToUpDelay.PRE) TON(_L21_ToUpDelay,?,?) XIC(_L21_ToUpDelay.DN) ONS(_L21_Ons_2) ,XIO(HMI_972_1120CP_Pb_DroppingPlatesSelect) XIC(I_PC_Stkr_StackOnPlatform) ]OTL(_L21_Auto_Up);
```

Rung 10, raise the clamp so a useful settle time can be dialled in. Optional, and only worth doing if
site trials show 0.3 s is not enough. Before:

```
[GRT(HMI_972_1120CPSettingDroppingPlatesUp,0.3) MOV(0.3,HMI_972_1120CPSettingDroppingPlatesUp) ,LES(HMI_972_1120CPStkrSetSideAlignmentStart,0) MOV(0,HMI_972_1120CPStkrSetSideAlignmentStart) ];
```

After:

```
[GRT(HMI_972_1120CPSettingDroppingPlatesUp,1.0) MOV(1.0,HMI_972_1120CPSettingDroppingPlatesUp) ,LES(HMI_972_1120CPStkrSetSideAlignmentStart,0) MOV(0,HMI_972_1120CPStkrSetSideAlignmentStart) ];
```

Then set `HMI_972_1120CPSettingDroppingPlatesUp` to 0.2 and trial upward. It is 0.0 today.

Also correct the tag description on `HMI_972_1120CPSettingDroppingPlatesUp`. It currently reads
"How long dropping plates keep panel before dropping, unit 0.1s", but rung 11 multiplies by 1000, which
treats the value as seconds. Fix the description, or change the multiplier to 100 and rescale the clamp
and the HMI setpoint together. Do not change one without the other.

## Fix E - both plate sensors must confirm the catching position

**`L21_DroppingPlates` rung 19.** Addresses finding 5.

Before:

```
[XIC(I_PR_Stkr_DroppingPlateLeftOn) ,XIC(I_PR_Stkr_DroppingPlateRightOn) ]OTE(_L21_Sensors_Sylinders_Out);
```

After:

```
XIC(I_PR_Stkr_DroppingPlateLeftOn)XIC(I_PR_Stkr_DroppingPlateRightOn)OTE(_L21_Sensors_Sylinders_Out);
```

Prove both proximity switches make reliably before applying this, or the line will stop. If one is
unreliable, fix the sensor rather than leaving the OR in place. This bit is `IN_IsOpenSensor` on the
valve AOI with `IN_OpenSensorInUse` tied to `Always_1`, so with the AND in place a failed extension also
raises the valve alarm through `_L21_AV_Stkr_DroppingPlates.OUT_AlarmOn` and `HMI_Alarm28.1` after the
10000 ms opening timeout, which is the behaviour you want.

## Fix F - stop ignoring the right-hand retracted sensor

**`L21_DroppingPlates` rung 20.** Addresses finding 6.

Before:

```
XIC(I_PR_Stkr_DroppingPlateLeftOff)[XIC(I_PR_Stkr_DroppingPlateRightOff) ,]OTE(_L21_Sensors_Sylinders_In);
```

After:

```
XIC(I_PR_Stkr_DroppingPlateLeftOff)XIC(I_PR_Stkr_DroppingPlateRightOff)OTE(_L21_Sensors_Sylinders_In);
```

Low risk today because rung 21 passes `Always_0` for `IN_ClosedSensorInUse`, so nothing acts on this bit.
Worth doing so the bit means what it says. If you want the valve to police the retracted position too,
change that `Always_0` to `Always_1` as a separate, later step.

## Fix G - the dead abort in rung 33

**`L25_StackerHoist_UpDown` rung 33.** Addresses finding 7. Pick one option, do not leave it as it is.

Current, with a contact and its own inverse in series:

```
[[XIC(_L25_DowmStopPlace) ,XIC(_L25_DowmStopPlace2) ] ,XIO(_L25_Auto) ,XIC(_L25_Auto) XIC(VFD1123M1.DevCtrl.Auto_Bw) TON(_L25_MaxRunTimetoDownWithoutPanels,?,?) XIC(_L25_MaxRunTimetoDownWithoutPanels.DN) XIO(_L25_MaxRunTimetoDownWithoutPanels.DN) ]OTU(VFD1123M1.DevCtrl.Auto_Bw);
```

**Option 1, recommended: re-scope the timer to actual motion and re-enable it.**

```
[[XIC(_L25_DowmStopPlace) ,XIC(_L25_DowmStopPlace2) ] ,XIO(_L25_Auto) ,XIC(_L25_Auto) XIC(_L25_StackerHoistUp_Down.OUT_RunningBw) TON(_L25_MaxRunTimetoDownWithoutPanels,?,?) XIC(_L25_MaxRunTimetoDownWithoutPanels.DN) ]OTU(VFD1123M1.DevCtrl.Auto_Bw);
```

Timing on `OUT_RunningBw` instead of the latched `Auto_Bw` is the whole point. `Auto_Bw` stays latched for
the entire stack build, which is why a 15 s timeout on it had to be disabled. `OUT_RunningBw` is only true
while the hoist is genuinely moving down, and it resets between index steps, so 15 s becomes 15 s of
continuous downward motion. The full pack-change descent is about 2100 mm at 2000, roughly one second, so
15 s is a generous abort. Commission this deliberately and watch for spurious trips on the first shift.

**Option 2: delete the branch and say so.** If nobody wants a downward run-time abort, remove the third
input branch entirely, delete `_L25_MaxRunTimetoDownWithoutPanels`, and put a rung comment saying there is
no run-time protection on the down move. Leaving a timer that runs, times out and does nothing is worse
than having none, because it reads as protection during a fault investigation.

## Fix H - remove the hard-coded pressure roll speed

**`L20_PressureRoll` rung 33.** Addresses finding 8.

Before:

```
MOV(70,_L20_Stkr_PressureRoll.IN_SpeedReferenceEU)CPS(AF1_1120M1:I,VFD1120M1.ModuleData.Input,1);
```

After:

```
CPS(AF1_1120M1:I,VFD1120M1.ModuleData.Input,1);
```

Before applying: read `HMI_972_1110CP_Setting_OutfeedSpeed`, which is in m/min, and check that
`value * 1000 / 60` is the speed you actually want the rolls to run at. Rung 32 does that conversion. If
the HMI setpoint has drifted while the override masked it, correct the setpoint first, then remove the
`MOV`. Removing the `MOV` also restores rung 31, which zeroes the reference when the drive is not
commanded to run.

## Fix I - do not declare the platform ready on the auto-start scan

**`L25_StackerHoist_UpDown` rung 27.** Addresses finding 9.

Before:

```
[XIC(VFD1123M1.DevCtrl.Auto_Fw) [XIO(I_PC_Stkr_MaxHeight) ,XIC(I_LS_Stkr_PlatformHoistUpLimit) ,XIC(_L25_StackerHoistUp_Down.OUT_MaxIdleTimeOver) ] ,XIC(_L25_AutoPulse) ,XIO(_L25_Auto) ,XIC(HMI_972CP_Pb_Reset_All) ][OTU(VFD1123M1.DevCtrl.Auto_Fw) ,XIC(_L25_Auto) XIO(HMI_972CP_Pb_Reset_All) XIO(_L25_StackerHoistUp_Down.OUT_MaxIdleTimeOver) [OTL(_L25_Platform_RdyInfeed) ,OTU(PackUnloading) ] ];
```

After:

```
[XIC(VFD1123M1.DevCtrl.Auto_Fw) [XIO(I_PC_Stkr_MaxHeight) ,XIC(I_LS_Stkr_PlatformHoistUpLimit) ,XIC(_L25_StackerHoistUp_Down.OUT_MaxIdleTimeOver) ] ,XIC(_L25_AutoPulse) ,XIO(_L25_Auto) ,XIC(HMI_972CP_Pb_Reset_All) ][OTU(VFD1123M1.DevCtrl.Auto_Fw) ,XIC(_L25_Auto) XIO(HMI_972CP_Pb_Reset_All) XIO(_L25_StackerHoistUp_Down.OUT_MaxIdleTimeOver) [XIO(I_PC_Stkr_MaxHeight) ,XIC(I_LS_Stkr_PlatformHoistUpLimit) ] [OTL(_L25_Platform_RdyInfeed) ,OTU(PackUnloading) ] ];
```

The added `[XIO(I_PC_Stkr_MaxHeight) ,XIC(I_LS_Stkr_PlatformHoistUpLimit) ]` only affects the
`_L25_AutoPulse` entry. Under `XIO(_L25_Auto)` the output branch's own `XIC(_L25_Auto)` already blocks it,
and under `HMI_972CP_Pb_Reset_All` its `XIO(HMI_972CP_Pb_Reset_All)` already blocks it. So the change is
narrow: switching to auto no longer claims the platform is up when it is not.

While you are in this rung, note that rung 26's branch
`XIC(_L25_AutoPulse) XIC(_L25_PromiceUP) XIO(I_PC_Stkr_StackOnPlatform) OTL(VFD1123M1.DevCtrl.Auto_Fw)`
is cancelled by rung 27's `XIC(_L25_AutoPulse)` on the same scan and has never done anything. Either
delete it or move the auto-start "send the platform up" logic somewhere it can survive. Do not leave it
looking functional.

---

## Commissioning order

Work down the list. Each stage is testable on its own, and stopping after stage 2 already addresses both
reported faults.

| Stage | Do | Why this order |
|---|---|---|
| 0 | Measurements 1 to 4 in the analysis document. No code changes. | Fix A needs the measured top position. Fixes E and H need field data. |
| 1 | Fix H. | Independent, and you want the feed speed known and tunable before tuning anything else. |
| 2 | Fixes A and B. | Root cause of both reported faults. Fix B references `_L21_PlatesCanAcceptPanel`, so create the tag with fix C's rung 25 in the same download. |
| 3 | Fix C in full. | Stops the third panel. Needs `MIS_StackingOnePanelPulse` proven first. |
| 4 | Fix D. | Landing quality. Tune the settle time only once the platform is arriving on time. |
| 5 | Fixes E and F. | Only after both plate sensors are proven. These will stop the line on a genuine fault, which is the intent. |
| 6 | Fixes G and I at the next planned outage. | Neither is causing the reported faults. Both change fault-handling behaviour and deserve their own watch. |

## Test plan

Run each of these in auto with the dropping plates selected.

1. **Normal stacking, no pack change.** Twenty panels. Confirm one plate drop per panel, the hoist
   indexing down as before, and `_L21_PanelsOnPlates` returning to 0 after every drop.
2. **Pack change, upstream starved.** Force a pack change with no panels available upstream. The platform
   should descend, discharge, and return without the plates cycling. Confirm `_L25_Platform_AlmostUp`
   comes true at the new threshold and `Out_PromiceFwd_1123M1` follows it.
3. **Pack change, upstream full.** The real test. Trend `_L21_PanelsOnPlates`, `Out_PromiceFwd_1123M1`,
   `_L20_PromiceFwd`, `I_PC_Stkr_PanelUnderPressingRoll` and `StackerHoistActualPosition`. Expected:
   the count reaches 2 and stops, the third panel halts under the pressure rolls, and the plates release
   only after the platform has stopped at the top.
4. **Hoist stall during the climb.** With the platform between the new threshold and the top, break a
   gap-free photocell by hand. The infeed promise should drop, the rolls should stop, and no panel should
   be released onto the plates. On restoring it, the hoist should complete its climb before the plates
   release.
5. **Auto start with the platform low.** Only after fix I. Switch to auto with the platform part way down.
   `_L25_Platform_RdyInfeed` must stay 0 until the platform is genuinely up.
6. **Single-plate failure.** Only after fix E, and with the line empty. Disconnect one plate proximity
   switch. The line should stop with the valve alarm, not feed a panel.

## Record these in the change log

Whatever is applied should go into `Changes History logs` in the existing format, with the date, initials
and rung reference. Two entries deserve a note in their own right, because the next person will otherwise
repeat them:

- The 1700 threshold was documented as "100mm from the top" and is not. Record the measured top position
  alongside the new `_L25_AlmostUpPosition` value.
- `L20` rung 33's `MOV(70, ...)` was never logged. If it is removed, log the removal and the HMI setpoint
  it hands control back to.
