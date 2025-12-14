# MOSIP Pre-Registration Configuration Guide

This guide documents the configuration changes required to set up a working MOSIP Pre-Registration portal.

## Quick Reference - Key Settings

| Setting | File | Value | Purpose |
|---------|------|-------|---------|
| Proxy OTP | config-server env | `MOSIP_KERNEL_AUTH_PROXY_OTP=true` | Enable mock OTP |
| Proxy OTP Value | config-server env | `MOSIP_KERNEL_AUTH_PROXY_OTP_VALUE=111111` | Set mock OTP value |
| Proxy Email | config-server env | `MOSIP_KERNEL_AUTH_PROXY_EMAIL=true` | Skip email sending |
| Proxy SMS | config-server env | `MOSIP_KERNEL_SMS_PROXY_SMS=true` | Skip SMS sending |
| CAPTCHA | pre-registration-default.properties | `mosip.preregistration.captcha.enable=false` | Disable CAPTCHA |
| Virus Scan | pre-registration-default.properties | `mosip.preregistration.document.scan=false` | Disable virus scan |

## Database Configuration

### Problem
MOSIP kernel services share `kernel-default.properties` but need different databases:
- **masterdata, syncdata** → `mosip_master` database
- **otpmanager, authmanager, idgenerator, pridgenerator** → `mosip_kernel` database

### Solution
Set `kernel-default.properties` to use `mosip_master` as default (for masterdata/syncdata):

```properties
# kernel-default.properties
javax.persistence.jdbc.url=jdbc:postgresql://${mosip.kernel.database.hostname}:${mosip.kernel.database.port}/mosip_master
javax.persistence.jdbc.user=masteruser
javax.persistence.jdbc.password=${db.dbuser.password}
```

Services needing `mosip_kernel` should use environment variable overrides (see below).

### Database Users
Ensure all database users have the correct password matching `db.dbuser.password`:

```bash
# Update passwords in PostgreSQL
kubectl exec postgres-postgresql-0 -n postgres -c postgresql -- \
  env PGPASSWORD=<admin-password> psql -U postgres -c \
  "ALTER USER masteruser WITH PASSWORD '<db.dbuser.password>';"

kubectl exec postgres-postgresql-0 -n postgres -c postgresql -- \
  env PGPASSWORD=<admin-password> psql -U postgres -c \
  "ALTER USER kerneluser WITH PASSWORD '<db.dbuser.password>';"
```

## Config-Server Overrides

Add these environment variables to the config-server deployment for essential settings:

```yaml
# Proxy OTP settings (for testing without real email/SMS)
- name: SPRING_CLOUD_CONFIG_SERVER_OVERRIDES_MOSIP_KERNEL_AUTH_PROXY_OTP
  value: "true"
- name: SPRING_CLOUD_CONFIG_SERVER_OVERRIDES_MOSIP_KERNEL_AUTH_PROXY_OTP_VALUE
  value: "111111"
- name: SPRING_CLOUD_CONFIG_SERVER_OVERRIDES_MOSIP_KERNEL_SMS_PROXY_SMS
  value: "true"
- name: SPRING_CLOUD_CONFIG_SERVER_OVERRIDES_MOSIP_KERNEL_AUTH_PROXY_EMAIL
  value: "true"

# Database password
- name: SPRING_CLOUD_CONFIG_SERVER_OVERRIDES_DB_DBUSER_PASSWORD
  value: "<your-db-password>"
```

## Pre-Registration Settings

### pre-registration-default.properties

```properties
# Disable CAPTCHA for testing
mosip.preregistration.captcha.enable=false

# Disable virus scanning (or fix ClamAV port to 3310)
mosip.preregistration.document.scan=false
```

### If enabling virus scanning in production
The ClamAV service runs on port **3310**, not port 80. Update:

```properties
# application-default.properties
mosip.kernel.virus-scanner.host=clamav
mosip.kernel.virus-scanner.port=3310
```

## Service-Specific Database Overrides

For services that need `mosip_kernel` database instead of the default `mosip_master`:

```bash
# Set SPRING_APPLICATION_JSON for kernel services needing mosip_kernel
for svc in otpmanager authmanager idgenerator pridgenerator; do
  kubectl set env deploy/$svc -n kernel \
    'SPRING_APPLICATION_JSON={"javax.persistence.jdbc.url":"jdbc:postgresql://postgres-postgresql.postgres:5432/mosip_kernel","javax.persistence.jdbc.user":"kerneluser","javax.persistence.jdbc.password":"<password>"}'
done
```

