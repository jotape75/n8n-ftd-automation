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

3. **Configure credentials** in n8n (Vault token, FMC host, SMTP settings)

4. **Set up Jenkins webhook** using the provided `Jenkinsfile`

## 📁 Repository Structure

```
fmc-n8n-workflows/
├── README.md                                                    # This file
├── Jenkinsfile                                                  # Jenkins pipeline for webhook trigger
├── workflows/
│   ├── ftd-main-orchestrator.json                              # 🎯 Main deployment orchestrator
│   ├── fmc-authentication-subworkflow.json                     # 🔐 FMC authentication handler
│   ├── fmc-device-registration-subworkflow.json                # 📝 Device registration manager
│   └── fmc-device-registration-monitoring-subworkflow.json     # 📊 Deployment monitoring & retry
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

## 📚 Documentation

- [� Jenkinsfile](Jenkinsfile) - Jenkins pipeline for triggering deployments via webhook

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