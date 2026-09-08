# Review of the KneeJ and StaggD adjustments

**Version 1.0.** Scope limited to the changes recorded in `Changes History logs` for
KneeJ (2026-05-21, 2026-06-11) and StaggD (2026-06-18). Nothing else in the routines is
reviewed here; the wider review is in `ANALYSIS_Hoist_DroppingPlate_Coordination.md`.

Four corrections have been applied to the L5X exports on this branch. Two logged adjustments were
checked and deliberately left alone. Four logged adjustments are in programs that are not in this
repository and could not be checked.

## Verdict summary

| Date | Who | Rung | Logged intent | Verdict | Action |
|---|---|---|---|---|---|
| 2026-05-21 | KneeJ | L21 R10 | Plates delay 0.3 s to 0 s | Rung is fine, the value is not | Corrected: minimum 0.2 s restored |
| 2026-06-11 | KneeJ | L21 R11 | Add `_L25_Platform_RdyInfeed` | Right idea, wrong position in the rung | Corrected: moved ahead of the timer |
| 2026-06-11 | KneeJ | L25 R37 | One slow-down at 1700, "100 mm from the top" | Distance is wrong, and the release bit was buried in a speed branch | Corrected: separate branch at 2000 |
| 2026-06-11 | KneeJ | L25 R37 | Down speed 1800 to 2000 | Sound | Left alone |
| 2026-06-11 | KneeJ | L25 R32, R37 | Bypass the delay when plates and aligners are done | Sound intent, one dependency worth knowing | Left alone |
| 2026-06-11 | KneeJ | L25 R47 | Restart when the platform is almost up | Primary regression, three defects | Corrected |
| 2026-06-11 | KneeJ | Saw 1 and Feeder Area | Four separate edits | Not in this repository | Not reviewed |
| 2026-06-18 | StaggD | L20 R26, R27 | Footpedal stop off-delay | Correctly implemented | Left alone, one field check |

---

## 2026-05-21, KneeJ, L21 rung 10

**Logged:** "Dropping plates delay was locked to 0.3sec. Have now set to 0sec, no delay."

**What the code does.** Rung 10 is a clamp, not a lock. It caps
`HMI_972_1120CPSettingDroppingPlatesUp` at 0.3 and leaves anything below that alone. The setpoint is
0.0 in the export. Rung 11 multiplies the setpoint by 1000 into `_L21_ToUpDelay.PRE`, so the preset is
0 ms and the timer's `.DN` is true on the scan it is enabled.

**Verdict.** Making the value adjustable instead of locked was an improvement. Setting it to zero was
not. With no delay the plates retract on the very scan `_L25_Platform_RdyInfeed` latches, and that bit
is latched by `L25` rung 27 when the max-height beam is broken, which is when the drive is *commanded*
off rather than when the platform has stopped. Panels are released onto a platform that is still
decelerating.

**Correction applied.** A minimum clamp of 0.2 s, alongside the existing maximum of 0.3 s.

```
[GRT(HMI_972_1120CPSettingDroppingPlatesUp,0.3) MOV(0.3,HMI_972_1120CPSettingDroppingPlatesUp) ,LES(HMI_972_1120CPSettingDroppingPlatesUp,0.2) MOV(0.2,HMI_972_1120CPSettingDroppingPlatesUp) ,LES(HMI_972_1120CPStkrSetSideAlignmentStart,0) MOV(0,HMI_972_1120CPStkrSetSideAlignmentStart) ];
```

Branches in one rung execute left to right, so a setpoint of 0.5 is capped to 0.3 by the first branch
and then passes the second unchanged. A setpoint of 0.0 or a negative value is raised to 0.2. The
operator keeps a 0.2 to 0.3 s adjustment range and can no longer select zero.

If you disagree that zero should be unavailable, deleting the added branch restores the previous
behaviour exactly.

**Note for later.** The tag description on `HMI_972_1120CPSettingDroppingPlatesUp` reads "unit 0.1s"
while rung 11 multiplies by 1000, which treats it as seconds. The description is wrong. Correcting
one without the other would change the delay by a factor of ten.

## 2026-06-11, KneeJ, L21 rung 11

**Logged:** "Add `_L25_Platform_RdyInfeed` so Plates wont drop until Lift is full up again."

**What the code does.** The contact was appended at the end of the branch, after the timer:

