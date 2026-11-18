# Hands REST API & Capabilities Specification

Hands provides a secure, encrypted HTTPS REST API on port `8766` (and optional plaintext HTTP on `8765` for localhost).

## 1. Authentication & Protocol Basics

- **Base URL**: `https://<device-ip>:8766`
- **Headers**:
  ```http
  Authorization: Bearer <client_token>
  Content-Type: application/json
  ```
- **Error Format**:
  All errors return a structured JSON response:
  ```json
  {
    "ok": false,
    "error": {
      "code": "disarmed",
      "message": "The phone is disarmed. Acquire an arming lease before sending commands."
    }
  }
  ```

---

## 2. API Endpoints

### `GET /v1/status`
Retrieves real-time status of the device, arm state, work screen target, battery, and capabilities.

#### Response Example:
```json
{
  "ok": true,
  "result": {
    "service": "hermes-hands",
    "protocol_version": 1,
    "armed": false,
    "backend": "hybrid",
    "target": "auto",
    "effective_target": "agent",
    "screen_control_available": true,
    "lanes": "l0+l1",
    "agent_screen": {
      "active": false,
      "display_id": -1
    },
    "shizuku": {
      "installed": true,
      "running": true,
      "permission_granted": true,
      "bound": true,
      "message": "Shizuku backend connected"
    },
    "keep_awake": {
      "enabled": false,
      "screen_on": false,
      "screen_locked": true
    },
    "capabilities": [
      "device.arm", "device.disarm", "device.snapshot", "device.screenshot",
      "device.click", "device.swipe", "device.type", "device.open_app", ...
    ]
  }
}
```

---

### `POST /v1/commands`
Submits a command to Hands. Requires a top-level `controller_id` and action-specific arguments.

#### Generic Request Body:
```json
{
  "controller_id": "agent-primary",
  "lease_id": "optional-lease-uuid",
  "action": "device.action_name",
  "arguments": {}
}
```

---

## 3. Core Action Catalog

### Arming & Leases

#### `device.arm`
Acquires an exclusive control lease to execute screen automation.
- **Arguments**:
  - `duration_seconds` (*integer*, optional, default: 3600): Length of control lease.
  - `target` (*string*, optional, `"auto"` | `"agent"` | `"primary"`): Preferred display target.
- **Response**: Returns `lease_id` and updated `status`.

#### `device.arm.renew`
Renews an active lease.
- **Arguments**:
  - `duration_seconds` (*integer*): Additional lease time.

#### `device.disarm`
Releases control lease and closes the agent screen.

---

### Screen Observation & Gestures

#### `device.snapshot`
Captures the interactive accessibility / UI tree of the target display.
- **Arguments**:
  - `interactive_only` (*boolean*, optional, default: false)
- **Response**:
  ```json
  {
    "ok": true,
    "result": {
      "app": "com.android.settings",
      "display_id": 34,
      "snapshot_id": "c138fd94-f6eb-460d-85fa-848ca98dc9c7",
      "size": {"w": 720, "h": 1560},
      "nodes": [
        {
          "ref": "e1",
          "role": "TextView",
          "text": "Connections",
          "clickable": true,
          "bounds": [108, 825, 256, 862]
        }
      ]
    }
  }
  ```

#### `device.screenshot`
Takes a PNG screenshot of the target display (Display 0 or Virtual Display $N$).
- **Response**:
  ```json
  {
    "ok": true,
    "result": {
      "mime": "image/png",
      "png_b64": "<base64_encoded_png>",
      "w": 720,
      "h": 1560,
      "display_id": 34
    }
  }
  ```

#### `device.click`
Clicks a specific element reference from the latest snapshot or absolute $(x, y)$ coordinates.
- **Arguments**:
  - `ref` (*string*, e.g., `"e1"`): Element ref from the previous snapshot.
  - `snapshot_id` (*string*): Snapshot ID that provided the reference.
  - *OR* `x`, `y` (*integer* / *float*): Screen coordinates.

#### `device.swipe`
Performs a swipe or drag gesture.
- **Arguments**:
  - `from_ref`, `to_ref` (*string*): References to drag between.
  - *OR* `x1`, `y1`, `x2`, `y2` (*integer*): Coordinate-based swipe.
  - `duration_ms` (*integer*, optional, default: 350)

#### `device.type`
Injects text into the currently focused input field.
- **Arguments**:
  - `text` (*string*): Text to type.

#### `device.press`
Simulates hardware key presses (`home`, `back`, `volume_up`, `volume_down`, `power`).
- **Arguments**:
  - `key` (*string*): `"home"`, `"back"`, `"recent"`, etc.

#### `device.open_app`
Launches an application by package name. On the Agent screen, launches isolated into the virtual display.
- **Arguments**:
  - `package` (*string*): E.g., `"com.android.settings"`

---

### System & Gate Operations

