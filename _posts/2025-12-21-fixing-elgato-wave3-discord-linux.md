---
layout: post
title: "Fixing Elgato Wave:3 Microphone Issues with Discord on Linux"
date: 2025-12-21
categories: [linux, audio, troubleshooting]
tags: [pipewire, discord, elgato, fedora, audio-fix]
excerpt: "Learn how to resolve audio issues with the Elgato Wave:3 microphone when using Discord on Linux systems, specifically Fedora, by configuring Pipewire settings."
---

## The Problem

After setting up a dual-boot configuration with Fedora alongside Windows 11, I encountered a frustrating audio issue with my Elgato Wave:3 microphone. While the classic NVIDIA driver problems (secure boot signature requirements) were straightforward to resolve, the microphone issue proved more challenging.

My setup involves an Elgato Wave:3 microphone with headphones connected directly to the mic's 3.5mm output jack.

### Symptoms:
The microphone exhibits these issues when using Discord:
- No audio transmission when initially joining voice channels
- The Wave:3 appears as the selected input device in Discord settings
- Audio only begins working after manually switching the output device in Discord
- The workaround must be repeated each time you join a voice channel

### Root Cause:
The Elgato Wave:3 enters a suspended state in PipeWire when idle to conserve system resources. When Discord attempts to use the microphone, PipeWire fails to wake the device properly. Switching the output device forces Discord to reinitialize all audio streams, which incidentally wakes the suspended input device as a side effect.

### System Configuration
This solution was developed and tested on:
- **Operating System:** Fedora 43
- **Audio Server:** PipeWire 1.4.9
- **Discord Version:** RPM package
- **Hardware:** Elgato Wave:3 USB Microphone

## The Solution
We'll implement a two-part fix that prevents the Wave:3 from suspending by configuring WirePlumber rules and creating a systemd service with a silent audio loopback.



### Step 1: Add WirePlumber Anti-Suspend Rule
WirePlumber is PipeWire's session manager. We'll add custom rules to prevent the Wave:3 from entering idle suspension.


```bash
mkdir -p ~/.config/wireplumber/main.lua.d
nano ~/.config/wireplumber/main.lua.d/51-elgato-wave3.lua
```

Add this content:
```lua
rule = {
  matches = {
    {
      { "node.name", "matches", "alsa_input.usb-Elgato_Systems_Elgato_Wave_3*" },
    },
  },
  apply_properties = {
    ["node.pause-on-idle"] = false,
    ["session.suspend-timeout-seconds"] = 0,
    ["api.alsa.period-size"] = 256,
    ["api.alsa.headroom"] = 1024,
  },
}

table.insert(alsa_monitor.rules, rule)

-- Also prevent suspension on Discord streams
stream_rule = {
  matches = {
    {
      { "application.name", "equals", "WEBRTC VoiceEngine" },
    },
  },
  apply_properties = {
    ["node.pause-on-idle"] = false,
    ["session.suspend-timeout-seconds"] = 0,
  },
}

table.insert(stream_rules, stream_rule)
```

**What this does:**
- Disables pause-on-idle for the Wave:3 input device
- Sets suspend timeout to zero (never suspend)
- Configures ALSA buffer parameters for better responsiveness
- Applies the same anti-suspend settings to Discord's WebRTC audio engine

Apply the changes by restarting WirePlumber:
```bash
systemctl --user restart wireplumber
```

### Step 2: Create Keepalive Service
The WirePlumber configuration alone isn't sufficient because PipeWire needs active audio routing to keep the device awake. We'll create a systemd user service that maintains a silent loopback connection.

Create the systemd user service directory and file:
```bash
mkdir -p ~/.config/systemd/user
vi ~/.config/systemd/user/discord-wave3-fix.service
```

Add this service configuration:
```ini
[Unit]
Description=Keep Wave:3 active for Discord
After=pipewire.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/sh -c 'sleep 2 && pactl load-module module-null-sink sink_name=wave3_dummy && pactl load-module module-loopback source=alsa_input.usb-Elgato_Systems_Elgato_Wave_3_BS10M1A09648-00.mono-fallback sink=wave3_dummy latency_msec=1'

[Install]
WantedBy=default.target
```

**Important:** Replace `BS10M1A09648-00` with your Wave:3's actual serial number. Find it using:
```bash
pactl list cards | grep "alsa_card.usb-Elgato"
```
Look for the full device name in the output, which will include your unique serial number.

**What this service does:**
- Creates a virtual null sink (audio destination that discards data)
- Establishes a loopback from the Wave:3 to the null sink
- Keeps the microphone in an active state without producing audible output
- Starts automatically when you log in

Enable and start the service:
```bash
systemctl --user daemon-reload
systemctl --user enable --now discord-wave3-fix.service
```

### Step 3: Restart Discord
For the changes to take full effect, restart Discord:
```bash
killall Discord
discord &
```
Alternatively, you can restart Discord through its GUI (Ctrl+Q to quit, then relaunch).


## Verification
Confirm that the Wave:3 is now running and not suspended:
```bash
pw-dump | jq '.[] | select(.info.props."node.name" | contains("alsa_input.usb-Elgato_Systems_Elgato_Wave_3")) | {state: .info.state, name: .info.props."node.name"}'
```
Expected output should show `"state": "running"`.

Join a Discord voice channel and test your microphone. It should work immediately without requiring any device switching.

## Performance Impact
This solution has minimal system overhead overhead:
- **CPU Usage:** Approximately 0.01% - the loopback merely copies audio samples to a null destination
- **Memory:** A few megabytes for audio buffers
- **Power Consumption:** Negligible - USB audio devices draw minimal power even when active

The Wave:3 will remain active rather than suspending, but this behavior is actually standard for professional audio interfaces, which are designed to stay ready for immediate use.

---

## Troubleshooting

If the fix doesn't work immediately:

1. **Verify the service is running:**
```bash
   systemctl --user status discord-wave3-fix.service
```

2. **Check for correct device names:**
```bash
   pactl list sources short | grep Elgato
```

3. **Restart both PipeWire and WirePlumber:**
```bash
   systemctl --user restart pipewire wireplumber
```

4. **View PipeWire logs for errors:**
```bash
   journalctl --user -u pipewire -f
```


## Conclusion
This solution eliminates the need to manually toggle Discord's output device by keeping the Elgato Wave:3 in an active state through a silent loopback connection. While it's unfortunate that this workaround is necessary, it provides a reliable, set-it-and-forget-it fix with virtually no performance penalty.

The root issue stems from how PipeWire's power management interacts with Discord's audio initialization sequence. Until Discord or PipeWire addresses this interaction more gracefully, this systemd service approach remains the most effective solution.

If you found this guide helpful, please share it with others experiencing Elgato audio issues on Linux. Feel free to leave a comment below if you have questions or discover any improvements to this method!

o7
