# Messaging Protocol

This document outlines the communication protocols between the backend and the UI.

## WebRTC Signaling

WebRTC signaling is the process of coordinating communication between the backend and the UI. There are two signaling protocols in use: a legacy HTTP-based protocol and a modern WebSocket-based protocol.

### Legacy HTTP Signaling

The legacy signaling protocol uses a simple HTTP POST request to exchange session descriptions. This protocol is maintained for backward compatibility.

**Channel:** HTTP
**Direction:** UI to Backend
**Endpoint:** `/webrtc/session`

#### Request

*   **Format:** JSON
*   **Description:** The UI sends a POST request with a JSON body containing the base64-encoded session description (SDP).
*   **Backend Code:**
    *   **File:** `web.go`
    *   **Function:** `handleWebRTCSession` (`web.go:207`)
    *   **Request Struct:** `WebRTCSessionRequest` (`web.go:34`)
*   **UI Code:**
    *   **File:** `ui/src/routes/devices.$id.tsx`
    *   **Function:** `legacyHTTPSignaling` (`ui/src/routes/devices.$id.tsx:437`)

```json
{
  "sd": "<base64-encoded session description>"
}
```

#### Response

*   **Format:** JSON
*   **Description:** The backend responds with a JSON body containing the base64-encoded session description.
*   **Backend Code:**
    *   **File:** `web.go`
    *   **Function:** `handleWebRTCSession` (`web.go:207`)
*   **UI Code:**
    *   **File:** `ui/src/routes/devices.$id.tsx`
    *   **Function:** `legacyHTTPSignaling` (`ui/src/routes/devices.$id.tsx:437`)

```json
{
  "sd": "<base64-encoded session description>"
}
```

### Modern WebSocket Signaling

The modern signaling protocol uses a persistent WebSocket connection to exchange messages between the backend and the UI.

**Channel:** WebSocket
**Direction:** Bidirectional
**Endpoint:** `/webrtc/signaling/client`
**Backend Code:**
*   **File:** `web.go`
*   **Functions:** `handleLocalWebRTCSignal` (`web.go:273`), `handleWebRTCSignalWsMessages` (`web.go:323`)
*   **UI Code:**
*   **File:** `ui/src/routes/devices.$id.tsx`

#### WebSocket Messages

##### `device-metadata`

*   **Direction:** Backend to UI
*   **Format:** JSON
*   **Description:** After the WebSocket connection is established, the backend sends this message to provide the device version to the UI. The UI uses this information to determine whether to use the modern or legacy signaling protocol.
*   **Backend Code:**
    *   **File:** `web.go` (`web.go:315`)
*   **UI Code:**
    *   **File:** `ui/src/routes/devices.$id.tsx` (`ui/src/routes/devices.$id.tsx:361`)

```json
{
  "type": "device-metadata",
  "data": {
    "deviceVersion": "v1.2.3"
  }
}
```

##### `offer`

*   **Direction:** UI to Backend
*   **Format:** JSON
*   **Description:** The UI sends this message to initiate a WebRTC session. The `sd` field contains the base64-encoded session description.
*   **UI Code:**
    *   **File:** `ui/src/routes/devices.$id.tsx` (`ui/src/routes/devices.$id.tsx:556`)
*   **Backend Code:**
    *   **File:** `web.go` (`web.go:471`)

```json
{
  "type": "offer",
  "data": {
    "sd": "<base64-encoded session description>"
  }
}
```

##### `answer`

*   **Direction:** Backend to UI
*   **Format:** JSON
*   **Description:** The backend responds to an `offer` message with an `answer` message containing its own session description.
*   **Backend Code:**
    *   **File:** `webrtc.go` (`webrtc.go:128`)
*   **UI Code:**
    *   **File:** `ui/src/routes/devices.$id.tsx` (`ui/src/routes/devices.$id.tsx:380`)

```json
{
  "type": "answer",
  "data": "<base64-encoded session description>"
}
```

##### `new-ice-candidate`

*   **Direction:** Bidirectional
*   **Format:** JSON
*   **Description:** Both the backend and the UI send this message to exchange ICE candidates.
*   **Backend Code:**
    *   **File:** `webrtc.go` (`webrtc.go:414`)
*   **UI Code:**
    *   **File:** `ui/src/routes/devices.$id.tsx` (`ui/src/routes/devices.$id.tsx:567`)

