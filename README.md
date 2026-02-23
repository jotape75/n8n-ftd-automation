# FMC Device Deployment Automation with n8n

> 🚀 **Automated workflows for Cisco FMC device deployment with Jenkins integration, real-time monitoring, and email notifications**

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![n8n](https://img.shields.io/badge/n8n-workflow-FF6D5A.svg)](https://n8n.io)
[![Cisco FMC](https://img.shields.io/badge/Cisco-FMC-1BA0D7.svg)](https://cisco.com)

## 📋 Overview

This project provides complete n8n workflows for automating Cisco FMC (Firewall Management Center) device deployment with:

- **Jenkins Integration**: Webhook-triggered deployment from CI/CD pipelines
- **Intelligent Monitoring**: Automated deployment status tracking with retry logic  
- **Health Verification**: Real-time device health status monitoring
- **Email Notifications**: Success/failure notifications with detailed reports
- **Fault Tolerance**: Automatic retry mechanisms with configurable timeouts

## 🏗️ Architecture

### Main Workflow
```
Jenkins Webhook → Variable extraction → Device Registration → Status Monitoring → Email Notification
```

### Monitoring Sub-Workflow  
```
Status Check → Deployment OK? → Success Email
     ↓              ↓ No           
Wait 30s ←── Retry Loop
```

## 🚀 Quick Start

### Prerequisites

- n8n instance (v1.0+)
- Cisco FMC with API access
- Jenkins with webhook capability
- HashiCorp Vault (for credential management)
- SMTP server (Gmail App Password supported)

### Installation

1. **Clone this repository**
   ```bash
   git clone https://github.com/yourusername/fmc-n8n-workflows.git
   cd fmc-n8n-workflows
   ```

2. **Import workflows to n8n**
   - In n8n interface, go to **Workflows** → **Import**
   - Import `workflows/ftd-main-orchestrator.json` (main workflow)
   - Import `workflows/fmc-authentication-subworkflow.json`
   - Import `workflows/fmc-device-registration-subworkflow.json`  
   - Import `workflows/fmc-device-registration-monitoring-subworkflow.json`
   - Optional: Import `workflows/ai-workflow-manager.json`

3. **Configure credentials** (see [Configuration Guide](docs/configuration.md))

4. **Set up Jenkins webhook** (see [Setup Guide](docs/setup-guide.md))

## 📁 Repository Structure

```
fmc-n8n-workflows/
├── README.md                                                    # This file
├── workflows/
│   ├── ftd-main-orchestrator.json                              # 🎯 Main deployment orchestrator
│   ├── fmc-authentication-subworkflow.json                     # 🔐 FMC authentication handler
│   ├── fmc-device-registration-subworkflow.json                # 📝 Device registration manager
│   ├── fmc-device-registration-monitoring-subworkflow.json     # 📊 Deployment monitoring & retry
│   └── ai-workflow-manager.json                                # 🤖 AI workflow manager (optional)
├── docs/
│   ├── setup-guide.md                                          # Complete setup instructions
│   ├── configuration.md                                        # Configuration reference
│   ├── troubleshooting.md                                      # Common issues and solutions
│   └── screenshots/                                            # Visual guides
└── .gitignore                                                  # Git ignore rules
```

## ⚙️ Key Features

### 🔄 Dynamic URL Handling
Handles different data structures between initial checks and retry loops with intelligent URL normalization.

### 🔐 Secure Authentication  
- Vault integration for credential management
- Persistent access token handling throughout workflow execution
- Secure header-based FMC API authentication

### 📧 Smart Email Notifications
- Formatted timestamps: `Feb 23, 2026 3:54 PM`
- Detailed deployment reports with device status
- Configurable recipient lists from Jenkins parameters

### 🔁 Intelligent Retry Logic
- **Max retries**: 40 attempts
- **Wait time**: 30 seconds between attempts
- **Total timeout**: ~20 minutes
- **Status tracking**: Full deployment and health monitoring

## 🛠️ Configuration

### Environment Variables

```bash
# FMC API Configuration
FMC_HOST=192.168.1.100
FMC_USERNAME=api_user
FMC_PASSWORD=secure_password

# Email Configuration  
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
EMAIL_FROM=alerts@yourcompany.com

# Vault Configuration
VAULT_URL=https://vault.company.com
VAULT_PATH=secret/data/cisco
```

### Jenkins Parameters

```groovy
parameters {
    string(name: 'DEVICE1_NAME', defaultValue: 'ciscoftd02')
    string(name: 'DEVICE1_HOST', defaultValue: '192.168.0.202') 
    string(name: 'DEVICE1_REGKEY', defaultValue: 'cisco123')
    string(name: 'DEVICE2_NAME', defaultValue: 'ciscoftd03')
    string(name: 'DEVICE2_HOST', defaultValue: '192.168.0.203')
    string(name: 'DEVICE2_REGKEY', defaultValue: 'cisco123')
    string(name: 'EMAIL_RECIPIENT', defaultValue: 'admin@company.com')
}
```

## 📊 Workflow Details

### FTD Main Orchestrator (7 nodes)

| Step | Node | Purpose |
|------|------|---------|  
| 1 | jenkins_webhook | Receives deployment trigger from Jenkins |
| 2 | Vars from Jenkins webhook call & Vault creds retrieval | Parses Jenkins parameters and gets credentials |  
| 3 | Extract Vars_01 (Main Orch workflow) | Extracts and prepares variables |
| 4 | Call 'FMC-Authentication-SubWorkflow' | Gets FMC access token |
| 5 | Call 'FMC-Device-Registration-SubWorkflow' | Registers devices with FMC |
| 6 | Execute Monitoring SubWorkflow | Triggers deployment monitoring |
| 7 | Email notification | Sends final status report |

### FMC Device Registration Monitoring SubWorkflow (26 nodes)

| Step | Node | Purpose |
|------|------|---------|  
| 1 | Wait1 | Receives monitoring parameters from main workflow |
| 2 | Extract Vars_01 | Processes input data and variables |
| 3 | Input data processing | Prepares monitoring data |
| 4 | Normalize URL Data | Handles different data structures |
| 5 | Get Device Deployment/Health Status | HTTP request to FMC API |
| 6 | Deployment Status = OK? | Conditional deployment check |
| 7 | Wait2 (30 sec) | Retry delay before next check |
| 8 | Email Success/Failure | Sends completion notifications |

## 🔧 Critical Code Snippets

### URL Normalization (Code Node)
```javascript
// Get all input items
const items = $input.all();

// Try to get access_token from Check Device Count node
const checkDeviceCount = $('Check Device Count').first();
let access_token = checkDeviceCount?.json?.access_token;

// Process each item
return items.map(item => {
  let url;
  
  // Check if data has 'items' object (from Check Device Count)
  if (item.json.items && item.json.items.links) {
    url = item.json.items.links.self;
  }
  // Otherwise it's from Wait2 (direct links.self)
  else if (item.json.links && item.json.links.self) {
    url = item.json.links.self;
  }
  
  // Return normalized structure
  return {
    json: {
      url: url,
      access_token: access_token,
      ...item.json
    }
  };
});
```

### Timestamp Formatting
```javascript
// Formatted timestamp for emails
{{ $now.format('MMM d, yyyy h:mm a') }}
// Output: Feb 23, 2026 3:54 PM
```

### Access Token Reference
```javascript
// Persistent access token in HTTP headers
{{ $('Check Device Count').first().json.access_token }}
```

## 📧 Email Templates

### Success Notification
```
✅ FMC Device Deployment - SUCCESS

Status: Completed Successfully

Devices Deployed:
• ciscoftd02 (192.168.0.202) - Deployment: OK, Health: GOOD  
• ciscoftd03 (192.168.0.203) - Deployment: OK, Health: GOOD

📊 Summary: All devices successfully deployed and operational

Timestamp: Feb 23, 2026 3:54 PM
```

### Failure Notification  
```
❌ FMC Device Deployment - FAILURE

Status: Deployment Failed

Devices:
• ciscoftd02 (192.168.0.202) - Deployment: FAILED
• ciscoftd03 (192.168.0.203) - Deployment: PENDING

📊 Summary: Deployment timeout after 40 retry attempts

Timestamp: Feb 23, 2026 4:24 PM
```

## 🐛 Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| "Access token not found" | Use `{{ $('Check Device Count').first().json.access_token }}` |
| Email shows "undefined" | Verify `email_destination` passes through all nodes |
| URL returns undefined | Check Normalize URL Data code handles both data sources |
| Loop doesn't work | Ensure Wait2 connects to Normalize URL Data |
| Wrong timestamp format | Use `{{ $now.format('MMM d, yyyy h:mm a') }}` |

### Debug Mode
Enable debug logging in n8n:
```env
N8N_LOG_LEVEL=debug
```

## 📚 Documentation

- [📖 Complete Setup Guide](docs/setup-guide.md) - Step-by-step installation
- [⚙️ Configuration Reference](docs/configuration.md) - All settings explained  
- [🔧 Troubleshooting Guide](docs/troubleshooting.md) - Solutions to common issues

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`) 
5. Open a Pull Request

## 📝 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🏷️ Tags

`n8n` `cisco` `fmc` `automation` `devops` `jenkins` `api` `monitoring` `workflows` `cicd`

## 👨‍💻 Author

**jotape75**
- GitHub: [@jotape75](https://github.com/jotape75)
- Created: February 23, 2026

## 🙏 Acknowledgments

- [n8n.io](https://n8n.io) for the excellent workflow automation platform
- Cisco for FMC API documentation and tools
- The DevOps community for automation best practices

---

**⭐ If this project helped you, please give it a star! ⭐**