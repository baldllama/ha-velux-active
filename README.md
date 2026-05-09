# VELUX Active with Netatmo — Home Assistant Integration

A Home Assistant custom integration for VELUX ACTIVE with NETATMO, supporting full control of:

- 🪟 **Roof windows** — open, close, set position, stop (requires one-time key extraction, see below)
- 🪟 **Roller shutters & awning blinds** — open, close, set position, stop (works out of the box)
- 🔒 **Departure mode** — lock/unlock all movement via the gateway (integrates with alarm systems)
- 🌡️ **Sensors** — rain, temperature, humidity, CO₂, light intensity (via the VELUX gateway)

---

## Installation

### Via HACS (recommended)

1. Open HACS in Home Assistant
2. Go to **Integrations** → click the three-dot menu → **Custom repositories**
3. Add this repository URL and select **Integration** as the category
4. Search for **Velux Active** and install it
5. Restart Home Assistant

### Manual

Copy the `velux_active` folder into your `/config/custom_components/` directory and restart Home Assistant.

---

## Setup

1. Go to **Settings → Devices & Services → Add Integration**
2. Search for **Velux Active with Netatmo**
3. Enter your VELUX ACTIVE account email and password
4. On the next screen you will be asked for a **Hash Sign Key** and **Sign Key ID** — these are required for roof window control. See [Obtaining your window signing keys](#obtaining-your-window-signing-keys) below.
   - If you only have roller shutters or blinds, leave these blank and click **Submit**

---

## Obtaining your window signing keys

Roof windows require cryptographic signing for security. The API verifies that commands come from a paired device before it allows a roof window to open or close.

You need to extract two values from the VELUX Android app once:

- **Hash Sign Key** — logged by the patched app as `velux-debug`
- **Sign Key ID** — logged by the patched app as `velux-key-id`

Older instructions used mitmproxy to capture `sign_key_id`. That is no longer needed when using the patched APK debug logging described below.

### What you need

- An **Android phone** connected to the same Wi-Fi as your VELUX gateway
- A **computer** with `adb`
- Your VELUX ACTIVE gateway powered on and accessible

> **iPhone users:** The iOS app cannot be used for this extraction flow. Use an Android device; a cheap secondhand device is enough.

---

### Step 1 — Install Android platform tools

```bash
# macOS
brew install android-platform-tools

# Linux
sudo apt install android-tools-adb
```

Verify `adb` is working:
```bash
adb version
```

---

### Step 2 — Enable USB debugging on your Android phone

1. Go to **Settings → About Phone**
2. Tap **Build number** (or **MIUI version** on Xiaomi) 7 times to enable Developer Options
3. Go to **Settings → Developer Options**
4. Enable **USB Debugging**
5. Connect the phone to your computer via USB and tap **Allow** when prompted

Verify the phone is detected:
```bash
adb devices
```
You should see your device listed.

---

### Step 3 — Install a patched Velux APK

Patch the VELUX Android app so it logs the signing values to Android logcat. The patched app does not need to trust a proxy certificate and the phone does not need a proxy configured.

The patch adds log output around the app's signing code:

- `velux-key-id` — the Sign Key ID to enter in Home Assistant
- `velux-debug` — the Hash Sign Key to enter in Home Assistant

If you already have a patched APK that emits these log tags, install it and skip to [Step 4](#step-4--capture-the-keys-from-logcat).

To build one yourself, install the VELUX ACTIVE app on the phone, then pull its APK files from the device:

```bash
mkdir -p ~/velux-apks

adb shell pm path com.velux.active | tr -d '\r' | while IFS= read -r line; do
  apk="${line#package:}"
  adb pull "$apk" "$HOME/velux-apks/$(basename "$apk")"
done
```

This stores `base.apk` and any split APKs in `~/velux-apks/`.

Now decompile, patch, and repack it:

```bash
# Install apktool
brew install apktool   # macOS
# or: sudo apt install apktool  # Linux

# Decompile
apktool d ~/velux-apks/base.apk -o ~/velux_patched
```

Patch the signing code to log the key values. In the tested APK, this code is in:

```bash
~/velux_patched/smali/android/br1.smali
```

The exact filename can change between app versions, so search for the signing mapper if needed:

```bash
grep -rn "HashMapperKey" ~/velux_patched/smali* | head
```

Open the mapper file and add two log statements in both signing methods.

#### Log the Sign Key ID

Search for this block:

```smali
    invoke-virtual/range {p3 .. p3}, Landroid/fid;->c()[B

    move-result-object v2

    invoke-virtual {v0, v2}, Landroid/br1;->a([B)Ljava/lang/String;

    move-result-object v2
```

Add the `velux-key-id` log immediately after it:

```smali
    invoke-virtual/range {p3 .. p3}, Landroid/fid;->c()[B

    move-result-object v2

    invoke-virtual {v0, v2}, Landroid/br1;->a([B)Ljava/lang/String;

    move-result-object v2

    const-string v3, "velux-key-id"

    invoke-static {v3, v2}, Landroid/util/Log;->w(Ljava/lang/String;Ljava/lang/String;)I
```

#### Log the Hash Sign Key

Search for this block:

```smali
    invoke-virtual/range {p3 .. p3}, Landroid/fid;->b()[B

    move-result-object v10

    invoke-virtual {v0, v10}, Landroid/br1;->a([B)Ljava/lang/String;

    move-result-object v10
```

Add the `velux-debug` log immediately after it:

```smali
    invoke-virtual/range {p3 .. p3}, Landroid/fid;->b()[B

    move-result-object v10

    invoke-virtual {v0, v10}, Landroid/br1;->a([B)Ljava/lang/String;

    move-result-object v10

    const-string v11, "velux-debug"

    invoke-static {v11, v10}, Landroid/util/Log;->w(Ljava/lang/String;Ljava/lang/String;)I
```

Repeat both additions once more lower in the same mapper file. The tested APK has two signing paths, so both `velux-key-id` and `velux-debug` should appear twice in the patched file.

If the registers are different in your APK version, use the register from the nearby `move-result-object` line. The log line must log that same value.

After patching, verify the tags exist:

```bash
grep -rn 'velux-key-id\|velux-debug' ~/velux_patched/smali*
```

You should see two `velux-key-id` entries and two `velux-debug` entries.

The final logcat output should look like this when a roof window is moved:

```text
W velux-key-id: sign_key_id
W velux-debug: hash_sign_key
```

Enter `hash_sign_key` as **Hash Sign Key** and `sign_key_id` as **Sign Key ID** in Home Assistant.

Then rebuild and sign the APK:

```bash
# Generate a signing key (one time only)
keytool -genkey -v -keystore ~/velux-key.keystore -alias velux \
  -keyalg RSA -keysize 2048 -validity 10000 \
  -storepass password123 -keypass password123 \
  -dname "CN=Velux, O=Test, C=GB"

# Rebuild
rm -rf ~/velux_patched/build
apktool b ~/velux_patched -o ~/velux-patched.apk

# Sign
mkdir -p ~/velux-signed

apksigner sign \
  --ks ~/velux-key.keystore \
  --ks-pass pass:password123 \
  --key-pass pass:password123 \
  --out ~/velux-signed/base.apk \
  ~/velux-patched.apk
```

> **Rebuild note:** If `apktool b` fails with drawable/resource errors, look for empty drawable entries such as `<drawable name="exo_..." />` in `res/values/drawables.xml` and replace them with transparent values like `<drawable name="exo_...">#00000000</drawable>`. Also check `res/drawable/thumb_manual_loading.xml`; if an `android:drawable="@null"` item causes rebuild errors, replace it with a transparent shape item.

Also sign any split APKs pulled from the device:
```bash
for apk in ~/velux-apks/split_config*.apk; do
  [ -e "$apk" ] || continue
  apksigner sign \
    --ks ~/velux-key.keystore \
    --ks-pass pass:password123 \
    --key-pass pass:password123 \
    --out "$HOME/velux-signed/$(basename "$apk")" \
    "$apk"
done
```

Uninstall the existing app and install the patched version:
```bash
adb uninstall com.velux.active

install_apks=(~/velux-signed/base.apk)
for apk in ~/velux-signed/split_config*.apk; do
  [ -e "$apk" ] || continue
  install_apks+=("$apk")
done

adb install-multiple "${install_apks[@]}"
```

> **Note for Xiaomi / MIUI users:** You may need to disable app verification in Developer Options before the install will succeed.

---

### Step 4 — Capture the keys from logcat

Start watching the patched app logs:

```bash
adb logcat -s velux-key-id:W velux-debug:W
```

Open the patched Velux app on your phone and log in. When prompted, press the button on your VELUX gateway to complete authentication.

Once authenticated, **tap a roof window to move it** (open or close it a little).

You should see output like this in your logcat terminal:
```text
W velux-key-id: sign_key_id
W velux-debug: hash_sign_key
```

Your two keys are:

- **Hash Sign Key** — the `velux-debug` value
- **Sign Key ID** — the `velux-key-id` value

---

### Step 5 — Enter the keys in Home Assistant

You can enter the keys during initial setup (Step 2 of the config flow), or add them to an existing config entry:

**For an existing installation**, run this on your HA host (replace the key values with yours):

```bash
# Docker / Home Assistant OS
docker exec homeassistant python3 -c "
import json
with open('/config/.storage/core.config_entries') as f:
    data = json.load(f)
for entry in data['data']['entries']:
    if 'velux' in entry.get('domain','').lower():
        entry['data']['hash_sign_key'] = 'YOUR_HASH_SIGN_KEY_HERE'
        entry['data']['sign_key_id'] = 'YOUR_SIGN_KEY_ID_HERE'
        print('Updated:', entry['title'])
with open('/config/.storage/core.config_entries', 'w') as f:
    json.dump(data, f)
"
```

Then restart Home Assistant. Your roof windows will now have full control.

---

### Step 6 — Clean up

You can uninstall the patched app and reinstall the regular Velux app from the Play Store. The keys are tied to your gateway pairing and do not change unless you re-pair your gateway.

---

## How the signing works

The Velux API requires roof window commands to be cryptographically signed using HMAC-SHA512. This prevents unauthorized control of windows (which are openings in your roof and pose a weather/security risk if operated without authorisation).

The signature is computed as:
```
msg    = f"target_position{position}{timestamp}{nonce}{device_id}"
hash   = HMAC-SHA512(key=base64decode(HashSignKey), msg=msg)
result = base64encode(hash).replace('+', '-').replace('/', '_')
```

The timestamp is Unix time in seconds. When multiple windows are commanded simultaneously (e.g. via a group), they are sent in a single API call with incrementing nonces (0, 1, 2, 3...) and the same timestamp — matching the behaviour of the official Velux app.

---

## Auto Ventilation

Each roof window has an **Auto Ventilation** switch (`switch.window_name_auto_ventilation`). When enabled, the VELUX gateway automatically adjusts the window position based on indoor CO₂ levels, temperature and humidity. When disabled, the window only moves in response to manual commands from HA or the Velux app.

This is useful for automations — for example, disabling auto ventilation when you leave home and re-enabling it when you return.

```yaml
# Disable auto ventilation when away
- action: switch.turn_off
  target:
    entity_id:
      - switch.window_1_auto_ventilation
      - switch.window_2_auto_ventilation
      - switch.window_3_auto_ventilation
      - switch.window_4_auto_ventilation
```

---

## Departure Mode (Lock)

The integration exposes the VELUX gateway's departure mode as a **lock entity** (`lock.velux_departure_mode`). When locked, the gateway disables all window and blind movement — useful for integrating with a home alarm system.

- **Locked** = away mode (all movement disabled)
- **Unlocked** = home mode (normal operation)

The lock state is read directly from the gateway on every poll — it survives HA restarts and reflects the true state even if departure mode was toggled from the Velux app.

### Locking requires signing keys

Activating departure mode (locking) works without signing keys. **Deactivating** departure mode (unlocking) requires the Hash Sign Key and Sign Key ID to be configured, as the API requires a signed command to re-enable movement for security reasons.

### Example automation — activate with alarm

```yaml
alias: Velux lock when alarm set
triggers:
  - trigger: state
    entity_id: alarm_control_panel.my_alarm
    to:
      - armed_away
      - armed_home
actions:
  - action: lock.lock
    target:
      entity_id: lock.velux_departure_mode

---

alias: Velux unlock when alarm disarmed
triggers:
  - trigger: state
    entity_id: alarm_control_panel.my_alarm
    to: disarmed
actions:
  - action: lock.unlock
    target:
      entity_id: lock.velux_departure_mode
```

---

## Entities

After setup the following entities are created:

**Per roof window** (requires signing keys):
- `cover.window_name` — open, close, set position, stop
- `switch.window_name_auto_ventilation` — enable/disable automatic ventilation algorithm

**Per roller shutter / awning blind:**
- `cover.blind_name` — open, close, set position, stop

**Gateway:**
- `lock.velux_departure_mode` — departure mode lock
- `binary_sensor.velux_gateway_rain_detected` — rain detection

> **Note:** Secure position and rain position sensors are planned for a future release pending pyatmo support for these fields.

---

## Troubleshooting

### API rate limit errors (error code 26 / 429)

The Netatmo API has rate limits shared across all integrations using your account. If you see errors like:

```
API limit exceeded. This could be your Application limit or User limit.
```

This means your account has been temporarily throttled. It will clear on its own within 30–60 minutes. To avoid triggering it:

- Do not restart Home Assistant repeatedly in quick succession
- Avoid setting a polling interval below 30 seconds
- Be aware that the fast polling mode (triggered after a movement command) is automatically cancelled if a rate limit is detected

If you are hitting rate limits regularly during normal use, check whether another integration is also polling the Netatmo API on the same account.

### Windows showing Unknown state on startup

This can happen if the gateway is temporarily offline or the API is rate limited when HA starts. The integration will automatically retry on the next poll. If windows remain Unknown after a few minutes, check that your VELUX gateway has a solid green light and internet access.

### Gateway goes offline

If the gateway loses its cloud connection (shown by the Velux app also being unable to control devices), the integration will keep retrying. Power cycling the gateway (unplug for 30 seconds) usually resolves this.

---

## Credits

Based on the original [ha-velux-active](https://github.com/Niek/ha-velux-active) integration by [@Niek](https://github.com/Niek).

Thanks to contributors who helped test VELUX ACTIVE roof window support, pyatmo compatibility, and Home Assistant signing behavior.

Window signing support was implemented using the VELUX signing algorithm documented by [@ZTHawk](https://github.com/ZTHawk), and validated with smali patching and Android logcat.
