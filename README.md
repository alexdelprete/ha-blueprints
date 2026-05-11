[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Falexdelprete%2Fha-blueprints%2Fmain%2Fha-blueprint-linked-entities.yaml)

**Linked Entities v2.0** 🔛

This blueprint allows you to easily create/maintain an automation that links the state of multiple entities:
  - turn ANY linked entity ON, it will turn ON ALL linked entities.
  - turn ANY linked entity OFF, it will turn OFF ALL linked entities.
  - set the brightness of any light entity, it will set the same brightness on ALL linked light entities.
  - set the color temperature of any light entity, it will set the same color temperature on ALL linked light entities.
  - set the speed (percentage) of any fan entity, it will set the speed on ALL linked fan entities.

**NOTE**: The entity selector is restricted to `light`, `switch`, `fan`, and `input_boolean`. Mixed-domain instances are supported: brightness/color/speed only propagate to entities of the matching domain.

My main use-case was for multiple light switches in the house controlling the same light, but I also use it for other things:
  - at dawn, when I turn on the external lights (a shelly switch), I also link the pool light mqtt switch.
  - when I want to open the external gate, I can use several linked switches in multiple rooms of the house
  - when I want to manually turn on/off the irrigation system, I use two switches (internal and external) to activate my RainMachine cycle

I'm sure you'll find many more use-cases. :slight_smile:

**Tuning for your network:**

The blueprint exposes three knobs. The defaults work well for most setups; adjust only if you hit the symptoms below.

| Input | Default | What it does |
|---|---|---|
| `Debounce` (`debounce_milliseconds`) | 100 ms | A state or attribute change must remain stable for this long before it propagates. Suppresses spurious echoes from flaky devices and absorbs micro-steps from brightness sliders. |
| `Delay` (`delay_milliseconds`) | 200 ms | Pause after each propagated service call. Throttles back-to-back triggers when `mode: queued` is processing a burst. |
| `Light Transition` (`transition_seconds`) | 0 (off) | Smooth fade duration for brightness/color updates on linked lights. |

**Symptom → fix:**