```
XIC(_L21_PanelComing) MUL(...,1000,_L21_ToUpDelay.PRE) TON(_L21_ToUpDelay,?,?) XIC(_L21_ToUpDelay.DN) XIC(Out_PromiceFwd_SideAlignment) XIC(_L25_Platform_RdyInfeed) ONS(_L21_Ons_2)
```

**Verdict.** The interlock itself is correct and necessary. It is the only thing that stops the plates
dropping while the hoist is still climbing, and it must stay.

Its position is wrong. Everything before the `TON` is the timer's enable, so the enable is
`_L21_Auto` and `Out_PromiceFwd_1123M1` and `_L21_PanelComing`. The delay whose description reads
"How long dropping plates keep panel before dropping" is therefore timed from the moment the infeed
promise is granted, not from the moment the panel has landed and the platform is in position. That is
harmless while the preset is 0 ms and wrong the moment anyone dials a value in, which rung 10 above now
forces.

**Correction applied.** The side alignment and platform ready contacts moved ahead of the timer.

```
XIC(_L21_Auto)XIC(Out_PromiceFwd_1123M1)[XIC(_L21_PanelComing) XIC(Out_PromiceFwd_SideAlignment) XIC(_L25_Platform_RdyInfeed) MUL(HMI_972_1120CPSettingDroppingPlatesUp,1000,_L21_ToUpDelay.PRE) TON(_L21_ToUpDelay,?,?) XIC(_L21_ToUpDelay.DN) ONS(_L21_Ons_2) ,XIO(HMI_972_1120CP_Pb_DroppingPlatesSelect) XIC(I_PC_Stkr_StackOnPlatform) ]OTL(_L21_Auto_Up);
```

The logic that decides *whether* to release is unchanged. Only the point from which the settle time is
measured has moved. This also makes the rung self-correcting when a panel is still arriving: the panel
engages the side aligners, `Out_PromiceFwd_SideAlignment` drops, the timer resets, and the release waits
for the aligners to finish.

## 2026-06-11, KneeJ, L25 rung 37

**Logged:** "Remove first slow speed when Hoist is returning up. Now just one slow down point at 1700
high (100mm from the top). Incresed Down Speed from 1800 to 2000."

**Two separate things happened in this rung.** The speed profile changed, and `_L25_Platform_AlmostUp`
was created inside the 400-speed branch.

**Verdict on the down speed.** 1800 to 2000 is sound and has been left alone.

**Verdict on the slow-down point.** 1700 is not 100 mm from the top. Three constants in the same
routine put the receiving position at 2100 to 2150:

| Reference | Value | Rung |
|---|---|---|
| Positioning target written for the hoist | 2100 | L25 R40 |
| Manual jog slow-down threshold on the way up | 2150 | L25 R36 |
| `StackerHoistActualPosition` in the export snapshot | 2087 | live data |

If the top were 1800, rung 36's manual slow-down at 2150 could never be reached and manual jog would
have no slow approach at all. So 1700 is roughly 400 mm below the top. The empty platform crawls that
whole distance at 400, which is about one extra second on every pack change.

**Verdict on `_L25_Platform_AlmostUp`.** It was written as an `OTE` inside the branch
`XIC(...IN_RunFw) GEQ(...,1700)`. Being qualified by `IN_RunFw` means it collapses the instant the drive
stops or stalls. That matters because rung 47 uses it to hold the infeed promise open, so the promise
flickers off whenever the hoist pauses.

**Correction applied.** The release bit moved to its own parallel branch, with its own threshold and no
`IN_RunFw` qualifier. The 1700 speed slow-down is untouched, because deceleration distance and infeed
release are two different decisions and should not share one comparison.

```
... ,XIC(_L25_StackerHoistUp_Down.IN_RunFw) GEQ(_L25_StackerHoistUp_Down.OUT_ActualPosition,1700) MOV(400,_L25_StackerHoistUp_Down.IN_SpeedReferenceEU) ,GEQ(_L25_StackerHoistUp_Down.OUT_ActualPosition,2000) OTE(_L25_Platform_AlmostUp) , ...
```

**2000 is a calculated starting value, not a measured one.** It is 100 mm below rung 40's target of
2100. Read `StackerHoistActualPosition` with the platform empty and fully up in auto, then set this
literal to that reading minus 100. A rung comment has been added saying so.

