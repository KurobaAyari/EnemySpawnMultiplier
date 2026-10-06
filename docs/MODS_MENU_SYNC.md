# F8 / MODS shared configuration (v23)

`config_service.lua` owns the committed profile independently of either UI.
It initializes from the shipped patch and restores `EnemySpawnMultiplier.cfg`
before the first gameplay update. `panel.lua` keeps a separate pending editor;
`mod_options.lua` bridges the public ModOptionsMenu API. Neither UI owns the
gameplay implementation or accesses game memory through this bridge.

## Submission and synchronization

- F8 Apply validates its full profile through the service and `patch.configure`.
- MODS `on_change` callbacks only stage changes. The next ESM update commits the
  whole APPLY batch once, before the gameplay check, and saves once.
- Applied changes update the other interface's same fields. Unrelated draft
  fields remain pending. `menu.set` replaces a same-field menu draft without
  dispatching callbacks; outgoing updates only set changed fields.
- Validation failure leaves the committed profile unchanged and returns the
  submitted menu fields to the committed values.
- Language changes commit only `language`, never staged gameplay values.
- Observer and menu errors are isolated. A failed optional menu or F8 window
  does not disable the other interface or the spawn patch.
- Startup uses ESM cfg when available. Otherwise it can import menu saved
  values once, unless F8 has already committed during deferred registration.

The menu is discovered during the first five update seconds, since addon load
order is unspecified. Missing or incompatible menus then stop polling. IDs are
stable `natsun.enemy_spawn_multiplier.<field>`; there are eight rows. The
bridge uses only `register_option`, `get`, `set` and `on_change`.

## Languages

`panel_model.lua` supplies both Chinese (`zh`, default) and English (`en`) UI
strings. F8 has an English / 中文 button; MODS has a Language choice. The shared
cfg saves the choice as an optional `language` field and keeps format 1, so
existing cfg files stay readable. Reset resets draft gameplay settings while
preserving the selected language and the committed profile.

Mod Options Menu v1.1+ (API `version >= 2`) receives text functions for labels,
descriptions and preset choices. Its public contract refreshes those functions
when the escape menu opens, so MODS text updates after closing and reopening
the escape menu. This bridge does not mutate menu internals to force a refresh.
The game's own MODS tab and Apply button follow the game's language setting.
Menu v1.0 receives static English strings and still synchronizes values.

## Dependencies and validation

- F8: Bingus Shared Loader v15+ / API 1.
- Native MODS: optional CowboyBingus Mod Options Menu; v1.1+ for translated
  labels, with Bingus Shared Loader v18+.
- HD2Runtime is not required or bundled.
- Public API source inspected and tested: CowboyBingus/ModOptionsMenu commit
  `fd0160807b6cc75c2eeb74dbf9793a3c5d30deb6` (v1.2 sources).
- Unit coverage includes full batched APPLY, both synchronization directions,
  false toggles, invalid input, restart precedence, late registration,
  missing/incompatible/full/failing menus, language-only updates and drafts.
- Upstream integration runs real registrations, option validation, Apply,
  translation refresh and `set` against the actual menu sources, with native
  integration inactive. It also attaches the real Win32 F8 panel.
- Loader integration proves restoration and menu changes still work when F8
  window creation fails.
- Captures in `build/panel-menus/panel-{zh,en}.png` show real Win32 rendering.

The package is built for the existing declared Steam build 25480438, EXE
1.8.46015.0. `runtime_verified` remains false: these checks do not establish
in-game native menu rendering, mission behavior or multiplayer stability.
