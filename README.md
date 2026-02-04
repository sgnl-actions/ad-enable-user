# Active Directory Enable User Action

This action enables a disabled user account in on-premise Active Directory using LDAP/LDAPS.

## Overview

The AD Enable User action re-enables disabled Active Directory accounts by clearing the `ACCOUNTDISABLE` bit (0x0002) in the `userAccountControl` attribute. It reads the current UAC value, checks if the account is disabled, and if so clears the disable bit while preserving all other flags. The operation is idempotent -- if the account is already enabled, it returns success without making changes.

## Prerequisites

- On-premise Active Directory domain controller accessible via LDAP or LDAPS
- A service account with permissions to modify the `userAccountControl` attribute on target user objects
- Network connectivity from the execution environment to the LDAP server

## Configuration

### Authentication

This action uses LDAP Simple Bind authentication with a service account.

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `BASIC_USERNAME` | Secret | Yes | Bind DN of the service account (e.g., `CN=svc-sgnl,OU=Service Accounts,DC=corp,DC=example,DC=com`) |
| `BASIC_PASSWORD` | Secret | Yes | Password for the service account |

### Environment Variables

| Variable | Required | Description | Example |
|----------|----------|-------------|---------|
| `ADDRESS` | Yes | LDAP server URL | `ldaps://ad.corp.example.com:636` |
| `TLS_SKIP_VERIFY` | No | Set to `true` to skip TLS certificate verification | `true` |

### Input Parameters

| Parameter | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| `userDN` | string | Yes | Distinguished Name of the user to enable | `CN=John Doe,OU=Users,DC=corp,DC=example,DC=com` |
| `address` | string | No | Optional LDAP server URL override | `ldaps://ad.corp.example.com:636` |

### Output Structure

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | Operation result (success, failed, etc.) |
| `userDN` | string | Distinguished Name of the user that was processed |
| `enabled` | boolean | Whether the user was newly enabled (false if already enabled) |
| `previousUAC` | number | The `userAccountControl` value before the operation |
| `newUAC` | number | The `userAccountControl` value after the operation |
| `address` | string | The LDAP server URL that was used |

## Usage Examples

### Basic Usage

```json
{
  "userDN": "CN=John Doe,OU=Users,DC=corp,DC=example,DC=com"
}
```

### Job Specification

```json
{
  "id": "enable-user-account",
  "type": "nodejs-22",
  "script": {
    "repository": "github.com/sgnl-actions/ad-enable-user",
    "version": "v1.0.0",
    "type": "nodejs"
  },
  "script_inputs": {
    "userDN": "CN=Disabled User,OU=Users,DC=corp,DC=example,DC=com"
  },
  "environment": {
    "ADDRESS": "ldaps://ad.corp.example.com:636"
  },
  "secrets": {
    "BASIC_USERNAME": "CN=svc-sgnl,OU=Service Accounts,DC=corp,DC=example,DC=com",
    "BASIC_PASSWORD": "your-service-account-password"
  }
}
```

### With TLS Skip Verify

For environments with self-signed certificates:

```json
{
  "id": "enable-user-account",
  "type": "nodejs-22",
  "script": {
    "repository": "github.com/sgnl-actions/ad-enable-user",
    "version": "v1.0.0",
    "type": "nodejs"
  },
  "script_inputs": {
    "userDN": "CN=Disabled User,OU=Users,DC=corp,DC=example,DC=com"
  },
  "environment": {
    "ADDRESS": "ldaps://ad.corp.example.com:636",
    "TLS_SKIP_VERIFY": "true"
  },
  "secrets": {
    "BASIC_USERNAME": "CN=svc-sgnl,OU=Service Accounts,DC=corp,DC=example,DC=com",
    "BASIC_PASSWORD": "your-service-account-password"
  }
}
```

## API Details

This action performs two LDAP operations:

