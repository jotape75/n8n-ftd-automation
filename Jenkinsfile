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
                        httpRequest(
                            url: 'http://your-n8n:5678/webhook/device-registration',
                            httpMode: 'POST',
                            contentType: 'APPLICATION_JSON',
                            requestBody: """
                            {
                                "devices": [
                                    {
                                        "name": "${params.DEVICE1_NAME}",
                                        "hostName": "${params.DEVICE1_HOST}",
                                        "regKey": {
                                            "plaintext": "${params.DEVICE1_REGKEY}"
                                        }
                                    },
                                    {
                                        "name": "${params.DEVICE2_NAME}",
                                        "hostName": "${params.DEVICE2_HOST}",
                                        "regKey": {
                                            "plaintext": "${params.DEVICE2_REGKEY}"
                                        }
                                    }
                                ],
                                "notification": {
                                    "email": "${params.EMAIL_RECIPIENT}"
                                }
                            }
                            """
                        )
                    }
                }
            }
        }
    }
}