#### `device.shizuku.connect`
Auto-rebinds or connects to the Shizuku service.
- **Arguments**: None.

#### `device.gates.get`
Returns the state of all safety gates, active permissions, and runtime modes.

#### `device.gates.set`
Configures gates or forces auto-connect:
- **Arguments**:
  - `shizuku_connect` (*boolean*, optional)
  - `work_screen_target` (*string*, optional, `"auto"` | `"agent"` | `"primary"`)

#### `device.shell.exec` (Privileged)
Executes a shell command via Shizuku (UID 2000).
- **Arguments**:
  - `command` (*string*)
  - `timeout_ms` (*integer*, optional, default: 10000)

#### `device.power.wake`
Wakes the physical device screen.

#### `device.keyguard.dismiss`
Dismisses the lockscreen and optionally inputs a PIN.
- **Arguments**:
  - `pin` (*string*, optional)

---

### Application Lifecycle

#### `device.apps.install`
Installs an Android application (.apk). When Shizuku is connected, performs a silent background install with runtime permissions auto-granted (`pm install -r -d -g <path>`) with zero user prompts. When Shizuku is unavailable, launches the system `PackageInstaller` intent sheet or Play Store.
- **Arguments**:
  - `path` (*string*, optional): Local filesystem path to the APK (e.g. `/sdcard/Download/app.apk`).
  - `url` (*string*, optional): Remote HTTP/HTTPS URL. Hands downloads the file to temporary storage and installs it.
  - `package` (*string*, optional): Package ID fallback to open in Play Store / Galaxy Store.
  - `grant_permissions` (*boolean*, optional, default: true): Auto-grant runtime permissions (Shizuku only).
  - `allow_downgrade` (*boolean*, optional, default: true): Permit installing older versions (Shizuku only).
  - `reinstall` (*boolean*, optional, default: true): Keep existing application data during update (Shizuku only).

#### `device.apps.uninstall`
Uninstalls an application. When Shizuku is connected and `silent: true`, removes the package silently in background via `pm uninstall`. Otherwise, triggers the system confirmation dialog.
- **Arguments**:
  - `package` (*string*, required): Package name to remove.
  - `silent` (*boolean*, optional, default: false): Silent uninstall via Shizuku.

---

### Hardware & Peripherals

#### `device.torch`
Toggles or sets the hardware camera flashlight. Genuine L0 tool, no Android permissions required, works directly from background or screen-off.
- **Arguments**:
  - `state` (*string*, optional, `"on"` | `"off"` | `"toggle"`, default: `"toggle"`): Target flashlight state.
- **Response**:
  ```json
  {
    "ok": true,
    "result": {
      "torch": "on",
      "enabled": true,
      "camera_id": "0"
    }
  }
  ```

---

### Clock & Timer Management

#### `device.alarm.set`
Schedules an alarm in the system clock app.
- **Arguments**:
  - `hour` (*integer*, required, 0–23): Alarm hour.
  - `minute` (*integer*, required, 0–59): Alarm minute.
  - `label` (*string*, optional): Alarm description.
  - `skip_ui` (*boolean*, optional, default: true): Request clock app to stay in background.

#### `device.alarm.dismiss`
Dismisses or cancels a scheduled or currently sounding alarm via `AlarmClock.ACTION_DISMISS_ALARM`.
- **Arguments**:
  - `search_mode` (*string*, optional, `"time"` | `"label"` | `"next"` | `"all"`): Targeting criteria.
  - `hour` (*integer*, optional, 0–23): For `search_mode="time"`.
  - `minute` (*integer*, optional, 0–59): For `search_mode="time"`.
  - `label` (*string*, optional): For `search_mode="label"`.
  - `skip_ui` (*boolean*, optional, default: true)

#### `device.alarm.show`
Directly opens the system clock app to the Alarms tab.

#### `device.alarm.snooze`
Snoozes a currently ringing alarm via `AlarmClock.ACTION_SNOOZE_ALARM`.
- **Arguments**:
  - `duration_minutes` (*integer*, optional): Snooze duration in minutes.

#### `device.timer.set`
Starts a countdown timer in the clock app.
- **Arguments**:
  - `seconds` (*integer*, required): Timer length in seconds.
  - `label` (*string*, optional): Timer label.
  - `skip_ui` (*boolean*, optional, default: true)

#### `device.timer.dismiss`
Dismisses an expired countdown timer via `AlarmClock.ACTION_DISMISS_TIMER`.
- **Arguments**:
  - `skip_ui` (*boolean*, optional, default: true)

#### `device.timer.reset`
Resets an active or expired countdown timer via `android.intent.action.RESET_TIMER`.
- **Arguments**:
  - `skip_ui` (*boolean*, optional, default: true)

#### `device.timer.show`
Directly opens the system clock app to the Timers tab via `AlarmClock.ACTION_SHOW_TIMERS`.
