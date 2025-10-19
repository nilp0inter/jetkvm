# JetKVM Local vs. Cloud Authentication

This document provides a technical analysis of the differences between local and cloud authentication in the JetKVM application.

## High-Level Overview

JetKVM supports two primary authentication methods:

*   **Local Authentication:** This method is designed for devices operating in a trusted local network. It relies on a simple password-based system, where the user sets a password that is stored locally on the device. When a user attempts to log in, the client sends the password to the device, which verifies it and grants access.

*   **Cloud Authentication:** This method is used when the device is connected to the JetKVM cloud service. It leverages OpenID Connect (OIDC) with Google as the identity provider, providing a more secure and robust authentication flow. When a user logs in through the cloud, they are redirected to Google to authenticate, and upon success, a token is issued to the client. This token is then used to register the device with the cloud service and establish a secure communication channel.

## Local Authentication in Detail

Local authentication is managed by the backend, which exposes a set of HTTP endpoints for handling the login process. The core logic for this authentication method resides in the `web.go` file.

### Endpoints

*   `POST /auth/login-local`: This endpoint is used to authenticate the user. It accepts a JSON payload with a `password` field and, upon successful authentication, issues a session cookie.
*   `POST /device/setup`: This endpoint is used to configure the initial authentication settings for the device. It allows setting the `localAuthMode` to either `password` or `noPassword`.

### Authentication Flow

1.  **Login Request:** The client sends a `POST` request to the `/auth/login-local` endpoint with the user's password.
2.  **Password Verification:** The backend compares the provided password with the hashed password stored in the device's configuration.
3.  **Session Cookie:** If the password is correct, the backend generates a unique `authToken` and sets it as a cookie in the client's browser.
4.  **Authenticated Session:** The client includes this cookie in all subsequent requests to protected endpoints, allowing the backend to verify the user's session.

### Code Snippets

The following snippet from `web.go` shows the implementation of the `handleLogin` function, which is responsible for verifying the user's password and issuing a session cookie:

```go
func handleLogin(c *gin.Context) {
	if config.LocalAuthMode == "noPassword" {
		c.JSON(http.StatusBadRequest, gin.H{"error": "Login is disabled in noPassword mode"})
		return
	}

	var req LoginRequest

	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	err := bcrypt.CompareHashAndPassword([]byte(config.HashedPassword), []byte(req.Password))
	if err != nil {
		c.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid password"})
		return
	}

	config.LocalAuthToken = uuid.New().String()

	// Set the cookie
	c.SetCookie("authToken", config.LocalAuthToken, 7*24*60*60, "/", "", false, true)

	c.JSON(http.StatusOK, gin.H{"message": "Login successful"})
}
```

The `protectedMiddleware` function in `web.go` ensures that all protected endpoints require a valid session cookie:

```go
func protectedMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		if config.LocalAuthMode == "noPassword" {
			c.Next()
			return
		}

		authToken, err := c.Cookie("authToken")
		if err != nil || authToken != config.LocalAuthToken || authToken == "" {
			c.JSON(http.StatusUnauthorized, gin.H{"error": "Unauthorized"})
			c.Abort()
			return
		}

		c.Next()
	}
}
```

## Cloud Onboarding and Device Authentication

The cloud authentication process involves two distinct phases: **user authentication** with the main cloud service and **device registration**. This document primarily focuses on the device registration aspect, as the user authentication is handled by the cloud service itself.

### User Journey

1.  **User Logs In:** The user first logs into the JetKVM cloud web application. This process is not detailed here, as it is part of the separate cloud service.
2.  **Initiates Device Onboarding:** From the cloud application, the user initiates the process of adding a new device.
3.  **Redirected to Device:** The user is then redirected to the local IP address of their JetKVM device. This is where the device registration flow, detailed below, begins.

The core logic for the device registration flow is in `cloud.go`.

### Endpoints

*   `POST /cloud/register`: This endpoint is used to register the device with the cloud service. It accepts an OIDC token from the client and exchanges it for a permanent `authToken`.

### Authentication Flow

1.  **OIDC Authentication:** The client initiates an OIDC flow with Google. The user is redirected to Google to authenticate and grant permissions.
2.  **Token Issuance:** Upon successful authentication, Google issues an OIDC token to the client.
3.  **Device Registration:** The client sends this token to the `/cloud/register` endpoint on the device.
4.  **Token Exchange:** The device backend exchanges the temporary OIDC token for a permanent `secretToken` with the cloud service.
5.  **Secure Connection:** This `secretToken` is then used to establish a secure WebSocket connection with the cloud service for ongoing communication.

