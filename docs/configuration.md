# ⚙️ Configuration Reference

This document provides comprehensive configuration details for all components of the FMC Device Deployment automation.

## 📋 Overview

The automation system requires configuration of several components:
- n8n workflow settings
- FMC API connections
- Vault credential management
- Jenkins integration
- Email notifications

## 🔧 n8n Configuration

### Environment Variables

```bash
# Basic n8n settings
N8N_HOST=0.0.0.0                    # Listen on all interfaces
N8N_PORT=5678                       # Default n8n port  
N8N_PROTOCOL=http                   # Use HTTPS in production

# Database (SQLite for development)
DB_TYPE=sqlite
DB_SQLITE_DATABASE=/home/node/.n8n/database.sqlite

# Execution settings
EXECUTIONS_PROCESS=main
EXECUTIONS_DATA_SAVE_ON_ERROR=all
EXECUTIONS_DATA_SAVE_ON_SUCCESS=all
EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS=true

# Security settings
N8N_BASIC_AUTH_ACTIVE=false         # Enable for production
N8N_BASIC_AUTH_USER=admin           # If auth enabled
N8N_BASIC_AUTH_PASSWORD=secure123   # If auth enabled

# Performance settings  
N8N_PAYLOAD_SIZE_MAX=16             # Max payload size (MB)
EXECUTIONS_TIMEOUT=3600             # Max execution time (seconds)

# Timezone
GENERIC_TIMEZONE=America/New_York   # Adjust as needed

# Logging and metrics
N8N_LOG_LEVEL=info                  # Use 'debug' for troubleshooting
N8N_METRICS=true                    # Enable metrics endpoint
```

### Workflow Settings

#### Webhook Configuration
```javascript
// Main workflow webhook settings
{
  "httpMethod": "POST",
  "path": "fmc-deployment",
  "authentication": "none",          // Or configure as needed
  "responseMode": "onReceived"
}
```

#### Execution Settings
```javascript
// Workflow execution settings
{
  "executionOrder": "v1",
  "saveDataErrorExecution": "all",
  "saveDataSuccessExecution": "all",
  "saveManualExecutions": true,
  "timeout": 3600,
  "timezone": "America/New_York"
}
```

## 🏛️ HashiCorp Vault Configuration

### Vault Setup

```bash
# Initialize Vault (if new installation)
vault operator init

# Unseal Vault  
vault operator unseal <unseal-key>

# Enable auth method
vault auth enable userpass

# Create policy for n8n access
vault policy write n8n-policy - <<EOF
path "secret/data/cisco*" {
  capabilities = ["read"]
}
EOF

# Create user for n8n
vault write auth/userpass/users/n8n-service \\
  password="secure-password" \\
  policies="n8n-policy"
```

### Store FMC Credentials

```bash
# Store FMC API credentials
vault kv put secret/data/cisco1234 \\
  username="api_user" \\
  password="Cisco1234!" \\
  description="FMC API credentials for device deployment"

# Verify storage
vault kv get secret/data/cisco1234
```

### Vault Access from n8n

```javascript
// Vault HTTP Request configuration
{
  "method": "GET",
  "url": "{{ $json.vault_url }}/v1/secret/data/cisco1234",
  "headers": {
    "X-Vault-Token": "{{ $json.vault_token }}"
  },
  "authentication": "none"
}
```

## 🔥 Cisco FMC Configuration  

### API User Setup

1. **Create API user in FMC**
   ```bash
   # FMC CLI Commands
   configure user api_user password Cisco1234! role admin
   ```

2. **Enable API access**
   - FMC Web Interface → System → Configuration → REST API Preferences
   - Enable REST API
   - Set session timeout (recommended: 30 minutes)

### API Authentication

```javascript
// FMC authentication request
{
  "method": "POST", 
  "url": "https://{{ $json.fmc_host }}/api/fmc_platform/v1/auth/generatetoken",
  "headers": {
    "Content-Type": "application/json"
  },
  "auth": {
    "username": "{{ $json.fmc_username }}",
    "password": "{{ $json.fmc_password }}"
  },
  "ignoreSSLIssues": true
}
```

