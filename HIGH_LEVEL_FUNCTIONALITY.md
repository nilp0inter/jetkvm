# JetKVM Client Feature Parity Analysis

This document compares the remote command functionality between the original Web UI and the Rust implementation.

**Legend:**
- ✅ Fully Implemented (Library + CLI)
- 🔶 Partially Implemented (Library only, no CLI)
- ❌ Not Implemented

---

## HID (Human Interface Device) Commands

### Keyboard

| Method | UI | Rust Library | CLI | Message | Status | Notes |
|--------|----|--------------|----|----|--------|-------|
| `keyboardReport` | ✅ | ✅ `rpc_keyboard_report()` | ✅ `keyboard-report` | `keyboardReport` | ✅ | Basic keyboard HID report |
| `getKeyboardLayout` | ✅ | ❌ | ❌ | `getKeyboardLayout` | ❌ | Get current keyboard layout |
| `setKeyboardLayout` | ✅ | ❌ | ❌ | `setKeyboardLayout` | ❌ | Set keyboard layout |
| `getKeyboardLedState` | ✅ | ❌ | ❌ | `getKeyboardLedState` | ❌ | Get LED state (Caps/Num Lock) |
| `getKeyDownState` | ✅ | ❌ | ❌ | `getKeyDownState` | ❌ | Get currently pressed keys |

**High-level keyboard helpers:**
| Function | UI | Rust Library | CLI | Message | Status |
|----------|----|--------------|----|----|--------|
| Send text (ASCII) | ✅ | ✅ `rpc_sendtext()` | ✅ `sendtext` | `keypressReport` | ✅ |
| Send text with layout | ✅ | ✅ `send_text_with_layout()` | ✅ `send-text-with-layout` | `keypressReport` | ✅ |
| Send Return/Enter | ✅ | ✅ `send_return()` | ✅ `send-return` | `keypressReport` | ✅ |
| Send Ctrl-C | ✅ | ✅ `send_ctrl_c()` | ✅ `send-ctrl-c` | `keyboardReport` | ✅ |
| Send Ctrl-V | ✅ | ✅ `send_ctrl_v()` | ✅ `send-ctrl-v` | `keyboardReport` | ✅ |
| Send Ctrl-X | ✅ | ✅ `send_ctrl_x()` | ✅ `send-ctrl-x` | `keyboardReport` | ✅ |
| Send Ctrl-A | ✅ | ✅ `send_ctrl_a()` | ✅ `send-ctrl-a` | `keyboardReport` | ✅ |
| Send Windows key | ✅ | ✅ `send_windows_key()` | ✅ `send-windows-key` | `keyboardReport` | ✅ |
| Key combinations | ✅ | ✅ `send_key_combinations()` | ❌ | `keyboardReport` | 🔶 |

### Mouse

| Method | UI | Rust Library | CLI | Message | Status | Notes |
|--------|----|--------------|----|----|--------|-------|
| `absMouseReport` | ✅ | ✅ `rpc_abs_mouse_report()` | ✅ `abs-mouse-report` | `absMouseReport` | ✅ | Absolute mouse positioning |
| `relMouseReport` | ✅ | ❌ | ❌ | `relMouseReport` | ❌ | Relative mouse movement |
| `wheelReport` | ✅ | ✅ `rpc_wheel_report()` | ✅ `wheel-report` | `wheelReport` | ✅ | Mouse wheel scrolling |

**High-level mouse helpers:**
| Function | UI | Rust Library | CLI | Message | Status |
|----------|----|--------------|----|----|--------|
| Move mouse | ✅ | ✅ `rpc_move_mouse()` | ✅ `move-mouse` | `absMouseReport` | ✅ |
| Left click | ✅ | ✅ `rpc_left_click()` | ✅ `left-click` | `absMouseReport` | ✅ |
| Right click | ✅ | ✅ `rpc_right_click()` | ✅ `right-click` | `absMouseReport` | ✅ |
| Middle click | ✅ | ✅ `rpc_middle_click()` | ✅ `middle-click` | `absMouseReport` | ✅ |
| Double click | ✅ | ✅ `rpc_double_click()` | ✅ `double-click` | `absMouseReport` | ✅ |
| Click and drag | ❌ | ✅ `rpc_left_click_and_drag_to_center()` | ❌ | `absMouseReport` | 🔶 |

### Mouse Jiggler

| Method | UI | Rust Library | CLI | Message | Status |
|--------|----|--------------|----|----|--------|
| `getJigglerState` | ✅ | ❌ | ❌ | `getJigglerState` | ❌ |
| `setJigglerState` | ✅ | ❌ | ❌ | `setJigglerState` | ❌ |
| `getJigglerConfig` | ✅ | ❌ | ❌ | `getJigglerConfig` | ❌ |
| `setJigglerConfig` | ✅ | ❌ | ❌ | `setJigglerConfig` | ❌ |

---

## Video Commands