The new branch is true whenever the platform is above 2000 in auto, moving or not. Two cases to check
if you want to satisfy yourself it is safe. During normal stacking near the top, rung 47 branch 1
already grants the promise through `_L25_Platform_RdyInfeed`, so nothing changes. During the pack-change
descent, rung 47's series `XIO(OUT_Stkr_Is_Full)` blocks branch 2, so nothing changes.

## 2026-06-11, KneeJ, L25 rungs 32 and 37, the delay bypass

**Logged:** "Add bypass rung, if Droping Plates and Side Aligners are done its ok to drop the pack."

**What the code does.** In both rungs a parallel branch `XIC(OUT_PromiceFromDroppingPlates)
XIC(Out_PromiceFwd_SideAlignment)` sits alongside the original timer, so the hoist no longer waits out
`_L25_StackDowndelay` (4000 ms) in rung 32 or `_L25StkrFull` (1000 ms) in rung 37.

**Verdict: sound. Left alone.** It affects the downward move only and is not implicated in either
reported fault.

**One dependency worth knowing.** `OUT_PromiceFromDroppingPlates` depends on
`_L21_Sensors_Sylinders_Out`, and rung 19 of `L21` builds that from
`[XIC(I_PR_Stkr_DroppingPlateLeftOn) ,XIC(I_PR_Stkr_DroppingPlateRightOn) ]`, an OR. Either plate
extended is enough to satisfy it. Before this bypass the 4 s and 1 s timers masked that. Now the
platform can dive away at 2000 with only one plate confirmed out. That rung is original vendor code and
outside this review, so it has not been touched, but it is the reason to trend both plate sensors
together. It is finding 5 in the wider analysis.

## 2026-06-11, KneeJ, L25 rung 47

**Logged:** "When dropping plates are in use, Restart when Platform is almost up (Load a panel onto the
plates)."

**Verdict: this is the primary regression.** Three defects.

1. **It releases the infeed about 400 mm early**, because `_L25_Platform_AlmostUp` was set at 1700.
   The panel that arrives then engages the side aligners and the gap-free photocells. Every one of
   `Out_PromiceFwd_SideAlignment`, `Out_PromiceFwd_SideAlignmentWagon`, `I_PC_Stkr_GapFreeLeftSide`,
   `I_PC_Stkr_GapFreeRightSide` and `_L31_AV_FrontAlignment.OUT_Closed` is a series contact in rung 3,
   which produces `_L25_PromiceUP`, which is a series contact for `IN_RunFw` in rung 34. The panel the
   hoist just called for removes the hoist's own permission to finish climbing. That is the reported
   "not fast enough, not close enough up".
2. **Branch 2 omits `XIO(_L25_HoistAutoMode_Resumed)`**, which branch 1 carries. With the plates
   selected, a manual to auto recovery that takes the hoist above the threshold grants the infeed
   promise during the recovery.
3. **Nothing checks whether the plates have room.** The log says "Load a panel onto the plates",
   singular, but the rung has no such limit. While the hoist is stalled per defect 1, panels keep
   arriving. That is the reported third panel.

**Correction applied.**

```
XIO(OUT_Stkr_Is_Full)XIO(_L25_HoistAutoMode_Resumed)[XIC(_L25_Platform_RdyInfeed) ,XIC(HMI_972_1120CP_Pb_DroppingPlatesSelect) XIC(_L25_Platform_AlmostUp) XIC(OUT_PromiceFromDroppingPlates) XIO(OUT_PanelOnDroppingPlates) ][OTE(Out_PromiceFwd_1123M1) ,OTE(HMI_972CP_STatusStkrRdyInfeed) ];
```

`XIO(_L25_HoistAutoMode_Resumed)` moved into the series section so it covers both branches.
`XIO(OUT_PanelOnDroppingPlates)` added to branch 2 so the early restart only fires while the plates are
empty. Defect 1 is addressed by the rung 37 correction above. `OUT_PanelOnDroppingPlates` is an existing
controller tag written by `L21` rung 14; its definition has been added to this export's context section
so the routine imports without an unresolved reference.

**What this does to throughput.** The early restart now loads one panel during the climb and stops,
which is what the log entry says it was for. Branch 1 is untouched, so normal stacking is unaffected.
In the usual case the plates are empty when a pack change starts, because the last panel has just been
dropped and feeding then stops, so the early restart still fires and the throughput gain is kept.

