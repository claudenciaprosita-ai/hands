# Shizuku Setup & Configuration Guide

Hands utilizes [Shizuku](https://shizuku.rikka.app/) to perform privileged system operations without requiring full device rooting:
- Managing and launching applications onto secondary virtual displays.
- High-fidelity coordinate and key event input injection.
- Reading multi-display window states and capturing display hardware buffers.
- Executing elevated shell commands as `com.android.shell` (UID 2000).

---

## 1. Installing Shizuku

1. Download **Shizuku** from Google Play or from GitHub releases:
   - [Google Play Store](https://play.google.com/store/apps/details?id=moe.shizuku.privileged.api)
   - [GitHub Releases](https://github.com/RikkaApps/Shizuku/releases)
2. Launch the Shizuku app on your Android device.

---

## 2. Starting Shizuku

### Method A: Wireless Debugging (No PC required after setup)
*(Requires Android 11+ and Wi-Fi connection during initial pairing)*

1. Enable **Developer Options** on your phone (Settings > About Phone > Tap "Build Number" 7 times).
2. Inside Developer Options, toggle **Wireless Debugging** ON.
3. Open Shizuku and tap **Pairing** under *Start via Wireless Debugging*.
4. In Android's Wireless Debugging screen, tap **Pair device with pairing code**.
5. Enter the 6-digit code into the Shizuku notification prompt.
6. Return to the Shizuku app and tap **Start**.

### Method B: ADB via Computer / Terminal
Connect your phone via USB or Tailscale wireless ADB and execute:

```bash
# Direct native starter (verified on Samsung Galaxy / modern Android):
adb shell /data/local/tmp/libshizuku.so

# Alternative legacy starter scripts:
adb shell sh /sdcard/Android/data/moe.shizuku.privileged.api/start.sh
```

### Method C: Root (Magisk / KernelSU / APatch)
If your device is rooted:
1. Open Shizuku.
2. Under *Start via Root*, tap **Start**.
3. Grant Superuser privileges when prompted.

---

## 3. Authorizing Hands

1. Open the **Shizuku** app.
2. Tap **Authorized Applications**.
3. Toggle the switch next to **Hands** to **ON**.
4. Open the **Hands** application on your device.
5. In Hands, navigate to the **Work screen** card. You will see:
   `Backend: Shizuku / Hybrid · Connected`

---

## 4. Hands Auto-Connect & Self-Healing

Hands features built-in self-healing for Shizuku:
- **Automatic Binder Binding**: Whenever Hands starts, receives an arming lease, or wakes from idle, `ShizukuController` verifies the binder connection and automatically re-binds.
- **Agent Tool (`mobile_shizuku_connect`)**: If the connection drops or is delayed during long reasoning, remote agents can call `mobile_shizuku_connect` to reconnect synchronously.

---

## 5. Wi-Fi Disconnects & Samsung One UI Process Lifetime

### Why does Shizuku stop when Wi-Fi is disconnected?
Under vanilla AOSP, processes started via ADB inherit ADB's session, but on certain OEM skins—particularly **Samsung One UI**:
1. **`sec_adbd` Process Group Teardown**: Samsung's proprietary ADB daemon (`sec_adbd`) actively monitors the network state. When Wi-Fi is toggled off or disconnected, `sec_adbd` shuts down its network listener and explicitly sends `SIGKILL` or `SIGHUP` to the entire child process group spawned under the wireless session.
2. **Android 12+ Phantom Process Killer**: Android enforces a strict limit (default 32) on background child processes spawned by apps or shell. If the phone is under memory pressure or changes network states, Android aggressively terminates background processes lacking foreground notification tokens.

### Best Practices for Maximum Uptime:
- **Keep Wi-Fi Active or Connected to Tailscale**: Leaving Wi-Fi enabled keeps `sec_adbd` running smoothly.
- **Quick Restart via ADB / Tailscale**: If Shizuku terminates, execute:
  ```bash
  adb shell /data/local/tmp/libshizuku.so
  ```
- **Fallback to Hybrid Mode**: In Hands' settings, keep the engine set to **Hybrid**. If Shizuku ever drops while you are away from Wi-Fi, Hands seamlessly falls back to the system Accessibility engine for primary screen control so you never lose remote access.
