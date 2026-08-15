# Personal changes

A personal fork of TM-CE. This branch is `master` plus everything below.

## Combo Training (the asm event)

Both of these live in
`ASM/training-mode/Custom Events/Custom Event Code - Rewrite.asm`.

### DK cargo throw DI

DK's cargo throws use `ThrownFF`..`ThrownFLw` (`0x10F`..`0x112`), not the normal
`ThrownF`..`ThrownLwWomen` (`0xEF`..`0xF3`). Two places hardcoded the normal
range, so cargo throw DI worked in Lab but not in Combo Training:

- `ComboTrainingCheckForThrowAngle` returned `-1` for cargo throws, so Survival DI
  and Combo DI fell back to the stored knockback angle at `0x1848`, which is never
  populated during a throw.
- The "always DI the direction of the angle" gate in ComboDI skipped them too.

Both now accept the cargo throw range as well.

Also fixes a typo just below the second gate: the `Above90` check compared `r3`,
still holding the action state, against `269` instead of `r24`, the backed up
angle. Normal throw states are 239–243 so it always took the same branch, and no
throw has an angle >= 269, making the correction a no-op for existing throws and
necessary for cargo throws, whose states are 271–274.

Sent upstream as [#350].

### Double Jump escape option

A fourth `Post Hitstun Action` that tap jumps out with no intangibility, so the
CPU stays punishable. The existing Invincible option also jumps out, but bundles
30 frames of intangibility.

Its reset timer only starts once the jump is actually out, and it hands off to
the spotdodge path once grounded, so landing doesn't cut the punish window short.

Worth knowing for anyone editing this: all three `PostHitstunAction` dispatches
fall through to their *second* handler on an unmatched value, so a new option
needs an explicit branch at each one. Miss a site and it silently behaves like
Invincible rather than erroring.

Sent upstream as [#351].

## Lab (main training mode)

### Combo Training menu

`Main Menu -> Combo Training`:

| Option | |
|---|---|
| Auto Reset | return to the saved position once the CPU recovers from a combo |
| Reset Delay | frames to wait before resetting |
| Escape Option | Custom / Airdodge / Double Jump / Attack |
| Percent Switch | swap the whole CPU and tech setup at this percent, 0 disables |
| Knockdown Move | which of your moves to read |
| Stale Level | how stale to assume that move is |
| Set Switch to Knockdown % | set the switch to where that move starts knocking down |
| Save as Low Percent | snapshot every CPU and tech option as the low set |
| Save as High Percent | snapshot every CPU and tech option as the high set |
| Randomize Position | Off / On Stage / On Platform / Anywhere |
| Randomize Facing | randomize which way you face |
| Randomize Percent | give the CPU a random percent in the range below |
| Percent Low / High | bounds of that range |
| Goal | Off / Hit Count / Kill |
| Goal Hit Count | hits needed to clear a setup |
| Goal Streak | times in a row a setup must be cleared |
| Show Success Rate | print how often the goal is being cleared |
| Show Move Data | print each move's real hitbox values |
| Survival DI Moves | submenu, per-move survival DI checklist |
| DK Options | submenu, Giant Punch charge randomization |

**Auto Reset** starts counting as soon as the CPU can act again, so Reset Delay
means frames after the CPU is free to act. Being hit again cancels a pending
reset. Entering recovery counts as combo over, and a 120 frame settle limit
backstops anything that will not resolve.

**Escape Option** writes into the real CPU counter options, so everything stays
editable in `CPU Options` afterwards. `Attack` picks a move that suits the CPU's
character, using the per-character table from the Combo Training asm event —
Lab's counter action otherwise applies one global move to everyone:

| | grounded | aerial |
|---|---|---|
| Fox / Falco | Down B | Nair |
| DK / Bowser / Samus / G&W | Up B | Up B |
| Jigglypuff | Down B (rest) | Dair |
| Zelda | Down Smash | Bair |
| Marth | Jab | Fair |
| Falcon / Pikachu / Ganon | Jab | Uair |
| Ness / Mewtwo | Down Tilt | Nair |
| Kirby | Jab | Bair |
| Ice Climbers | Jab | Dair |

**Randomized setups.** Every piece is independent, so any combination works.
Position picks a spot by raycasting for ground, so it works on any stage; the
zone filter tells platforms from the stage by height above the main floor. The
CPU is always placed just in front of you, the same as pressing DPad down.
Facing and percent are separate toggles; turning around brings the CPU round to
the front with you rather than leaving it behind.

**Goals.** A setup can require a number of hits or a kill, and `Goal Streak` sets
how many times in a row it must be cleared. Until then the same setup repeats,
so a missed attempt means another go at it rather than a new one. With no goal
set every attempt counts as cleared. Once a setup is cleared the new one is saved
over the reset state, so plain resets return to it.

**DK Options** randomize Giant Punch charge on each new setup. `Charge Mode` is
either `Range`, picking anywhere between a lowest and highest number of windups,
or `None or Full`, which only ever gives 0 or a full punch — the two cases that
change what DK actually threatens. `Show Charge` puts the result on screen: full,
none, or `n/10`.

**Survival DI Moves** is a checklist, one row per move. Whatever you are
currently attacking with is looked up in it, and only the ticked moves make the
CPU DI for survival — everything else keeps whatever `Trajectory DI` is set to.
That way the CPU DIs your kill move the way a real opponent would while still
taking your combo starters normally, so a string ends in a realistic kill
attempt rather than the CPU DIing everything or nothing.

Survival DI here is the same perpendicular-toward-centre DI as the
`Toward Center` trajectory option. A ticked move also wins over `Slide Off`.

`Preset` fills the checklist in:

| preset | ticks |
|---|---|
| None | nothing |
| DK Kill Moves | Forward Air, Up B, Neutral B |
| All | every move |

Editing any move by hand drops `Preset` back to `Custom`. Moves are matched by
the attacker's action state through the same lookup the `Knockdown Move` list
uses, so specials resolve to neutral/side/up/down B from the stick direction
held when B was pressed.

Survival DI is the highest priority DI decision. It is resolved first, and
nothing after it reads the DI setting again, so a ticked move beats `Slide Off`
and the percent profiles. The check runs in `CPUOnHit`, on the frame of the hit,
so the attacker is still in the attacking state. Throws are not on the list and
are left alone. `Show When Used` prints which move triggered it, to confirm it
is firing.

**Show Move Data** prints a move's real hitbox values the first time you throw
it out:

```
Forward Air  d16.0 a361 g100 b20 w0
```

damage, angle, knockback growth, base knockback, set knockback. These come off
the live hitbox rather than a frame data site, so they are what this build
actually uses. Turning the option on forgets every move already seen, so
everything reprints rather than staying silent. Angle 361 is Melee's Sakurai
angle sentinel.

It reuses the recorder behind `Knockdown Move`, which walks the player's active
hitboxes every frame and maps them to a move through the action state. Only the
first active hitbox of a move is recorded, so multi-hit moves report their first
hit.

**Show Success Rate** prints how often you are clearing the `Goal`, as
`Success: cleared/attempts (pct)`. This is the overall record, separate from
`Goal Streak` — the streak is about repeating one setup until it is clean,
this is every attempt you have made. Only attempts where you actually landed a
hit count, so idling does not dilute it, and the tally only clears when you
leave training mode.

The point is playing the punish against something external, an AI agent on
another port being the case it was built for.

`phillip.sh` in the workspace launches a slippi-ai agent against this build. It
passes `--dolphin.iso` pointed at `TM-CE.iso`, so the agent plays inside the
training mode with everything above still live. Agents are read from a folder
rather than configured, so adding one is dropping the file in - run it with no
arguments to see what is there. The `basic-*` agents are human-imitation
trained; the rest are reinforcement trained per matchup and stronger, so picking
the agent is the difficulty dial.

**Set Switch to Knockdown %** solves melee's knockback formula for the move
picked in `Knockdown Move`, against the CPU's actual weight, and reports the
percent where knockback first reaches 80 — the point the victim is taken off its
feet. Computed from the real hitbox and the real matchup, so it works for any
move on any character rather than needing a table.

Melee only exposes a move's damage, knockback growth and base knockback while
its hitbox is live, so there is nothing to read from cold. Every move is instead
recorded the first time its hitboxes come out, whether or not it connects — so
throwing the move once in the air is enough, and normal practice fills the list
in on its own. Picking a move that has not been used yet says so.

Specials are character specific action states and cannot be identified by state
id, so they are matched by the direction held when B was pressed.

`Stale Level` is how many recent uses of the move to assume when working the
percent out. 0 is fresh; each step takes another slice off the damage using
melee's staling weights, so level 3 is the damage after three uses in a row.

Staling follows the 9 place queue: places 1 through 9 reduce damage by 0.09x
down to 0.01x, repeat appearances have their multipliers summed before being
applied, and the result is `base - (base * sum)`. Damage is kept fractional
throughout, since a 14 damage move at level 1 deals 12.74 and rounding it to 12
moves the answer.
That is usually the number you want, since a move that knocks down at 14% fresh
does not knock down there once it has been used a few times in a combo.

Set knockback moves report that they never knock down, since they ignore percent
by definition.

**Percent profiles** snapshot the `LabOptions_CPU` and `LabOptions_Tech` value
sets. Configure a low percent setup, save it, configure a kill percent one, save
that, set the switch percent.

The active set is only stamped over the options when the CPU crosses the switch
percent, not on every hit, so anything changed in the menu in between stays put.
Re-saving a profile takes effect at the next crossing.

The percent row and the move-CPU row are left out of both the snapshot and the
restore: the first is rewritten every frame from the CPU's actual damage and the
second is a menu action rather than a setting, so carrying either between
profiles only fights whatever else owns them.

### New Trajectory DI options

`Slight Random`, `Slight Towards` and `Down and Away`, from the Combo Training
asm event.

`Toward Center` takes whichever of the two perpendiculars to the knockback
carries the CPU back toward the middle of the stage rather than out toward a
blast zone. This is the "DI to center stage when you cannot slide off" answer,
and the same one for surviving a kill move.

`Slide Off` performs slideoff DI. ASDI down does the drop, so it forces the
C-stick down for that hit regardless of the ASDI option. The trajectory DI takes
whichever perpendicular to the knockback carries the CPU toward the edge — the
"DI away" that supplies the horizontal momentum to slide off with. It has to be a
real DI angle rather than a raw sideways stick, since DI shifts the knockback
trajectory rather than moving the CPU directly.

Sliding off edge cancels the knockdown and leaves the CPU actionable, which is
what makes it a reversal out of platform tech chases.

It only fires straight out of a tech roll or a getup roll from a missed tech,
which is when it comes up in play; only when the CPU is actually near the edge,
since rolling inward does not set it up; and only when the hit does not send the
CPU into tumble, because then there is no knockdown to edge cancel and it is
simply launched away. Works off the stage edge as well as platforms.
`Slide Off Else` names the DI to use the rest of the time.

The ground under the CPU is found by raycast rather than from its own ground
index, which is cleared as soon as knockback lifts it — that is, on exactly the
hits this cares about.

### New Smash DI direction: Toward Ground

Aims the CPU at ground it could actually reach: SDIs straight down if there is something directly below, or
diagonally toward the shorter drop if the ground is off to one side.

Range is `6 units x N`, where `N` is the lower of Smash DI Amount and the move's
hitlag. SDI only happens during hitlag, so the move caps how many inputs are
actually available however many the option asks for — a 4 frame hitlag move
cannot be SDIed 7 times.

Options that depend on another option — the two `Else` rows, the reset delay,
the percent range, the goal settings, the DK charge limits — are greyed out when
that option is not selected, so it is clear which ones are actually in play. This
updates live, from `Event_Update` rather than `Event_Think`, because the latter
does not run while the game is paused with the menu open.

`Toward Ground Else` sets the direction to use when nothing is in range,
offering every direction except Toward Ground itself. It resolves before the
main switch, so the fallback runs through the same code as picking that
direction outright.

## Banner

`opening.bnr` carries the name, maker and description shown alongside the game.
Those now read TM-CE v1.4 20DK and UnclePunch, Aitch, SNEAKY_URKEL.

The credits inside the banner picture are pixels in a 96x32 texture rather than
text, so they were redrawn: the two line "UnclePunch and Aitch" is now three
lines reading UNCLEPUNCH, AITCH, SNEAKY_URKEL.

The band is only 48x16 with the border and the MODE lettering around it, which
leaves room for a 3x4 pixel font - four rows rather than five, so there is a
blank row between lines to keep them apart. `banner.png` is the texture decoded,
if it wants redoing.

## Building

```
MSYSTEM=MSYS DEVKITPPC="C:/devkitPro/devkitPPC" ./build.sh path/to/vanilla-melee.iso
```

`MSYSTEM=MSYS` is needed under Git Bash, whose `uname` reports `MINGW64` and so
misses the check in `build.sh` that selects the bundled Windows binaries.

[#350]: https://github.com/AlexanderHarrison/TrainingMode-CommunityEdition/pull/350
[#351]: https://github.com/AlexanderHarrison/TrainingMode-CommunityEdition/pull/351