1. **SEARCH** the user DN (base scope) to read the current `userAccountControl` value
2. **MODIFY** the `userAccountControl` attribute with the `ACCOUNTDISABLE` bit cleared (if it was set)

```
SEARCH userDN (scope=base, attrs=userAccountControl)
MODIFY userDN
  REPLACE userAccountControl: <value with bit 0x0002 cleared>
```

The connection lifecycle is stateless: each invocation binds to the LDAP server, performs the search/modify operations, and unbinds in a `finally` block.

## Error Handling

### Success Scenarios

- **User was disabled**: Account enabled successfully (`enabled: true`, UAC updated)
- **User already enabled**: No changes made (`enabled: false`, UAC unchanged)

### Retryable Errors

The framework automatically retries on transient errors such as:
- Network connectivity issues
- LDAP server temporarily unavailable
- Connection timeouts

### Fatal Errors

The following errors will not be retried:
- **Invalid credentials**: Incorrect bind DN or password
- **Insufficient access rights**: Service account lacks permission to modify `userAccountControl`
- **No such object** (LDAP code 32): The user DN does not exist
- **Invalid DN syntax**: Malformed Distinguished Name
- **User not found**: Search returned no entries for the given DN

## Security Considerations

- **Authentication**: Uses LDAP Simple Bind with a dedicated service account
- **Transport Security**: Supports LDAPS (LDAP over TLS) for encrypted connections
- **TLS Verification**: Certificate verification is enabled by default; `TLS_SKIP_VERIFY` should only be used in development or with self-signed certificates
- **Credential Security**: Bind credentials are provided via secrets and are never logged
- **Connection Lifecycle**: Connections are unbound in a `finally` block to prevent resource leaks

## Development

### Local Testing

```bash
# Run with mock parameters
npm run dev

# Run unit tests
npm test

# Check test coverage
npm run test:coverage
```

### Building

```bash
# Build distribution bundle
npm run build

# Validate metadata
npm run validate

# Lint code
npm run lint
```

## Troubleshooting

### Common Issues

1. **"Missing LDAP bind credentials"**
   - Ensure `BASIC_USERNAME` and `BASIC_PASSWORD` are set in secrets
   - Verify the bind DN is a valid Distinguished Name

2. **"No URL specified"**
   - Ensure the `ADDRESS` environment variable is set or `address` is provided in params
   - Verify the URL format (e.g., `ldaps://ad.corp.example.com:636`)

3. **"Invalid credentials"**
   - Verify the service account DN and password are correct
   - Check that the account is not locked or expired in Active Directory

4. **"Insufficient access rights"**
   - Verify the service account has Write permission on the `userAccountControl` attribute
   - Check if there are any deny ACEs blocking the operation

5. **"User not found"**
   - Verify the user DN exists in Active Directory
   - Check for typos in the Distinguished Name

6. **TLS/SSL connection errors**
   - Verify the LDAP server is accessible on the configured port
   - For LDAPS, ensure the server certificate is trusted or set `TLS_SKIP_VERIFY=true` for testing
   - Check that the correct port is used (389 for LDAP, 636 for LDAPS)

### Verifying Account Status

To verify the action worked correctly, you can check the account status using:

```bash
# Using ldapsearch
ldapsearch -H ldaps://ad.corp.example.com:636 \
  -D "CN=svc-sgnl,OU=Service Accounts,DC=corp,DC=example,DC=com" \
  -W -b "CN=John Doe,OU=Users,DC=corp,DC=example,DC=com" \
  "(objectClass=user)" userAccountControl

# Using PowerShell
Get-ADUser -Identity "John Doe" -Properties Enabled, userAccountControl | Select-Object Name, Enabled, userAccountControl
```

## Support

- [ldapts Documentation](https://github.com/ldapts/ldapts)
- [Active Directory LDAP Reference](https://docs.microsoft.com/en-us/windows/win32/ad/active-directory-domain-services)
- [SGNL Actions Documentation](https://github.com/sgnl-actions)
