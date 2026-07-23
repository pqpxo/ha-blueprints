# IKEA STYRBAR E2313 | Two Light Controller with Colour Matching

<p align="center">
  <img src="https://www.zigbee2mqtt.io/images/devices/E2001-E2002-E2313.png" width="230" alt="Notify Studio logo">
</p>

A Home Assistant automation blueprint for controlling two lights from a Zigbee2MQTT four-button remote, where the second light inherits the brightness and colour of the first.

Built for the IKEA STYRBAR (E2313) but works with any Z2M remote that publishes the same action strings. See [Other remotes](#other-remotes).

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fpqpxo%2Fha-blueprints%2Fblob%2Fmain%2Fikea-styrbar-e2313%2Ftwo_light_controller.yaml)

## What it does

| Gesture | Action |
| --- | --- |
| Left arrow, short press | Toggles the left light |
| Right arrow, short press | Toggles the right light |
| Up, short press | Brightens every light that is currently on |
| Down, short press | Dims every light that is currently on |
| Up, hold | Sets a warm colour temperature on the lights that are on |
| Down, hold | Sets a cool colour temperature on the lights that are on |

Two behaviours make this different from a plain button-to-action blueprint.

**Matched turn-on.** Switch one light on while the other is already on and it comes up at the other light's brightness and colour, whether that is an RGB colour or a colour temperature. With both lights off, it comes on at a brightness you configure.

**Independent brightness stepping.** Up and down adjust each light that is on relative to its own current level, rather than forcing both to a shared value. Lights that are off are left alone rather than being switched on.

**Live colour sync (optional).** While both lights are on, changing the colour of either one from anywhere, a dashboard, voice assistant or another automation, mirrors it to the other. Turn this off if you want the two lights set independently.

## Requirements

- Home Assistant 2024.10 or newer (the blueprint uses input sections)
- Zigbee2MQTT with the MQTT integration configured in Home Assistant
- Two light entities, ideally both colour capable

Mixed capabilities are handled safely. If one light does colour and the other only does colour temperature, colour is copied when the receiving light supports it and falls back to matching brightness alone when it does not. Nothing errors.

## Installation

Click the import badge above, or add it manually:

1. Copy `two_light_controller.yaml` into `config/blueprints/automation/pqpxo/`
2. Reload automations, or restart Home Assistant
3. Go to **Settings > Automations & Scenes > Blueprints** and create an automation from it

## Configuration

### Controller

| Input | Default | Notes |
| --- | --- | --- |
| Controller name | none | The device's friendly name in Zigbee2MQTT, for example `Office Lights Switch` |
| Base MQTT topic | `zigbee2mqtt` | The base topic configured in Z2M |

### Lights

| Input | Notes |
| --- | --- |
| Left button light | Toggled by the left arrow |
| Right button light | Toggled by the right arrow |

### Behaviour

| Input | Default | Notes |
| --- | --- | --- |
| Brightness step | 20% | Change per short press of up or down |
| Brightness when both lights are off | 100% | Used when there is nothing to copy from |
| Warm colour temperature | 2000K | Applied on hold up |
| Cool colour temperature | 6666K | Applied on hold down |
| Keep colours in sync | on | Live mirroring while both lights are on |
| Transition | 0.3s | Applied to on, off and brightness changes only |

## How the colour matching works

Two implementation details are worth knowing, because both were the result of debugging real bulb behaviour rather than preference.

**Colour is sent as a separate command.** Many Zigbee bulbs silently discard a colour that arrives in the same command that switches them on. The blueprint switches the light on with brightness only, waits for it to report `on`, waits a further 300ms, then sends the colour on its own with no transition attached. It reads back what the light reports and retries up to three times if the colour did not take.

**Capability checks are computed inline, never stored in a variable.** `supported_color_modes` is a list of `StrEnum` members. Rendering that list through a template variable produces text like `[<ColorMode.HS: 'hs'>]`, which cannot be parsed back into a list, so the value silently becomes a string and every membership test against it fails. Jinja's `string` filter does not fix this, because `soft_str` returns `StrEnum` values untouched. The awkward part is that this only affects some integrations, so a setup can work in one direction and fail in the other. If you refactor this blueprint, leave those checks inline.

**Loop safety on the sync.** Mirroring a colour changes the receiving light, which re-triggers the automation with source and target swapped. That second run finds the pair already within tolerance, matches no branch, and stops. Tolerances are 5 on hue and saturation and 100K on colour temperature, wide enough to absorb the difference between what you send a bulb and what it reports back.

## Other remotes

The action strings are hardcoded to the STYRBAR set:

```
on, off, arrow_left_click, arrow_right_click,
brightness_move_up, brightness_move_down
```

To use a different Z2M remote, check what it publishes on its MQTT topic (Zigbee2MQTT frontend, or listen to `zigbee2mqtt/YOUR_DEVICE` in **Developer Tools > Events**) and edit those strings in the `choose` conditions.

## Troubleshooting

**Nothing happens on a button press.** Confirm the controller name matches the Z2M friendly name exactly, and listen on `zigbee2mqtt/YOUR_DEVICE` to check the actions arriving are the ones listed above.

**The colour does not match.** Open the automation trace for that press and check the variables step. `copy_colour` or `copy_temp` should be `true`. If both are false, the source light's `color_mode` is outside the expected set, or the target does not report a compatible mode in `supported_color_modes`.

**The colour is sent but does not stick.** The trace will show all three retry attempts running. That means the commands are leaving Home Assistant and something is undoing them. Adaptive Lighting is the usual cause, as is any other automation that reacts to a light turning on.

**The sync runs continuously.** Two runs per colour change, the real one and the echo that matches nothing, is correct. A continuous stream means a bulb is reporting back a value it cannot reproduce, so widen the tolerances in the sync branch conditions.

## Licence

MIT
