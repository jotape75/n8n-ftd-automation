# 🚀 Complete Setup Guide

This guide walks you through setting up the FMC Device Deployment automation from scratch.

## 📋 Prerequisites

Before starting, ensure you have:

### Required Infrastructure
- **n8n instance** (v1.0+) running with web interface access
- **Cisco FMC** with API access enabled
- **Jenkins server** with webhook capability
- **HashiCorp Vault** for secure credential storage
- **SMTP server** (Gmail App Password recommended)

### Network Requirements
- n8n server accessible from Jenkins (for webhooks)
- n8n server can reach FMC API (usually port 443)
- Email SMTP connectivity (port 587/465)

### Access Requirements
- FMC API user credentials
- Jenkins administrator access
- Vault write permissions for secrets
- SMTP authentication credentials

## 🛠️ Step 1: n8n Setup

### Option A: Docker Installation (Recommended)

1. **Create n8n directory**
   ```bash
   mkdir n8n-fmc-automation
   cd n8n-fmc-automation
   ```

2. **Create docker-compose.yml**
   ```yaml
   version: '3.8'
   services:
     n8n:
       image: n8nio/n8n:latest
       container_name: n8n-fmc
       restart: unless-stopped
       ports:
         - "5678:5678"
       environment:
         - N8N_HOST=0.0.0.0
         - N8N_PORT=5678
         - N8N_PROTOCOL=http
         - DB_TYPE=sqlite
         - DB_SQLITE_DATABASE=/home/node/.n8n/database.sqlite
         - EXECUTIONS_PROCESS=main
         - EXECUTIONS_DATA_SAVE_ON_ERROR=all
         - EXECUTIONS_DATA_SAVE_ON_SUCCESS=all
         - N8N_METRICS=true
         - N8N_LOG_LEVEL=info
         - GENERIC_TIMEZONE=America/New_York
       volumes:
         - ./n8n-data:/home/node/.n8n
   ```

3. **Start n8n**
   ```bash
   docker-compose up -d
   ```

4. **Access web interface**
   ```
   http://your-server:5678
   ```

### Option B: npm Installation

```bash
# Install n8n globally
npm install n8n -g

# Start n8n
n8n start

# Access at http://localhost:5678
```

## 🔐 Step 2: Configure Vault Credentials

### Set up FMC API Credentials in Vault

1. **Store FMC credentials**
   ```bash
   # Using Vault CLI
   vault kv put secret/data/cisco1234 \\
     username="api_user" \\
     password="Cisco1234!"
   ```

2. **Store Vault access token securely**
   ```bash
   # Create vault token for n8n access
   vault auth -method=userpass username=n8n-service
   ```

3. **Test vault access**
   ```bash
   curl -X GET \\
     -H "X-Vault-Token: your-vault-token" \\
     http://your-vault:8200/v1/secret/data/cisco1234
   ```

## 📧 Step 3: Configure Email (Gmail Example)

1. **Enable 2-Factor Authentication** on your Gmail account

2. **Create App Password**
   - Go to Google Account Settings
   - Security → 2-Step Verification
   - App passwords → Generate new password
   - Save the 16-character password

3. **SMTP Settings**
   ```
   Host: smtp.gmail.com
   Port: 587 (TLS) or 465 (SSL)
   Username: your-email@gmail.com
   Password: your-app-password (not Gmail password)
   ```

## 🔧 Step 4: Import and Configure Workflows

### Import Workflows

1. **Download workflow files**
   ```bash
   wget https://raw.githubusercontent.com/yourusername/fmc-n8n-workflows/main/workflows/main-orchestration.json
   wget https://raw.githubusercontent.com/yourusername/fmc-n8n-workflows/main/workflows/monitoring-subworkflow.json
   ```

2. **Import to n8n**
   - Open n8n web interface
   - Go to **Workflows** → **Import from file**
   - Import both JSON files

### Configure Credentials in n8n

1. **Add HashiCorp Vault Credentials**
   - Go to **Credentials** → **Create New**
   - Choose **HTTP Basic Auth** or **HTTP Header Auth**
   - Name: `Vault_Access`
   - Configure vault token header

2. **Add FMC HTTP Authentication**
   - **Credentials** → **Create New**
   - Choose **HTTP Basic Auth**
   - Name: `FMC_API_Credentials`
   - Username: `{{ $json.data.data.username }}` (from Vault)
   - Password: `{{ $json.data.data.password }}` (from Vault)

