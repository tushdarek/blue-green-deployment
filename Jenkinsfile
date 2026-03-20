pipeline {
    agent any
    
    parameters {
        choice(name: 'DEPLOY_TO', choices: ['blue', 'green'], description: 'Which environment is INACTIVE right now?')
        booleanParam(name: 'AUTO_SWITCH', defaultValue: true, description: 'Switch traffic after success?')
        choice(name: 'VERSION', choices: ['v1', 'v2'], description: 'Which version to deploy?')
    }

    environment {
        // YOUR AWS RESOURCES - Update these with your actual values
        LISTENER_ARN = 'arn:aws:elasticloadbalancing:ap-south-1:098688552647:loadbalancer/app/BlueGreen-ALB/c43469f376ace0e9'
        BLUE_TG_ARN   = 'arn:aws:elasticloadbalancing:ap-south-1:098688552647:targetgroup/TG-Blue/1b9b9dc59587436b'
        GREEN_TG_ARN  = 'arn:aws:elasticloadbalancing:ap-south-1:098688552647:targetgroup/TG-Green/00e48e7f78dbd3ac'
        
        // Your EC2 Instance IPs
        BLUE_IP       = '13.201.124.219'
        GREEN_IP      = '65.0.55.222'
        
        // Application Configuration
        APP_PORT = '80'
        HEALTH_ENDPOINT = '/health'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', 
                    url: 'https://github.com/tushdarek/blue-green-deployment.git'
                echo "✅ Code checked out from GitHub"
            }
        }
        
        stage('Determine Environment') {
            steps {
                script {
                    if (params.DEPLOY_TO == 'blue') {
                        env.INACTIVE_IP = env.BLUE_IP
                        env.INACTIVE_TG = env.BLUE_TG_ARN
                        env.ACTIVE_TG = env.GREEN_TG_ARN
                        env.INACTIVE_ENV = 'Blue'
                        env.ACTIVE_ENV = 'Green'
                    } else {
                        env.INACTIVE_IP = env.GREEN_IP
                        env.INACTIVE_TG = env.GREEN_TG_ARN
                        env.ACTIVE_TG = env.BLUE_TG_ARN
                        env.INACTIVE_ENV = 'Green'
                        env.ACTIVE_ENV = 'Blue'
                    }
                    
                    echo """
                    ========================================
                    DEPLOYMENT PLAN
                    ========================================
                    Deploying to: ${env.INACTIVE_ENV} Environment
                    Target IP: ${env.INACTIVE_IP}
                    Current Active: ${env.ACTIVE_ENV} Environment
                    Version: ${params.VERSION}
                    Auto Switch: ${params.AUTO_SWITCH}
                    ========================================
                    """
                }
            }
        }
        
        stage('Create Deployment Package') {
            steps {
                script {
                    // Create version-specific content
                    sh """
                        # Create index.html with version info
                        cat > index.html <<EOF
                        <!DOCTYPE html>
                        <html>
                        <head>
                            <title>Blue-Green Deployment - Version ${params.VERSION}</title>
                            <style>
                                body { font-family: Arial, sans-serif; text-align: center; padding: 50px; }
                                .version { color: ${params.VERSION == 'v1' ? 'blue' : 'green'}; font-size: 24px; }
                                .env { color: #666; margin-top: 20px; }
                            </style>
                        </head>
                        <body>
                            <h1>Blue-Green Deployment Demo</h1>
                            <div class="version">Version: ${params.VERSION}</div>
                            <div class="env">Environment: ${env.INACTIVE_ENV}</div>
                            <div>Deployed at: \$(date)</div>
                            <div>Host: \$(hostname)</div>
                        </body>
                        </html>
                        EOF
                        
                        # Create health check endpoint
                        echo "OK" > health.html
                        
                        # Create a simple status page
                        cat > status.html <<EOF
                        <html>
                        <body>
                            <h2>Status Page</h2>
                            <p>Environment: ${env.INACTIVE_ENV}</p>
                            <p>Version: ${params.VERSION}</p>
                            <p>Status: Healthy</p>
                        </body>
                        </html>
                        EOF
                    """
                    echo "✅ Deployment package created with ${params.VERSION}"
                }
            }
        }
        
        stage('Deploy to Inactive Environment') {
            steps {
                script {
                    echo "📦 Deploying to ${env.INACTIVE_ENV} environment (${env.INACTIVE_IP})..."
                    
                    sshagent(credentials: ['ec2-ssh-key']) {
                        try {
                            sh """
                                # Backup current deployment on target instance
                                ssh -o StrictHostKeyChecking=no ubuntu@${env.INACTIVE_IP} '
                                    timestamp=\$(date +%Y%m%d_%H%M%S)
                                    if [ -d /var/www/html ]; then
                                        sudo cp -r /var/www/html /var/www/html_backup_\${timestamp}
                                        echo "Backup created: html_backup_\${timestamp}"
                                    fi
                                    sudo rm -rf /var/www/html/*
                                    sudo mkdir -p /var/www/html
                                '
                                
                                # Copy new files
                                echo "Copying files to ${env.INACTIVE_ENV}..."
                                scp -r * ubuntu@${env.INACTIVE_IP}:/var/www/html/
                                
                                # Set permissions and restart Nginx
                                ssh ubuntu@${env.INACTIVE_IP} '
                                    sudo chown -R www-data:www-data /var/www/html
                                    sudo chmod -R 755 /var/www/html
                                    sudo systemctl restart nginx
                                    sleep 3
                                    echo "Nginx restarted successfully"
                                '
                            """
                            echo "✅ Deployment completed to ${env.INACTIVE_ENV}"
                        } catch (Exception e) {
                            error("Deployment failed: ${e.getMessage()}")
                        }
                    }
                }
            }
        }
        
        stage('Health Check Validation') {
            steps {
                script {
                    echo "🔍 Performing health checks on ${env.INACTIVE_ENV}..."
                    
                    def maxRetries = 6
                    def healthCheckPassed = false
                    
                    // Give time for application to stabilize
                    sleep(10)
                    
                    for (int i = 1; i <= maxRetries; i++) {
                        echo "Health check attempt ${i}/${maxRetries}..."
                        
                        try {
                            // Check 1: Target Group Health via AWS
                            def tgHealth = sh(
                                script: "aws elbv2 describe-target-health --target-group-arn ${env.INACTIVE_TG} --query 'TargetHealthDescriptions[0].TargetHealth.State' --output text",
                                returnStdout: true
                            ).trim()
                            echo "  ✓ AWS Target Group Health: ${tgHealth}"
                            
                            // Check 2: Application Health Endpoint
                            def healthStatus = sh(
                                script: "curl -s -o /dev/null -w '%{http_code}' http://${env.INACTIVE_IP}${HEALTH_ENDPOINT}",
                                returnStdout: true
                            ).trim()
                            echo "  ✓ Health Endpoint Status: ${healthStatus}"
                            
                            // Check 3: Main Page Accessibility
                            def mainPageStatus = sh(
                                script: "curl -s -o /dev/null -w '%{http_code}' http://${env.INACTIVE_IP}/",
                                returnStdout: true
                            ).trim()
                            echo "  ✓ Main Page Status: ${mainPageStatus}"
                            
                            // Check 4: Version Verification
                            def deployedVersion = sh(
                                script: "curl -s http://${env.INACTIVE_IP}/ | grep -o 'Version: [^<]*' | cut -d' ' -f2",
                                returnStdout: true
                            ).trim()
                            echo "  ✓ Deployed Version: ${deployedVersion}"
                            
                            // All checks passed
                            if (tgHealth == "healthy" && healthStatus == "200" && mainPageStatus == "200") {
                                healthCheckPassed = true
                                echo """
                                ✅ ALL HEALTH CHECKS PASSED!
                                =================================
                                Target Group: ${tgHealth}
                                Health Endpoint: ${healthStatus}
                                Main Page: ${mainPageStatus}
                                Version: ${deployedVersion}
                                =================================
                                """
                                break
                            }
                        } catch (Exception e) {
                            echo "  ✗ Health check attempt ${i} failed: ${e.getMessage()}"
                        }
                        
                        if (i < maxRetries) {
                            echo "Waiting 10 seconds before next attempt..."
                            sleep(10)
                        }
                    }
                    
                    if (!healthCheckPassed) {
                        error("Health check failed after ${maxRetries} attempts")
                    }
                }
            }
        }
        
        stage('Switch Traffic') {
            when { expression { return params.AUTO_SWITCH } }
            steps {
                script {
                    echo "🔄 Switching traffic from ${env.ACTIVE_ENV} to ${env.INACTIVE_ENV}..."
                    
                    try {
                        // Get current listener configuration
                        def currentTG = sh(
                            script: "aws elbv2 describe-listeners --listener-arns ${LISTENER_ARN} --query 'Listeners[0].DefaultActions[0].TargetGroupArn' --output text",
                            returnStdout: true
                        ).trim()
                        echo "Current active target group: ${currentTG}"
                        
                        // Switch traffic to inactive environment
                        sh """
                            aws elbv2 modify-listener \
                                --listener-arn ${LISTENER_ARN} \
                                --default-actions Type=forward,TargetGroupArn=${INACTIVE_TG}
                        """
                        
                        // Verify the switch
                        sleep(5)
                        def newTG = sh(
                            script: "aws elbv2 describe-listeners --listener-arns ${LISTENER_ARN} --query 'Listeners[0].DefaultActions[0].TargetGroupArn' --output text",
                            returnStdout: true
                        ).trim()
                        
                        if (newTG == INACTIVE_TG) {
                            echo """
                            ✅ TRAFFIC SWITCHED SUCCESSFULLY!
                            =================================
                            Previous Active: ${env.ACTIVE_ENV}
                            New Active: ${env.INACTIVE_ENV}
                            Target Group: ${newTG}
                            =================================
                            """
                        } else {
                            error("Traffic switch verification failed")
                        }
                    } catch (Exception e) {
                        error("Traffic switch failed: ${e.getMessage()}")
                    }
                }
            }
        }
        
        stage('Final Verification') {
            steps {
                script {
                    echo "🔍 Performing final verification on production..."
                    sleep(10)
                    
                    // Get ALB DNS
                    def albDNS = sh(
                        script: "aws elbv2 describe-load-balancers --names BlueGreen-ALB --query 'LoadBalancers[0].DNSName' --output text",
                        returnStdout: true
                    ).trim()
                    
                    echo "ALB DNS: ${albDNS}"
                    
                    // Test production endpoint
                    def response = sh(
                        script: "curl -s http://${albDNS}/ | grep -i version",
                        returnStdout: true
                    ).trim()
                    
                    echo "Production Response: ${response}"
                    
                    if (response.contains(params.VERSION)) {
                        echo "✅ Final verification passed - Correct version is live!"
                    } else {
                        echo "⚠️ Warning: Production may not have the latest version"
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo """
            ╔══════════════════════════════════════════════════════╗
            ║              ✅ DEPLOYMENT SUCCESSFUL!                ║
            ╠══════════════════════════════════════════════════════╣
            ║ Version: ${params.VERSION}                                    ║
            ║ Active Environment: ${env.INACTIVE_ENV}                         ║
            ║ Previous Environment: ${env.ACTIVE_ENV}                         ║
            ║ Auto Switch: ${params.AUTO_SWITCH}                                 ║
            ╚══════════════════════════════════════════════════════╝
            
            🌐 Access your application at: http://${env.BLUE_IP} or http://${env.GREEN_IP}
            🔄 ALB DNS: http://<your-alb-dns>
            """
        }
        
        failure {
            echo """
            ╔══════════════════════════════════════════════════════╗
            ║              ❌ DEPLOYMENT FAILED!                    ║
            ╠══════════════════════════════════════════════════════╣
            ║ Initiating automatic rollback...                     ║
            ╚══════════════════════════════════════════════════════╝
            """
            
            script {
                echo "🔄 Rolling back to ${env.ACTIVE_ENV} environment..."
                
                try {
                    // Get current active before rollback
                    def failedTG = sh(
                        script: "aws elbv2 describe-listeners --listener-arns ${LISTENER_ARN} --query 'Listeners[0].DefaultActions[0].TargetGroupArn' --output text",
                        returnStdout: true
                    ).trim()
                    
                    // Switch back to previous environment
                    sh """
                        aws elbv2 modify-listener \
                            --listener-arn ${LISTENER_ARN} \
                            --default-actions Type=forward,TargetGroupArn=${ACTIVE_TG}
                    """
                    
                    sleep(5)
                    
                    // Verify rollback
                    def rolledBackTG = sh(
                        script: "aws elbv2 describe-listeners --listener-arns ${LISTENER_ARN} --query 'Listeners[0].DefaultActions[0].TargetGroupArn' --output text",
                        returnStdout: true
                    ).trim()
                    
                    echo """
                    ✅ ROLLBACK COMPLETED!
                    =================================
                    Failed deployment to: ${env.INACTIVE_ENV}
                    Reverted to: ${env.ACTIVE_ENV}
                    Version attempted: ${params.VERSION}
                    Current active: ${rolledBackTG}
                    =================================
                    
                    📋 NEXT STEPS:
                    1. Check application logs on ${env.INACTIVE_IP}
                    2. Verify Nginx configuration
                    3. Review deployment package
                    4. Fix issues and retry deployment
                    """
                } catch (Exception e) {
                    echo """
                    ⚠️ CRITICAL: Rollback also failed!
                    Error: ${e.getMessage()}
                    
                    MANUAL INTERVENTION REQUIRED:
                    Run this command to manually rollback:
                    aws elbv2 modify-listener --listener-arn ${LISTENER_ARN} --default-actions Type=forward,TargetGroupArn=${ACTIVE_TG}
                    """
                    currentBuild.result = 'UNSTABLE'
                }
            }
        }
        
        always {
            script {
                echo """
                📊 DEPLOYMENT SUMMARY
                =================================
                Job: ${env.JOB_NAME}
                Build: #${env.BUILD_NUMBER}
                Status: ${currentBuild.currentResult}
                Version: ${params.VERSION}
                Target: ${env.INACTIVE_ENV}
                Time: ${new Date()}
                Duration: ${currentBuild.durationString}
                =================================
                """
                
                // Cleanup old backups (keep last 3)
                sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${INACTIVE_IP} '
                        cd /var/www && ls -t html_backup_* 2>/dev/null | tail -n +4 | xargs -r sudo rm -rf
                        echo "Cleaned up old backups"
                    ' || true
                """
            }
        }
    }
}
