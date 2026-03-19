# changes_kokeshi

This note summarizes the four changes currently reflected in the codebase, with implementation points.

## 1) Removed Game Freak intro at boot

- Boot entry is set to skip the normal boot copyright/intro route and go straight to title init:
  - `ld_script_modern.ld`: `gInitialMainCB2 = CB2_InitBootSkipToTitleScreen;`
- The skip callback loads save/options context, initializes heap/audio option, then jumps directly to title:
  - `src/intro.c`: `CB2_InitBootSkipToTitleScreen()`
  - Final step there is `SetMainCallback2(CB2_InitTitleScreen);`

Result: startup bypasses the default boot intro sequence and reaches title screen faster.

## 2) Overworld speed-up + in-game options

- New save options were added in `SaveBlock2`:
  - `include/global.h`: `optionsFastForward`, `optionsFastForwardEnabled`
- Option values are defined here:
  - `include/constants/global.h`: `OPTIONS_FAST_FORWARD_1_25X`, `OPTIONS_FAST_FORWARD_1_5X`, `OPTIONS_FAST_FORWARD_2X`
- Option menu UI and persistence were added:
  - `src/option_menu.c`: menu items `FAST FORWARD` and `FAST SPEED`
  - `src/option_menu.c`: reads/writes `gSaveBlock2Ptr->optionsFastForwardEnabled` and `optionsFastForward`
- Default values on new save:
  - `src/new_game.c`: defaults to enabled + `1.25x`
- Runtime speed logic:
  - `src/main.c`: `GetMainCallbackRunsPerFrame()`
  - Applies only when in overworld callbacks (`CB2_Overworld` / `CB2_OverworldBasic`), not in battle/menus.
  - Uses accumulator stepping to run the main callback loop multiple times per rendered frame.

Result: configurable overworld fast-forward that is safe-scoped away from battle/menu state machines.

## 3) Catch probability based on ball + enemy state

- Ball-specific modifiers are centralized in:
  - `src/battle_script_commands.c`: `ComputeBallData(...)`
  - Handles ball multipliers/conditions (Great/Ultra/Quick/Timer/Dusk/Net/Level/Love/Heavy/etc.), including edge cases like guaranteed capture.
- Final capture odds combine:
  - Wild catch rate
  - Current HP factor
  - Ball multiplier / divisor
  - Status condition bonuses
  - Additional config-based bonuses/maluses (badge and level related)
  - Implemented in `src/battle_script_commands.c`: `ComputeCaptureOdds(...)`
- Ball shake and critical capture are then computed from those odds:
  - `CriticalCapture(...)`, `ComputeBallShakeOdds(...)`, `Cmd_handleballthrow(...)`

Result: catch chance is dynamically affected by both chosen ball and target battle state (HP/status/etc.).

## 4) Overworld auto-heal for fainted Pokemon (this session)

- Config knobs:
  - `include/config/overworld.h`
  - `OW_FAINTED_AUTOHEAL = TRUE`
  - `OW_FAINTED_AUTOHEAL_TICK_FRAMES = 600` (10s at 60 FPS)
  - `OW_FAINTED_AUTOHEAL_MAX_PERCENT = 25`
  - `OW_FAINTED_AUTOHEAL_TICK_PERCENT = 5`
- Runtime logic:
  - `src/overworld.c`: `TryAutoHealFaintedPartyMons()`
  - Called from overworld loop (`OverworldBasic`)
  - Every tick, valid non-egg party mons recover `+5%` HP, capped at `25%` max HP.
  - Fainted mons regain consciousness on the first heal tick.

Result: passive, slow recovery in overworld only, with configurable timing and cap.
