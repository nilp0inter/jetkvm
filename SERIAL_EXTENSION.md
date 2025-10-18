# Serial Extension Protocol

This document outlines the protocol for transmitting serial extension data between the backend and the UI.

## Overview

The serial extension protocol uses a combination of a raw data channel and JSON-RPC messages over a separate data channel to transmit data between the backend and the UI.

## Data Channels

Two WebRTC data channels are utilized:

-   `serial`: A raw data channel for transmitting serial data to and from the serial console.
-   `rpc`: A data channel for sending and receiving JSON-RPC messages.

## Raw `serial` Data Channel

The `serial` data channel is used for bidirectional communication with the serial console.

### Backend

-   **Handler:** `handleSerialChannel` in `serial.go:315`
-   **Description:** This function handles the lifecycle of the `serial` data channel.
    -   On `open`, it starts a goroutine that reads from the serial port and sends the data over the data channel.
    -   On `message`, it writes the received data to the serial port.

### UI

-   **Initializer:** The `KvmIdRoute` component in `ui/src/routes/devices.$id.tsx:885` creates the `serial` data channel and passes it to the `Terminal` component.
-   **Consumer:** The `Terminal` component (defined in `ui/src/components/Terminal.tsx`) uses this data channel to display serial output to the user and send user input back to the device.

## JSON-RPC `rpc` Data Channel

### Backend-to-UI Communication

#### ATX Power Control

-   **Event:** `atxState`
-   **Generator:** The `runATXControl` function in `serial.go:70` sends the `atxState` event using `writeJSONRPCEvent`.
-   **Consumer:** The `useJsonRpc` hook in `ui/src/components/extensions/ATXPowerControl.tsx:26` receives the `atxState` event.
-   **Payload Structure:**
    ```json
    {
      "power": <boolean>,
      "hdd": <boolean>
    }
    ```

#### DC Power Control

-   **Method:** `getDCPowerState`
-   **Description:** The UI polls the backend for the current DC power state. The backend does not send unsolicited `dcState` events; instead, it continuously updates an internal state that is returned when the UI calls the `getDCPowerState` RPC method.
-   **Caller:** The `DCPowerControl` component in `ui/src/components/extensions/DCPowerControl.tsx:34` polls for the `dcState` by calling the `getDCPowerState` method.
-   **Handler:** The `rpcGetDCPowerState` function in `jsonrpc.go:740`.
-   **Payload Structure:**
    ```json
    {
      "isOn": <boolean>,
      "voltage": <number>,
      "current": <number>,
      "power": <number>,
      "restoreState": <number>
    }
    ```

### UI-to-Backend Communication

#### ATX Power Control

-   **Method:** `setATXPowerAction`
-   **Caller:** The `ATXPowerControl` component in `ui/src/components/extensions/ATXPowerControl.tsx` calls this method when the user interacts with the power or reset buttons.
-   **Handler:** The `rpcSetATXPowerAction` function in `jsonrpc.go:796`.
-   **Payload Structure:**
    ```json
    {
      "action": <"power-short" | "power-long" | "reset">
    }
    ```

#### DC Power Control

-   **Method:** `setDCPowerState`
-   **Caller:** The `DCPowerControl` component in `ui/src/components/extensions/DCPowerControl.tsx:48` calls this method when the user toggles the power.
-   **Handler:** The `rpcSetDCPowerState` function in `jsonrpc.go:744`.
-   **Payload Structure:**
    ```json
    {
      "enabled": <boolean>
    }
    ```

-   **Method:** `setDCRestoreState`
-   **Caller:** The `DCPowerControl` component in `ui/src/components/extensions/DCPowerControl.tsx:59` calls this method when the user changes the restore state.
-   **Handler:** The `rpcSetDCRestoreState` function in `jsonrpc.go:753`.
-   **Payload Structure:**
    ```json
    {
      "state": <number>
    }
    ```

#### Serial Console

-   **Method:** `getSerialSettings`
-   **Caller:** The `SerialConsole` component in `ui/src/components/extensions/SerialConsole.tsx:23` calls this method on component mount to get the initial settings.
-   **Handler:** The `rpcGetSerialSettings` function in `jsonrpc.go:817`.
-   **Response Payload Structure:**
    ```json
    {
      "baudRate": <string>,
      "dataBits": <string>,
      "stopBits": <string>,
      "parity":   <string>
    }
    ```

-   **Method:** `setSerialSettings`
-   **Caller:** The `SerialConsole` component in `ui/src/components/extensions/SerialConsole.tsx:42` calls this method when the user changes a setting.
-   **Handler:** The `rpcSetSerialSettings` function in `jsonrpc.go:858`.
-   **Request Payload Structure:**
    ```json
    {
      "settings": {
        "baudRate": <string>,
        "dataBits": <string>,
        "stopBits": <string>,
        "parity":   <string>
      }
    }
    ```
