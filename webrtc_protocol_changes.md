# WebRTC Protocol Changes

This document outlines the key differences between the legacy and current WebRTC protocols used in this application.

## Legacy WebRTC Protocol

The legacy WebRTC protocol was characterized by a simple, HTTP-based signaling mechanism.

### Signaling

The signaling process for the legacy protocol relied on a single HTTP POST request to the `/webrtc/session` endpoint. The client would create an SDP offer, encode it, and send it in the body of the POST request. The server would then process the offer, generate an SDP answer, and return it in the HTTP response. This synchronous exchange was simple to implement but lacked the robustness of a persistent connection.

### Message Formats

**Client to Server (POST `/webrtc/session`)**

The client sends a JSON object containing the base64-encoded SDP offer and the device ID.

```json
{
  "sd": "...", // base64-encoded SDP offer
  "id": "..."  // device ID
}
```

**Server to Client (HTTP Response)**

The server responds with a JSON object containing the base64-encoded SDP answer.

```json
{
  "sd": "..." // base64-encoded SDP answer
}
```

### Data Channels

Once the peer connection was established, the following data channels were created for communication:

- **`rpc`**: A reliable, ordered data channel used for JSON RPC communication. This channel is the primary means of communication between the client and the device, used for sending commands and receiving state updates.
- **`hidrpc`**: A reliable, ordered data channel used for HID (Human Interface Device) reports, such as keyboard and mouse events.
- **`hidrpc-unreliable-ordered`**: An unreliable, ordered data channel for HID reports that can tolerate some packet loss but require ordering.
- **`hidrpc-unreliable-nonordered`**: An unreliable, unordered data channel for HID reports where neither ordering nor reliability is critical.

## New WebRTC Protocol

The new WebRTC protocol modernizes the signaling process by using a persistent WebSocket connection, which provides a more robust and efficient communication channel.

### Signaling

The signaling process for the new protocol is handled over a WebSocket connection to the `/webrtc/signaling/client` endpoint. This allows for a more interactive and real-time exchange of signaling messages, including the SDP offer/answer and ICE candidates. The persistent nature of the WebSocket connection eliminates the need for repeated HTTP requests and provides a more stable foundation for the WebRTC session.

### Message Formats

The WebSocket messages are JSON objects with a `type` and `data` field.

**Server to Client (`device-metadata`)**

After the WebSocket connection is established, the server sends a `device-metadata` message to the client, which is used to determine if the device supports the new signaling protocol.

```json
{
  "type": "device-metadata",
  "data": {
    "deviceVersion": "..."
  }
}
```

**Client to Server (`offer`)**

The client sends the SDP offer to the server.

```json
{
  "type": "offer",
  "data": {
    "sd": "..." // base64-encoded SDP offer
  }
}
```

**Server to Client (`answer`)**

The server responds with the SDP answer.

```json
{
  "type": "answer",
  "data": "..." // base64-encoded SDP answer
}
```

**Client and Server (`new-ice-candidate`)**

Both the client and server exchange ICE candidates as they are discovered.

```json
{
  "type": "new-ice-candidate",
  "data": {
    // ICE candidate object
  }
}
```

### Data Channels

The data channels created in the new protocol are identical to those in the legacy protocol:

- **`rpc`**
- **`hidrpc`**
- **`hidrpc-unreliable-ordered`**
- **`hidrpc-unreliable-nonordered`**

The core communication logic over these channels, including the JSON RPC protocol on the `rpc` channel, remains unchanged.

## Key Differences

The primary difference between the legacy and new WebRTC protocols is the **signaling mechanism**:

- **Legacy**: Uses a single, synchronous HTTP POST request for the entire SDP offer/answer exchange.
- **New**: Uses a persistent WebSocket connection for a more robust, real-time signaling process.

While the signaling process has been modernized, the **data channels and the JSON RPC communication protocol remain consistent** between the two versions. The evolution of the protocol was focused on improving the reliability and efficiency of the connection establishment process, not on changing the core communication logic.