### Device Registration Payload

```json
{
  "type": "DeviceRecord",
  "name": "{{ $json.device1_name }}",
  "hostName": "{{ $json.device1_host }}",  
  "regKey": "{{ $json.device1_regkey }}",
  "type": "Device",
  "license_caps": ["MALWARE", "URLFilter", "THREAT"],
  "metadata": {
    "domain": {
      "name": "Global",
      "id": "e276abec-56ab-11e0-8f1e-00200ec81234"
    }
  }
}
```

### FMC API Endpoints

```bash
# Authentication
POST /api/fmc_platform/v1/auth/generatetoken

# Device registration  
POST /api/fmc_config/v1/domain/{domain}/devices/devicerecords

# Device status
GET /api/fmc_config/v1/domain/{domain}/devices/devicerecords/{device-id}

# Deployment status
GET /api/fmc_config/v1/domain/{domain}/deployment/deployabledevices/{device-id}
```

## 📧 Email Configuration

### SMTP Settings

```javascript
// Gmail SMTP configuration
{
  "host": "smtp.gmail.com",
  "port": 587,
  "secure": false,              // Use STARTTLS
  "auth": {
    "user": "your-email@gmail.com",
    "pass": "app-password"      // Gmail App Password, not regular password
  }
}

// Alternative: SSL configuration
{
  "host": "smtp.gmail.com", 
  "port": 465,
  "secure": true,             // Use SSL
  "auth": {
    "user": "your-email@gmail.com",
    "pass": "app-password"
  }
}
```

### Email Templates

#### Success Email Template
```html
<h2>✅ FMC Device Deployment - SUCCESS</h2>
<p><strong>Status:</strong> Completed Successfully</p>

<h3>Devices Deployed:</h3>
<ul>
  <li><strong>{{ $json.device1_name }}</strong>
    <br>IP: {{ $json.device1_host }}
    <br>Deployment: {{ $json.device1_deployment }}
    <br>Health: {{ $json.device1_health }}
  </li>
  <li><strong>{{ $json.device2_name }}</strong>
    <br>IP: {{ $json.device2_host }}
    <br>Deployment: {{ $json.device2_deployment }}  
    <br>Health: {{ $json.device2_health }}
  </li>
</ul>

<h3>📊 Summary:</h3>
<p>{{ $json.summary_message }}</p>

<p><strong>Timestamp:</strong> {{ $now.format('MMM d, yyyy h:mm a') }}</p>
```

#### Failure Email Template
```html
<h2>❌ FMC Device Deployment - FAILURE</h2>
<p><strong>Status:</strong> Deployment Failed</p>

<h3>Devices:</h3>
<ul>
  <li><strong>{{ $json.device1_name }}</strong> ({{ $json.device1_host }}) - {{ $json.device1_status }}
  <li><strong>{{ $json.device2_name }}</strong> ({{ $json.device2_host }}) - {{ $json.device2_status }}
</ul>

<h3>📊 Summary:</h3>
<p>{{ $json.error_message }}</p>

<p><strong>Retry attempts:</strong> {{ $json.retry_count }}/40</p>
<p><strong>Timestamp:</strong> {{ $now.format('MMM d, yyyy h:mm a') }}</p>
```

## 🚀 Jenkins Configuration

### Pipeline Parameters

