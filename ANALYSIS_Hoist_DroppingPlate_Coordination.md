# Stacker hoist / dropping plate coordination - code review

> **Scope note.** This is the full review of all three routines. The active work is narrowed to the
> KneeJ and StaggD adjustments recorded in `Changes History logs`; see `KNEEJ_STAGGD_REVIEW.md` for that
> review and for the corrections applied to the exports on this branch. Findings here that fall outside
> those adjustments are parked, not withdrawn.

**Version 1.0** (first issue; no prior version to compare against).
**Scope:** `L20_PressureRoll`, `L21_DroppingPlates`, `L25_StackerHoist_UpDown` (Studio 5000 L5X exports,
controller `P2850_CHH_Trimming_Repairing`, exported 08 Sep 2026) plus `Changes History logs`.
**Reported symptoms:** (1) the hoist does not come up fast enough or close enough to catch panels released
by the dropping plates; (2) a third panel is fed on top of the two panels already resting on the plates.

---

## Read this first: what would make this analysis wrong

Leading with the counter-case, because three of the findings below depend on things I cannot see in
this repository.

1. **Three routines is not the whole interlock.** `L22`/`L23` (side alignment, alignment wagon),
   `L24` (rear stopper / carriage), `L31` (front alignment) and the outfeed pack conveyor are not in
   the export. Signals such as `Out_PromiceFwd_SideAlignment`, `Out_PromiceFwd_SideAlignmentWagon`,
   `I_PC_Stkr_GapFreeLeftSide/RightSide`, `_L31_AV_FrontAlignment.OUT_Closed`, `Out_PanelCominToStacker`
   and `OUT_StackerIsEmpty` are consumed here but produced elsewhere. Finding 1 assumes the side
   aligners drop their forward promise while they are squaring an incoming panel. That is the normal
   convention on this style of stacker, but it is an **inference**, not something I verified.
2. **The tag values in the export are not a coherent snapshot.** `_L25_StackHeight` reads 136 while
   `HMI_972Cp_Stkr_StatusStackHeight`, which rung 29 copies from it in the same scan, reads 51. The
   decorated data was collected over several scans. Timer presets and HMI setpoints are configuration
   and are reliable; live bits and positions are indicative only.
3. **The mechanical geometry is unverified.** I have no drawing giving the platform receiving height,
   the plate stroke, or the panel entry plane. Finding 2 rests on code constants (rung 40 writes a
   target of 2100, rung 36 slows manual jog above 2150) rather than a measurement.

Where a finding survives these caveats using only the three exported routines, it is labelled
**established fact**. Everything else is labelled **inference** or **speculation**.

---

## How the machine is meant to work

Think of the dropping plates as two shelf brackets that swing into the panel path just under where a
panel arrives. The hoist platform sits below them, holding the growing stack. Normally the shelf takes
the panel, the side aligners square it up on the shelf, the shelf pulls out, and the panel drops a few
millimetres onto the stack. The shelf then swings back in for the next panel.

When a pack is finished the platform has to leave: it drives all the way down, the pack rolls off, and
the platform comes back up empty. During that trip the shelf is the only thing holding incoming panels,
so panels queue on it. When the platform is back in position the shelf pulls out and the queued panels
drop together.

Two conditions have to hold at that moment, and both are what the reported faults are about:

- the platform must be in position **before** the shelf lets go, and
- no more panels may be fed while the shelf already has its two.

Naming in the code is the reverse of intuition and is worth stating once. `_L21_Auto_Down` /
`IN_DriveOpen` / `_L21_Sensors_Sylinders_Out` is the **catching** position (cylinders extended, plates in
the panel path). `_L21_Auto_Up` / `IN_DriveClose` / `_L21_Sensors_Sylinders_In` is the **released**
position (cylinders retracted, panel falls). Rungs 17 to 20 of `L21` establish this.

---

## The pack-change sequence as the code actually runs it