3. **Add SMTP Credentials**
   - **Credentials** → **Create New** 
   - Choose **SMTP**
   - Name: `Email_SMTP`
   - Host: `smtp.gmail.com`
   - Port: `587`
   - Username: your Gmail address
   - Password: your App Password

### Configure Webhook

1. **In Main Orchestration Workflow**
   - Open **Jenkins Webhook** node
   - Note the webhook URL (e.g., `http://your-n8n:5678/webhook/fmc-deployment`)
   - Set authentication if needed

## 🚀 Step 5: Configure Jenkins

### Add Webhook Pipeline Stage

1. **Pipeline Parameters**
   ```groovy
   pipeline {
       parameters {
           string(name: 'DEVICE1_NAME', defaultValue: 'ciscoftd02', description: 'First FTD device name')
           string(name: 'DEVICE1_HOST', defaultValue: '192.168.0.202', description: 'First FTD IP address')
           string(name: 'DEVICE1_REGKEY', defaultValue: 'cisco123', description: 'First FTD registration key')
           string(name: 'DEVICE2_NAME', defaultValue: 'ciscoftd03', description: 'Second FTD device name')
           string(name: 'DEVICE2_HOST', defaultValue: '192.168.0.203', description: 'Second FTD IP address')
           string(name: 'DEVICE2_REGKEY', defaultValue: 'cisco123', description: 'Second FTD registration key')
           string(name: 'EMAIL_RECIPIENT', defaultValue: 'admin@company.com', description: 'Notification email')
       }
       
       stages {
           stage('Deploy Devices') {
               steps {
                   // Your deployment steps here
               }
               post {
                   always {
                       script {
                           // Webhook call to n8n
                           httpRequest(
                               url: 'http://your-n8n:5678/webhook/fmc-deployment',
                               httpMode: 'POST',
                               contentType: 'APPLICATION_JSON',
                               requestBody: \"\"\"
                               {
                                   \"devices\": [
                                       {
                                           \"name\": \"${params.DEVICE1_NAME}\",
                                           \"hostName\": \"${params.DEVICE1_HOST}\",
                                           \"regKey\": {
                                               \"plaintext\": \"${params.DEVICE1_REGKEY}\"
                                           }
                                       },
                                       {
                                           \"name\": \"${params.DEVICE2_NAME}\",
                                           \"hostName\": \"${params.DEVICE2_HOST}\",
                                           \"regKey\": {
                                               \"plaintext\": \"${params.DEVICE2_REGKEY}\"
                                           }
                                       }
                                   ],
                                   \"notification\": {
                                       \"email\": \"${params.EMAIL_RECIPIENT}\"
                                   }
                               }
                               \"\"\"
                           )
                       }
                   }
               }
           }
       }
   }
   ```

## ⚙️ Step 6: Configure Workflow Variables

### Main Orchestration Workflow

1. **Vars from Jenkins Webhook Node**
   ```javascript
   // Extract variables from webhook
   device1_name = {{ $('jenkins_webhook').item.json.body.devices[0].name }}
   device1_host = {{ $('jenkins_webhook').item.json.body.devices[0].hostName }}
   device1_regkey = {{ $('jenkins_webhook').item.json.body.devices[0].regKey.plaintext }}
   device2_name = {{ $('jenkins_webhook').item.json.body.devices[1].name }}
   device2_host = {{ $('jenkins_webhook').item.json.body.devices[1].hostName }}
   device2_regkey = {{ $('jenkins_webhook').item.json.body.devices[1].regKey.plaintext }}
   email_destination = {{ $('jenkins_webhook').item.json.body.notification.email }}
   ```

2. **Vault Credential Retrieval**
   - URL: `http://your-vault:8200/v1/secret/data/cisco1234`
   - Headers: `X-Vault-Token: your-vault-token`

3. **FMC Device Registration**
   - Configure device registration payloads
   - Use extracted variables from Jenkins

### Monitoring Sub-Workflow

1. **Wait1 Node Configuration**
   - Ensure it receives parameters from main workflow

