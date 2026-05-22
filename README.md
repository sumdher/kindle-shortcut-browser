## Shortcut Browser

Scriptlet that can turn your kindle into a kiosk or setup a shortcut to open a website of any choice!

**For Kindle Firmwares 5.16.4+ ONLY**

[Git Repo](https://github.com/mitchellurgero/kindle-shortcut-browser)

## Using:

1. Download the [Latest Zip](https://github.com/mitchellurgero/kindle-shortcut-browser/releases)
2. Extract the contents to the root of your kindle
3. Modify `https://example.com` in `<YourKindle>\documents\shortcutbrowser\index.html` OR change the `FULLSCREEN_SITE` variable in `<YourKindle\documents\shortcut_browser.sh` to your site of choice.
4. Run the app!

**NOW WITH KEYBOARD SUPPORT** *Only tested on Kindle 10th Gen (KT4) on 5.18.1 Firmware.*

You can also change the `GO_FULLSCREEN` to `false` to keep the UI in the background. 

A new script, `shortcut_stop.sh` is included in case the browser processes get out of control.

You can name the script whatever you want as well if you want multiple copies of it on the home screen.

## Known Issues:

- If the browser does funky shit, just hit the `Stop Shortcut Browser` scriptlet in your library or restart the device.
    - SOLVED: see [Button Control](#button-control) section
- Since this is using an iframe some websites will NOT load if using the included index.html file - simply change `FULLSCREEN_SITE` to your preferred site.

## Button Control

The `documents/shortcutbrowser/button_handler` script maps physical page turn buttons to browser actions. It starts automatically with the browser and is killed by `shortcut_stop.sh`.

On Kindle Oasis (10th Gen), the physical page turn buttons are mapped to browser controls.

On Oasis:
| Button | Short Press | Long Press (>0.8s) | Combo |
|--------|-------------|-------------------|-------|
| Back | Decrease brightness | Reload page | — |
| Next | Increase brightness | Stop browser | — |
| Back held + Next press&release | — | — | Toggle display inversion (Y8 ↔ Y8INV) |
| Next held + Back press&release | — | — | *same as above; can be changed* Toggle display inversion (Y8 ↔ Y8INV) |

Reload uses Chrome DevTools Protocol (CDP) over localhost.

### Adding button support for other Kindle models

Every Kindle model exposes its hardware buttons as a Linux input device. To find the right device and key codes for your model:

> *helps if you have SSH access to your kindle device, or else, pipe the output to a file and read later*

**Step 1 — Find the button input device:**
```sh
cat /proc/bus/input/devices | grep -A3 "gpio-keys"
```
Note the event number in the `Handlers` line. If `gpio-keys` doesn't appear, run without the grep and look for a device with `KEY=` in its capabilities and a name suggesting physical buttons. (for Kindle Oasis, it was: `event3`).

**Step 2 — Capture button events:**

Ideally, run this in an SSH session and don't close it until values are captured:

```sh
cat /dev/input/eventX | hexdump -v -e '16/1 "%02X"'
```

Replace `eventX` with the event number from Step 1.

Press each button and note the hex output. Bytes 8-9 are the key code, byte 12 is `01` for press and `00` for release.

> **Tip:** The output is raw hex and not very human-readable. Paste the lines into an AI assistant with a prompt like: "these are Linux input events in hexdump format (16 bytes each). Bytes 8-9 are the event type, bytes 10-11 are the key code, bytes 12-15 are the value (01=press, 00=release). What are the key codes for each button press?"

**Known key codes:**

| Device | Back button | Next button |
|--------|-------------|-------------|
| Kindle Oasis 10th Gen (KOA3) | `6D00` | `6800` |

**Step 3 — Update the handler:**

Open `documents/shortcutbrowser/button_handler`,

set `DEV=/dev/input/eventX` and replace `6D00`/`6800` with your key codes

That's it! It should now work according to your configuration.

## Tips for Always-On / Kiosk Use

**Prevent screen sleep:**
```sh
lipc-set-prop com.lab126.powerd preventScreenSaver 1
```

**Persist across reboots** — create `/etc/upstart/prevent_sleep.conf`:
```sh
start on started lab126_gui
script
    lipc-set-prop com.lab126.powerd preventScreenSaver 1
end script
```

**Keep WiFi alive** — add a background ping to your startup script:
```sh
while true; do ping -c 1 <router-ip> > /dev/null 2>&1; sleep 120; done &
```