| Method | UI | Rust Library | CLI | Message | Status | Notes |
|--------|----|--------------|----|----|--------|-------|
| `getVideoState` | ✅ | ❌ | ❌ | `getVideoState` | ❌ | Get video stream state |
| `getStreamQualityFactor` | ✅ | ❌ | ❌ | `getStreamQualityFactor` | ❌ | Get video quality factor |
| `getEDID` | ✅ | ✅ `rpc_get_edid()` | ✅ `get-edid` | `getEDID` | ✅ | Get EDID data |
| `setEDID` | ✅ | ✅ `rpc_set_edid()` | ✅ `set-edid` | `setEDID` | ✅ | Set EDID configuration |
| `getVideoLogStatus` | ✅ | ❌ | ❌ | `getVideoLogStatus` | ❌ | Get video logging status |
| Screenshot | ✅ | ✅ `VideoFrameCapture::capture_screenshot_png()` | ✅ `screenshot` | | ✅ | Capture PNG screenshot. Not implemented via JSON-RPC. |

---

## Storage / Virtual Media Commands

| Method | UI | Rust Library | CLI | Message | Status | Notes |
|--------|----|--------------|----|----|--------|-------|
| `getVirtualMediaState` | ✅ | ❌ | ❌ | `getVirtualMediaState` | ❌ | Get mounted media state |
| `mountWithHTTP` | ✅ | ❌ | ❌ | `mountWithHTTP` | ❌ | Mount image from HTTP URL |
| `mountWithStorage` | ✅ | ❌ | ❌ | `mountWithStorage` | ❌ | Mount image from storage |
| `unmountImage` | ✅ | ❌ | ❌ | `unmountImage` | ❌ | Unmount virtual media |
| `listStorageFiles` | ✅ | ❌ | ❌ | `listStorageFiles` | ❌ | List stored ISO/image files |
| `getStorageSpace` | ✅ | ❌ | ❌ | `getStorageSpace` | ❌ | Get available storage space |
| `deleteStorageFile` | ✅ | ❌ | ❌ | `deleteStorageFile` | ❌ | Delete file from storage |
| `startStorageFileUpload` | ✅ | ❌ | ❌ | `startStorageFileUpload` | ❌ | Upload file to storage |

---

## Network Commands

| Method | UI | Rust Library | CLI | Message | Status | Notes |
|--------|----|--------------|----|----|--------|-------|
| `getNetworkSettings` | ✅ | ❌ | ❌ | `getNetworkSettings` | ❌ | Get network configuration |
| `setNetworkSettings` | ✅ | ❌ | ❌ | `setNetworkSettings` | ❌ | Set network configuration |
| `getNetworkState` | ✅ | ❌ | ❌ | `getNetworkState` | ❌ | Get current network state |
| `renewDHCPLease` | ✅ | ❌ | ❌ | `renewDHCPLease` | ❌ | Renew DHCP lease |

### Wake-on-LAN

| Method | UI | Rust Library | CLI | Message | Status |
|--------|----|--------------|----|----|--------|
| `getWakeOnLanDevices` | ✅ | ❌ | ❌ | `getWakeOnLanDevices` | ❌ |
| `setWakeOnLanDevices` | ✅ | ❌ | ❌ | `setWakeOnLanDevices` | ❌ |
| `sendWOLMagicPacket` | ✅ | ❌ | ❌ | `sendWOLMagicPacket` | ❌ |

---

## Power Control Commands

### ATX Power Control

| Method | UI | Rust Library | CLI | Message | Status | Notes |
|--------|----|--------------|----|----|--------|-------|
| `getATXState` | ✅ | ❌ | ❌ | `getATXState` | ❌ | Get ATX power state |
| `setATXPowerAction` | ✅ | ❌ | ❌ | `setATXPowerAction` | ❌ | Power on/off/reset actions |

### DC Power Control

| Method | UI | Rust Library | CLI | Message | Status |
|--------|----|--------------|----|----|--------|
| `getDCPowerState` | ✅ | ❌ | ❌ | `getDCPowerState` | ❌ |
| `setDCPowerState` | ✅ | ❌ | ❌ | `setDCPowerState` | ❌ |
| `setDCRestoreState` | ✅ | ❌ | ❌ | `setDCRestoreState` | ❌ |

---

## USB Commands

| Method | UI | Rust Library | CLI | Message | Status | Notes |
|--------|----|--------------|----|----|--------|-------|
| `getUsbConfig` | ✅ | ❌ | ❌ | `getUsbConfig` | ❌ | Get USB device configuration |
| `setUsbConfig` | ✅ | ❌ | ❌ | `setUsbConfig` | ❌ | Set USB device configuration |
| `getUsbDevices` | ✅ | ❌ | ❌ | `getUsbDevices` | ❌ | List USB devices |
| `setUsbDevices` | ✅ | ❌ | ❌ | `setUsbDevices` | ❌ | Configure USB devices |
| `getUsbEmulationState` | ✅ | ❌ | ❌ | `getUsbEmulationState` | ❌ | Get USB emulation state |
| `setUsbEmulationState` | ✅ | ❌ | ❌ | `setUsbEmulationState` | ❌ | Set USB emulation state |