```json
{
  "type": "new-ice-candidate",
  "data": {
    "candidate": "candidate:1234567890 1 udp 2122260223 192.168.1.100 12345 typ host generation 0 ufrag abcdefgh network-id 1",
    "sdpMLineIndex": 0,
    "sdpMid": "0"
  }
}
```

## WebRTC Data Channels

Once the WebRTC peer connection is established, several data channels are used for communication between the backend and the UI.

**Backend Code:**
*   **File:** `webrtc.go` (`webrtc.go:343`)
**UI Code:**
*   **File:** `ui/src/routes/devices.$id.tsx` (`ui/src/routes/devices.$id.tsx:589`)

### `rpc`

*   **Description:** This channel is used for JSON-RPC communication. The UI sends requests to the backend, and the backend sends responses and events.

### `hidrpc`

*   **Description:** This channel is used for HID (Human Interface Device) messages, such as keyboard and mouse reports.

### `hidrpc-unreliable-ordered`

*   **Description:** An unreliable, ordered channel for HID messages.

### `hidrpc-unreliable-nonordered`

*   **Description:** An unreliable, non-ordered channel for HID messages.

### `terminal`

*   **Description:** This channel is used for the KVM terminal.

### `serial`

*   **Description:** This channel is used for the serial console.

## JSON-RPC Messages

The `rpc` data channel is used for JSON-RPC communication.

**Backend Code:**
*   **File:** `jsonrpc.go` (`jsonrpc.go:120`)
**UI Code:**
*   **File:** `ui/src/hooks/useJsonRpc.ts`

### UI to Backend

