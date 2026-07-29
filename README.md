# GeckoLib Reborn Plugin — custom animation controllers

A patched build of [GeckoLib Reborn](https://github.com/CBJaxxx/GeckoLib-Reborn) (v7.3.0, MIT)
for MCreator 2026.2, adding user-defined GeckoLib animation controllers to animated entities.

| Generator | GeckoLib | Asset layout |
|-----------|----------|--------------|
| **NeoForge 1.21.1** | **4.9.2** | `geo/` + `animations/` |
| **NeoForge 26.1.2** | **5.5.2** | `geckolib/models/` + `geckolib/animations/` |

This **replaces** the upstream plugin zip in `~/.mcreator/plugins/` — it is not a
side-by-side overlay. Adding controllers means changing the mod element's data class
and its editor GUI, both of which are compiled Java inside the plugin, so a separate
higher-weight plugin could not do it. The untouched original is kept in `upstream/`.

## What was added

**Animations page of the animated entity editor** — a *Custom Animation Controllers*
list where each row is a controller name, an *Additive* checkbox, and an optional
*Own transition* override. Rows can be reordered, because order matters (see below).

*Own transition* replaces the entity's global "Animation transition ticks" for that
controller. Left unchecked the controller inherits the global value, which is also
what rows saved before this option existed do. Additive layers usually want `0`:
otherwise GeckoLib ramps into the animation's first keyframe from the stopped state,
which shows up on scale channels as a brief squash or stretch before the animation
proper begins.

**The three animation procedure blocks** now take a controller name:

| Block | Now reads |
|---|---|
| play | If entity element %1 is the %3 then play the animation %2 **on controller %4** |
| stop | If entity element %1 is the %2 then stop the animation **on controller %3** |
| get | If entity element %1 is the %2 then get current animation **on controller %3** |

A blank or unknown controller name falls back to the built-in `procedure` controller,
so the blocks behave exactly as before when the field is left at its default.

Both generators are patched.

## What "additive" does

On the **26.1.2 generator (GeckoLib 5)** the checkbox emits
`AnimationController.additiveAnimations()`, which is real additive blending: the
controller's animation is **added** on top of what earlier controllers produced
instead of assigning over it.

That makes the animation's keyframes **offsets from the base pose**, and a keyframe
of `0` means "leave the base pose alone". So an attack authored from the model's
rest pose plays correctly over a held fight stance and settles back into it — no
need to bake the stance into every attack.

```
stance holds shoulder at 45°
jab adds another 45°          ->  90° during the jab
jab returns to 0°             ->  back to the 45° stance
```

Controllers are registered in this order:

```
non-additive controllers
movement                      (built-in: idle/walk/sprint/...)
attacking                     (built-in, if an attack animation is set)
procedure                     (built-in, legacy animation blocks)
additive controllers          <- added on top of everything above
```

Additive controllers go last on purpose: a non-additive controller *assigns*, so one
registered after them would discard the accumulated offset.

### On the 1.21.1 generator this degrades

GeckoLib **4.9.2 has no `additiveAnimations()`** — additive blending was added in
GeckoLib 5. There the flag only controls layering order, so a later controller
overrides the bones its animation keyframes. In that mode **an additive animation
must only keyframe the bones it needs**, or it fully replaces the base pose instead
of layering over it. The codegen test asserts that this generator never emits
`additiveAnimations()`.

Controller names must be plain identifiers (`[a-zA-Z_][a-zA-Z0-9_]*`) because they
become part of generated Java identifiers. `movement`, `attacking` and `procedure`
are reserved — they are always registered automatically.

## Migrating existing procedures

Animation blocks saved before this change have no controller input. MCreator will
flag them as missing an input; drop a text block in (or leave the default
`procedure`) to restore the previous behaviour. Existing animated entities need no
changes — they deserialize with an empty controller list.

## Requirements

- **MCreator 2026.2** (supported version id `2026002`)
- **Java plugins must be enabled** in MCreator: *Preferences → Plugins → Enable Java
  plugins*, then restart. This plugin registers its mod element types from Java, so with
  the setting off it simply does not load — that is off by default on a fresh install.
- Enable the **GeckoLib** API in workspace settings (*Workspace settings → External APIs*)
- Java **25** only for *building* the plugin (the MCreator-bundled JBR is used)

## Building

```bash
./build-plugin.sh
```

Produces `build/libs/GeckoLib_Reborn_Plugin.zip`. Copy it over
`~/.mcreator/plugins/GeckoLib_Reborn_Plugin.zip` and restart MCreator.

The build deliberately does not use Gradle (upstream's `build.gradle` still works but
wants a ~150 MB Gradle 9.6 download). It compiles against `libs/mcreator-api.jar`,
a repackaged copy of the installed MCreator's classes; regenerate it after an
MCreator update with `./regen-api-jar.sh`.

### Checks

```bash
./tools/validate-templates.sh        # FreeMarker syntax of all 123 templates
./tools/test-controller-codegen.sh   # filtering, ordering and template output
```

The codegen test asserts the registration order above, that reserved/duplicate/
malformed names are dropped, and that entities with no controller list still work.

## Note on upstream sources

Upstream's git tree (branch `CBJaxx`) is behind its own v7.3.0 release: it does not
contain the `neoforge-26.1.2` generator at all, and some `neoforge-1.21.1` templates
are stale. The Java sources match the release, so `src/main/resources` here was taken
from the published release zip instead. A verified-unmodified build of that baseline
reproduced the release artifact exactly (identical class sizes, zero resource diffs)
before any changes were applied.

When updating to a newer upstream release, take resources from the release zip again
rather than from git.

Localized `blockly.block.*` strings in `lang/texts_*.properties` had the new
placeholder appended in English (`(controller %N)`) rather than machine-translated —
a missing placeholder is a hard Blockly error, a partly-English label is cosmetic.