**Note**: `SPRING_APPLICATION_JSON` may not override config-server properties in all cases due to Spring Cloud Config precedence rules.

## Authentication Configuration

### Keycloak Internal URLs
For service-to-service communication, use internal Keycloak URLs to avoid SSL issues:

```properties
# kernel-default.properties
mosip.keycloak.url=${keycloak.internal.url}
keycloak.external.host=${keycloak.internal.url}
keycloak.external.url=${keycloak.internal.url}
```

### Token Validation
For testing environments, you may need to disable strict issuer validation:

```yaml
# config-server env
- name: SPRING_CLOUD_CONFIG_SERVER_OVERRIDES_MOSIP_IAM_ADAPTER_VALIDATE_ISSUER
  value: "false"
- name: SPRING_CLOUD_CONFIG_SERVER_OVERRIDES_AUTH_TOKEN_OFFLINE_VALIDATE_ISS
  value: "false"
```

## Troubleshooting

### OTP Login Issues

1. **"Missing Input Parameter" error**
   - Check if otpmanager is connecting to `mosip_kernel` database
   - Verify `kernel.otp_transaction` table exists

2. **"OTP failed to send through channel" error**
   - This is notification error, but OTP is still generated
   - User can still login with mock OTP (111111) if proxy mode is enabled

3. **OTP validation fails**
   - Check if OTP has expired (default: 3 minutes)
   - Regenerate OTP and validate immediately

### Document Upload Issues

1. **"File could not be uploaded" error**
   - Check ClamAV connectivity: `kubectl logs deploy/prereg-application -n prereg | grep -i virus`
   - Either fix ClamAV port (3310) or disable virus scanning

2. **Check ClamAV status**
   ```bash
   kubectl get pods -n clamav
   kubectl get svc -n clamav  # Should show port 3310
   ```

### Database Connection Issues

1. **"relation does not exist" errors**
   - Service is connected to wrong database
   - Check which database the service should use
   - Verify JDBC URL in logs: `kubectl logs deploy/<service> -n <namespace> | grep "Located environment"`

2. **Password authentication failed**
   - Update database user password to match `db.dbuser.password`

### Loading Spinner / UI Issues

1. **Infinite loading after login**
   - Check browser console for errors
   - May be related to masterdata API calls failing
   - Verify masterdata service is connecting to `mosip_master`

## Restart Order

After configuration changes, restart services in this order:

```bash
# 1. Config server (to pick up git changes)
kubectl rollout restart deploy/config-server -n config-server
kubectl rollout status deploy/config-server -n config-server --timeout=120s

# 2. Kernel services
kubectl rollout restart deploy/masterdata deploy/otpmanager deploy/authmanager -n kernel

# 3. Prereg services
kubectl rollout restart deploy/prereg-application -n prereg
```

## Testing the Setup

### Test OTP Flow
```bash
# 1. Send OTP (may show notification error but OTP is generated)
curl -sk "https://<host>/preregistration/v1/login/sendOtp" \
  -X POST -H "Content-Type: application/json" \
  -d '{"id":"mosip.pre-registration.login.sendotp","version":"1.0","requesttime":"2025-01-01T00:00:00.000Z","request":{"userId":"test@example.com"}}'

# 2. Validate OTP with mock value
curl -sk "https://<host>/preregistration/v1/login/validateOtp" \
  -X POST -H "Content-Type: application/json" \
  -d '{"id":"mosip.pre-registration.login.useridotp","version":"1.0","requesttime":"2025-01-01T00:00:00.000Z","request":{"userId":"test@example.com","otp":"111111"}}'
```

### Expected Response
```json
{"response":{"message":"VALIDATION_SUCCESSFUL","status":"success"},"errors":null}
```

## Summary of Changes Made

1. **kernel-default.properties**: Changed default database to `mosip_master` for masterdata
2. **pre-registration-default.properties**: Disabled CAPTCHA and virus scanning
3. **PostgreSQL**: Updated `masteruser` and `kerneluser` passwords
4. **Config-server**: Proxy OTP/email/SMS settings already configured

---
*Last updated: 2025-12-14*
