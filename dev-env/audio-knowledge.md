---
Docs: https://wiki.archlinux.org/title/PipeWire  ·  wpctl: https://man.archlinux.org/man/wpctl.1
---

## wpctl (WirePlumber) — defaults

```bash
wpctl status                    # tree: Sinks (outputs) + Sources (inputs); * marks the default
wpctl set-default <ID>          # set default — reads node type, works for sink OR source
wpctl set-mute   <ID> toggle
```
The CHOICE persists per-device (WirePlumber state) across reboots/reconnects even though the numeric
ID changes. Setting a default is a system-wide, DE-independent change.

## pactl — card profiles (Bluetooth mic tradeoff)

Bluetooth earbuds expose EITHER good stereo playback (A2DP) OR a headset profile with a mic (HSP/HFP),
never both. So using BT earbuds AS a mic forces mono call-quality output.
```bash
pactl list cards | grep -A40 bluez_card                       # find profile names
pactl set-card-profile bluez_card.XX headset-head-unit        # mic ON, output → mono
pactl set-card-profile bluez_card.XX a2dp-sink                # back to good stereo output
pactl set-card-profile <card> off                             # temporarily DISABLE a device
```