---

## System / Device Commands

| Method | UI | Rust Library | CLI | Message | Status | Notes |
|--------|----|--------------|----|----|--------|-------|
| `ping` | ✅ | ✅ `rpc_ping()` | ✅ `ping` | `ping` | ✅ | Basic connectivity test |
| `getDeviceID` | ✅ | ✅ `rpc_get_device_id()` | ✅ `get-device-id` | `getDeviceID` | ✅ | Get device identifier |
| `reboot` | ✅ | ❌ | ❌ | `reboot` | ❌ | Reboot the device |
| `getLocalVersion` | ✅ | ❌ | ❌ | `getLocalVersion` | ❌ | Get firmware version |
| `getUpdateStatus` | ✅ | ❌ | ❌ | `getUpdateStatus` | ❌ | Get firmware update status |
| `tryUpdate` | ✅ | ❌ | ❌ | `tryUpdate` | ❌ | Attempt firmware update |
| `getAutoUpdateState` | ✅ | ❌ | ❌ | `getAutoUpdateState` | ❌ | Get auto-update setting |
| `setAutoUpdateState` | ✅ | ❌ | ❌ | `setAutoUpdateState` | ❌ | Set auto-update setting |
| `getTimezones` | ✅ | ❌ | ❌ | `getTimezones` | ❌ | List available timezones |

---

## Hardware Settings Commands

| Method | UI | Rust Library | CLI | Message | Status | Notes |
|--------|----|--------------|----|----|--------|-------|
| `setDisplayRotation` | ✅ | ❌ | ❌ | `setDisplayRotation` | ❌ | Rotate display orientation |
| `getBacklightSettings` | ✅ | ❌ | ❌ | `getBacklightSettings` | ❌ | Get backlight configuration |
| `setBacklightSettings` | ✅ | ❌ | ❌ | `setBacklightSettings` | ❌ | Set backlight configuration |

---

## Cloud / Access Commands

| Method | UI | Rust Library | CLI | Message | Status | Notes |
|--------|----|--------------|----|----|--------|-------|
| `getCloudState` | ✅ | ❌ | ❌ | `getCloudState` | ❌ | Get cloud connection state |
| `setCloudUrl` | ✅ | ❌ | ❌ | `setCloudUrl` | ❌ | Set cloud URL |
| `getTLSState` | ✅ | ❌ | ❌ | `getTLSState` | ❌ | Get TLS/SSL state |
| `setTLSState` | ✅ | ❌ | ❌ | `setTLSState` | ❌ | Set TLS/SSL state |
| `deregisterDevice` | ✅ | ❌ | ❌ | `deregisterDevice` | ❌ | Deregister from cloud |

---

## Extension Commands

| Method | UI | Rust Library | CLI | Message | Status | Notes |
|--------|----|--------------|----|----|--------|-------|
| `getActiveExtension` | ✅ | ❌ | ❌ | `getActiveExtension` | ❌ | Get active extension ID |
| `setActiveExtension` | ✅ | ❌ | ❌ | `setActiveExtension` | ❌ | Set active extension |
| `getSerialSettings` | ✅ | ❌ | ❌ | `getSerialSettings` | ❌ | Get serial console settings |
| `setSerialSettings` | ✅ | ❌ | ❌ | `setSerialSettings` | ❌ | Set serial console settings |

---

## Advanced Settings Commands

| Method | UI | Rust Library | CLI | Message | Status | Notes |
|--------|----|--------------|----|----|--------|-------|
| `getDevModeState` | ✅ | ❌ | ❌ | `getDevModeState` | ❌ | Get developer mode state |
| `setDevModeState` | ✅ | ❌ | ❌ | `setDevModeState` | ❌ | Set developer mode state |
| `getSSHKeyState` | ✅ | ❌ | ❌ | `getSSHKeyState` | ❌ | Get SSH key configuration |
| `setSSHKeyState` | ✅ | ❌ | ❌ | `setSSHKeyState` | ❌ | Set SSH key |
| `getDevChannelState` | ✅ | ❌ | ❌ | `getDevChannelState` | ❌ | Get dev channel state |
| `setDevChannelState` | ✅ | ❌ | ❌ | `setDevChannelState` | ❌ | Set dev channel state |
| `getLocalLoopbackOnly` | ✅ | ❌ | ❌ | `getLocalLoopbackOnly` | ❌ | Get loopback-only setting |
| `setLocalLoopbackOnly` | ✅ | ❌ | ❌ | `setLocalLoopbackOnly` | ❌ | Set loopback-only setting |
| `getUsbEmulationState` | ✅ | ❌ | ❌ | `getUsbEmulationState` | ❌ | Get USB emulation state |
| `setUsbEmulationState` | ✅ | ❌ | ❌ | `setUsbEmulationState` | ❌ | Set USB emulation state |
| `resetConfig` | ✅ | ❌ | ❌ | `resetConfig` | ❌ | Reset to factory defaults |

---
