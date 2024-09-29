[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Falexdelprete%2Fha-blueprints%2Fmain%2Fha-blueprint-linked-entities.yaml)

**Linked Entities v1.3** 🔛

This blueprint allows you to easily create/maintain an automation that links the state of multiple entities:
  - turn ANY linked entity ON, it will turn ON ALL linked entities.
  - turn ANY linked entity OFF, it will turn OFF ALL linked entities.
  - set the brightness of any light entity, it will set the same brightness of ALL linked light entities.
  - set the color temp of any light entity, it will set the same color temperature of ALL linked light entities.
  - set the speed (percentage) of any fan entity, it will set the speed of ALL linked fan entities.

**NOTE**: You can select any entity with an ON/OFF state (switches, lights, etc.), lights with brightness or color temperature attributes, and Fans with speed (percentage) attributes.

My main use-case was for multiple light switches in the house controlling the same light, but I also use it for other things:
  - at dawn, when I turn on the external lights (a shelly switch), I also link the pool light mqtt switch.
  - when I want to open the external gate, I can use several linked switches in multiple rooms of the house
  - when I want to manually turn on/off the irrigation system, I use two switches (internal and external) to activate my RainMachine cycle

I'm sure you'll find many more use-cases. :slight_smile:

**CHANGELOG:**
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
