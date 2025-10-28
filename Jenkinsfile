pipeline {
    agent any
    
    environment {
        APACHE_LOG_PATH = '/var/log/apache2/access.log'
        ERROR_LOG_PATH = '/var/log/apache2/error.log'
        REPORT_FILE = "apache_error_report_${BUILD_NUMBER}.txt"
        APACHE_SERVICE = 'apache2'
    }
    
    parameters {
        choice(
            name: 'ACTION',
            choices: ['install_and_check', 'check_logs_only', 'restart', 'full_deployment'],
            description: 'Select action to perform'
        )
        booleanParam(
            name: 'ANALYZE_LOGS',
            defaultValue: true,
            description: 'Analyze Apache logs for 4xx and 5xx errors'
        )
        string(
            name: 'LOG_LINES',
            defaultValue: '1000',
            description: 'Number of recent log lines to analyze'
        )
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
    }
    
    stages {
        stage('Initialize') {
            steps {
                script {
                    echo """
                    ==========================================
                    Apache Deployment & Monitoring Pipeline
                    ==========================================
                    Build Number: ${BUILD_NUMBER}
                    Date: ${new Date()}
                    Action: ${params.ACTION}
                    Analyze Logs: ${params.ANALYZE_LOGS}
                    ==========================================
                    """
                }
            }
        }
        
        stage('Check Prerequisites') {
            steps {
                script {
                    echo "Checking system prerequisites..."
                    sh '''
                        echo "Operating System: $(lsb_release -d | cut -f2)"
                        echo "Kernel: $(uname -r)"
                        echo "User: $(whoami)"
                        
                        # Check if sudo is available
                        which sudo || (echo "ERROR: sudo not found" && exit 1)
                        
                        # Check disk space
                        df -h / | tail -1
                    '''
                }
            }
        }
        
        stage('Install Apache2') {
            when {
                expression { params.ACTION in ['install_and_check', 'full_deployment'] }
            }
            steps {
                script {
                    echo "Installing Apache2 server..."
                    sh '''
                        set -e
                        
                        # Check if Apache is already installed
                        if dpkg -l | grep -qw apache2; then
                            echo "Apache2 is already installed"
                            APACHE_VERSION=$(apache2 -v | head -1)
                            echo "Current version: ${APACHE_VERSION}"
                        else
                            echo "Installing Apache2..."
                            sudo apt-get update -qq
                            sudo DEBIAN_FRONTEND=noninteractive apt-get install apache2 -y
                            echo "Apache2 installation completed"
                        fi
                        
                        # Enable Apache service
                        sudo systemctl enable apache2
                        
                        # Start Apache service
                        sudo systemctl start apache2
                        
                        echo "Apache2 is now running"
                    '''
                }
            }
        }
        
        stage('Configure Apache') {
            when {
                expression { params.ACTION == 'full_deployment' }
            }
            steps {
                script {
                    echo "Configuring Apache2..."
                    sh '''
                        # Enable useful modules
                        sudo a2enmod rewrite || true
                        sudo a2enmod headers || true
                        
                        # Create custom index page
                        cat <<HTML | sudo tee /var/www/html/index.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Apache Deployed by Jenkins</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            display: flex;
            align-items: center;
            justify-content: center;
            padding: 20px;
        }
        .container {
            background: white;
            padding: 40px;
            border-radius: 20px;
            box-shadow: 0 20px 60px rgba(0,0,0,0.3);
            max-width: 600px;
            text-align: center;
        }
        h1 {
            color: #667eea;
            margin-bottom: 20px;
            font-size: 2.5em;
        }
        .badge {
            display: inline-block;
            background: #10b981;
            color: white;
            padding: 8px 16px;
            border-radius: 20px;
            margin: 10px 0;
            font-weight: bold;
        }
        .info {
            background: #f3f4f6;
            padding: 20px;
            border-radius: 10px;
            margin: 20px 0;
            text-align: left;
        }
        .info-item {
            padding: 8px 0;
            border-bottom: 1px solid #e5e7eb;
        }
        .info-item:last-child { border-bottom: none; }
        .label {
            font-weight: bold;
            color: #6b7280;
            display: inline-block;
            width: 120px;
        }
        .emoji { font-size: 3em; margin: 20px 0; }
    </style>
</head>
<body>
    <div class="container">
        <div class="emoji">🚀</div>
        <h1>Success!</h1>
        <div class="badge">Deployed by Jenkins</div>
        <p style="margin: 20px 0; color: #6b7280;">
            Apache2 web server has been successfully deployed and configured
        </p>
        <div class="info">
            <div class="info-item">
                <span class="label">Build:</span>
                <span>#${BUILD_NUMBER}</span>
            </div>
            <div class="info-item">
                <span class="label">Server:</span>
                <span>$(hostname)</span>
            </div>
            <div class="info-item">
                <span class="label">Deployed:</span>
                <span>$(date)</span>
            </div>
            <div class="info-item">
                <span class="label">Status:</span>
                <span style="color: #10b981;">● Active</span>
            </div>
        </div>
    </div>
</body>
</html>
HTML
                        
                        # Set proper permissions
                        sudo chown -R www-data:www-data /var/www/html/
                        sudo chmod -R 755 /var/www/html/
                        
                        # Test Apache configuration
                        sudo apache2ctl configtest
                        
                        # Reload Apache to apply changes
                        sudo systemctl reload apache2
                        
                        echo "Apache configuration completed successfully"
                    '''
                }
            }
        }
        
        stage('Verify Apache Status') {
            steps {
                script {
                    echo "Verifying Apache2 installation..."
                    sh '''
                        echo "=== Apache Service Status ==="
                        sudo systemctl status apache2 --no-pager | head -15
                        
                        echo ""
                        echo "=== Apache Version ==="
                        apache2 -v
                        
                        echo ""
                        echo "=== Listening Ports ==="
                        sudo ss -tuln | grep :80 || sudo netstat -tuln | grep :80
                        
                        echo ""
                        echo "=== Apache Process Count ==="
                        ps aux | grep apache2 | grep -v grep | wc -l
                        
                        echo ""
                        echo "=== Testing HTTP Response ==="
                        HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost)
                        echo "HTTP Status Code: ${HTTP_CODE}"
                        
                        if [ "${HTTP_CODE}" != "200" ]; then
                            echo "ERROR: Apache is not responding correctly!"
                            exit 1
                        fi
                        
                        echo "✓ Apache is running and responding correctly"
                    '''
                }
            }
        }
        
        stage('Analyze Apache Logs') {
            when {
                expression { params.ANALYZE_LOGS == true }
            }
            steps {
                script {
                    echo "Analyzing Apache logs for 4xx and 5xx errors..."
                    sh '''
                        set +e  # Don't exit on errors in this stage
                        
                        # Create report header
                        cat > ${REPORT_FILE} << 'HEADER'
==========================================
  Apache Error Analysis Report
==========================================
HEADER
                        
                        echo "Generated: $(date)" >> ${REPORT_FILE}
                        echo "Build Number: ${BUILD_NUMBER}" >> ${REPORT_FILE}
                        echo "Log Lines Analyzed: ${LOG_LINES}" >> ${REPORT_FILE}
                        echo "==========================================\n" >> ${REPORT_FILE}
                        
                        # Generate some test traffic if logs are empty
                        if [ ! -f "${APACHE_LOG_PATH}" ] || [ ! -s "${APACHE_LOG_PATH}" ]; then
                            echo "Generating test traffic..." | tee -a ${REPORT_FILE}
                            for i in {1..20}; do
                                curl -s http://localhost/ > /dev/null 2>&1
                                curl -s http://localhost/nonexistent 2>&1 | head -1
                            done
                            sleep 2
                        fi
                        
                        # Check if access log exists
                        if [ -f "${APACHE_LOG_PATH}" ]; then
                            echo "=== ACCESS LOG ANALYSIS ===" >> ${REPORT_FILE}
                            echo "Log File: ${APACHE_LOG_PATH}\n" >> ${REPORT_FILE}
                            
                            # Get total requests
                            TOTAL_REQUESTS=$(sudo tail -${LOG_LINES} ${APACHE_LOG_PATH} 2>/dev/null | wc -l)
                            echo "Total Requests Analyzed: ${TOTAL_REQUESTS}" >> ${REPORT_FILE}
                            
                            # Count 4xx errors
                            FOUR_XX_COUNT=$(sudo tail -${LOG_LINES} ${APACHE_LOG_PATH} 2>/dev/null | grep -oE ' (4[0-9]{2}) ' | wc -l)
                            echo "4xx Client Errors Found: ${FOUR_XX_COUNT}" >> ${REPORT_FILE}
                            
                            # Count 5xx errors
                            FIVE_XX_COUNT=$(sudo tail -${LOG_LINES} ${APACHE_LOG_PATH} 2>/dev/null | grep -oE ' (5[0-9]{2}) ' | wc -l)
                            echo "5xx Server Errors Found: ${FIVE_XX_COUNT}" >> ${REPORT_FILE}
                            
                            # Calculate error rate
                            if [ ${TOTAL_REQUESTS} -gt 0 ]; then
                                TOTAL_ERRORS=$((FOUR_XX_COUNT + FIVE_XX_COUNT))
                                ERROR_RATE=$(awk "BEGIN {printf \"%.2f\", (${TOTAL_ERRORS}/${TOTAL_REQUESTS})*100}")
                                echo "Overall Error Rate: ${ERROR_RATE}%" >> ${REPORT_FILE}
                            fi
                            
                            echo "" >> ${REPORT_FILE}
                            
                            # Status code breakdown
                            echo "=== STATUS CODE BREAKDOWN ===" >> ${REPORT_FILE}
                            sudo tail -${LOG_LINES} ${APACHE_LOG_PATH} 2>/dev/null | \
                                awk '{print $9}' | grep -E '^[0-9]{3}$' | sort | uniq -c | sort -rn >> ${REPORT_FILE}
                            
                            echo "" >> ${REPORT_FILE}
                            
                            # Show 4xx errors in detail
                            echo "=== DETAILED 4XX ERRORS ===" >> ${REPORT_FILE}
                            if [ ${FOUR_XX_COUNT} -gt 0 ]; then
                                sudo tail -${LOG_LINES} ${APACHE_LOG_PATH} 2>/dev/null | \
                                    grep -E ' (4[0-9]{2}) ' | tail -10 >> ${REPORT_FILE}
                            else
                                echo "No 4xx errors found" >> ${REPORT_FILE}
                            fi
                            
                            echo "" >> ${REPORT_FILE}
                            
                            # Show 5xx errors in detail
                            echo "=== DETAILED 5XX ERRORS ===" >> ${REPORT_FILE}
                            if [ ${FIVE_XX_COUNT} -gt 0 ]; then
                                sudo tail -${LOG_LINES} ${APACHE_LOG_PATH} 2>/dev/null | \
                                    grep -E ' (5[0-9]{2}) ' | tail -10 >> ${REPORT_FILE}
                            else
                                echo "No 5xx errors found" >> ${REPORT_FILE}
                            fi
                            
                            echo "" >> ${REPORT_FILE}
                            
                            # Most requested URLs
                            echo "=== TOP 10 REQUESTED URLs ===" >> ${REPORT_FILE}
                            sudo tail -${LOG_LINES} ${APACHE_LOG_PATH} 2>/dev/null | \
                                awk '{print $7}' | sort | uniq -c | sort -rn | head -10 >> ${REPORT_FILE}
                            
                            echo "" >> ${REPORT_FILE}
                            
                            # Top client IPs
                            echo "=== TOP 10 CLIENT IPs ===" >> ${REPORT_FILE}
                            sudo tail -${LOG_LINES} ${APACHE_LOG_PATH} 2>/dev/null | \
                                awk '{print $1}' | sort | uniq -c | sort -rn | head -10 >> ${REPORT_FILE}
                            
                        else
                            echo "WARNING: Access log not found at ${APACHE_LOG_PATH}" >> ${REPORT_FILE}
                        fi
                        
                        echo "" >> ${REPORT_FILE}
                        
                        # Check error log
                        if [ -f "${ERROR_LOG_PATH}" ]; then
                            echo "=== ERROR LOG (Last 30 lines) ===" >> ${REPORT_FILE}
                            sudo tail -30 ${ERROR_LOG_PATH} >> ${REPORT_FILE} 2>/dev/null
                        else
                            echo "WARNING: Error log not found at ${ERROR_LOG_PATH}" >> ${REPORT_FILE}
                        fi
                        
                        echo "" >> ${REPORT_FILE}
                        echo "==========================================" >> ${REPORT_FILE}
                        echo "Report generation completed" >> ${REPORT_FILE}
                        echo "==========================================" >> ${REPORT_FILE}
                        
                        # Display the report in console
                        cat ${REPORT_FILE}
                    '''
                    
                    // Archive the report as an artifact
                    archiveArtifacts artifacts: "${env.REPORT_FILE}",
                                     fingerprint: true,
                                     allowEmptyArchive: true
                }
            }
        }
        
        stage('Restart Apache') {
            when {
                expression { params.ACTION == 'restart' }
            }
            steps {
                script {
                    echo "Restarting Apache2 service..."
                    sh '''
                        sudo systemctl restart apache2
                        sleep 3
                        sudo systemctl status apache2 --no-pager | head -10
                    '''
                }
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    echo "Performing final health check..."
                    sh '''
                        echo "=== Final Health Check ==="
                        
                        # Test HTTP response
                        echo "Testing HTTP response..."
                        curl -I http://localhost 2>&1 | head -10
                        
                        # Check disk usage
                        echo ""
                        echo "Disk Usage:"
                        df -h / | tail -1
                        
                        # Check memory
                        echo ""
                        echo "Memory Usage:"
                        free -h | grep Mem
                        
                        # Apache resource usage
                        echo ""
                        echo "Apache Resource Usage:"
                        ps aux | grep apache2 | grep -v grep | \
                            awk '{cpu+=$3; mem+=$4} END {printf "CPU: %.1f%%, Memory: %.1f%%\n", cpu, mem}'
                        
                        echo ""
                        echo "✓ Health check completed successfully"
                    '''
                }
            }
        }
    }
    
    post {
        success {
            script {
                echo """
                ==========================================
                ✓✓✓ PIPELINE COMPLETED SUCCESSFULLY ✓✓✓
                ==========================================
                Build: #${BUILD_NUMBER}
                Duration: ${currentBuild.durationString}
                Apache Status: RUNNING
                Access: http://localhost
                Report: ${env.REPORT_FILE}
                ==========================================
                """
            }
        }
        
        failure {
            script {
                echo """
                ==========================================
                ✗✗✗ PIPELINE FAILED ✗✗✗
                ==========================================
                Build: #${BUILD_NUMBER}
                Check the console output above for details
                ==========================================
                """
            }
        }
        
        always {
            script {
                echo "Pipeline execution completed at: ${new Date()}"
            }
        }
    }
}
