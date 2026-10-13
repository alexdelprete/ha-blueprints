[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Falexdelprete%2Fha-blueprints%2Fmain%2Fha-blueprint-linked-entities.yaml)

**Linked Entities v2.2.0** 🔛

This blueprint allows you to easily create/maintain an automation that links the state of multiple entities:
  - turn ANY linked entity ON, it will turn ON ALL linked entities.
  - turn ANY linked entity OFF, it will turn OFF ALL linked entities.
  - set the brightness of any light entity, it will set the same brightness on ALL linked light entities.
  - set the color of any light entity (color temperature / hs / rgb / xy), it will mirror the color on ALL linked light entities.
  - set the speed (percentage) of any fan entity, it will set the speed on ALL linked fan entities.
  - when an entity comes back from `unavailable`/`unknown`, it adopts the state (and brightness/color/speed) of the linked entities that stayed online.

**Requires Home Assistant 2025.4 or newer.**

My main use-case was for multiple light switches in the house controlling the same light, but I also use it for other things:
  - at dawn, when I turn on the external lights (a shelly switch), I also link the pool light mqtt switch.
  - when I want to open the external gate, I can use several linked switches in multiple rooms of the house
  - when I want to manually turn on/off the irrigation system, I use two switches (internal and external) to activate my RainMachine cycle

I'm sure you'll find many more use-cases. :slight_smile:

**Supported domains:**

| Domain | On/off sync | Attribute sync |
|---|---|---|
| `light` | yes | brightness, color (color temperature / hs / rgb / xy) |
| `fan` | yes | speed (percentage) |
| `switch` | yes | — |
| `input_boolean` | yes | — |
| `siren` | yes | — |
| `humidifier` | yes | — |
| `remote` | yes | — |

