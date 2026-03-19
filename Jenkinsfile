pipeline {
    agent any

    environment {
        AWS_REGION           = 'eu-north-1'
        ALB_ARN              = credentials('ALB_ARN')
        LISTENER_ARN         = credentials('LISTENER_ARN')
        BLUE_TG_ARN          = credentials('BLUE_TG_ARN')
        GREEN_TG_ARN         = credentials('GREEN_TG_ARN')
        BLUE_EC2_IP          = credentials('BLUE_EC2_IP')
        GREEN_EC2_IP         = credentials('GREEN_EC2_IP')
        HEALTH_CHECK_RETRIES = '5'
        HEALTH_CHECK_DELAY   = '10'
        APP_PORT             = '80'
    }

    parameters {
        string(name: 'APP_VERSION', defaultValue: 'v2.0',    description: 'Version to deploy')
        string(name: 'GITHUB_REPO', defaultValue: 'https://github.com/tushdarek/blue-green-deployment.git', description: 'Source repo URL')
        string(name: 'BRANCH',      defaultValue: 'main',    description: 'Branch to deploy')
    }

    stages {

        // ─────────────────────────────────────────────
        // STAGE 1 - Detect Active Environment
        // ─────────────────────────────────────────────
        stage('Detect Active Environment') {
            steps {
                script {
                    echo "🔍 Detecting active environment..."

                    def activeArn = sh(
                        script: """
                            aws elbv2 describe-listeners \
                                --listener-arns ${env.LISTENER_ARN} \
                                --region ${env.AWS_REGION} \
                                --query 'Listeners[0].DefaultActions[0].ForwardConfig.TargetGroups[0].TargetGroupArn' \
                                --output text
                        """,
                        returnStdout: true
                    ).trim()

                    if (activeArn == env.BLUE_TG_ARN) {
                        env.ACTIVE_ENV      = 'blue'
                        env.INACTIVE_ENV    = 'green'
                        env.ACTIVE_TG_ARN   = env.BLUE_TG_ARN
                        env.INACTIVE_TG_ARN = env.GREEN_TG_ARN
                        env.DEPLOY_IP       = env.GREEN_EC2_IP
                    } else {
                        env.ACTIVE_ENV      = 'green'
                        env.INACTIVE_ENV    = 'blue'
                        env.ACTIVE_TG_ARN   = env.GREEN_TG_ARN
                        env.INACTIVE_TG_ARN = env.BLUE_TG_ARN
                        env.DEPLOY_IP       = env.BLUE_EC2_IP
                    }

                    echo "✅ Active: ${env.ACTIVE_ENV.toUpperCase()} → Deploying to: ${env.INACTIVE_ENV.toUpperCase()}"
                }
            }
        }

        // ─────────────────────────────────────────────
        // STAGE 2 - Checkout Code
        // ─────────────────────────────────────────────
        stage('Checkout Code') {
            steps {
                echo "📥 Code already checked out by Jenkins SCM."
            }
        }

        // ─────────────────────────────────────────────
        // STAGE 3 - Build Application
        // ─────────────────────────────────────────────
        stage('Build Application') {
            steps {
                echo "🔨 Building version ${params.APP_VERSION}..."
                sh '''
                    echo "Build complete."
                    ls -la dist/
                '''
            }
        }

        // ─────────────────────────────────────────────
        // STAGE 4 - Deploy to Inactive Environment
        // ─────────────────────────────────────────────
        stage('Deploy to Inactive Environment') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'EC2_SSH_KEY', keyFileVariable: 'SSH_KEY_FILE')]) {
                    sh """
                        chmod 400 \$SSH_KEY_FILE

                        echo "🚢 Deploying ${params.APP_VERSION} to ${env.INACTIVE_ENV.toUpperCase()} (${env.DEPLOY_IP})..."

                        # Copy application files
                        scp -i \$SSH_KEY_FILE \
                            -o StrictHostKeyChecking=no \
                            -r ./dist/health ./dist/index.html ./dist/version \
                            ec2-user@${env.DEPLOY_IP}:/var/www/html/

                        # Update version and restart web server
                        ssh -i \$SSH_KEY_FILE \
                            -o StrictHostKeyChecking=no \
                            ec2-user@${env.DEPLOY_IP} '
                                echo "${params.APP_VERSION}" | sudo tee /var/www/html/version
                                sudo systemctl restart httpd
                                sudo systemctl status httpd --no-pager
                            '

                        echo "✅ Deployment to ${env.INACTIVE_ENV.toUpperCase()} complete."
                    """
                }
            }
        }

        // ─────────────────────────────────────────────
        // STAGE 5 - Health Check on Inactive Env
        // ─────────────────────────────────────────────
        stage('Health Check') {
            steps {
                script {
                    echo "🏥 Running health checks on ${env.INACTIVE_ENV.toUpperCase()} (${env.DEPLOY_IP})..."

                    def healthy = false
                    def retries = env.HEALTH_CHECK_RETRIES.toInteger()
                    def delay   = env.HEALTH_CHECK_DELAY.toInteger()

                    for (int i = 1; i <= retries; i++) {
                        echo "Attempt ${i} of ${retries}..."

                        def httpStatus = sh(
                            script: """
                                curl -s -o /dev/null -w "%{http_code}" \
                                    --connect-timeout 5 \
                                    http://${env.DEPLOY_IP}:${env.APP_PORT}/health
                            """,
                            returnStdout: true
                        ).trim()

                        echo "HTTP status: ${httpStatus}"

                        if (httpStatus == '200') {
                            def deployedVersion = sh(
                                script: "curl -s http://${env.DEPLOY_IP}:${env.APP_PORT}/version",
                                returnStdout: true
                            ).trim()

                            echo "Deployed version: ${deployedVersion}"

                            if (deployedVersion.contains(params.APP_VERSION)) {
                                healthy = true
                                echo "✅ Health check PASSED on attempt ${i}."
                                break
                            } else {
                                echo "⚠️  Version mismatch — expected ${params.APP_VERSION}, got ${deployedVersion}"
                            }
                        }

                        if (i < retries) {
                            echo "Waiting ${delay}s before retry..."
                            sleep delay
                        }
                    }

                    if (!healthy) {
                        error("❌ Health check FAILED after ${retries} attempts. Triggering rollback.")
                    }
                }
            }
        }

        // ─────────────────────────────────────────────
        // STAGE 6 - Switch ALB Traffic
        // ─────────────────────────────────────────────
        stage('Switch ALB Traffic') {
            steps {
                script {
                    echo "🔀 Switching traffic: ${env.ACTIVE_ENV.toUpperCase()} → ${env.INACTIVE_ENV.toUpperCase()}..."

                    sh """
                        aws elbv2 modify-listener \
                            --listener-arn ${env.LISTENER_ARN} \
                            --region ${env.AWS_REGION} \
                            --default-actions '[{"Type":"forward","ForwardConfig":{"TargetGroups":[{"TargetGroupArn":"${env.INACTIVE_TG_ARN}","Weight":100},{"TargetGroupArn":"${env.ACTIVE_TG_ARN}","Weight":0}]}}]'
                    """

                    echo "✅ Traffic switched. ${env.INACTIVE_ENV.toUpperCase()} is now LIVE."
                }
            }
        }

        // ─────────────────────────────────────────────
        // STAGE 7 - Post-Switch Validation with Retries
        // ─────────────────────────────────────────────
        stage('Post-Switch Validation') {
            steps {
                script {
                    echo "🔎 Waiting for ALB to mark targets healthy..."

                    def albDns = sh(
                        script: """
                            aws elbv2 describe-load-balancers \
                                --load-balancer-arns ${env.ALB_ARN} \
                                --region ${env.AWS_REGION} \
                                --query 'LoadBalancers[0].DNSName' \
                                --output text
                        """,
                        returnStdout: true
                    ).trim()

                    echo "ALB DNS: http://${albDns}"

                    // Wait for ALB target registration and health check cycle
                    echo "⏳ Waiting 90 seconds for ALB target health propagation..."
                    sleep 90

                    // Retry validation up to 5 times with 20s gap
                    def validated = false
                    for (int i = 1; i <= 5; i++) {
                        echo "Validation attempt ${i} of 5..."

                        def liveStatus = sh(
                            script: "curl -s -o /dev/null -w \"%{http_code}\" http://${albDns}/health",
                            returnStdout: true
                        ).trim()

                        echo "ALB response: HTTP ${liveStatus}"

                        if (liveStatus == '200') {
                            validated = true
                            echo "✅ Validation passed! Live URL: http://${albDns}"
                            break
                        }

                        echo "⏳ Not ready yet (HTTP ${liveStatus}) — waiting 20s before retry..."
                        sleep 20
                    }

                    if (!validated) {
                        error("❌ Post-switch validation failed after all retries. Initiating rollback.")
                    }
                }
            }
        }
    }

    // ─────────────────────────────────────────────
    // POST - Rollback on failure / notify on success
    // ─────────────────────────────────────────────
    post {
        failure {
            script {
                echo "🔄 Pipeline failed — initiating AUTOMATIC ROLLBACK..."
                if (env.ACTIVE_TG_ARN && env.INACTIVE_TG_ARN) {
                    sh """
                        aws elbv2 modify-listener \
                            --listener-arn ${env.LISTENER_ARN} \
                            --region ${env.AWS_REGION} \
                            --default-actions '[{"Type":"forward","ForwardConfig":{"TargetGroups":[{"TargetGroupArn":"${env.ACTIVE_TG_ARN}","Weight":100},{"TargetGroupArn":"${env.INACTIVE_TG_ARN}","Weight":0}]}}]'
                    """
                    echo "✅ Rollback complete. ${env.ACTIVE_ENV?.toUpperCase()} is still live."
                } else {
                    echo "⚠️  No traffic switch occurred — rollback not needed."
                }
            }
        }
        success {
            echo "🎉 Deployment of ${params.APP_VERSION} to ${env.INACTIVE_ENV?.toUpperCase()} succeeded!"
        }
        always {
            node('built-in') {
                cleanWs()
            }
        }
    }
}