| Method                 | Parameters                  | Description                                      | Backend Handler                 | UI Caller                           |
| ---------------------- | --------------------------- | ------------------------------------------------ | ------------------------------- | ----------------------------------- |
| `ping`                 | None                        | Pings the backend.                               | `rpcPing` (`jsonrpc.go:203`)      | `useJsonRpc` (`useJsonRpc.ts:8`)      |
| `reboot`               | `force` (boolean)           | Reboots the device.                              | `rpcReboot` (`jsonrpc.go:211`)    | `SettingsGeneralRebootRoute` (`devices.$id.settings.general.reboot.tsx:13`) |
| `getDeviceID`          | None                        | Gets the device ID.                              | `rpcGetDeviceID` (`jsonrpc.go:207`) | `UsbInfoSetting` (`UsbInfoSetting.tsx:139`) |
| `deregisterDevice`     | None                        | Deregisters the device from the cloud.           | `rpcDeregisterDevice` (`cloud.go:139`) | `SettingsAccessIndex` (`devices.$id.settings.access._index.tsx:92`) |
| `getCloudState`        | None                        | Gets the cloud connection state.                 | `rpcGetCloudState` (`cloud.go:135`) | `SettingsAccessIndex` (`devices.$id.settings.access._index.tsx:60`) |
| `getNetworkState`      | None                        | Gets the network state.                          | `rpcGetNetworkState` (`network.go:24`) | `SettingsNetwork` (`devices.$id.settings.network.tsx:120`) |
| `getNetworkSettings`   | None                        | Gets the network settings.                       | `rpcGetNetworkSettings` (`network.go:28`) | `SettingsNetwork` (`devices.$id.settings.network.tsx:106`) |
| `setNetworkSettings`   | `settings` (object)         | Sets the network settings.                       | `rpcSetNetworkSettings` (`network.go:32`) | `SettingsNetwork` (`devices.$id.settings.network.tsx:131`) |
| `renewDHCPLease`       | None                        | Renews the DHCP lease.                           | `rpcRenewDHCPLease` (`network.go:36`) | `SettingsNetwork` (`devices.$id.settings.network.tsx:153`) |
| `getKeyboardLedState`  | None                        | Gets the keyboard LED state.                     | `rpcGetKeyboardLedState` (`hidrpc.go:20`) | `KvmIdRoute` (`devices.$id.tsx:692`) |
| `getKeyDownState`      | None                        | Gets the keyboard key down state.                | `rpcGetKeysDownState` (`hidrpc.go:24`) | `KvmIdRoute` (`devices.$id.tsx:713`) |
| `keyboardReport`       | `modifier`, `keys`          | Sends a keyboard report.                         | `rpcKeyboardReport` (`hidrpc.go:28`) | `useKeyboard` (`useKeyboard.ts:87`) |
| `keypressReport`       | `key`, `press`              | Sends a keypress report.                         | `rpcKeypressReport` (`hidrpc.go:32`) | `useKeyboard` (`useKeyboard.ts:107`) |
| `absMouseReport`       | `x`, `y`, `buttons`         | Sends an absolute mouse report.                  | `rpcAbsMouseReport` (`hidrpc.go:36`) | `useMouse` (`useMouse.ts:70`) |
| `relMouseReport`       | `dx`, `dy`, `buttons`       | Sends a relative mouse report.                   | `rpcRelMouseReport` (`hidrpc.go:40`) | `useMouse` (`useMouse.ts:39`) |
| `wheelReport`          | `wheelY`                    | Sends a mouse wheel report.                      | `rpcWheelReport` (`hidrpc.go:44`) | `useMouse` (`useMouse.ts:151`) |
| `getVideoState`        | None                        | Gets the video state.                            | `rpcGetVideoState` (`video.go:20`) | `KvmIdRoute` (`devices.$id.tsx:676`) |
| `getUSBState`          | None                        | Gets the USB state.                              | `rpcGetUSBState` (`usb.go:20`) | `KvmIdRoute` (`devices.$id.tsx:734`) |
| `unmountImage`         | None                        | Unmounts the virtual media image.                | `rpcUnmountImage` (`usb_mass_storage.go:20`) | `MountPopover` (`MountPopover.tsx:38`) |
| `rpcMountBuiltInImage` | `filename` (string)         | Mounts a built-in virtual media image.           | `rpcMountBuiltInImage` (`usb_mass_storage.go:24`) | `MountRoute` (`devices.$id.mount.tsx:107`) |
| `setJigglerState`      | `enabled` (boolean)         | Sets the jiggler state.                          | `rpcSetJigglerState` (`jiggler.go:20`) | `SettingsMouse` (`devices.$id.settings.mouse.tsx:121`) |
| `getJigglerState`      | None                        | Gets the jiggler state.                          | `rpcGetJigglerState` (`jiggler.go:24`) | `SettingsMouse` (`devices.$id.settings.mouse.tsx:89`) |
| `setJigglerConfig`     | `jigglerConfig` (object)    | Sets the jiggler configuration.                  | `rpcSetJigglerConfig` (`jiggler.go:28`) | `SettingsMouse` (`devices.$id.settings.mouse.tsx:129`) |
| `getJigglerConfig`     | None                        | Gets the jiggler configuration.                  | `rpcGetJigglerConfig` (`jiggler.go:32`) | `SettingsMouse` (`devices.$id.settings.mouse.tsx:97`) |
| `getTimezones`         | None                        | Gets the available timezones.                    | `rpcGetTimezones` (`timesync.go:20`) | `JigglerSetting` (`JigglerSetting.tsx:37`) |
| `sendWOLMagicPacket`   | `macAddress` (string)       | Sends a Wake-on-LAN magic packet.                | `rpcSendWOLMagicPacket` (`wol.go:20`) | `WakeOnLanIndex` (`Index.tsx:34`) |
| `getStreamQualityFactor` | None                        | Gets the stream quality factor.                  | `rpcGetStreamQualityFactor` (`jsonrpc.go:223`) | `SettingsVideo` (`devices.$id.settings.video.tsx:67`) |
| `setStreamQualityFactor` | `factor` (number)           | Sets the stream quality factor.                  | `rpcSetStreamQualityFactor` (`jsonrpc.go:227`) | `SettingsVideo` (`devices.$id.settings.video.tsx:98`) |
| `getAutoUpdateState`   | None                        | Gets the auto-update state.                      | `rpcGetAutoUpdateState` (`jsonrpc.go:235`) | `SettingsGeneralIndex` (`devices.$id.settings.general._index.tsx:27`) |
| `setAutoUpdateState`   | `enabled` (boolean)         | Sets the auto-update state.                      | `rpcSetAutoUpdateState` (`jsonrpc.go:239`) | `SettingsGeneralIndex` (`devices.$id.settings.general._index.tsx:34`) |
| `getEDID`              | None                        | Gets the EDID.                                   | `rpcGetEDID` (`jsonrpc.go:247`) | `SettingsVideo` (`devices.$id.settings.video.tsx:72`) |
| `setEDID`              | `edid` (string)             | Sets the EDID.                                   | `rpcSetEDID` (`jsonrpc.go:251`) | `SettingsVideo` (`devices.$id.settings.video.tsx:119`) |
| `getVideoLogStatus`    | None                        | Gets the video log status.                       | `rpcGetVideoLogStatus` (`jsonrpc.go:263`) | `SettingsVideo` (`devices.$id.settings.video.tsx:138`) |
| `getVideoSleepMode`    | None                        | Gets the video sleep mode.                       | `rpcGetVideoSleepMode` (`video.go:28`) | `KvmIdRoute` (`devices.$id.tsx:734`) |
| `setVideoSleepMode`    | `duration` (number)         | Sets the video sleep mode.                       | `rpcSetVideoSleepMode` (`video.go:32`) | `KvmIdRoute` (`devices.$id.tsx:734`) |
| `getDevChannelState`   | None                        | Gets the dev channel state.                      | `rpcGetDevChannelState` (`jsonrpc.go:267`) | `SettingsAdvanced` (`devices.$id.settings.advanced.tsx:45`) |
| `setDevChannelState`   | `enabled` (boolean)         | Sets the dev channel state.                      | `rpcSetDevChannelState` (`jsonrpc.go:271`) | `SettingsAdvanced` (`devices.$id.settings.advanced.tsx:120`) |
| `getLocalVersion`      | None                        | Gets the local version.                          | `rpcGetLocalVersion` (`jsonrpc.go:283`) | `useVersion` (`useVersion.tsx:52`) |
| `getUpdateStatus`      | None                        | Gets the update status.                          | `rpcGetUpdateStatus` (`jsonrpc.go:275`) | `useVersion` (`useVersion.tsx:30`) |
| `tryUpdate`            | None                        | Tries to update the device.                      | `rpcTryUpdate` (`jsonrpc.go:291`) | `SettingsGeneralUpdate` (`devices.$id.settings.general.update.tsx:22`) |
| `getDevModeState`      | None                        | Gets the dev mode state.                         | `rpcGetDevModeState` (`jsonrpc.go:321`) | `SettingsAdvanced` (`devices.$id.settings.advanced.tsx:29`) |
| `setDevModeState`      | `enabled` (boolean)         | Sets the dev mode state.                         | `rpcSetDevModeState` (`jsonrpc.go:337`) | `SettingsAdvanced` (`devices.$id.settings.advanced.tsx:105`) |
| `getSSHKeyState`       | None                        | Gets the SSH key state.                          | `rpcGetSSHKeyState` (`jsonrpc.go:363`) | `SettingsAdvanced` (`devices.$id.settings.advanced.tsx:35`) |
| `setSSHKeyState`       | `sshKey` (string)           | Sets the SSH key state.                          | `rpcSetSSHKeyState` (`jsonrpc.go:371`) | `SettingsAdvanced` (`devices.$id.settings.advanced.tsx:92`) |
| `getTLSState`          | None                        | Gets the TLS state.                              | `rpcGetTLSState` (`jsonrpc.go:387`) | `SettingsAccessIndex` (`devices.$id.settings.access._index.tsx:81`) |
| `setTLSState`          | `state` (object)            | Sets the TLS state.                              | `rpcSetTLSState` (`jsonrpc.go:391`) | `SettingsAccessIndex` (`devices.$id.settings.access._index.tsx:160`) |
| `setMassStorageMode`   | `mode` (string)             | Sets the mass storage mode.                      | `rpcSetMassStorageMode` (`jsonrpc.go:556`) | `MountRoute` (`devices.$id.mount.tsx:88`) |
| `getMassStorageMode`   | None                        | Gets the mass storage mode.                      | `rpcGetMassStorageMode` (`jsonrpc.go:578`) | `MountRoute` (`devices.$id.mount.tsx:66`) |
| `isUpdatePending`      | None                        | Checks if an update is pending.                  | `rpcIsUpdatePending` (`jsonrpc.go:589`) | `KvmIdRoute` (`devices.$id.tsx:734`) |
| `getUsbEmulationState` | None                        | Gets the USB emulation state.                    | `rpcGetUsbEmulationState` (`jsonrpc.go:593`) | `SettingsAdvanced` (`devices.$id.settings.advanced.tsx:40`) |
| `setUsbEmulationState` | `enabled` (boolean)         | Sets the USB emulation state.                    | `rpcSetUsbEmulationState` (`jsonrpc.go:597`) | `SettingsAdvanced` (`devices.$id.settings.advanced.tsx:65`) |
| `getUsbConfig`         | None                        | Gets the USB configuration.                      | `rpcGetUsbConfig` (`jsonrpc.go:605`) | `UsbInfoSetting` (`UsbInfoSetting.tsx:96`) |
| `setUsbConfig`         | `usbConfig` (object)        | Sets the USB configuration.                      | `rpcSetUsbConfig` (`jsonrpc.go:610`) | `UsbInfoSetting` (`UsbInfoSetting.tsx:116`) |
| `checkMountUrl`        | `url` (string)              | Checks if a URL is mountable.                    | `rpcCheckMountUrl` (`usb_mass_storage.go:28`) | `MountRoute` (`devices.$id.mount.tsx:88`) |
| `getVirtualMediaState` | None                        | Gets the virtual media state.                    | `rpcGetVirtualMediaState` (`usb_mass_storage.go:32`) | `MountPopover` (`MountPopover.tsx:26`) |
| `getStorageSpace`      | None                        | Gets the storage space.                          | `rpcGetStorageSpace` (`usb_mass_storage.go:36`) | `MountRoute` (`devices.$id.mount.tsx:569`) |
| `mountWithHTTP`        | `url`, `mode`               | Mounts virtual media from a URL.                 | `rpcMountWithHTTP` (`usb_mass_storage.go:40`) | `MountRoute` (`devices.$id.mount.tsx:88`) |
| `mountWithStorage`     | `filename`, `mode`          | Mounts virtual media from storage.               | `rpcMountWithStorage` (`usb_mass_storage.go:44`) | `MountRoute` (`devices.$id.mount.tsx:107`) |
| `listStorageFiles`     | None                        | Lists files in storage.                          | `rpcListStorageFiles` (`usb_mass_storage.go:48`) | `MountRoute` (`devices.$id.mount.tsx:554`) |
| `deleteStorageFile`    | `filename` (string)         | Deletes a file from storage.                     | `rpcDeleteStorageFile` (`usb_mass_storage.go:52`) | `MountRoute` (`devices.$id.mount.tsx:598`) |
| `startStorageFileUpload` | `filename`, `size`          | Starts a file upload to storage.                 | `rpcStartStorageFileUpload` (`usb_mass_storage.go:56`) | `MountRoute` (`devices.$id.mount.tsx:1052`) |
| `getWakeOnLanDevices`  | None                        | Gets the Wake-on-LAN devices.                    | `rpcGetWakeOnLanDevices` (`jsonrpc.go:618`) | `WakeOnLanIndex` (`Index.tsx:53`) |
| `setWakeOnLanDevices`  | `params` (object)           | Sets the Wake-on-LAN devices.                    | `rpcSetWakeOnLanDevices` (`jsonrpc.go:626`) | `WakeOnLanIndex` (`Index.tsx:71`) |
| `resetConfig`          | None                        | Resets the device configuration.                 | `rpcResetConfig` (`jsonrpc.go:634`) | `SettingsAdvanced` (`devices.$id.settings.advanced.tsx:80`) |
| `setDisplayRotation`   | `params` (object)           | Sets the display rotation.                       | `rpcSetDisplayRotation` (`jsonrpc.go:395`) | `SettingsHardware` (`devices.$id.settings.hardware.tsx:25`) |
| `getDisplayRotation`   | None                        | Gets the display rotation.                       | `rpcGetDisplayRotation` (`jsonrpc.go:413`) | `SettingsHardware` (`devices.$id.settings.hardware.tsx:62`) |
| `setBacklightSettings` | `params` (object)           | Sets the backlight settings.                     | `rpcSetBacklightSettings` (`jsonrpc.go:419`) | `SettingsHardware` (`devices.$id.settings.hardware.tsx:50`) |
| `getBacklightSettings` | None                        | Gets the backlight settings.                     | `rpcGetBacklightSettings` (`jsonrpc.go:445`) | `SettingsHardware` (`devices.$id.settings.hardware.tsx:62`) |
| `getDCPowerState`      | None                        | Gets the DC power state.                         | `rpcGetDCPowerState` (`jsonrpc.go:653`) | `DCPowerControl` (`DCPowerControl.tsx:26`) |
| `setDCPowerState`      | `enabled` (boolean)         | Sets the DC power state.                         | `rpcSetDCPowerState` (`jsonrpc.go:657`) | `DCPowerControl` (`DCPowerControl.tsx:38`) |
| `setDCRestoreState`    | `state` (number)            | Sets the DC restore state.                       | `rpcSetDCRestoreState` (`jsonrpc.go:665`) | `DCPowerControl` (`DCPowerControl.tsx:50`) |
| `getActiveExtension`   | None                        | Gets the active extension.                       | `rpcGetActiveExtension` (`jsonrpc.go:673`) | `ExtensionPopover` (`ExtensionPopover.tsx:47`) |
| `setActiveExtension`   | `extensionId` (string)      | Sets the active extension.                       | `rpcSetActiveExtension` (`jsonrpc.go:677`) | `ExtensionPopover` (`ExtensionPopover.tsx:60`) |
| `getATXState`          | None                        | Gets the ATX state.                              | `rpcGetATXState` (`jsonrpc.go:704`) | `ATXPowerControl` (`ATXPowerControl.tsx:34`) |
| `setATXPowerAction`    | `action` (string)           | Sets the ATX power action.                       | `rpcSetATXPowerAction` (`jsonrpc.go:692`) | `ATXPowerControl` (`ATXPowerControl.tsx:57`) |
| `getSerialSettings`    | None                        | Gets the serial settings.                        | `rpcGetSerialSettings` (`jsonrpc.go:712`) | `SerialConsole` (`SerialConsole.tsx:29`) |
| `setSerialSettings`    | `settings` (object)         | Sets the serial settings.                        | `rpcSetSerialSettings` (`jsonrpc.go:742`) | `SerialConsole` (`SerialConsole.tsx:42`) |
| `getUsbDevices`        | None                        | Gets the USB devices.                            | `rpcGetUsbDevices` (`jsonrpc.go:782`) | `UsbDeviceSetting` (`UsbDeviceSetting.tsx:71`) |
| `setUsbDevices`        | `devices` (object)          | Sets the USB devices.                            | `rpcSetUsbDevices` (`jsonrpc.go:794`) | `UsbDeviceSetting` (`UsbDeviceSetting.tsx:101`) |
| `setUsbDeviceState`    | `device`, `enabled`         | Sets the state of a USB device.                  | `rpcSetUsbDeviceState` (`jsonrpc.go:800`) | `KvmIdRoute` (`devices.$id.tsx:734`) |
| `setCloudUrl`          | `apiUrl`, `appUrl`          | Sets the cloud URL.                              | `rpcSetCloudUrl` (`jsonrpc.go:816`) | `SettingsAccessIndex` (`devices.$id.settings.access._index.tsx:114`) |
| `getKeyboardLayout`    | None                        | Gets the keyboard layout.                        | `rpcGetKeyboardLayout` (`jsonrpc.go:828`) | `SettingsKeyboard` (`devices.$id.settings.keyboard.tsx:20`) |
| `setKeyboardLayout`    | `layout` (string)           | Sets the keyboard layout.                        | `rpcSetKeyboardLayout` (`jsonrpc.go:832`) | `SettingsKeyboard` (`devices.$id.settings.keyboard.tsx:33`) |
| `getKeyboardMacros`    | None                        | Gets the keyboard macros.                        | `getKeyboardMacros` (`jsonrpc.go:840`) | `KvmIdRoute` (`devices.$id.tsx:734`) |
| `setKeyboardMacros`    | `params` (object)           | Sets the keyboard macros.                        | `setKeyboardMacros` (`jsonrpc.go:847`) | `KvmIdRoute` (`devices.$id.tsx:734`) |
| `getLocalLoopbackOnly` | None                        | Gets the local loopback only setting.            | `rpcGetLocalLoopbackOnly` (`jsonrpc.go:924`) | `SettingsAdvanced` (`devices.$id.settings.advanced.tsx:50`) |
| `setLocalLoopbackOnly` | `enabled` (boolean)         | Sets the local loopback only setting.            | `rpcSetLocalLoopbackOnly` (`jsonrpc.go:928`) | `SettingsAdvanced` (`devices.$id.settings.advanced.tsx:135`) |

