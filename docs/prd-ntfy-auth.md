# Guide: Adding ntfy Authentication Support to `claude-code-ntfy`

## Overview

By default, `claude-code-ntfy` does not support sending authenticated notifications to ntfy servers that require a token (e.g., via Bearer, Basic Auth, or custom HTTP headers). This guide explains how to add support for token-based authentication (with a focus on Bearer tokens) and outlines how to make it configurable via environment variable and config file.

---

## 1. Decide on Authentication Mechanism

ntfy.sh (and self-hosted ntfy) supports:
- **Bearer tokens** (via the `Authorization: Bearer ...` header)
- **Basic Auth** (via `Authorization: Basic ...`)
- (Optionally) **Custom Headers**

This guide focuses on **Bearer token** support, but the approach for Basic Auth is similar.

---

## 2. Extend Configuration

### a. Environment Variable

- Add `CLAUDE_NOTIFY_AUTH_TOKEN` (for bearer tokens)
- Optionally support `CLAUDE_NOTIFY_AUTH_TYPE` (defaults to `Bearer`)

### b. Config File

Add:
```yaml
ntfy_auth_token: "YOUR_TOKEN_HERE"
ntfy_auth_type: "Bearer"  # Or "Basic"
```

Update your config loader to read these new fields.

---

## 3. Update `NtfyClient` Struct

Add two new fields to `NtfyClient`:

```go
authToken string
authType  string // e.g., "Bearer"
```

Update the constructor:

```go
func NewNtfyClient(server, topic, authToken, authType string) *NtfyClient
```

Or, if using options struct, add those fields to the config struct.

---

## 4. Set the Authorization Header

In `pkg/notification/ntfy_client.go`, in the `Send()` method (before sending the request):

```go
if c.authToken != "" {
    authType := c.authType
    if authType == "" {
        authType = "Bearer"
    }
    req.Header.Set("Authorization", fmt.Sprintf("%s %s", authType, c.authToken))
}
```

---

## 5. Update Config Loader

- When loading from environment, read `CLAUDE_NOTIFY_AUTH_TOKEN` and `CLAUDE_NOTIFY_AUTH_TYPE`.
- When loading from YAML, read `ntfy_auth_token` and `ntfy_auth_type`.

Pass these values to the `NtfyClient` constructor.

---

## 6. Update Documentation

Update README and config examples to show the new options.

### Example (Environment):

```bash
export CLAUDE_NOTIFY_AUTH_TOKEN="my-secret-token"
export CLAUDE_NOTIFY_AUTH_TYPE="Bearer"   # Optional, defaults to Bearer
```

### Example (YAML):

```yaml
ntfy_auth_token: "my-secret-token"
ntfy_auth_type: "Bearer"
```

---

## 7. Test

- Use a private/auth-gated topic on your ntfy server.
- Set the token in your environment or config.
- Run `claude-code-ntfy` and confirm that notifications succeed.

Add tests for:
- No token (should fail with 401)
- Valid token (should succeed)
- Invalid token (should fail with 401)

---

## 8. (Optional) Support Basic Auth

For Basic Auth, encode username:password in base64 and set `Authorization: Basic ...`.

---

## Example Patch

Below is a minimal change sketch:

```go
// Add to struct
type NtfyClient struct {
    server     string
    topic      string
    authToken  string
    authType   string
    httpClient *http.Client
}

// In constructor
func NewNtfyClient(server, topic, authToken, authType string) *NtfyClient {
    return &NtfyClient{
        server: server,
        topic: topic,
        authToken: authToken,
        authType: authType,
        httpClient: &http.Client{Timeout: 10 * time.Second},
    }
}

// In Send()
if c.authToken != "" {
    typ := c.authType
    if typ == "" {
        typ = "Bearer"
    }
    req.Header.Set("Authorization", fmt.Sprintf("%s %s", typ, c.authToken))
}
```

---

## 9. PR Checklist

- [ ] Update `NtfyClient`
- [ ] Update config loader
- [ ] Add documentation & examples
- [ ] Add tests for authenticated/unauthenticated/invalid cases

---

## References

- [ntfy/sh authentication docs](https://docs.ntfy.sh/publish/#authentication)
- [Go http.Request headers](https://pkg.go.dev/net/http#Request)

---

Feel free to ask for code snippets or a ready-to-merge PR!