### Code Snippets

The `handleCloudRegister` function in `cloud.go` manages the device registration process:

```go
func handleCloudRegister(c *gin.Context) {
	var req CloudRegisterRequest

	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(400, gin.H{"error": "Invalid request body"})
		return
	}

	// Exchange the temporary token for a permanent auth token
	payload := struct {
		TempToken string `json:"tempToken"`
	}{
		TempToken: req.Token,
	}
	jsonPayload, err := json.Marshal(payload)
	if err != nil {
		c.JSON(500, gin.H{"error": "Failed to encode JSON payload: " + err.Error()})
		return
	}

	client := &http.Client{Timeout: CloudAPIRequestTimeout}

	apiReq, err := http.NewRequest(http.MethodPost, config.CloudURL+"/devices/token", bytes.NewBuffer(jsonPayload))
	if err != nil {
		c.JSON(500, gin.H{"error": "Failed to create register request: " + err.Error()})
		return
	}
	apiReq.Header.Set("Content-Type", "application/json")

	apiResp, err := client.Do(apiReq)
	if err != nil {
		c.JSON(500, gin.H{"error": "Failed to exchange token: " + err.Error()})
		return
	}
	defer apiResp.Body.Close()

	if apiResp.StatusCode != http.StatusOK {
		c.JSON(apiResp.StatusCode, gin.H{"error": "Failed to exchange token: " + apiResp.Status})
		return
	}

	var tokenResp struct {
		SecretToken string `json:"secretToken"`
	}
	if err := json.NewDecoder(apiResp.Body).Decode(&tokenResp); err != nil {
		c.JSON(500, gin.H{"error": "Failed to parse token response: " + err.Error()})
		return
	}

	if tokenResp.SecretToken == "" {
		c.JSON(500, gin.H{"error": "Received empty secret token"})
		return
	}

	config.CloudToken = tokenResp.SecretToken

	provider, err := oidc.NewProvider(c, "https://accounts.google.com")
	if err != nil {
		c.JSON(500, gin.H{"error": "Failed to initialize OIDC provider: " + err.Error()})
		return
	}

	oidcConfig := &oidc.Config{
		ClientID: req.ClientId,
	}

	verifier := provider.Verifier(oidcConfig)
	idToken, err := verifier.Verify(c, req.OidcGoogle)
	if err != nil {
		c.JSON(400, gin.H{"error": "Invalid OIDC token: " + err.Error()})
		return
	}

	config.GoogleIdentity = idToken.Audience[0] + ":" + idToken.Subject

	// Save the updated configuration
	if err := SaveConfig(); err != nil {
		c.JSON(500, gin.H{"error": "Failed to save configuration"})
		return
	}

	c.JSON(200, gin.H{"message": "Cloud registration successful"})
}
```

The `runWebsocketClient` function in `cloud.go` establishes the secure WebSocket connection using the `cloudToken`:

```go
func runWebsocketClient() error {
	if config.CloudToken == "" {
		time.Sleep(5 * time.Second)
		return fmt.Errorf("cloud token is not set")
	}

	wsURL, err := url.Parse(config.CloudURL)
	if err != nil {
		return fmt.Errorf("failed to parse config.CloudURL: %w", err)
	}

	if wsURL.Scheme == "http" {
		wsURL.Scheme = "ws"
	} else {
		wsURL.Scheme = "wss"
	}

	setCloudConnectionState(CloudConnectionStateConnecting)

	header := http.Header{}
	header.Set("X-Device-ID", GetDeviceID())
	header.Set("X-App-Version", builtAppVersion)
	header.Set("Authorization", "Bearer "+config.CloudToken)
	dialCtx, cancelDial := context.WithTimeout(context.Background(), CloudWebSocketConnectTimeout)
	defer cancelDial()
	c, _, err := websocket.Dial(dialCtx, wsURL.String(), &websocket.DialOptions{
		HTTPHeader: header,
	})
	if err != nil {
		return err
	}
	defer c.CloseNow()

	cloudLogger.Info().Msg("websocket connected")

	// set the metrics when we successfully connect to the cloud.
	wsResetMetrics(true, "cloud", wsURL.Host)

	// we don't have a source for the cloud connection
	return handleWebRTCSignalWsMessages(c, true, wsURL.Host, "", nil)
}
```