### Backend to UI

| Method                | Parameters        | Description                                      | Backend Emitter              | UI Listener                       |
| --------------------- | ----------------- | ------------------------------------------------ | ---------------------------- | --------------------------------- |
| `otherSessionConnected` | None              | Notifies the UI that another session connected.  | `handleWebRTCSession` (`web.go:219`) | `KvmIdRoute` (`devices.$id.tsx:734`) |
| `usbState`            | `USBStates`       | Notifies the UI of a USB state change.           | `triggerUSBStateUpdate` (`usb.go:28`) | `KvmIdRoute` (`devices.$id.tsx:734`) |
| `videoInputState`     | `hdmiState`       | Notifies the UI of a video input state change.   | `triggerVideoStateUpdate` (`video.go:24`) | `KvmIdRoute` (`devices.$id.tsx:734`) |
| `networkState`        | `NetworkState`    | Notifies the UI of a network state change.       | `triggerNetworkStateUpdate` (`network.go:40`) | `KvmIdRoute` (`devices.$id.tsx:734`) |
| `keyboardLedState`    | `KeyboardLedState`| Notifies the UI of a keyboard LED state change.  | `reportHidRPCKeyboardLedState` (`hidrpc.go:193`) | `KvmIdRoute` (`devices.$id.tsx:734`) |
| `keysDownState`       | `KeysDownState`   | Notifies the UI of a key down state change.      | `reportHidRPCKeysDownState` (`hidrpc.go:201`) | `KvmIdRoute` (`devices.$id.tsx:734`) |
| `otaState`            | `OtaState`        | Notifies the UI of an OTA update state change.   | `triggerOTAStateUpdate` (`ota.go:20`) | `KvmIdRoute` (`devices.$id.tsx:734`) |