```groovy
parameters {
    string(
        name: 'DEVICE1_NAME', 
        defaultValue: 'ciscoftd02',
        description: 'Name of the first FTD device'
    )
    string(
        name: 'DEVICE1_HOST',
        defaultValue: '192.168.0.202', 
        description: 'IP address of the first FTD device'
    )
    string(
        name: 'DEVICE1_REGKEY',
        defaultValue: 'cisco123',
        description: 'Registration key for first FTD device'
    )
    string(
        name: 'DEVICE2_NAME',
        defaultValue: 'ciscoftd03',
        description: 'Name of the second FTD device'  
    )
    string(
        name: 'DEVICE2_HOST',
        defaultValue: '192.168.0.203',
        description: 'IP address of the second FTD device'
    )
    string(
        name: 'DEVICE2_REGKEY', 
        defaultValue: 'cisco123',
        description: 'Registration key for second FTD device'
    )
    string(
        name: 'EMAIL_RECIPIENT',
        defaultValue: 'admin@company.com',
        description: 'Email address for deployment notifications'
    )
    choice(
        name: 'ENVIRONMENT',
        choices: ['development', 'staging', 'production'],
        description: 'Target environment for deployment'
    )
}
```

### Webhook Configuration

```groovy
// Jenkins webhook post-build action
post {
    always {
        script {
            def webhookUrl = 'http://n8n.company.com:5678/webhook/fmc-deployment'
            def payload = [
                devices: [
                    [
                        name: params.DEVICE1_NAME,
                        hostName: params.DEVICE1_HOST,
                        regKey: [plaintext: params.DEVICE1_REGKEY]
                    ],
                    [
                        name: params.DEVICE2_NAME,
                        hostName: params.DEVICE2_HOST,
                        regKey: [plaintext: params.DEVICE2_REGKEY]
                    ]
                ],
                notification: [
                    email: params.EMAIL_RECIPIENT
                ],
                metadata: [
                    jenkins_build: env.BUILD_NUMBER,
                    jenkins_job: env.JOB_NAME,
                    environment: params.ENVIRONMENT,
                    timestamp: new Date().format('yyyy-MM-dd HH:mm:ss')
                ]
            ]
            
            httpRequest(
                url: webhookUrl,
                httpMode: 'POST',
                contentType: 'APPLICATION_JSON',
                requestBody: groovy.json.JsonBuilder(payload).toString(),
                validResponseCodes: '200:299'
            )
        }
    }
}
```

## 🔄 Workflow Variables Reference

### Main Workflow Variables

```javascript
// Variables from Jenkins webhook
device1_name = {{ $('jenkins_webhook').item.json.body.devices[0].name }}
device1_host = {{ $('jenkins_webhook').item.json.body.devices[0].hostName }}  
device1_regkey = {{ $('jenkins_webhook').item.json.body.devices[0].regKey.plaintext }}
device2_name = {{ $('jenkins_webhook').item.json.body.devices[1].name }}
device2_host = {{ $('jenkins_webhook').item.json.body.devices[1].hostName }}
device2_regkey = {{ $('jenkins_webhook').item.json.body.devices[1].regKey.plaintext }}
email_destination = {{ $('jenkins_webhook').item.json.body.notification.email }}

// Variables from Vault
fmc_username = {{ $json.data.data.username }}
fmc_password = {{ $json.data.data.password }}

// FMC API variables
access_token = {{ $json.access_token }}
domain_uuid = {{ $json.items[0].domain.id }}
fmc_host = "192.168.1.100"  // Configure for your environment
```

### Sub-Workflow Variables

```javascript
// Variables passed from main workflow
device1_name = {{ $('Wait1').first().json.device1_name }}
device2_name = {{ $('Wait1').first().json.device2_name }}
access_token = {{ $('Wait1').first().json.access_token }}
fmc = {{ $('Wait1').first().json.fmc }}
email_destination = {{ $('Wait1').first().json.email_destination }}

// Retry configuration
retryAttempts = [1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40]
waitTime = 30  // seconds
```

## 🕰️ Timing Configuration

### Retry Logic Settings

```javascript
// Retry configuration
const retryConfig = {
  maxRetries: 40,               // Maximum number of retry attempts
  waitBetween: 30,              // Seconds to wait between retries  
  totalMaxWait: 1200,           // Total max wait time (20 minutes)
  exponentialBackoff: false     // Use fixed wait time
};

// Wait calculation
const totalWaitTime = retryConfig.maxRetries * retryConfig.waitBetween;
// Result: 40 * 30 = 1200 seconds = 20 minutes
```