**Residual risk, stated plainly.** `OUT_PanelOnDroppingPlates` follows `_L21_PanelComing`, which `L21`
rung 9 latches when the panel is *announced*, not when it lands. So the promise drops earlier than the
panel arrives. The panel is then drawn up to `I_PC_Stkr_PanelUnderPressingRoll` and held there by the
interlock chain, and it is released onto the plates when the platform tops out and branch 1 takes over.
If the panel takes longer than the settle time to reach the side aligners after that, the plates can
open on an empty shelf and the panel falls straight through onto the platform. That is untidy but it is
not a crash, and the platform is in position by then. Test 3 below is aimed at exactly this. If it
shows up, raise the settle time toward 0.3 s, or delete the single `XIO(OUT_PanelOnDroppingPlates)`
contact and rely on the threshold change alone.

## 2026-06-18, StaggD, L20 rungs 26 and 27

**Logged:** rung 27 a new off-delay timer driven by `OUT_OutfeedConvStoppedFromFootPedal` examine off,
rung 26 the examine-off contact replaced by `TMR_Footpedal_StopOffDelay.DN` for 1500 ms, so the pressure
rolls do not stop until the board is in the stacker.

**What the code does.**

```
R27  XIO(OUT_OutfeedConvStoppedFromFootPedal)TOF(TMR_Footpedal_StopOffDelay,?,?);
R26  [XIC(_L20_Manual) XIC(VFD1120M1.DevCtrl.Man_Fw) ,XIC(_L20_Auto) XIC(OUT_Stacker_Feeding_ON) XIC(VFD1120M1.DevCtrl.Auto_Fw) XIC(TMR_Footpedal_StopOffDelay.DN) ]XIC(_L20_PromiceFwd)GEQ(VFD1120M1.Info.SpeedActual,0)OTE(_L20_Stkr_PressureRoll.IN_RunFw);
```

**Verdict: correctly implemented. Left alone.** Checked four ways.

- The `TOF` semantics are right. `.DN` is true while the rung is true and is held for the preset after
  it goes false, so the rolls keep running for 1500 ms after the pedal is pressed.
- `TMR_Footpedal_StopOffDelay.PRE` is 1500, matching the log.
- The contact was placed in the auto branch only, so manual jog is still under direct operator control.
  That is correct.
- The off-delay is not defeated by the rest of the rung. `_L20_PromiceFwd` sits in the series section and
  could in principle cut the delay short, but it is built in rung 4 from the downstream promises and the
  stacker-full latch. The footpedal stops the outfeed belt conveyors, which are upstream, so it does not
  reach `_L20_PromiceFwd`. The `VFD1120M1.DevCtrl.Auto_Fw` latch in rungs 23 and 24 is likewise not
  affected by the pedal.

**One field check.** Whether 1500 ms is long enough is a mechanical question this code cannot answer.
Time a board from the moment the pedal is pressed to the moment its trailing edge clears
`I_PC_Stkr_PanelUnderPressingRoll` at production speed. If that is longer than 1500 ms, raise the preset.
I could not calculate it: the drive AOI is called with `IN_SpeedRefPoint` and `IN_SpeedAtRefPointEU` both
set to 1, so the engineering-unit calibration described in the AOI parameter help was never done and the
speed reference cannot be converted to mm/s from the code alone.

**Log wording only.** The log names the tag `TMR_Footpedal_StopOffDelay_Dn`; the code uses
`TMR_Footpedal_StopOffDelay.DN`. Same thing.

## Not reviewable from this repository

Four of KneeJ's 2026-06-11 edits are in programs that are not in this export.

| Entry | Status |
|---|---|
| `P_Saw_1 > L11_Saw1_Chain > R41`, disable the saw gap increase during unloading | Not present |
| `P_FeederArea > L23_FeederHoistConv > R30`, swap `OUT_RunningFw` for `IN_RunFw` and `IN_RunBw` | Not present |
| `P_FeederArea > L24_FeederHoist > R24`, bypass board under press rolls to say stack is empty | Not present |
| `P_FeederArea > L25_FeederCarriage > R44`, allow first pickup before the hoist reaches feeding height | Not present |

