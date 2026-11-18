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