Mixed-domain instances are supported: on/off propagates to every linked entity via `homeassistant.turn_on/off`; brightness/color only reach lights, speed only reaches fans. Anything else (`cover`, `climate`, `media_player`, `number`, …) has a different state vocabulary or no `turn_on`/`turn_off` services and is deliberately not accepted. If you need to link such an entity, wrap it in a [template switch](https://www.home-assistant.io/integrations/switch.template/) helper and link the switch.

**Tuning for your network:**

The blueprint exposes four knobs. The defaults work well for most setups; adjust only if you hit the symptoms below.

| Input | Default | What it does |
|---|---|---|
| `Debounce (on/off)` (`debounce_milliseconds`) | 50 ms | An on/off change must remain stable for this long before it propagates. This is the only fixed cost in the reaction time of linked entities; everything else is the device round trip. Absorbs split/duplicate reports and spurious off-after-on glitches from flaky devices. |
| `Debounce (brightness/color/speed)` (`attribute_debounce_milliseconds`) | 100 ms | Wait before sampling and propagating a brightness, color or fan-speed change. During a slider drag the linked lights follow in samples spaced by this value plus `Delay` plus the device round trip, and always end on the final value. |
| `Delay` (`delay_milliseconds`) | 200 ms | Pause after each propagated service call. Adds no latency to a toggle; it throttles back-to-back calls and keeps the automation busy long enough for the state echoes of its own actions to be dropped. |
| `Light Transition` (`transition_seconds`) | 0 (off) | Smooth fade duration for brightness/color updates on linked lights. |

Recovery (see below) uses a fixed 1 s settle time and is not affected by these inputs.

**Symptom → fix:**

| Symptom | Try |
|---|---|
| Lights occasionally turn on and immediately back off on their own (or vice versa) | **Raise `Debounce (on/off)` to 500–1000 ms**. This is the Matter/Tapo/slow-Zigbee echo bug from [#3](https://github.com/alexdelprete/ha-blueprints/issues/3). |
| Linked entities feel a bit sluggish to react | **Lower `Debounce (on/off)` to 0**. Only do this on fast, reliable local devices (ESPHome, Tasmota, Shelly, Z2M on a wired host). What remains is the device round trip, typically 80–120 ms on WiFi, which no automation setting can shorten. |
| Home Assistant log shows service-call errors or "too many requests" warnings when toggling | **Raise `Delay` to 300–500 ms**. Some integrations (cloud-backed Tuya, large Z-Wave meshes) don't like rapid back-to-back calls. |
| Dragging a brightness slider on one light produces visible "stepping" on the others | Expected: linked lights follow in samples every `Debounce (brightness/color/speed)` + `Delay` + device round trip (~350 ms at defaults) and land on the final value. Raise the attribute debounce to 200–300 ms for fewer intermediate steps, or set `Light Transition` to 0.5 s so each step fades. |
| Linked lights jump abruptly when brightness/color changes | **Set `Light Transition` to 0.5–2 seconds** for smooth fades. |
| You linked a Zigbee bulb to a smart switch and the bulb misfires when the switch is also under load | jnrcorp's advice on [#3](https://github.com/alexdelprete/ha-blueprints/issues/3): consider whether you actually want the bulb to follow the switch's reported state, or if the switch should be the only controller (in which case you don't need this blueprint). If you do want sync, raise `Debounce (on/off)` to 1000+ ms. |

**Profiles by setup:**

Values are `Debounce (on/off)` / `Debounce (brightness/color/speed)` / `Delay` / `Transition`.

- **Fast & local** (ESPHome over Ethernet, Tasmota/Shelly on a clean 2.4 GHz, Z2M on a wired host): `0 / 100 / 200 / 0`.
- **Default home** (mixed Zigbee + WiFi, Hue bridge, Sonoff): leave at defaults (`50 / 100 / 200 / 0`).
- **Flaky network or cloud-backed devices** (Tapo via Matter, Tuya cloud, large Zigbee mesh): `500 / 200 / 300 / 0`.
- **Smooth lighting scenes** (RGB bulbs syncing color across a room): defaults + `Transition: 1` for a 1-second fade.

If you still hit weird behavior after tuning, open an issue with: your devices, integration types, the symptom, and your current input values.

**Recovery behavior (since v2.1.0):**

An entity that has just come back from `unavailable` or `unknown` is the *least* trustworthy member of its group: it was offline, so it missed whatever happened in the meantime. Since v2.1.0 the blueprint therefore never pushes a recovering entity's state to the others. Instead:

1. When an entity recovers, the blueprint waits a fixed 1 second (so the device can finish its restore sequence and its peers have a chance to come online too), then looks at the linked entities that are currently online.
2. If they **all agree** (all `on` or all `off`), the recovered entity is set to that state. If it is a light or a fan and the peers are on, it also adopts brightness/color or speed from a same-domain peer.
3. If the online peers disagree, or no peer is online, nothing happens.

Consequences worth knowing:

- After a **Home Assistant restart**, the first entity to come back finds no online peer and does nothing. Each later one adopts the state of those already online. No fan-out, no ping-pong, and boot order no longer decides the outcome of the whole group.
- A **single device reboot** (bulb power-cycled, integration reloaded) resyncs the device to the group, which is almost always what you want — a rebooted bulb typically comes back at its power-on default rather than the brightness the group was at.
- An **`input_boolean` in the group** never goes unavailable and restores its value across restarts, so it acts as a durable memory of the group's state. Useful if your physical devices tend to come back with random states.
- **Momentary / pulse relays** (gate openers) are inherently awkward to sync; recovery only acts under unanimity and never fans out, so this scheme is strictly safer for them than v2.0's.

**Upgrading:**

HA doesn't auto-pull blueprint updates. Re-import the blueprint manually:

1. **Settings → Automations & Scenes → Blueprints → Linked Entities → ⋮ → Re-import blueprint** (or click the My-HA badge at the top of this README again).
2. HA fetches the new YAML, overwrites the local copy and reloads every automation built on it. All existing instances immediately run the new logic — **no re-save needed**.

*From v2.1.0 to v2.2.0:* nothing to re-enter. The only visible change is how linked lights follow a slider drag (see the FAQ), and the Spook repair disappears.

*From v2.0.x to v2.1.0:* no input was renamed or removed, so nothing to re-enter. Requires HA 2025.4+. Behavioral changes you may notice: the recovery direction described above, and on/off sync reacting ~50 ms faster (the on/off debounce default dropped from 100 to 50 ms; if you had set `Debounce` yourself, your value still applies to on/off changes, and the new `Debounce (brightness/color/speed)` starts at 100 ms). If you have `siren`, `humidifier` or `remote` entities you previously had to leave out, you can now add them to your instances.

*From v1.3 to v2.x:* open an existing automation only if one of these applies to you:

- **You had customized the `Delay` field** to anything other than 200 ms. The input was renamed `delay_miliseconds` → `delay_milliseconds` (typo fix), so HA silently falls back to the new default of 200 ms for your existing instances. Re-open the automation and re-enter your value under the new `Delay` field.
- **You had non-supported entities in the linked list** (`cover`, `climate`, `media_player`, `binary_sensor`, etc.). They generally still load, but the branches only act on the supported domains listed above. Remove anything outside that list if it was relying on the old generic behavior.
- **You want to tune the `Debounce (on/off)`, `Debounce (brightness/color/speed)` or `Light Transition` inputs** for your specific network. Defaults work fine for most setups — see the *Tuning* section above for cases where they don't.

After upgrading, verify:

- Toggle one entity in a linked group and confirm the rest follow.
- For light groups: turn off, then back on at a specific brightness — linked lights should match (this is the [#3](https://github.com/alexdelprete/ha-blueprints/issues/3) fix).
- For color bulbs: change color on one, confirm the others mirror (this is the [#5](https://github.com/alexdelprete/ha-blueprints/issues/5) fix).
- For fan groups: turn a fan on at a given speed — linked fans should come up at that speed (v2.1.0 fix).
- Watch the HA log for the first day. If anything looks off, raise `Debounce (on/off)` first.

**FAQ:**

**Q: Can I link entities of different types (e.g. a light, a switch, and an input_boolean)?**
Yes. On/off propagates across all of them via `homeassistant.turn_on/off`. Brightness and color only propagate to lights; fan speed only propagates to fans. Mixed-domain instances no longer error since v2.0.

**Q: My entity isn't showing in the selector — why?**
The selector accepts `light`, `switch`, `fan`, `input_boolean`, `siren`, `humidifier` and `remote`. Other domains (`cover`, `climate`, `media_player`, `number`, `binary_sensor`, …) either don't have an `on`/`off` state or don't have `turn_on`/`turn_off` services, so the blueprint could not sync them correctly. Wrap the entity in a template switch helper and link the switch instead. If you think a domain is missing from the list, open an issue.

**Q: Can the same entity appear in multiple linked-entity instances?**
Yes, but think it through. If instance A links `[X, Y]` and instance B links `[Y, Z]`, then toggling X propagates to Y (via A), and Y's change propagates to Z (via B). That's usually fine — the context-id guard prevents loops back to A. But if both instances include the *same pair*, you'll get redundant service calls. Stick to one instance per logical group when you can.

**Q: Will this cause feedback loops?**
Three layers of protection:
1. **`mode: single` + trailing `Delay`** — while the blueprint is running its propagation, any echo triggers arriving during the trailing delay window get silently dropped at the queue gate. This is the primary defense, especially against auto-off / pulse switches (Shelly gate openers, momentary relays) where every echoed `homeassistant.turn_on` would otherwise re-pulse the relay.
2. **Context self-trigger guard** — state changes whose context is the automation's last run, a child of it, or shares the same originating context (the user action or device event that started the last run) are rejected at the condition. Since v2.1.0 this also covers echoes arriving *after* the trailing delay, which v2.0.x could let through when several triggers co-fired for the same event.
3. **`Debounce (on/off)` window** (default 50 ms) — filters spurious state echoes from slow/flaky networks before they reach the trigger.
Recovery (see above) never fans out to the other entities, so it cannot start a loop either. If you still see misfires on Matter/Tapo/slow-Zigbee setups, raise `Debounce (on/off)` to 500–2000 ms. See the *Tuning* section above.

**Q: When I drag a brightness slider, the linked lights jump in steps instead of following smoothly. Is this broken?**
No. Since v2.2.0 an attribute change starts one run that samples the source light, sends the value to the linked lights, waits `Delay`, and repeats while the source keeps changing. So during a drag the linked lights follow in samples (roughly every 350 ms at defaults) and are guaranteed to end on the final value. Raise `Debounce (brightness/color/speed)` to 200–300 ms for fewer intermediate steps, or set `Light Transition` to 0.5–2 seconds so each step fades instead of jumping.

**Q: Spook reports "unknown attributes" (e.g. `percentage` on a light) for my Linked Entities automations.**
Fixed in v2.2.0 ([#11](https://github.com/alexdelprete/ha-blueprints/issues/11)). Earlier versions watched every attribute on every linked entity, because a blueprint cannot filter an input per domain; the repair was harmless but noisy. v2.2.0 uses a single plain state trigger and works out the changed attributes inside the run. Re-import the blueprint and the repair goes away.

**Q: How fast can linked entities react?**
The reaction time is `Debounce (on/off)` plus the device round trip (command out, device acts, state report back). With the default 50 ms debounce and typical local WiFi/Zigbee devices that is 130–170 ms end to end; with debounce 0 it is the round trip alone, 80–120 ms. The `Delay` input adds nothing to a single toggle. Note that the logbook shows when the device *confirmed* the new state, which is a few tens of milliseconds after the relay actually switched.

**Q: Can I turn the light on at a specific brightness depending on which switch toggled it (e.g., dim at night entrance switch, bright at day kitchen switch)?**
Not directly with this blueprint — it propagates whatever brightness the source has, with no per-source logic. Workaround: create a small per-source automation that sets the desired brightness on the *first* light in the linked group when that specific switch fires, then this blueprint propagates that brightness to the rest. See [#7](https://github.com/alexdelprete/ha-blueprints/issues/7).

**Q: I have a smart switch (wired to power) controlling a smart bulb in "decoupled" mode. Will linking them work reliably?**
Mostly yes, with caveats. The combo of a wall switch + a bulb in decoupled mode is fundamentally a sync problem — both devices report their own state independently, and neither is authoritative. The blueprint will keep them aligned 99% of the time, but spurious state echoes on slow networks (especially Matter/Tapo dimmers) can occasionally cause the bulb to flicker off-on. Raise `Debounce (on/off)` to 500–1000 ms to absorb these. If you only ever want the switch to control the bulb (one-way sync), you may not need this blueprint at all — see [@jnrcorp's note in #3](https://github.com/alexdelprete/ha-blueprints/issues/3#issuecomment-4413889870).

**Q: What happens during a Home Assistant restart?**
Each linked entity comes back from `unavailable`/`unknown` in whatever order its integration loads. Since v2.1.0 a recovering entity adopts the state of the peers that are already online, and only if they agree — see *Recovery behavior* above. The first entity to appear does nothing; each later one aligns itself to the group. No redundant fan-out calls, and no risk of the first device to boot overwriting the real state of the others.

**Q: How do I know which version I'm running?**
Open the blueprint in **Settings → Automations & Scenes → Blueprints → Linked Entities** and check the description — the version is in the first line. The full changelog is below this FAQ.

**CHANGELOG:**
  - **2.2.0**: (2026-10-13)
    - **Fix ([#11](https://github.com/alexdelprete/ha-blueprints/issues/11))**: the six per-attribute triggers (brightness, color, fan speed) are replaced by a single plain state trigger on the linked entities. A blueprint cannot restrict an input per domain, so the old triggers watched e.g. `percentage` on lights and `brightness` on switches, which Spook's "unknown attributes" check reports as ghosts. Which attributes changed is now worked out inside the run. Thanks @junk67 for the report.
    - **Fix**: a brightness and color change arriving in one update (scenes, adaptive lighting, RGBW controllers) now propagates both. Previously the brightness trigger won the `mode: single` slot and the color change was dropped. Also reported, with a proposed fix, by @butsify in [PR #10](https://github.com/alexdelprete/ha-blueprints/pull/10).
    - **Change**: during a slider drag the linked lights now follow in samples (every attribute debounce + delay + device round trip) and are guaranteed to end on the final value. The previous design propagated only the final value and could miss it when it landed inside another run's trailing delay.
    - **Improvement**: attribute sync also turns on linked entities that are still off, so a turn-on dropped by `mode: single` can no longer leave a group split.
  - **2.1.0**: (2026-10-06)
    - **New**: supported domains extended with `siren`, `humidifier` and `remote` (on/off sync).
    - **Change**: recovery from `unavailable`/`unknown` no longer pushes the recovering entity's state to the group. The recovering entity now *adopts* the state of the linked entities that stayed online, and only when they all agree; lights and fans also adopt brightness/color/speed from a same-domain peer. Fixes state flips after HA restarts or integration reloads, when entities came back in arbitrary order and the first one to appear overwrote the others.
    - **Fix**: self-trigger guard hardened. State changes that share the originating context of the automation's last run are now rejected too. Previously, the attribute triggers that co-fire with a light turn-on were dropped by `mode: single` but still overwrote the stored context, so echoes arriving after the trailing delay slipped through.
    - **Fix**: fan speed is now carried when a fan turns on (same class of bug as [#3](https://github.com/alexdelprete/ha-blueprints/issues/3) for lights). Linked fans no longer come up at their previous speed.
    - **Fix**: linked lights and fans receive a single service call on turn-on, carrying brightness/color or speed directly. Previously a plain `turn_on` was followed by a second call with the attributes, causing a brief flash at the previous brightness/color.
    - **Change**: the debounce is split in two. `Debounce (on/off)` (existing input `debounce_milliseconds`, so values you already tuned keep applying) now covers only on/off changes and its default drops from 100 to 50 ms, which is the only fixed cost in the reaction time of linked entities. The new `Debounce (brightness/color/speed)` (`attribute_debounce_milliseconds`, default 100 ms) covers attribute changes, where it smooths slider drags.
    - **Change**: recovery waits a fixed 1 s settle before adopting the group state, instead of the on/off debounce, so the recovering device can finish its restore sequence and its peers have time to come online too.
    - **Improvement**: declared `min_version` 2025.4.0 (the `combine` template filter) and switched to the modern `triggers:` / `conditions:` / `actions:` syntax.
    - **Docs**: the `Delay` description no longer references `queued` mode.
    - **Improvement**: 20 stored traces per automation (HA default is 5), so the real run's trace is no longer evicted by the dropped co-firing triggers and rejected echoes of the same toggle.
  - **2.0.2**: (2026-05-13) — color_temp regression fix
    - **Fix**: linked lights no longer revert to a dim blueish tint when the source is set to a white color temperature. v2.0's priority chain (`hs > rgb > xy > color_temp_kelvin`) always picked `hs_color`, but Home Assistant *derives* `hs_color`/`rgb_color`/`xy_color` from kelvin when a light is in `color_mode: color_temp` — and that derived hs sits in a low-saturation blue range. Linked bulbs then ran their RGB LEDs at that hue instead of their dedicated white LEDs, producing the dim-blue look. The color attribute to propagate is now chosen from the source's `color_mode`, with the old priority chain kept only as a fallback if `color_mode` is missing. Thanks @mcmasterp for the report.
  - **2.0.1**: (2026-05-11) — regression fix
    - **Fix**: feedback loop on auto-off / pulse switches (Shelly gate openers, momentary relays). v2.0's `mode: queued` exposed a latent flaw in the context-id self-trigger guard — echoes from the blueprint's own service calls slipped through because each queued run had its own context. Reverted to `mode: single`, which keeps the trailing delay active long enough for echoes to be silently dropped. v2.0's other fixes (the `turn_on` branch now carries brightness/color directly) make the loss of queued mode harmless for lights.
    - **Fix**: tightened the self-trigger guard to also reject state events whose `context.parent_id` matches the automation's last context — defense-in-depth in case `mode: single` ever lets a one-hop echo through.
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
    - **Improvement**: trailing per-branch delay retained for throttling and echo absorption under `mode: single`.
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