## HID-RPC Messages

The `hidrpc` data channel is used for HID communication.

**Backend Code:**
*   **File:** `hidrpc.go` (`hidrpc.go:13`)

### UI to Backend

| Type                        | Description                                      | Backend Handler                 | UI Caller                           |
| --------------------------- | ------------------------------------------------ | ------------------------------- | ----------------------------------- |
| `Handshake`                 | Initiates the HID-RPC handshake.                 | `handleHidRPCMessage` (`hidrpc.go:16`) | `useHidRpc` (`useHidRpc.ts:94`) |
| `KeypressReport`            | Sends a keypress report.                         | `handleHidRPCKeyboardInput` (`hidrpc.go:138`) | `useHidRpc` (`useHidRpc.ts:94`) |
| `KeyboardReport`            | Sends a keyboard report.                         | `handleHidRPCKeyboardInput` (`hidrpc.go:138`) | `useHidRpc` (`useHidRpc.ts:94`) |
| `KeyboardMacroReport`       | Sends a keyboard macro.                          | `handleHidRPCMessage` (`hidrpc.go:24`) | `useHidRpc` (`useHidRpc.ts:94`) |
| `CancelKeyboardMacroReport` | Cancels a keyboard macro.                        | `handleHidRPCMessage` (`hidrpc.go:30`) | `useHidRpc` (`useHidRpc.ts:94`) |
| `KeypressKeepAliveReport`   | Sends a keypress keep-alive.                     | `handleHidRPCKeypressKeepAlive` (`hidrpc.go:102`) | `useHidRpc` (`useHidRpc.ts:94`) |
| `PointerReport`             | Sends an absolute mouse report.                  | `handleHidRPCMessage` (`hidrpc.go:33`) | `useHidRpc` (`useHidRpc.ts:87`) |
| `MouseReport`               | Sends a relative mouse report.                   | `handleHidRPCMessage` (`hidrpc.go:40`) | `useHidRpc` (`useHidRpc.ts:89`) |

### Backend to UI

| Type                 | Description                                      | Backend Emitter                  | UI Listener                       |
| -------------------- | ------------------------------------------------ | -------------------------------- | --------------------------------- |
| `Handshake`          | Responds to the HID-RPC handshake.               | `handleHidRPCMessage` (`hidrpc.go:16`) | `useHidRpc` (`useHidRpc.ts:94`) |
| `KeyboardLedMessage` | Notifies the UI of a keyboard LED state change.  | `reportHidRPCKeyboardLedState` (`hidrpc.go:193`) | `useHidRpc` (`useHidRpc.ts:94`) |
| `KeydownStateMessage`| Notifies the UI of a key down state change.    | `reportHidRPCKeysDownState` (`hidrpc.go:201`) | `useHidRpc` (`useHidRpc.ts:94`) |