| Symptom | Try |
|---|---|
| Lights occasionally turn on and immediately back off on their own (or vice versa) | **Raise `Debounce` to 500–1000 ms**. This is the Matter/Tapo/slow-Zigbee echo bug from [#3](https://github.com/alexdelprete/ha-blueprints/issues/3). |
| Linked entities feel a bit sluggish to react | **Lower `Debounce` to 0** and `Delay` to 50–100 ms. Only do this on fast, reliable local networks (Ethernet ESPHome, Z2M on a wired host). |
| Home Assistant log shows service-call errors or "too many requests" warnings when toggling | **Raise `Delay` to 300–500 ms**. Some integrations (cloud-backed Tuya, large Z-Wave meshes) don't like rapid back-to-back calls. |
| Dragging a brightness slider on one light produces visible "stepping" on the others | This is `Debounce` working as intended — only values that hold for 100 ms+ propagate. Raise it to 200–300 ms if you want even fewer intermediate steps, or lower to 0 if you want every step mirrored. |
| Linked lights jump abruptly when brightness/color changes | **Set `Light Transition` to 0.5–2 seconds** for smooth fades. |
| You linked a Zigbee bulb to a smart switch and the bulb misfires when the switch is also under load | jnrcorp's advice on [#3](https://github.com/alexdelprete/ha-blueprints/issues/3): consider whether you actually want the bulb to follow the switch's reported state, or if the switch should be the only controller (in which case you don't need this blueprint). If you do want sync, raise `Debounce` to 1000+ ms. |

**Profiles by setup:**

- **Fast & local** (ESPHome over Ethernet, Z2M on a wired host, Shellies on a clean 2.4 GHz): `Debounce: 0`, `Delay: 100`, `Transition: 0`.
- **Default home** (mixed Zigbee + WiFi, Hue bridge, Sonoff): leave at defaults (`100 / 200 / 0`).
- **Flaky network or cloud-backed devices** (Tapo via Matter, Tuya cloud, large Zigbee mesh): `Debounce: 500`, `Delay: 300`, `Transition: 0`.
- **Smooth lighting scenes** (RGB bulbs syncing color across a room): defaults + `Transition: 1` for a 1-second fade.

If you still hit weird behavior after tuning, open an issue with: your devices, integration types, the symptom, and your current input values.

**Migrating from v1.3 to v2.0:**

HA doesn't auto-pull blueprint updates. Re-import the blueprint manually:

1. **Settings → Automations & Scenes → Blueprints → Linked Entities → ⋮ → Re-import blueprint** (or click the My-HA badge at the top of this README again).
2. HA fetches the new YAML and overwrites the local copy. All existing automation instances pointing to this blueprint now use v2.0 logic.

Then for each existing automation built from this blueprint, open it in **Settings → Automations & Scenes** and:

1. **Check the `Delay` field** — the input was renamed `delay_miliseconds` → `delay_milliseconds` (typo fix), so HA can't find the old stored value. If you had customized it (say, 500 ms), it silently reverts to the new default of 200 ms. Re-enter your previous value if you cared about it.
2. **Review the linked entities list** — the selector now only offers `light`, `switch`, `fan`, `input_boolean`. Stored values for other domains (`cover`, `climate`, `media_player`, etc.) generally still load, but the new branches only operate on the supported domains. Remove anything that isn't one of those four.
3. **New `Debounce` field** appears with default 100 ms — no action needed unless you have a flaky network. See the *Tuning* section above for recommended values.
4. **New `Light Transition` field** appears with default 0 (off) — no action needed unless you want smooth fades on brightness/color changes.
5. **Click Save** on each instance — this forces HA to re-validate against the v2.0 schema and persist the values under the new keys.

After upgrading, verify:

- Toggle one entity in a linked group and confirm the rest follow.
- For light groups: turn off, then back on at a specific brightness — linked lights should match (this is the [#3](https://github.com/alexdelprete/ha-blueprints/issues/3) fix).
- For color bulbs: change color on one, confirm the others mirror (this is the [#5](https://github.com/alexdelprete/ha-blueprints/issues/5) fix).
- Watch the HA log for the first day. If anything looks off, raise `Debounce` first.

What does **not** break: the blueprint stays at the same path/URL, so all existing instance references survive. The context-id self-trigger guard is semantically unchanged. `homeassistant.turn_on/off` is still used in the on/off branches, so `input_boolean` / `switch` / `fan` domains keep working exactly as before.

**FAQ:**

**Q: Can I link entities of different types (e.g. a light, a switch, and an input_boolean)?**
Yes. On/off propagates across all of them via `homeassistant.turn_on/off`. Brightness and color only propagate to lights; fan speed only propagates to fans. Mixed-domain instances no longer error since v2.0.

**Q: My entity isn't showing in the selector — why?**
The selector is restricted to `light`, `switch`, `fan`, and `input_boolean`. Other domains (`cover`, `climate`, `media_player`, `binary_sensor`, etc.) aren't supported. If you have a use case for one of those, open an issue describing what you'd expect the blueprint to do for it.

**Q: Can the same entity appear in multiple linked-entity instances?**
Yes, but think it through. If instance A links `[X, Y]` and instance B links `[Y, Z]`, then toggling X propagates to Y (via A), and Y's change propagates to Z (via B). That's usually fine — the context-id guard prevents loops back to A. But if both instances include the *same pair*, you'll get redundant service calls. Stick to one instance per logical group when you can.

**Q: Will this cause feedback loops?**
Two layers of protection:
1. The context-id guard ignores state changes whose context matches the blueprint's last firing context — this catches the common case where the blueprint's own service calls produce echoes.
2. The `Debounce` window (default 100 ms) filters spurious state echoes from slow/flaky networks before they reach the action.
If you still see misfires on Matter/Tapo/slow-Zigbee setups, raise `Debounce` to 500–2000 ms. See the *Tuning* section above.

**Q: When I drag a brightness slider, the linked lights jump in steps instead of following smoothly. Is this broken?**
No — that's `Debounce` working as intended. Only brightness values that hold for the debounce window (default 100 ms) propagate, so micro-steps during a slider drag are absorbed. If you want every step mirrored, lower `Debounce` to 0. If you want even fewer steps, raise it to 200–300 ms. For smooth fades to the final value, set `Light Transition` to 0.5–2 seconds.

**Q: Can I turn the light on at a specific brightness depending on which switch toggled it (e.g., dim at night entrance switch, bright at day kitchen switch)?**
Not directly with this blueprint — it propagates whatever brightness the source has, with no per-source logic. Workaround: create a small per-source automation that sets the desired brightness on the *first* light in the linked group when that specific switch fires, then this blueprint propagates that brightness to the rest. See [#7](https://github.com/alexdelprete/ha-blueprints/issues/7).

**Q: I have a smart switch (wired to power) controlling a smart bulb in "decoupled" mode. Will linking them work reliably?**
Mostly yes, with caveats. The combo of a wall switch + a bulb in decoupled mode is fundamentally a sync problem — both devices report their own state independently, and neither is authoritative. The blueprint will keep them aligned 99% of the time, but spurious state echoes on slow networks (especially Matter/Tapo dimmers) can occasionally cause the bulb to flicker off-on. Raise `Debounce` to 500–1000 ms to absorb these. If you only ever want the switch to control the bulb (one-way sync), you may not need this blueprint at all — see [@jnrcorp's note in #3](https://github.com/alexdelprete/ha-blueprints/issues/3#issuecomment-4413889870).

**Q: What happens during a Home Assistant restart?**
Each linked entity transitions through `unknown → on/off` as it restores. The blueprint's triggers accept `unknown` and `unavailable` as valid starting states (since v2.0, [#6](https://github.com/alexdelprete/ha-blueprints/issues/6)), so the blueprint fires once per entity as they come back online. Service calls on already-correct state are idempotent, so the result is consistent sync rather than flapping. Worst case: a couple of seconds of harmless redundant `homeassistant.turn_on/off` calls in the log right after startup.

**Q: How do I know I'm running v2.0?**
Open the blueprint in **Settings → Automations & Scenes → Blueprints → Linked Entities** and check the description — it should say "Linked Entities v2.0" near the top. The full changelog is below this FAQ.

**CHANGELOG:**
  - **2.0**: (2026-05-11)
    - **Fix ([#6](https://github.com/alexdelprete/ha-blueprints/issues/6))**: linked entities now sync correctly when an entity recovers from `unavailable`/`unknown` to `on` or `off` — covers device-restart and integration-reload cases. Thanks @nsitt for the report.
    - **Fix ([#3](https://github.com/alexdelprete/ha-blueprints/issues/3))**: brightness and color are now correctly propagated when a linked light turns on. Previously the `turn_on` branch dropped attribute triggers due to `mode: single` racing with state-change fan-out, so linked lights came up at their previous brightness/color. Thanks @jrosspaperless for the report and @jnrcorp / @richard-berg for the diagnosis and prototypes.
    - **Fix**: full color support — `hs_color`, `rgb_color`, `xy_color` are now propagated alongside `color_temp_kelvin` (priority: hs > rgb > xy > color_temp_kelvin), so RGB/Hue/Lifx-style bulbs sync color, not just whites. Inspired by @richard-berg's fork.
    - **Fix**: color flicker on multi-attribute color changes is eliminated — when a light updates several color attributes at once (e.g. hs + rgb + xy together), all color triggers route through a single branch with priority-resolved color, so linked lights receive identical updates instead of competing partial writes.
    - **Fix**: echo loop mitigation for slow/flaky networks — added a configurable per-trigger debounce. Default 100 ms swallows most spurious state echoes (e.g., Matter/Tapo dimmers reporting a momentary off after on). Users can raise it up to 5000 ms if needed. Thanks @AbbieDoobie for the report.
    - **Fix**: domain-aware service calls — `light.turn_on` and `fan.set_percentage` only run against entities of the matching domain. Mixed-domain instances no longer error.
    - **Fix**: switched from deprecated `color_temp` (mireds) to `color_temp_kelvin` (HA 2024.3+).
    - **Fix**: fan speed branch now ignores non-fan sources (was firing on any entity with a `percentage` attribute).
    - **Breaking**: input renamed `delay_miliseconds` → `delay_milliseconds`. Existing instances will revert to the default (200 ms) until re-saved.
    - **Breaking**: entity selector restricted to `light`, `switch`, `fan`, `input_boolean` (thanks @Aaroneisele55 for [PR #9](https://github.com/alexdelprete/ha-blueprints/pull/9)). Remove any other entity types from existing instances.
    - **Improvement**: `mode: queued` (max 10) for clean state fan-out; trailing per-branch delay retained for throttling.
    - **Improvement**: optional `transition_seconds` input for smooth brightness/color changes.
    - **Improvement**: modern `target:` service-call syntax.
  - **1.3**: (2024-07-10 - thanks @jsenecal for PR #2)
    - Ignore changes to entities triggered by this automation
    - Do not change the state of the entity triggering the automation
  - **1.2**: (2024-01-28 - thanks @phrak / @TimU for PR #1)
    - Changed the action to a "Choose" building block to support fan, light and future attributes
    - Added support for linked fan speeds
    - Added support for linked light brightness and light color temperature
  - **1.1**: (2024-01-09)
    - Optimized service call leveraging trigger.id
    - Introduced max_exceeded to avoid warnings in log due to possible self-triggering
    - Introduced a small delay after the service call to throttle the automation
  - **1.0**: (2023-12-29)
    - First official release

**If you like to show your support please hit click here:**
[![contributions - welcome](https://img.shields.io/badge/contributions-welcome-blue)](https://www.paypal.com/donate/?hosted_button_id=8V9YE6S5E869G "PayPal Donation")

**Source:**
[![alexdelprete - ha-blueprints](https://img.shields.io/static/v1?label=alexdelprete&message=ha-blueprints&color=blue&logo=github)](https://github.com/alexdelprete/ha-blueprints/blob/main/ha-blueprint-linked-entities.yaml "Go to GitHub repo") [![stars - ha-blueprints](https://img.shields.io/github/stars/alexdelprete/ha-blueprints?style=social)](https://github.com/alexdelprete/ha-blueprints) [![forks - ha-blueprints](https://img.shields.io/github/forks/alexdelprete/ha-blueprints?style=social)](https://github.com/alexdelprete/ha-blueprints)

**Support:**
[![HA Community - Topic](https://img.shields.io/static/v1?label=HA+Community&message=Topic&color=2ea44f&logo=home-assistant)](https://community.home-assistant.io/t/linked-entities-keep-mutlple-entities-binary-state-in-sync-lights-switches-etc/662836?u=alexdelprete)
