# OpenRig: an open-source controller for the Dr. Dabber Switch²

A fast, login-free web app for the Switch² e-rig. It's a single HTML file with no build step, no
account, and no tracking.

> Unofficial. Not affiliated with or endorsed by Dr. Dabber. Use at your own risk. Firmware updates
> and factory reset are deliberately left to the official app.

## What it does

- Live temperature ring, session start/stop, battery and charging state
- Five presets: temperature, hold time and heating mode (Steady, Ascent, Descent, Valley, Hill, Custom)
- Six-point custom heat curve editor with a live preview
- All 28 light modes plus brightness
- Unit (°F/°C), haptics, auto shut-off, session extend, cleaning assist
- Usage stats, device info, rename, and a packet log for debugging
- **Demo mode** with a simulated rig, so you can try everything without hardware

## Why it's faster

The official app waits up to a full second between Bluetooth writes. OpenRig updates the screen
instantly, sends only what changed with short pacing, and confirms against the rig's reply. If a
change ever doesn't stick, set **Device → Write pacing → Safe**.

## Where it runs

| Platform | How |
|---|---|
| Android | Chrome or Edge |
| Mac / Windows / Linux / ChromeOS | Chrome or Edge |
| iPhone / iPad | Safari can't do Bluetooth. Open the page in **Bluefy** or **LabLink** (App Store). |

The page must be served over **HTTPS** for Bluetooth to work, or from `localhost`.

## Host it (2 minutes, free)

**GitHub Pages:** create a repo, upload `index.html`, then Settings → Pages → Deploy from branch
→ `main` / root. Your app will be at `https://<you>.github.io/<repo>/`.

Netlify Drop, Cloudflare Pages or any static host also works. On your phone, use "Add to Home Screen"
so it opens like an app.

## Protocol

See [PROTOCOL.md](PROTOCOL.md) for the full write-up of the Bluetooth protocol: service UUIDs, auth
handshake, every command and notification.

## Status

Confirmed working on a real Switch² from iPhone (iOS 18, Bluefy): pairing, presets, heating and
lights. Broader testing is ongoing. Android/Chrome and the usage stats haven't been confirmed yet.
If something misbehaves, open **Device → Packet log**, reproduce it, and include the log in an issue.

## License

MIT
