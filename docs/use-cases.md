# Hands Real-World Use Cases

Hands provides a production-grade automation substrate for modern agentic workflows. Here are key use cases unlocked by Hands' sovereign architecture:

---

## 1. Headless Background Assistant (Screen-Off & Locked Automation)

### Problem
Traditional mobile automation requires the phone's physical screen to be turned on and unlocked. If the user puts the phone in their pocket, locks the device, or goes to sleep, automation halts immediately or interrupts the user.

### Hands Solution
Using `target: auto` or `target: agent`:
- Hands detects when `!power.isInteractive` or the device is locked.
- Automatically establishes a headless virtual display (`Display N`) in <50ms backed by `ImageReader`.
- AI agents launch apps, complete workflows, order food, reply to messages, or check flight status headlessly without lighting up the phone or consuming unnecessary display battery.

---

## 2. Non-Intrusive Parallel Workspaces

### Problem
Human users cannot use their phone while an AI agent is tapping, swiping, or typing on the screen.

### Hands Solution
- The Agent Screen runs concurrently as an independent display.
- While the human operator is watching YouTube or chatting with friends on Display 0, the AI agent is running tasks on Display $N$ in parallel without stealing keyboard focus or overriding touches.

---

## 3. End-to-End Automated Mobile App Testing

### Problem
Testing Android apps on real hardware typically requires tethered USB cables, invasive instrumentation frameworks (Appium, Espresso), or flaky accessibility scripts.

### Hands Solution
- QA agents can interact with native Android applications via clean REST or Model Context Protocol (MCP) calls.
- High-fidelity UI snapshots (`device.snapshot`) deliver verified coordinate bounds, clickable states, roles, and text.
- Full screenshot verification (`device.screenshot`) captures true hardware display buffers for visual regression testing.

---

## 4. Autonomous Notification & Telephony Hub

### Problem
Agents frequently need situational awareness (e.g. 2FA codes, incoming message alerts, call status) without polling or storing intrusive credentials.

### Hands Solution
- Hands streams system notifications passively via `CompanionNotificationListener` over WebSocket/HTTP.
- Dedicated tools allow reading recent SMS, logging incoming calls, composing messages, and triggering direct telephony actions under strict user-controlled permission gates.

---

## 5. Local Security & Private Entry Isolation

### Problem
AI agents must never observe or store banking passwords, cryptographic keys, or biometric prompts.

### Hands Solution
- The `DevicePrivacy` engine detects when password fields or private credentials receive focus.
- Automatically replaces text with `[private]` and suspends remote command dispatch until the local operator completes entry on the physical device.