### Timeout Settings

```javascript
// HTTP request timeouts
const httpConfig = {
  fmcApiTimeout: 30000,         // 30 seconds for FMC API calls
  vaultTimeout: 10000,          // 10 seconds for Vault calls
  emailTimeout: 15000           // 15 seconds for email sending
};
```

## 📊 Monitoring Configuration

### Execution Data Retention

```bash
# n8n execution data settings
EXECUTIONS_DATA_SAVE_ON_ERROR=all
EXECUTIONS_DATA_SAVE_ON_SUCCESS=all  
EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS=true
EXECUTIONS_DATA_PRUNE=true
EXECUTIONS_DATA_MAX_AGE=336  # Hours (14 days)
```

### Metrics Configuration

```bash
# Enable n8n metrics endpoint
N8N_METRICS=true

# Access metrics at:
# http://your-n8n:5678/metrics
```

## 🔐 Security Configuration

### SSL/TLS Settings

```javascript
// FMC API TLS configuration
{
  "ignoreSSLIssues": true,      // For lab environments
  "rejectUnauthorized": false   // For self-signed certificates
}

// Production settings
{
  "ignoreSSLIssues": false,
  "rejectUnauthorized": true,
  "ca": "path/to/ca-cert.pem"   // Custom CA if needed
}
```

### Credential Security

```javascript
// Secure credential references
{
  "fmc_username": "{{ $credentials.FMC_API.username }}",
  "fmc_password": "{{ $credentials.FMC_API.password }}",
  "vault_token": "{{ $credentials.Vault_Access.token }}"
}
```

## 🧪 Environment-Specific Configuration

### Development Environment

```bash
# Development settings
N8N_LOG_LEVEL=debug
N8N_BASIC_AUTH_ACTIVE=false
EXECUTIONS_TIMEOUT=7200
N8N_SECURE_COOKIE=false

# FMC settings
FMC_HOST=192.168.1.100
FMC_VERIFY_SSL=false
```

### Production Environment

```bash
# Production settings  
N8N_LOG_LEVEL=warn
N8N_BASIC_AUTH_ACTIVE=true
N8N_SECURE_COOKIE=true
EXECUTIONS_TIMEOUT=3600

# Database (PostgreSQL recommended)
DB_TYPE=postgresdb
DB_POSTGRESDB_HOST=postgres.company.com
DB_POSTGRESDB_DATABASE=n8n_prod
DB_POSTGRESDB_USER=n8n_user
DB_POSTGRESDB_PASSWORD=secure_password

# FMC settings
FMC_HOST=fmc.company.com
FMC_VERIFY_SSL=true
```

## 📋 Configuration Checklist

### Pre-Deploy Checklist

- [ ] n8n environment variables configured
- [ ] Vault policies and credentials set up
- [ ] FMC API user created and tested
- [ ] SMTP credentials configured and tested
- [ ] Jenkins pipeline parameters defined
- [ ] Webhook URLs accessible
- [ ] SSL certificates configured (production)
- [ ] Network connectivity verified
- [ ] Backup procedures configured

### Post-Deploy Verification

- [ ] Workflows import successfully  
- [ ] All credentials resolve correctly
- [ ] Webhook triggers workflow execution
- [ ] FMC API calls succeed
- [ ] Email notifications deliver
- [ ] Retry logic functions properly  
- [ ] Monitoring data collected
- [ ] Error handling works as expected

## 🚨 Common Configuration Issues

### Issue: Variables not resolving
**Fix**: Check node references and JSON paths

### Issue: Credentials not found
**Fix**: Verify credential names match exactly

### Issue: API calls timing out
**Fix**: Adjust timeout values and check network connectivity

### Issue: Email delivery fails
**Fix**: Verify SMTP settings and Gmail App Password

### Issue: Webhook not accessible
**Fix**: Check firewall rules and n8n binding address

---

**💡 Tip**: Always test configuration changes in a development environment before applying to production!