| Step | Where | What happens |
|---|---|---|
| 1 | L25 R30 | Count or height reached. `_L25_Platform_RdyInfeed` unlatched, `OUT_Stkr_Is_Full` and `PackUnloading` latched. |
| 2 | L21 R15 | `XIC(OUT_Stkr_Is_Full)` holds `_L21_Auto_Down`, so the plates stay extended and catch panels. |
| 3 | L25 R32, R37 | Hoist latches `Auto_Bw` and descends. Down speed 2000. |
| 4 | L25 R33 | At the down limit `_L25_DowmStopPlace2` unlatches `Auto_Bw`. Pack discharges. |
| 5 | L25 R30 | `OUT_StackerIsEmpty` unlatches `OUT_Stkr_Is_Full`. |
| 6 | L25 R6, R26, R34, R37 | Hoist rises. 2500 while `OUT_StackerIsEmpty`, then 400 once position >= 1700. |
| 7 | L25 R37 | At position >= 1700, `_L25_Platform_AlmostUp` energises. |
| 8 | L25 R47 | `_L25_Platform_AlmostUp` grants `Out_PromiceFwd_1123M1`, which releases the pressure rolls in L20 R4. |
| 9 | L25 R27 | Max-height beam blocked. `Auto_Fw` unlatched, `_L25_Platform_RdyInfeed` latched. |
| 10 | L21 R11 | Plates retract, panels drop. |
| 11 | L21 R13, R15 | After 1.2 s the plates re-extend. |

Steps 7 and 8 are the 11 Jun 2026 change. They are where both reported faults originate.

---

## Finding 1 - the infeed is released while the hoist still needs its own permission to finish climbing

**Classification: inference** (the code paths are established fact; the aligner behaviour is inferred).
**Severity: high. This is the primary cause of "the hoist does not get up in time".**

`L25` rung 47, branch 2, added on 11 Jun 2026:

```
XIC(HMI_972_1120CP_Pb_DroppingPlatesSelect) XIC(_L25_Platform_AlmostUp) XIC(OUT_PromiceFromDroppingPlates)
```

grants `Out_PromiceFwd_1123M1`. That signal is a series contact in `L20` rung 4, so the pressure rolls
start and a panel enters the stacker while the platform is still climbing.

That panel then sits in the side-alignment zone. `L25` rung 3 builds `_L25_PromiceUP` from, among others:

```
XIC(I_PC_Stkr_GapFreeRightSide) XIC(I_PC_Stkr_GapFreeLeftSide) XIC(Out_PromiceFwd_SideAlignment)
[XIC(Out_PromiceFwd_SideAlignmentWagon) , XIC(_L25_Manual)] XIC(_L31_AV_FrontAlignment.OUT_Closed)
```

and `_L25_PromiceUP` is a series contact in rung 34, which produces `IN_RunFw`. So the panel that the
hoist just called for can remove the hoist's own permission to finish its climb. The hoist stops short
of the receiving position and waits for the aligners.

It gets worse on the way back. `_L25_Platform_AlmostUp` is an OTE written **inside** the rung 37 branch
`XIC(_L25_Auto) XIC(...IN_RunFw) GEQ(...OUT_ActualPosition,1700)`. It is not latched. The instant
`IN_RunFw` drops, `_L25_Platform_AlmostUp` drops, `Out_PromiceFwd_1123M1` drops, and the pressure rolls
stop mid-panel. When the aligners release, the hoist restarts from standstill and, because it is already
above 1700, creeps the rest of the way at 400.

Net effect at the machine: the platform arrives late and low, exactly as reported.

## Finding 2 - the slow-approach point is roughly 400 mm from the top, not 100 mm

**Classification: inference from code constants. Verify on the machine before changing.**
**Severity: high. This is the other half of "not fast enough, not close enough".**

The change log states the 11 Jun 2026 edit left "just one slow down point at 1700 high (100mm from the
top)". Three constants in the same routine disagree:

| Reference | Value | Rung |
|---|---|---|
| Positioning target written for the hoist | 2100 | L25 R40 |
| Manual jog slow-down threshold on the way up | 2150 | L25 R36 |
| Auto slow-down and "almost up" threshold | 1700 | L25 R37 |
| `StackerHoistActualPosition` in the export snapshot | 2087 | live data |

If the receiving position were 1800, rung 36's manual slow-down at 2150 could never be reached and
manual jog would have no slow approach at the top at all. The consistent reading is that the top is
around 2100 to 2150 and that **1700 is about 400 mm below it, not 100 mm**.

Two consequences, and they compound:

- The empty platform crawls the last 400 mm at 400 instead of covering most of it at 2500. At 400 that
  is roughly one second of crawl added to every pack change.
- The infeed release in finding 1 fires a full 400 mm early, which is long enough for a second and
  often a third panel to be fed. That is the direct link to the third-panel fault.

## Finding 3 - nothing counts the panels on the plates, and the one interlock that could stop a third panel is bypassed exactly when the plates are in use

**Classification: established fact.**
**Severity: high. This is the cause of the third-panel fault.**

