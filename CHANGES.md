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
| Save as Low Percent | snapshot every CPU and tech option as the low set |
| Save as High Percent | snapshot every CPU and tech option as the high set |
| Randomize Position | Off / On Stage / On Platform / Anywhere |
| Randomize Facing | randomize which way you face |
| Randomize Percent | give the CPU a random percent in the range below |
| Percent Low / High | bounds of that range |
| Goal | Off / Hit Count / Kill |
| Goal Hit Count | hits needed to clear a setup |
| Goal Streak | times in a row a setup must be cleared |
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
Facing and percent are separate toggles.

**Goals.** A setup can require a number of hits or a kill, and `Goal Streak` sets
how many times in a row it must be cleared. Until then the same setup repeats,
so a missed attempt means another go at it rather than a new one. With no goal
set every attempt counts as cleared. Once a setup is cleared the new one is saved
over the reset state, so plain resets return to it.

**DK Options** randomize Giant Punch charge on each new setup, between a lowest
and highest number of windups so partial charges can be practiced. `Show Charge`
puts the result on screen — full, none, or `n/10`.

**Percent profiles** snapshot the entire `LabOptions_CPU` and `LabOptions_Tech`
value sets. Configure a low percent setup, save it, configure a kill percent one,
save that, set the switch percent. The active set is chosen when the CPU is hit,
not every frame, so it never fights menu edits.

### New Trajectory DI options

`Slight Random`, `Slight Towards` and `Down and Away`, from the Combo Training
asm event.

### New Smash DI direction: Toward Ground

If there is ground the CPU could drop onto, SDI at it: straight down, or
diagonally toward the shorter drop if it is off to one side.

Range is `6 units x N`, where `N` is the lower of Smash DI Amount and the move's
hitlag. SDI only happens during hitlag, so the move caps how many inputs are
actually available however many the option asks for — a 4 frame hitlag move
cannot be SDIed 7 times.

`Toward Ground Else` sets the direction to use when nothing is in range,
offering every direction except Toward Ground itself. It resolves before the
main switch, so the fallback runs through the same code as picking that
direction outright.

## Building

```
MSYSTEM=MSYS DEVKITPPC="C:/devkitPro/devkitPPC" ./build.sh path/to/vanilla-melee.iso
```

`MSYSTEM=MSYS` is needed under Git Bash, whose `uname` reports `MINGW64` and so
misses the check in `build.sh` that selects the bundled Windows binaries.

[#350]: https://github.com/AlexanderHarrison/TrainingMode-CommunityEdition/pull/350
[#351]: https://github.com/AlexanderHarrison/TrainingMode-CommunityEdition/pull/351
