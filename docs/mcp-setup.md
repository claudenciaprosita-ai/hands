# Hands MCP Setup Guide

Hands comes with a first-class **Model Context Protocol (MCP)** server that exposes all 86 mobile tools to any MCP-compliant AI assistant, such as **Claude Desktop**, **Antigravity**, **Cursor**, or **Hermes Agent**.

---

## 1. Prerequisites

- Python 3.10+
- Network connectivity to the phone running Hands (Local Wi-Fi, WireGuard, or Tailscale)
- Hands Client Token (obtained from the Hands phone app UI under **Clients** tab)

---

## 2. Server Configuration

The MCP server entry point is located in `hands-mcp/hands_mcp.py`.

### Configuration Options
- `--endpoint`: The Hands HTTPS endpoint (e.g. `https://100.105.213.54:8766`).
- `--token`: The Bearer token issued by Hands.
- `--token-file`: Path to a file containing the token (e.g. `~/.config/hands/token`).
- `--server-fingerprint`: Optional SHA-256 certificate pin for strict TLS verification.
- `--insecure`: Allow self-signed TLS certificates without fingerprint verification.

---

## 3. Integrating with MCP Clients

### Claude Desktop
Add the following to your `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "hands": {
      "command": "python3",
      "args": [
        "/path/to/hands/hands-mcp/hands_mcp.py",
        "--endpoint", "https://100.105.213.54:8766",
        "--token-file", "/Users/yourname/.config/hands/token",
        "--insecure"
      ]
    }
  }
}
```

### Hermes Agent Gateway
Add to your `~/.hermes/config.yaml`:

```yaml
mcp_servers:
  hands:
    command: python3
    args:
      - /path/to/hands/hands-mcp/hands_mcp.py
      - --endpoint
      - https://100.105.213.54:8766
      - --token-file
      - ~/.config/hands/token
      - --insecure
```

### Direct CLI / Stdio Testing
You can run the MCP server interactively or test device smoke directly:

```bash
# Run standalone device smoke test
python3 hands-mcp/tests/device_smoke.py \
  --endpoint https://100.105.213.54:8766 \
  --token-file ~/.config/hands/token \
  --insecure
```

---

## 4. Available MCP Tools

The Hands MCP catalog exports **86 tools**:
- **Lifecycle**: `mobile_arm`, `mobile_disarm`, `mobile_status`
- **Vision & Navigation**: `mobile_snapshot`, `mobile_screenshot`, `mobile_open_app`, `mobile_apps`
- **Interaction**: `mobile_click`, `mobile_swipe`, `mobile_scroll`, `mobile_type`, `mobile_press`
- **Privileged Controls**: `mobile_shizuku_connect`, `mobile_shell_exec`, `mobile_power_wake`, `mobile_keyguard_dismiss`
- **Communication & PIM**: `mobile_sms_list`, `mobile_sms_send`, `mobile_calls_log`, `mobile_calls_dial`, `mobile_contacts_search`, `mobile_calendar_list`, `mobile_calendar_add`
- **Device Management**: `mobile_battery`, `mobile_thermal`, `mobile_volume_set`, `mobile_ringer`, `mobile_brightness_set`, `mobile_dnd_set`
- **Notifications**: `mobile_notifications_list`, `mobile_notifications_stream`, `mobile_notifications_reply`, `mobile_notifications_dismiss`