## Cloud-Exclusive Features

Several parts of the codebase are exclusively used for the cloud implementation:

*   **`cloud.go`:** This file contains the core logic for the cloud integration, including device registration, token management, and the WebSocket client for communicating with the cloud service.
*   **WebSocket Signaling:** The cloud implementation uses a WebSocket-based signaling client to establish and maintain a persistent connection with the cloud service. This is in contrast to the local implementation, which uses a more traditional HTTP-based approach.
*   **OIDC Integration:** The OIDC integration with Google is exclusive to the cloud authentication flow. This provides a secure and standardized way to handle user authentication and identity verification.
*   **Frontend Environment Variables:** The `ui` directory contains several `.env.cloud-*` files that are used to configure the frontend for different cloud environments. These files define the API endpoints and other settings that the client uses to communicate with the cloud service.

## Client-Side Authentication

The frontend application, located in the `ui/` directory, handles both local and cloud authentication flows. The core logic for these flows is implemented in the React components found in the `ui/src/routes/` directory.

### Local Authentication (Client-Side)

The `login-local.tsx` component is responsible for handling the local login process.

**Authentication Flow:**

1.  **User Input:** The user enters their password into a form.
2.  **API Request:** Upon form submission, the `action` function sends a `POST` request to the `/auth/login-local` endpoint with the password.
3.  **Redirection:** If the login is successful, the user is redirected to the main page (`/`).

**Code Snippet:**

The following snippet from `ui/src/routes/login-local.tsx` shows the `action` function that handles the form submission:

```tsx
const action: ActionFunction = async ({ request }: ActionFunctionArgs) => {
  const formData = await request.formData();
  const password = formData.get("password");

  try {
    const response = await api.POST(`${DEVICE_API}/auth/login-local`, {
      password,
    });

    if (response.ok) {
      return redirect("/");
    } else {
      return { error: "Invalid password" };
    }
  } catch (error) {
    console.error(error);
    return { error: "An error occurred while logging in" };
  }
};
```

### Cloud Authentication (Client-Side)

The cloud authentication flow is initiated by the `AuthLayout.tsx` component and completed by the `adopt.tsx` component.

**Authentication Flow:**

1.  **OIDC Initiation:** The `AuthLayout.tsx` component renders a form that, when submitted, sends a `POST` request to the cloud service's OIDC endpoint (`${CLOUD_API}/oidc/google`).
2.  **Redirect and Callback:** The user is redirected to Google for authentication. After successful authentication, they are redirected back to the `adopt` route of the application.
3.  **Device Registration:** The `loader` function in `adopt.tsx` extracts the `tempToken`, `oidcGoogle`, and `clientId` from the URL and sends a `POST` request to the `/cloud/register` endpoint on the device.
4.  **Final Redirection:** Upon successful registration, the user is redirected to the device's setup page in the cloud application.

**Code Snippets:**

The form in `ui/src/components/AuthLayout.tsx` that starts the OIDC flow:

```tsx
<form action={`${CLOUD_API}/oidc/google`} method="POST">
  {/* ... hidden input fields for deviceId and returnTo ... */}
  <Button
    // ...
    type="submit"
  />
</form>
```

The `loader` function in `ui/src/routes/adopt.tsx` that handles the device registration:

```tsx
const loader: LoaderFunction = async ({ request }: LoaderFunctionArgs) => {
  const url = new URL(request.url);
  const searchParams = url.searchParams;

  const tempToken = searchParams.get("tempToken");
  const deviceId = searchParams.get("deviceId");
  const oidcGoogle = searchParams.get("oidcGoogle");
  const clientId = searchParams.get("clientId");

  const [cloudStateResponse, registerResponse] = await Promise.all([
    api.GET(`${DEVICE_API}/cloud/state`),
    api.POST(`${DEVICE_API}/cloud/register`, {
      token: tempToken,
      oidcGoogle,
      clientId,
    }),
  ]);

  if (!cloudStateResponse.ok) throw new Error("Failed to get cloud state");
  const cloudState = (await cloudStateResponse.json()) as CloudState;

  if (!registerResponse.ok) throw new Error("Failed to register device");

  return redirect(cloudState.appUrl + `/devices/${deviceId}/setup`);
};
```
