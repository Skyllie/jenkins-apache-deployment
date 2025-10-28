pipeline {
    agent any

    parameters {
        choice(
            name: 'ACTION',
            choices: ['full_deployment', 'restart_only', 'config_update'],
            description: 'Select deployment action'
        )
        booleanParam(
            name: 'ANALYZE_LOGS',
            defaultValue: true,
            description: 'Enable Apache log analysis'
        )
        string(
            name: 'LOG_LINES',
            defaultValue: '1000',
            description: 'Number of log lines to analyze'
        )
    }

    environment {
        REPORT_FILE = "apache_error_report_${BUILD_NUMBER}.txt"
    }

    stages {

        stage('Detect OS & Apache Paths') {
            steps {
                script {
                    echo '🔍 Detecting OS and Apache installation...'
                    env.OS_ID = sh(
                        script: "grep '^ID=' /etc/os-release | cut -d= -f2 | tr -d '\"'",
                        returnStdout: true
                    ).trim()

                    if (fileExists('/usr/sbin/apache2')) {
                        env.APACHE_CMD = 'apache2'
                        env.SERVICE_NAME = 'apache2'
                        env.ACCESS_LOG = '/var/log/apache2/access.log'
                        env.ERROR_LOG = '/var/log/apache2/error.log'
                    } else if (fileExists('/usr/sbin/httpd')) {
                        env.APACHE_CMD = 'httpd'
                        env.SERVICE_NAME = 'httpd'
                        env.ACCESS_LOG = '/var/log/httpd/access_log'
                        env.ERROR_LOG = '/var/log/httpd/error_log'
                    } else {
                        env.APACHE_CMD = ''
                    }

                    echo "Detected OS: ${env.OS_ID}"
                    echo "Apache binary: ${env.APACHE_CMD ?: 'Not found'}"
                }
            }
        }

        stage('Install Apache') {
            when {
                expression { params.ACTION == 'full_deployment' && !env.APACHE_CMD }
            }
            steps {
                script {
                    echo '📦 Installing Apache Web Server...'
                    if (env.OS_ID in ['ubuntu', 'debian']) {
                        sh 'sudo apt-get update && sudo apt-get install -y apache2'
                    } else if (env.OS_ID in ['centos', 'rhel', 'fedora']) {
                        sh 'sudo yum install -y httpd || sudo dnf install -y httpd'
                    } else {
                        error("Unsupported OS: ${env.OS_ID}")
                    }
                }
            }
        }

        stage('Configure & Start Apache') {
            when {
                expression { params.ACTION in ['full_deployment', 'config_update'] }
            }
            steps {
                script {
                    echo '⚙️ Configuring and starting Apache...'
                    sh """
                        sudo systemctl enable ${env.SERVICE_NAME}
                        sudo systemctl restart ${env.SERVICE_NAME}
                        sudo systemctl status ${env.SERVICE_NAME} --no-pager
                    """
                }
            }
        }

        stage('Restart Apache') {
            when {
                expression { params.ACTION == 'restart_only' }
            }
            steps {
                echo '🔄 Restarting Apache service...'
                sh "sudo systemctl restart ${env.SERVICE_NAME}"
            }
        }

        stage('Verify Apache') {
            steps {
                echo '✅ Verifying Apache installation and status...'
                sh """
                    ${env.APACHE_CMD} -v
                    sudo ss -tuln | grep ':80' || echo 'Apache not listening on port 80!'
                    curl -I http://localhost || echo 'Warning: Apache not responding'
                """
            }
        }

        stage('Analyze Logs') {
            when {
                expression { params.ANALYZE_LOGS }
            }
            steps {
                script {
                    echo "📊 Generating Apache log report (${params.LOG_LINES} lines)..."
                    def logScript = """
                        set -e
                        REPORT="${env.REPORT_FILE}"
                        ACCESS="${env.ACCESS_LOG}"
                        ERROR="${env.ERROR_LOG}"

                        echo "Apache Log Report - Build #${env.BUILD_NUMBER}" > $REPORT
                        echo "Generated: $(date)" >> $REPORT
                        echo "" >> $REPORT

                        if [ -f "$ACCESS" ]; then
                            echo "--- 4xx Client Errors ---" >> $REPORT
                            sudo tail -n ${params.LOG_LINES} "$ACCESS" | grep ' 4[0-9][0-9] ' | wc -l >> $REPORT
                            echo "--- 5xx Server Errors ---" >> $REPORT
                            sudo tail -n ${params.LOG_LINES} "$ACCESS" | grep ' 5[0-9][0-9] ' | wc -l >> $REPORT
                        else
                            echo "Access log not found" >> $REPORT
                        fi

                        echo "" >> $REPORT
                        echo "--- Recent Error Log Entries ---" >> $REPORT
                        [ -f "$ERROR" ] && sudo tail -n 20 "$ERROR" >> $REPORT || echo "Error log not found" >> $REPORT

                        cat $REPORT
                    """
                    writeFile file: 'analyze_logs.sh', text: logScript
                    sh 'chmod +x analyze_logs.sh && ./analyze_logs.sh'
                }
            }
        }
    }

    post {
        always {
            echo '📁 Archiving artifacts...'
            script {
                if (fileExists(env.REPORT_FILE)) {
                    archiveArtifacts artifacts: env.REPORT_FILE, fingerprint: true
                }
            }
        }
        success {
            echo '🎉 Apache pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed. Please check logs.'
        }
    }
}