2. **Normalize URL Data Code Node**
   ```javascript
   // Paste the normalize URL code from the summary
   const items = $input.all();
   const checkDeviceCount = $('Check Device Count').first();
   let access_token = checkDeviceCount?.json?.access_token;
   
   return items.map(item => {
     let url;
     if (item.json.items && item.json.items.links) {
       url = item.json.items.links.self;
     } else if (item.json.links && item.json.links.self) {
       url = item.json.links.self;
     }
     
     return {
       json: {
         url: url,
         access_token: access_token,
         ...item.json
       }
     };
   });
   ```

3. **HTTP Request Configuration**
   - Method: GET
   - URL: `{{ $json.url }}`
   - Headers: `X-auth-access-token: {{ $('Check Device Count').first().json.access_token }}`

## ✅ Step 7: Testing

### Test Individual Components

1. **Test Vault Access**
   ```bash
   # Test from n8n server
   curl -X GET \\
     -H "X-Vault-Token: your-token" \\
     http://vault:8200/v1/secret/data/cisco1234
   ```

2. **Test FMC API**
   ```bash
   # Test FMC authentication
   curl -X POST \\
     https://your-fmc/api/fmc_platform/v1/auth/generatetoken \\
     -H "Content-Type: application/json" \\
     -u "api_user:password" -k
   ```

3. **Test Email**
   - Send test email from SMTP node in n8n
   - Verify delivery and formatting

### Test Workflows

1. **Manual Webhook Test**
   ```bash
   # Test webhook directly
   curl -X POST \\
     http://your-n8n:5678/webhook/fmc-deployment \\
     -H "Content-Type: application/json" \\
     -d '{
       "devices": [
         {"name": "test1", "hostName": "192.168.1.100", "regKey": {"plaintext": "test123"}},
         {"name": "test2", "hostName": "192.168.1.101", "regKey": {"plaintext": "test123"}}
       ],
       "notification": {"email": "test@company.com"}
     }'
   ```

2. **End-to-End Test**
   - Run Jenkins pipeline with test parameters
   - Monitor n8n execution logs
   - Verify email notifications

## 🔍 Step 8: Monitoring and Logs

### Enable Debug Logging

1. **n8n Debug Mode**
   ```bash
   # In docker-compose.yml or environment
   N8N_LOG_LEVEL=debug
   ```

2. **View Execution Logs**
   - n8n interface → **Executions**
   - Click on each execution to see details
   - Check for errors in nodes

### Set up Monitoring

1. **Workflow Monitoring**
   - Enable n8n execution retention
   - Set up workflow failure alerts
   - Monitor execution times

2. **Infrastructure Monitoring**
   - Monitor n8n server resources
   - Check Vault connectivity
   - Monitor FMC API response times

## 🚨 Troubleshooting Common Issues

### Issue: Webhook not triggering
```bash
# Check n8n is accessible
curl -I http://your-n8n:5678

# Check webhook URL in workflow
# Verify Jenkins can reach n8n server
```

### Issue: FMC authentication fails
```bash
# Test FMC credentials manually
# Check Vault path and token
# Verify FMC API user permissions
```

### Issue: Email not sending
```bash
# Test SMTP credentials
# Check app password (not regular Gmail password)
# Verify firewall rules for SMTP ports
```

### Issue: Workflows fail to import
```bash
# Check n8n version compatibility
# Validate JSON syntax
# Verify credential names match
```

## 📋 Final Checklist

- [ ] n8n instance running and accessible
- [ ] Vault configured with FMC credentials  
- [ ] SMTP credentials configured and tested
- [ ] Workflows imported and configured
- [ ] Jenkins pipeline configured with webhook
- [ ] All credentials properly referenced
- [ ] Webhook URL accessible from Jenkins
- [ ] Test execution completed successfully
- [ ] Email notifications working
- [ ] Monitoring and logging configured

## 🎯 Next Steps

After successful setup:

1. **Create deployment schedules** in Jenkins
2. **Set up monitoring dashboards** for workflow metrics
3. **Configure backup procedures** for n8n data
4. **Document custom modifications** for your environment
5. **Train team members** on workflow monitoring

## 📞 Need Help?

If you encounter issues:

1. Check the [Troubleshooting Guide](troubleshooting.md)
2. Review n8n execution logs
3. Test individual components
4. Create an issue in the GitHub repository

---

**🎉 Congratulations! Your FMC Device Deployment automation is now ready for production use!**