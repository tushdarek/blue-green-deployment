pipeline {
    agent any

    environment {
        AWS_REGION          = 'us-east-1'
        ALB_ARN             = credentials('ALB_ARN')
        LISTENER_ARN        = credentials('LISTENER_ARN')
        BLUE_TG_ARN         = credentials('BLUE_TG_ARN')
        GREEN_TG_ARN        = credentials('GREEN_TG_ARN')
        BLUE_EC2_IP         = credentials('BLUE_EC2_IP')
        GREEN_EC2_IP        = credentials('GREEN_EC2_IP')
        SSH_KEY             = credentials('EC2_SSH_KEY')
        HEALTH_CHECK_RETRIES = '5'
        HEALTH_CHECK_DELAY  = '10'
        APP_PORT            = '80'
        SLACK_CHANNEL       = '#deployments'
    }

    parameters {
        string(name: 'APP_VERSION', defaultValue: 'v2', description: 'Version to deploy (e.g. v2)')
        string(name: 'GITHUB_REPO', defaultValue: 'https://github.com/your-org/your-app.git', description: 'Source repo URL')
        string(name: 'BRANCH', defaultValue: 'main', description: 'Branch to deploy')
    }

    stages {

        // ─────────────────────────────────────────────────
        // STAGE 1 – Determine which environment is inactive
        // ─────────────────────────────────────────────────
        stage('Detect Active Environment') {
            steps {
                script {
                    echo "🔍 Querying ALB listener to find the active target group..."

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

                    echo "✅ Active environment  : ${env.ACTIVE_ENV.toUpperCase()}"
                    echo "🚀 Deploy target       : ${env.INACTIVE_ENV.toUpperCase()} (${env.DEPLOY_IP})"
                }
            }
        }

        // ─────────────────────────────────────────────────
        // STAGE 2 – Pull source code from GitHub
        // ─────────────────────────────────────────────────
        stage('Checkout Code') {
            steps {
                echo "📥 Cloning ${params.GITHUB_REPO} @ ${params.BRANCH}..."
                git branch: params.BRANCH, url: params.GITHUB_REPO
                echo "✅ Code checked out successfully."
            }
        }

        // ─────────────────────────────────────────────────
        // STAGE 3 – Build / package the application
        // ─────────────────────────────────────────────────
        stage('Build Application') {
            steps {
                echo "🔨 Building application version ${params.APP_VERSION}..."
                sh '''
                    # Adjust build commands to your stack:
                    # npm ci && npm run build          # Node.js
                    # mvn clean package -DskipTests    # Java/Maven
                    # pip install -r requirements.txt  # Python

                    echo "Build step complete (customise for your stack)."
                '''
            }
        }

        // ─────────────────────────────────────────────────
        // STAGE 4 – Deploy to inactive environment
        // ─────────────────────────────────────────────────
        stage('Deploy to Inactive Environment') {
            steps {
                script {
                    echo "🚢 Deploying ${params.APP_VERSION} to ${env.INACTIVE_ENV.toUpperCase()} (${env.DEPLOY_IP})..."

                    // Write temporary SSH key
                    writeFile file: '/tmp/deploy_key.pem', text: env.SSH_KEY
                    sh 'chmod 400 /tmp/deploy_key.pem'

                    sh """
                        # ── Copy artefacts ──────────────────────────────────
                        scp -i /tmp/deploy_key.pem \
                            -o StrictHostKeyChecking=no \
                            -r ./dist/* \
                            ec2-user@${env.DEPLOY_IP}:/var/www/html/

                        # ── Remote deployment commands ───────────────────────
                        ssh -i /tmp/deploy_key.pem \
                            -o StrictHostKeyChecking=no \
                            ec2-user@${env.DEPLOY_IP} '
                                # Update version file (shown by the app)
                                echo "${params.APP_VERSION}" | sudo tee /var/www/html/version.txt

                                # Restart web server
                                sudo systemctl restart httpd || sudo systemctl restart nginx

                                # Confirm service is running
                                sudo systemctl status httpd  || sudo systemctl status nginx
                            '
                    """

                    echo "✅ Deployment to ${env.INACTIVE_ENV.toUpperCase()} complete."
                }
            }
            post {
                always {
                    sh 'rm -f /tmp/deploy_key.pem'
                }
            }
        }

        // ─────────────────────────────────────────────────
        // STAGE 5 – Health check on inactive environment
        // ─────────────────────────────────────────────────
        stage('Health Check') {
            steps {
                script {
                    echo "🏥 Running health checks on ${env.INACTIVE_ENV.toUpperCase()} (${env.DEPLOY_IP})..."

                    def healthy = false
                    def retries = env.HEALTH_CHECK_RETRIES.toInteger()
                    def delay   = env.HEALTH_CHECK_DELAY.toInteger()

                    for (int i = 1; i <= retries; i++) {
                        echo "   Attempt ${i} of ${retries}..."

                        def httpStatus = sh(
                            script: """
                                curl -s -o /dev/null -w "%{http_code}" \
                                    --connect-timeout 5 \
                                    http://${env.DEPLOY_IP}:${env.APP_PORT}/health
                            """,
                            returnStdout: true
                        ).trim()

                        echo "   HTTP status: ${httpStatus}"

                        if (httpStatus == '200') {
                            // Also verify the deployed version
                            def deployedVersion = sh(
                                script: """
                                    curl -s http://${env.DEPLOY_IP}:${env.APP_PORT}/version
                                """,
                                returnStdout: true
                            ).trim()

                            echo "   Deployed version reported: ${deployedVersion}"

                            if (deployedVersion.contains(params.APP_VERSION)) {
                                healthy = true
                                echo "✅ Health check PASSED on attempt ${i}."
                                break
                            } else {
                                echo "⚠️  Version mismatch (expected ${params.APP_VERSION}, got ${deployedVersion})."
                            }
                        }

                        if (i < retries) {
                            echo "   Waiting ${delay}s before retry..."
                            sleep delay
                        }
                    }

                    if (!healthy) {
                        error("❌ Health check FAILED after ${retries} attempts. Triggering rollback.")
                    }
                }
            }
        }

        // ─────────────────────────────────────────────────
        // STAGE 6 – Switch ALB traffic to inactive env
        // ─────────────────────────────────────────────────
        stage('Switch Traffic') {
            steps {
                script {
                    echo "🔀 Switching ALB traffic from ${env.ACTIVE_ENV.toUpperCase()} → ${env.INACTIVE_ENV.toUpperCase()}..."

                    sh """
                        aws elbv2 modify-listener \
                            --listener-arn ${env.LISTENER_ARN} \
                            --region ${env.AWS_REGION} \
                            --default-actions '[
                                {
                                    "Type": "forward",
                                    "ForwardConfig": {
                                        "TargetGroups": [
                                            {
                                                "TargetGroupArn": "${env.INACTIVE_TG_ARN}",
                                                "Weight": 100
                                            },
                                            {
                                                "TargetGroupArn": "${env.ACTIVE_TG_ARN}",
                                                "Weight": 0
                                            }
                                        ]
                                    }
                                }
                            ]'
                    """

                    echo "✅ Traffic switched. ${env.INACTIVE_ENV.toUpperCase()} is now LIVE."
                    env.NEW_ACTIVE_ENV = env.INACTIVE_ENV
                }
            }
        }

        // ─────────────────────────────────────────────────
        // STAGE 7 – Post-switch validation
        // ─────────────────────────────────────────────────
        stage('Post-Switch Validation') {
            steps {
                script {
                    echo "🔎 Validating live traffic via ALB endpoint..."

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

                    sleep 5   // Allow ALB to propagate

                    def liveStatus = sh(
                        script: "curl -s -o /dev/null -w \"%{http_code}\" http://${albDns}/health",
                        returnStdout: true
                    ).trim()

                    if (liveStatus != '200') {
                        error("❌ Post-switch validation failed (HTTP ${liveStatus}). Initiating rollback.")
                    }

                    echo "✅ Post-switch validation passed. ALB is serving on ${env.INACTIVE_ENV.toUpperCase()}."
                    echo "🌐 Application URL: http://${albDns}"
                }
            }
        }
    }

    // ─────────────────────────────────────────────────────
    // POST – Rollback on failure, notify on all outcomes
    // ─────────────────────────────────────────────────────
    post {
        failure {
            script {
                echo "🔄 Pipeline failed — initiating AUTOMATIC ROLLBACK to ${env.ACTIVE_ENV?.toUpperCase() ?: 'PREVIOUS'} environment..."

                if (env.ACTIVE_TG_ARN && env.INACTIVE_TG_ARN) {
                    sh """
                        aws elbv2 modify-listener \
                            --listener-arn ${env.LISTENER_ARN} \
                            --region ${env.AWS_REGION} \
                            --default-actions '[
                                {
                                    "Type": "forward",
                                    "ForwardConfig": {
                                        "TargetGroups": [
                                            {
                                                "TargetGroupArn": "${env.ACTIVE_TG_ARN}",
                                                "Weight": 100
                                            },
                                            {
                                                "TargetGroupArn": "${env.INACTIVE_TG_ARN}",
                                                "Weight": 0
                                            }
                                        ]
                                    }
                                }
                            ]'
                    """
                    echo "✅ Rollback complete. Traffic is back on ${env.ACTIVE_ENV?.toUpperCase()}."
                } else {
                    echo "⚠️  Environment variables not set — rollback skipped (no traffic switch had occurred)."
                }

                // Slack / email notification
                echo "📣 Sending failure notification..."
                // slackSend channel: env.SLACK_CHANNEL,
                //            color: 'danger',
                //            message: "❌ Deployment of ${params.APP_VERSION} FAILED. Rolled back to ${env.ACTIVE_ENV?.toUpperCase()}."
            }
        }

        success {
            echo "🎉 Deployment of ${params.APP_VERSION} to ${env.INACTIVE_ENV?.toUpperCase()} succeeded!"
            echo "📣 Sending success notification..."
            // slackSend channel: env.SLACK_CHANNEL,
            //            color: 'good',
            //            message: "✅ Deployment of ${params.APP_VERSION} succeeded. ${env.INACTIVE_ENV?.toUpperCase()} is now LIVE."
        }

        always {
            cleanWs()
        }
    }
}