`L21` rung 22 builds `OUT_PromiceFromDroppingPlates`:

```
[[XIC(HMI_972_1120CP_Pb_DroppingPlatesSelect) XIO(OUT_PanelOnDroppingPlates)
 ,XIO(I_PC_Stkr_StackOnPlatform)
 ,XIC(OUT_Stkr_Is_Full) ] XIC(_L21_Sensors_Sylinders_Out) , ... ]
```

The first sub-branch is the real interlock: plates selected and no panel on them. The other two
sub-branches sit in parallel and short it out. Through the whole plate-buffering window the platform is
empty, so `XIO(I_PC_Stkr_StackOnPlatform)` is true and the interlock does nothing. During the descent
`XIC(OUT_Stkr_Is_Full)` does the same job.

`OUT_PromiceFromDroppingPlates` then feeds both `L20` rung 4 (rolls run) and `L25` rung 47 branch 2
(infeed promise). With finding 1 holding the promise open from 1700 upward, panels keep arriving.

There is also no count anywhere. `OUT_PanelOnDroppingPlates` (`L21` rung 14) is a single BOOL meaning
"at least one panel". `_L21_PanelComing` (rung 9) is a latch, not a counter. Nothing in these three
routines knows the plates hold two panels, so nothing can stop the third.

This bypass was harmless before 11 Jun 2026. The old rung 47 gated the infeed on `_L25_Platform_RdyInfeed`,
which only latches at the top, so the rolls could not run during the climb. The new branch 2 exposed a
latent defect rather than creating one.

## Finding 4 - the plates release with zero settle time, on the same scan the beam is broken

**Classification: established fact.**
**Severity: medium to high.**

`HMI_972_1120CPSettingDroppingPlatesUp` is 0.0 in the export, so `L21` rung 11 computes
`_L21_ToUpDelay.PRE = 0` and the timer's `.DN` is true on the scan it is enabled. The remaining series
contacts are `Out_PromiceFwd_SideAlignment`, `_L25_Platform_RdyInfeed` and a one-shot, so the plates
retract on the very scan `_L25_Platform_RdyInfeed` latches.

`_L25_Platform_RdyInfeed` is latched by `L25` rung 27 when the max-height beam is broken. That is when
`Auto_Fw` is **commanded** off, not when the platform has stopped. The panels are therefore released
onto a platform that is still decelerating through its final approach.

There is a second problem in the same rung. The `_L21_ToUpDelay` timer is enabled by
`XIC(_L21_Auto) XIC(Out_PromiceFwd_1123M1) XIC(_L21_PanelComing)` only. It does **not** include
`_L25_Platform_RdyInfeed` or `Out_PromiceFwd_SideAlignment`, both of which sit after the timer in the
rung. So the setting whose description reads "How long dropping plates keep panel before dropping" is
now timed from "the hoist passed 1700", not from "the panel has landed and been squared". Even if the
setting is restored to a non-zero value it will not do what its name says.

Separately, the tag description says the unit is 0.1 s while rung 11 multiplies by 1000, which treats it
as seconds. Rung 10 clamps it to 0.3. One of the description, the clamp or the scaling is wrong. At the
current value of 0.0 this makes no numeric difference, but it will the moment someone dials a value in.

## Finding 5 - `_L21_Sensors_Sylinders_Out` is an OR of the two plate sensors

**Classification: established fact.**
**Severity: high, and it interacts badly with the increased down speed.**

`L21` rung 19:

```
[XIC(I_PR_Stkr_DroppingPlateLeftOn) ,XIC(I_PR_Stkr_DroppingPlateRightOn) ]OTE(_L21_Sensors_Sylinders_Out);
```

Either plate extended is enough to declare the plates in the catching position. That bit is used as
`IN_IsOpenSensor` on the valve AOI with `IN_OpenSensorInUse` tied to `Always_1`, and it is a series
contact in rung 22.

The consequence became sharper with the 11 Jun 2026 change. `L25` rungs 32 and 37 now let the hoist drop
away at 2000 as soon as `OUT_PromiceFromDroppingPlates` and `Out_PromiceFwd_SideAlignment` are true,
instead of waiting out the 4 s `_L25_StackDowndelay` and the 1 s `_L25StkrFull`. With rung 19 as an OR,
the platform can leave at full speed while only one plate has actually extended, leaving the incoming
panel unsupported on one side. That is a credible mechanism for panels landing skewed or dropping through.

