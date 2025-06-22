# PRD: Adding ntfy Bearer Authentication Support to `claude-code-ntfy`

## Overview

This PRD outlines the implementation of Bearer token authentication support for `claude-code-ntfy`, enabling secure notifications to private ntfy topics.

## Goals

1. Enable Bearer token authentication for private ntfy topics
2. Maintain backward compatibility (auth should be optional)
3. Follow existing project patterns and conventions
4. Ensure secure handling of authentication credentials
5. Provide comprehensive test coverage

## Implementation Plan

### 1. Configuration Schema Updates

#### Environment Variables
- `CLAUDE_NOTIFY_AUTH_TOKEN` - Bearer authentication token

#### Config File (YAML)
```yaml
ntfy_auth_token: "YOUR_TOKEN_HERE"
```

### 2. Code Changes

#### A. Update Config Structure (`pkg/config/config.go`)

Add new field to the Config struct:
```go
// Authentication settings
NtfyAuthToken string `yaml:"ntfy_auth_token" env:"CLAUDE_NOTIFY_AUTH_TOKEN"`
```

#### B. Update Config Loading (`pkg/config/config.go`)

In `loadFromEnv()`, add:
```go
if authToken := os.Getenv("CLAUDE_NOTIFY_AUTH_TOKEN"); authToken != "" {
    cfg.NtfyAuthToken = authToken
}
```

#### C. Update NtfyClient (`pkg/notification/ntfy_client.go`)

1. Add field to struct:
```go
type NtfyClient struct {
    server     string
    topic      string
    authToken  string
    httpClient *http.Client
}
```

2. Update constructor signature:
```go
func NewNtfyClient(server, topic, authToken string) *NtfyClient {
    return &NtfyClient{
        server:     server,
        topic:      topic,
        authToken:  authToken,
        httpClient: &http.Client{Timeout: 10 * time.Second},
    }
}
```

3. Add authentication to Send() method:
```go
// Add authorization header if token is provided
if c.authToken != "" {
    req.Header.Set("Authorization", "Bearer " + c.authToken)
}
```

#### D. Update Dependencies Creation (`cmd/claude-code-ntfy/app.go`)

Update the notification manager creation to pass the auth token:
```go
// In NewDependencies
ntfyClient := notification.NewNtfyClient(cfg.NtfyServer, cfg.NtfyTopic, cfg.NtfyAuthToken)
```

### 3. Security Considerations

1. **No Logging of Credentials**: Never log auth tokens
2. **Secure Storage**: Document that users should use appropriate file permissions for config files containing credentials
3. **Environment Variable Priority**: Env vars override file config for security
4. **Error Messages**: Don't expose auth tokens in error messages

### 4. Testing Strategy

#### A. Unit Tests for Config Loading
- Test loading auth token from environment
- Test loading auth token from YAML
- Test that empty token is allowed (backward compatibility)

#### B. Unit Tests for NtfyClient
- Test Bearer token is added to request header when provided
- Test no auth header when token is empty
- Test 401 Unauthorized response handling
- Verify token is not logged or exposed in errors

#### C. Update Existing Tests
- Update all `NewNtfyClient` calls to include the auth parameter
- Ensure backward compatibility in tests

### 5. Documentation Updates

#### README.md
Add to configuration section:
```markdown
### Authentication

For private ntfy topics that require authentication:

```bash
# Via environment variable
export CLAUDE_NOTIFY_AUTH_TOKEN="tk_your_token_here"

# Or in config.yaml
ntfy_auth_token: "tk_your_token_here"
```

**Security Note**: Store tokens securely and never commit them to version control.
```

#### Update existing examples to show auth usage

### 6. Migration & Compatibility

- Existing users without auth continue to work (auth token is optional)
- No breaking changes to external API
- Clear error messages when auth fails (401 responses)

## Success Criteria

1. Can send notifications to authenticated topics using Bearer tokens
2. All existing functionality continues to work without auth
3. Tests pass with maintained coverage levels
4. No auth tokens are logged or exposed
5. Documentation clearly explains auth setup

## Implementation Notes

- Keep the implementation minimal and focused on Bearer auth only
- Follow the existing pattern of optional configuration fields
- Ensure the change is backward compatible
- Add proper test coverage for the new functionality

## References

- [ntfy authentication docs](https://docs.ntfy.sh/publish/#authentication)
- [RFC 6750 - Bearer Token Usage](https://tools.ietf.org/html/rfc6750)