# Hands Architecture

**Hands** (`app.hermes.hands`) is a sovereign, dual-backend Android automation service designed for autonomous AI agents. Unlike traditional ADB tethering or single-screen accessibility tools, Hands treats mobile automation as an independent, multi-tenant platform service with strict permission isolation, dual-display capability, and headless operation.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Remote / Host AI Agents                         │
│             (Hermes Gateway, Claude Desktop, Antigravity)              │
└────────────────────────────────────┬───────────────────────────────────┘
                                     │  Model Context Protocol (MCP)
                                     ▼
┌────────────────────────────────────────────────────────────────────────┐
│                           Hands MCP Server                             │
│                  (Python stdio or standalone HTTPS)                    │
└────────────────────────────────────┬───────────────────────────────────┘
                                     │  Mutual TLS / Bearer Token
                                     ▼
┌────────────────────────────────────────────────────────────────────────┐
│                        Hands Android Runtime                           │
│                      (Listening on port 8766)                          │
├────────────────────────────────────┬───────────────────────────────────┤
│          Safety & Gates            │          Session Control          │
│  - Arm-First Control Leases        │  - Target: Auto / Agent / Primary │
│  - Protected Package Denylist      │  - Adaptive Headless Switch       │
│  - Local Device Privacy Barrier    │  - Partial Wake Lock (Doze Proof) │
└─────────────────┬──────────────────┴─────────────────┬─────────────────┘
                  │                                    │
                  ▼                                    ▼
┌──────────────────────────────────┐ ┌───────────────────────────────────┐
│     Shizuku Engine (Privileged)  │ │   Accessibility Engine (L0/L1)   │
│ - Virtual Display Manager        │ │ - CompanionAccessibilityService   │
│ - Headless ImageReader (<50ms)   │ │ - UI Tree Node Snapshot           │
│ - Hardware Display Capture       │ │ - System Gesture Dispatch         │
│ - Multi-Display Input Injection  │ │ - CompanionNotificationListener   │
└─────────────────┬────────────────┘ └─────────────────┬─────────────────┘
                  │                                    │
                  ▼                                    ▼
┌──────────────────────────────────┐ ┌───────────────────────────────────┐
│       Display N (Agent Screen)   │ │      Display 0 (Primary Screen)   │
│ - Virtual Display (720x1560)     │ │ - Physical AMOLED Display         │
│ - Runs Headlessly when Screen Off│ │ - User Interface / Home Launcher  │
│ - Never Steals Physical Focus    │ │ - Optional Floating Live Preview  │
└──────────────────────────────────┘ └───────────────────────────────────┘
```

---

## 1. Dual-Display & Headless Virtual Screen Engine

Hands features a unique virtual display engine that enables AI agents to work on an Android phone **without taking over the user's physical screen** and **even when the device is locked or the physical screen is completely off**.

### The Two Displays
- **Display 0 (Primary Display)**: The physical smartphone screen. Used by the human operator.
- **Display $N$ (Agent Display)**: A secondary virtual display created programmatically via Android's `DisplayManager.createVirtualDisplay`. Target resolution defaults to $720 \times 1560$ at 320 DPI.

### Work Screen Selection (`target`)
Hands provides three work screen target modes:
1. `agent`: Remote actions strictly execute on the isolated Agent screen. The physical phone remains unaffected and can be locked, sleeping, or used by the operator simultaneously.
2. `primary`: Remote actions execute on the physical screen (Display 0), requiring screen-on and accessibility privileges.
3. `auto` (**Default & Recommended**): Hands dynamically routes actions based on physical phone state:
   - When the phone screen is **ON and unlocked**: Uses the primary screen.
   - When the phone screen is **LOCKED or OFF (`!isInteractive`)**: Automatically switches to the headless Agent screen.

### Headless `ImageReader` Composition
Traditional virtual display implementations attach a `TextureView` overlay to WindowManager on Display 0. However, when the physical display sleeps, Android halts composition on Display 0, causing overlay listeners to freeze.

Hands overcomes this by maintaining a headless `ImageReader` surface:
- Created directly on a background worker thread (`PixelFormat.RGBA_8888`, 2 buffers).
- Establishes a hardware-backed virtual display in **<50 milliseconds**.
- Renders and accepts input continuously while the phone's physical display remains completely dark.

---

## 2. Privileged Shizuku Engine

For high-speed, secure, and isolated execution, Hands integrates directly with [Shizuku](https://shizuku.rikka.app/):
- **UserService IPC via AIDL**: Hands defines [`IHandsService.aidl`](file:///Users/nyx/Projects/hermes-hands/hands-engine/src/main/aidl/app/hermes/hands/engine/IHandsService.aidl) running in Shizuku's privileged process context (system/shell UID 2000).
- **Direct Multi-Display Input**: Taps, swipes, key events, and text inputs are delivered directly to the target display using `/system/bin/input -d <displayId>`.
- **Multi-Display Activity Launching**: Apps are launched specifically onto the virtual display using `am start --display <displayId> -f 0x18000000` (`FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_MULTIPLE_TASK`), isolating the agent's tasks from the user's recents.
- **Hardware Screenshot Capture**: Screenshots are captured via `android.window.ScreenCapture` directly from the hardware buffer of the target display.

---

## 3. Fallback Accessibility Engine (L0 & L1)

When Shizuku is unavailable, Hands gracefully falls back to its built-in Accessibility Service:
- **L0 (Passive Lane)**: Read-only observation, notification streaming, telemetry, battery, volume, ringer, and device info.
- **L1 (Interactive Lane)**: UI node tree inspection (`SnapshotNode`), clickable element resolution, gesture dispatch, and app switching.
- **Secure Accessibility Toggle**: If the operator has granted `android.permission.WRITE_SECURE_SETTINGS` to Hands via ADB, Hands can automatically self-heal and re-enable its Accessibility Service without requiring manual navigation into Android Settings.

---

## 4. Security & Safety Model

Hands implements defense-in-depth safety boundaries to ensure agents remain strictly within authorized parameters:

### Arm-First Leases
- Control tools (`device.click`, `device.type`, `device.open_app`, etc.) cannot be called without an active control lease.
- The agent must acquire a lease via `device.arm(duration_seconds=N)`.
- If a lease expires, all input injection is revoked instantly.

### Protected Packages Denylist
- Hands protects sensitive packages from remote automation.
- `app.hermes.hands` is permanently protected so an agent cannot reconfigure Hands permissions or approve clients remotely.
- Operators can add custom packages (e.g. banking apps, password managers) via the Hands UI.

### Private Entry Mode (`DevicePrivacy`)
- When the user taps a private or password field on their phone, Hands immediately pauses all remote agent operations and masks sensitive text fields with `[private]`.
- Resumes only after the operator finishes private entry.

### Encrypted Client Tokens
- Authentication tokens use SHA-256 digests stored in Android `EncryptedSharedPreferences` backed by hardware Android Keystore (AES-256-GCM / AES-256-SIV).
- Revoking a client immediately invalidates all active sessions.

### Partial Wake Lock (Doze Prevention)
- `HandsHostService` acquires a `PARTIAL_WAKE_LOCK` while serving (`runtime.state.value.serving`).
- Prevents Android's power management and Doze mode from sleeping the HTTPS server socket (port 8766) while the physical screen is off.