## Finding 6 - `_L21_Sensors_Sylinders_In` ignores the right-hand plate

**Classification: established fact.**
**Severity: low as currently wired.**

`L21` rung 20:

```
XIC(I_PR_Stkr_DroppingPlateLeftOff)[XIC(I_PR_Stkr_DroppingPlateRightOff) ,]OTE(_L21_Sensors_Sylinders_In);
```

The empty parallel branch unconditionally shorts out `XIC(I_PR_Stkr_DroppingPlateRightOff)`, so the bit
is the left sensor alone. Impact is limited today because the valve AOI call in rung 21 passes
`Always_0` for `IN_ClosedSensorInUse`, so the retracted feedback is time-based rather than sensor-based.
The bit is still wrong and will mislead anyone reading it on the HMI or in a fault trace.

## Finding 7 - the "max run time to down" abort is dead logic

**Classification: established fact.**
**Severity: medium. Do not simply re-enable it.**

`L25` rung 33 ends with:

```
TON(_L25_MaxRunTimetoDownWithoutPanels,?,?) XIC(_L25_MaxRunTimetoDownWithoutPanels.DN) XIO(_L25_MaxRunTimetoDownWithoutPanels.DN)
```

A contact and its own inverse in series. That branch can never be true, so the 15 s abort never unlatches
`Auto_Bw`. The export confirms the timer runs and times out in service: `.PRE` 15000, `.ACC` 15014,
`.DN` 1.

The reason it was probably killed matters. Rung 32 latches `Auto_Bw` on `_L25_Platform_RdyInfeed`, and
rung 33 otherwise only unlatches it at the down limit, so `Auto_Bw` stays latched for the whole stack
build while the actual motion is gated by `_L25_PromiceDown` and by the speed branches in rung 37. The
timer is enabled by `XIC(_L25_Auto) XIC(Auto_Bw)`, which is true for that entire period. Re-enabling the
branch as written would abort `Auto_Bw` 15 s into every stack and break the indexing. The fix is to
re-scope the timer, not to remove the contradictory contact. See the fixes document.

## Finding 8 - `L20` rung 33 overwrites the pressure roll speed with a hard-coded 70

**Classification: established fact.**
**Severity: medium, and directly relevant to feed timing.**

```
RUNG 29  XIC(_L20_Manual)MOV(HMI_972_1110CP_Setting_OutfeedSpeed,_L20_Stkr_PressureRoll.IN_SpeedReferenceEU);
RUNG 30  XIC(_L20_Auto)  MOV(HMI_972_1110CP_Setting_OutfeedSpeed,_L20_Stkr_PressureRoll.IN_SpeedReferenceEU);
RUNG 31  XIO(...IN_RunFw)XIO(...IN_RunBw)MOV(0,_L20_Stkr_PressureRoll.IN_SpeedReferenceEU);
RUNG 32  MUL(...,1000.0,...)DIV(...,60.0,...);        // m/min -> mm/s
RUNG 33  MOV(70,_L20_Stkr_PressureRoll.IN_SpeedReferenceEU)CPS(AF1_1120M1:I,VFD1120M1.ModuleData.Input,1);
RUNG 34  _VFDControl_CT(... _L20_Stkr_PressureRoll.IN_SpeedReferenceEU ...);
```

Rung 33 runs unconditionally, after everything that computes the value and before the drive AOI consumes
it. So the HMI outfeed-speed setting does nothing for the pressure rolls, rung 31's stop-at-zero does
nothing, and rung 32's unit conversion is pointless. The rolls are always commanded 70 EU.

This is not in the change log, and it sits in a rung whose comment is about copying drive input data,
which is how it has stayed invisible. It matters here because panel arrival timing at the stacker cannot
be tuned from the HMI, and coordinating arrival with the plate drop is exactly the problem being chased.

## Finding 9 - on the auto-start scan the hoist's "go up" is cancelled and the platform is declared ready wherever it is

**Classification: established fact.**
**Severity: medium. Pre-existing, not from the recent changes.**

`_L25_AutoPulse` is a one-shot, true for one scan when auto turns on.

- Rung 26 uses it: `XIC(_L25_AutoPulse) XIC(_L25_PromiceUP) XIO(I_PC_Stkr_StackOnPlatform) OTL(Auto_Fw)`.
- Rung 27, scanned immediately after, also uses it as an input branch and unlatches `Auto_Fw`.