The last two are worth flagging even unseen. Both are the same pattern as `L25` rung 47: granting a
permission earlier than the mechanism is actually ready. "Allow first pickup of new pack to happen
before hoist fully reaches the feeding height" is almost word for word the stacker change that caused
the fault being investigated here. Export `P_FeederArea` and it can be checked the same way.

---

## Commissioning

1. **Measure first.** Put the platform empty and fully up in auto and read
   `StackerHoistActualPosition`. Set the `2000` literal in `L25` rung 37 to that reading minus 100.
   Do this before downloading anything else.
2. Download `L21` and `L25`. `L20` is unchanged.
3. Confirm `HMI_972_1120CPSettingDroppingPlatesUp` has moved to 0.2 on the first scan. Rung 10 forces it.
4. Run the tests below before releasing the line to production.

## Tests

Run in auto with the dropping plates selected.

1. **Normal stacking, twenty panels, no pack change.** One plate drop per panel, hoist indexing as
   before. Nothing in these corrections touches this path, so it is the regression check.
2. **Pack change, upstream starved.** Platform descends, discharges, returns, no plate cycle. Trend
   `_L25_Platform_AlmostUp` and confirm it comes true at the new threshold and stays true once the
   platform stops, instead of dropping out.
3. **Pack change, upstream full.** The main test. Trend `Out_PromiceFwd_1123M1`,
   `OUT_PanelOnDroppingPlates`, `_L21_PanelComing`, `I_PC_Stkr_PanelUnderPressingRoll`,
   `Out_PromiceFwd_SideAlignment`, `_L21_Auto_Up` and `StackerHoistActualPosition`. Expected: one panel
   released during the climb, the next held under the pressure rolls, the plates releasing only after
   the platform has stopped and the aligners have finished. Watch specifically for the plates opening
   before the panel reaches the aligners, which is the residual risk noted under rung 47.
4. **Hoist stalled during the climb.** With the platform between the threshold and the top, break a
   gap-free photocell by hand. The infeed promise should stay off, no further panel should be fed, and
   the plates should not release. Restore it and the hoist should finish its climb before the plates
   open.
5. **Manual to auto recovery.** Jog the hoist in manual, return to auto, and confirm no infeed promise
   is granted during the recovery climb. This is the branch 2 lockout.
6. **Footpedal.** With a board in the pressure rolls, press an outfeed belt conveyor stop footswitch.
   Time how long the rolls keep running and whether the board clears. This is StaggD's rung, unchanged;
   the test is to confirm 1500 ms is enough.

## Change log entry to add once downloaded

Draft, in the site's existing format. Do not add it to `Changes History logs` until the download is
done, so the log keeps meaning what is in the controller.

```
<date>	<initials>	Corrections to the 2026-05-21 and 2026-06-11 Outfeed Pack Change adjustments
			P_Stacker > L21_DroppingPlates > R10
				Restore a minimum plates settle time of 0.2sec. Max stays 0.3sec.
				Zero delay was releasing panels onto a platform still slowing down.
			P_Stacker > L21_DroppingPlates > R11
				Move Out_PromiceFwd_SideAlignment and _L25_Platform_RdyInfeed ahead of the
				settle timer so the delay times from the platform being in position.
			P_Stacker > L25_StackerHoist_UpDown > R37
				_L25_Platform_AlmostUp moved to its own branch at <measured top - 100>,
				no longer qualified by IN_RunFw so it holds when the hoist stops.
				Speed slow down point stays at 1700. Down speed unchanged at 2000.
			P_Stacker > L25_StackerHoist_UpDown > R47
				XIO _L25_HoistAutoMode_Resumed now covers both branches.
				Early restart only while the plates are empty, one panel per pack change.
```

## Still open, outside this review

Six defects found in the wider review are untouched here because they are original vendor code, not
KneeJ or StaggD adjustments. They are documented with proposed corrections in
`ANALYSIS_Hoist_DroppingPlate_Coordination.md` and `PROPOSED_FIXES_Rung_Level.md`. In severity order:
the OR on the two plate position sensors (`L21` R19); no counter bounding the plates to two panels
(`L21` R22 plus new logic); the hard-coded `MOV(70, ...)` overriding the pressure roll speed
(`L20` R33); `_L25_Platform_RdyInfeed` latched on the auto-start scan regardless of position
(`L25` R27); the dead abort branch carrying a contact and its own inverse (`L25` R33); and the
right-hand plate retracted sensor shorted out by an empty branch (`L21` R20).
