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
            description: 'Enable log analysis'
        )
        string(
            name: 'LOG_LINES',
            defaultValue: '1000',
            description: 'Number of log lines to analyze'
        )
    }
    
    environment {
        APACHE_LOG = '/var/log/apache2/access.log'
        ERROR_LOG = '/var/log/apache2/error.log'
        REPORT_FILE = "apache_error_report_${BUILD_NUMBER}.txt"
    }
    
    stages {
        stage('Check Apache Installation') {
            steps {
                script {
                    echo '=== Checking if Apache is installed ==='
                    def apacheInstalled = sh(
                        script: 'which apache2 || which httpd',
                        returnStatus: true
                    )
                    
                    if (apacheInstalled != 0) {
                        echo 'Apache not found, will install...'
                        env.NEEDS_INSTALL = 'true'
                    } else {
                        echo 'Apache already installed'
                        env.NEEDS_INSTALL = 'false'
                    }
                }
            }
        }
        
        stage('Install Apache') {
            when {
                expression { params.ACTION == 'full_deployment' || env.NEEDS_INSTALL == 'true' }
            }
            steps {
                script {
                    echo '=== Installing Apache Web Server ==='
                    
                    // Detect OS and install accordingly
                    def osType = sh(script: 'cat /etc/os-release | grep "^ID=" | cut -d= -f2', returnStdout: true).trim()
                    
                    if (osType.contains('ubuntu') || osType.contains('debian')) {
                        echo 'Detected Debian/Ubuntu system'
                        sh '''
                            sudo apt-get update
                            sudo apt-get install -y apache2
                        '''
                        env.APACHE_LOG = '/var/log/apache2/access.log'
                        env.ERROR_LOG = '/var/log/apache2/error.log'
                    } else if (osType.contains('centos') || osType.contains('rhel') || osType.contains('fedora')) {
                        echo 'Detected RedHat/CentOS/Fedora system'
                        sh '''
                            sudo yum install -y httpd || sudo dnf install -y httpd
                        '''
                        env.APACHE_LOG = '/var/log/httpd/access_log'
                        env.ERROR_LOG = '/var/log/httpd/error_log'
                    } else {
                        error("Unsupported operating system: ${osType}")
                    }
                }
            }
        }
        
        stage('Configure Apache') {
            when {
                expression { params.ACTION == 'full_deployment' || params.ACTION == 'config_update' }
            }
            steps {
                echo '=== Configuring Apache ==='
                script {
                    sh '''
                        # Check which service command to use
                        if command -v apache2 &> /dev/null; then
                            SERVICE_NAME="apache2"
                        else
                            SERVICE_NAME="httpd"
                        fi
                        
                        # Enable and start Apache
                        sudo systemctl enable $SERVICE_NAME
                        sudo systemctl start $SERVICE_NAME
                        
                        # Verify Apache is running
                        sudo systemctl status $SERVICE_NAME --no-pager
                    '''
                }
            }
        }
        
        stage('Restart Apache') {
            when {
                expression { params.ACTION == 'restart_only' }
            }
            steps {
                echo '=== Restarting Apache ==='
                sh '''
                    if command -v apache2 &> /dev/null; then
                        sudo systemctl restart apache2
                    else
                        sudo systemctl restart httpd
                    fi
                '''
            }
        }
        
        stage('Verify Apache') {
            steps {
                echo '=== Verifying Apache Installation ==='
                script {
                    sh '''
                        # Check Apache version
                        if command -v apache2 &> /dev/null; then
                            apache2 -v
                        else
                            httpd -v
                        fi
                        
                        # Check if Apache is listening on port 80
                        sudo netstat -tuln | grep :80 || sudo ss -tuln | grep :80
                        
                        # Test local connection
                        curl -I http://localhost || echo "Warning: Could not connect to localhost"
                    '''
                }
            }
        }
        
        stage('Analyze Logs') {
            when {
                expression { params.ANALYZE_LOGS == true }
            }
            steps {
                echo "=== Analyzing Apache Logs (last ${params.LOG_LINES} lines) ==="
                script {
                    sh '''
                        # Determine log file location
                        if [ -f /var/log/apache2/access.log ]; then
                            ACCESS_LOG="/var/log/apache2/access.log"
                            ERROR_LOG="/var/log/apache2/error.log"
                        elif [ -f /var/log/httpd/access_log ]; then
                            ACCESS_LOG="/var/log/httpd/access_log"
                            ERROR_LOG="/var/log/httpd/error_log"
                        else
                            echo "Warning: Apache logs not found"
                            ACCESS_LOG=""
                            ERROR_LOG=""
                        fi
                        
                        # Create report
                        echo "==================================" > ''' + env.REPORT_FILE + '''
                        echo "Apache Error Analysis Report" >> ''' + env.REPORT_FILE + '''
                        echo "Build: ''' + env.BUILD_NUMBER + '''" >> ''' + env.REPORT_FILE + '''
                        echo "Date: $(date)" >> ''' + env.REPORT_FILE + '''
                        echo "==================================" >> ''' + env.REPORT_FILE + '''
                        echo "" >> ''' + env.REPORT_FILE + '''
                        
                        if [ -f "$ACCESS_LOG" ]; then
                            echo "--- 4xx Client Errors ---" >> ''' + env.REPORT_FILE + '''
                            sudo tail -n ''' + params.LOG_LINES + ''' "$ACCESS_LOG" | grep " 4[0-9][0-9] " | wc -l >> ''' + env.REPORT_FILE + '''
                            echo "Total 4xx errors found" >> ''' + env.REPORT_FILE + '''
                            echo "" >> ''' + env.REPORT_FILE + '''
                            
                            echo "Sample 4xx errors:" >> ''' + env.REPORT_FILE + '''
                            sudo tail -n ''' + params.LOG_LINES + ''' "$ACCESS_LOG" | grep " 4[0-9][0-9] " | head -10 >> ''' + env.REPORT_FILE + '''
                            echo "" >> ''' + env.REPORT_FILE + '''
                            
                            echo "--- 5xx Server Errors ---" >> ''' + env.REPORT_FILE + '''
                            sudo tail -n ''' + params.LOG_LINES + ''' "$ACCESS_LOG" | grep " 5[0-9][0-9] " | wc -l >> ''' + env.REPORT_FILE + '''
                            echo "Total 5xx errors found" >> ''' + env.REPORT_FILE + '''
                            echo "" >> ''' + env.REPORT_FILE + '''
                            
                            echo "Sample 5xx errors:" >> ''' + env.REPORT_FILE + '''
                            sudo tail -n ''' + params.LOG_LINES + ''' "$ACCESS_LOG" | grep " 5[0-9][0-9] " | head -10 >> ''' + env.REPORT_FILE + '''
                        else
                            echo "Access log not found or not readable" >> ''' + env.REPORT_FILE + '''
                        fi
                        
                        echo "" >> ''' + env.REPORT_FILE + '''
                        echo "--- Recent Error Log Entries ---" >> ''' + env.REPORT_FILE + '''
                        if [ -f "$ERROR_LOG" ]; then
                            sudo tail -n 20 "$ERROR_LOG" >> ''' + env.REPORT_FILE + '''
                        else
                            echo "Error log not found" >> ''' + env.REPORT_FILE + '''
                        fi
                        
                        # Display report
                        cat ''' + env.REPORT_FILE + '''
                    '''
                }
            }
        }
    }
    
    post {
        always {
            echo '=== Pipeline Execution Complete ==='
            script {
                if (fileExists(env.REPORT_FILE)) {
                    archiveArtifacts artifacts: env.REPORT_FILE, fingerprint: true
                    echo "Report archived: ${env.REPORT_FILE}"
                }
            }
        }
        success {
            echo '✓ Pipeline completed successfully!'
        }
        failure {
            echo '✗ Pipeline failed. Check logs above for details.'
        }
    }
}