Rung 26's latch is undone on the same scan, so the "send the platform up at auto start" logic can never
take effect. Worse, rung 27's output branch
`XIC(_L25_Auto) XIO(HMI_972CP_Pb_Reset_All) XIO(...OUT_MaxIdleTimeOver) OTL(_L25_Platform_RdyInfeed)`
is energised by that same `_L25_AutoPulse` branch with **no position or beam condition**. Switching to
auto therefore latches "platform is up and ready to receive panels" regardless of where the platform
actually is, which in turn latches `Auto_Bw` in rung 32, grants the infeed promise in rung 47 branch 1,
and permits the plates to drop in `L21` rung 11.

## Finding 10 - rung 47 branch 2 omits the manual-recovery lockout

**Classification: established fact.**
**Severity: low to medium.**

Branch 1 carries `XIO(_L25_HoistAutoMode_Resumed)`, which suppresses the infeed promise while the hoist
is recovering from manual intervention. Branch 2 does not. With the plates selected, a manual-to-auto
recovery that takes the hoist above 1700 will grant the infeed promise during the recovery.

---

## Ranked conclusion

Both reported faults trace to the same root cause: **the point at which the code decides the platform is
"almost up" is roughly 400 mm below the receiving position, and that point is used to release the infeed.**

- The 400 mm of crawl at 400 is the "not fast enough" complaint (finding 2).
- Releasing a panel 400 mm early lets that panel's own alignment cycle stall the hoist through
  `_L25_PromiceUP`, which is the "not close enough up" complaint (finding 1).
- The same 400 mm of early release, with the panel-on-plates interlock bypassed and no counter anywhere,
  is what admits the third panel (findings 2 and 3).
- Zero settle time (finding 4) and the OR'd plate sensors (finding 5) determine how badly the panels land
  once the plates do let go.

Findings 6 to 10 are real defects that are not the primary cause. Finding 8 is the closest to it:
a pressure roll speed that cannot be tuned from the HMI limits how well arrival can ever be coordinated
with the plate drop, even after findings 1 to 5 are fixed.

## What was done wrong, in change-log terms

| Log entry | Verdict |
|---|---|
| 21 May, L21 R10, delay 0.3 s to 0 s | Wrong in effect. Removed the only settle time before the plates release. See finding 4. |
| 11 Jun, L25 R37, single slow-down at 1700 "100 mm from the top" | The distance is wrong. The routine's own constants put the top at 2100 to 2150. See finding 2. |
| 11 Jun, L25 R32/R37, bypass the delay when plates and aligners are done | Sound intent, but it relies on `_L21_Sensors_Sylinders_Out`, which is an OR of the two plate sensors. See finding 5. |
| 11 Jun, L25 R47, restart when the platform is almost up | This is the primary regression. Releases the infeed 400 mm early, uses a non-latched bit, and omits the manual-recovery lockout. See findings 1, 2, 10. |
| 11 Jun, L21 R11, add `_L25_Platform_RdyInfeed` | Correct and necessary. It is the only reason the plates do not drop mid-climb. Keep it. |
| 18 Jun, L20 R26/R27, footpedal off-delay | Unrelated to these faults and looks sound. `TMR_Footpedal_StopOffDelay.PRE` is 1500 as documented. |

The 11 Jun package was a throughput optimisation. Each individual edit is defensible; the combination
removed every timing margin the pack change had, at the same time as it moved the infeed release earlier
than intended.

---

## Verification to do at the machine before changing anything

1. Put the platform empty and fully up in auto, and record `StackerHoistActualPosition`. This settles
   finding 2. Everything in the fixes document keys off that number.
2. Trend `_L25_StackerHoistUp_Down.IN_RunFw`, `_L25_PromiceUP`, `Out_PromiceFwd_SideAlignment`,
   `I_PC_Stkr_GapFreeLeftSide`, `I_PC_Stkr_GapFreeRightSide` and `StackerHoistActualPosition` through one
   pack change. If `IN_RunFw` drops between 1700 and the top, finding 1 is confirmed at the machine.
3. Trend `I_PR_Stkr_DroppingPlateLeftOn` and `I_PR_Stkr_DroppingPlateRightOn` together. If they do not
   make within a few tens of milliseconds of each other, finding 5 is already biting.
4. Read `HMI_972_1110CP_Setting_OutfeedSpeed` and compare with the 70 in `L20` rung 33 before touching
   finding 8.

Proposed rung-level corrections are in `PROPOSED_FIXES_Rung_Level.md`. None of them have been applied to
the exports in this repository.
