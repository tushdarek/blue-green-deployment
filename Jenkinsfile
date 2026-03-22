pipeline {
    agent any
    
    parameters {
        choice(name: 'DEPLOY_TO', choices: ['blue', 'green'], description: 'Which environment is INACTIVE right now?')
        booleanParam(name: 'AUTO_SWITCH', defaultValue: true, description: 'Switch traffic after success?')
        choice(name: 'VERSION', choices: ['v1', 'v2'], description: 'Which version to deploy?')
    }

    environment {
        // ✅ Correct Listener ARN (FIXED)
        LISTENER_ARN = 'arn:aws:elasticloadbalancing:ap-south-1:098688552647:listener/app/BlueGreen-ALB/c43469f376ace0e9/15b37439cafd643a'

        BLUE_TG_ARN   = 'arn:aws:elasticloadbalancing:ap-south-1:098688552647:targetgroup/TG-Blue/1b9b9dc59587436b'
        GREEN_TG_ARN  = 'arn:aws:elasticloadbalancing:ap-south-1:098688552647:targetgroup/TG-Green/00e48e7f78dbd3ac'
        
        BLUE_IP  = '13.201.124.219'
        GREEN_IP = '65.0.55.222'
        
        APP_PORT = '80'
        HEALTH_ENDPOINT = '/health'
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main', 
                    url: 'https://github.com/tushdarek/blue-green-deployment.git'
                echo "✅ Code checked out"
            }
        }

        stage('Determine Target Environment') {
            steps {
                script {
                    if (params.DEPLOY_TO == 'blue') {
                        env.INACTIVE_IP = env.BLUE_IP
                        env.INACTIVE_TG = env.BLUE_TG_ARN
                        env.ACTIVE_TG   = env.GREEN_TG_ARN
                    } else {
                        env.INACTIVE_IP = env.GREEN_IP
                        env.INACTIVE_TG = env.GREEN_TG_ARN
                        env.ACTIVE_TG   = env.BLUE_TG_ARN
                    }

                    echo "🎯 Deploying to ${params.DEPLOY_TO} (${env.INACTIVE_IP})"
                }
            }
        }

        stage('Deploy Application') {
            steps {
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh """
                        echo "📦 Preparing server..."
                        ssh -o StrictHostKeyChecking=no ubuntu@${INACTIVE_IP} '
                            sudo rm -rf /var/www/html/*
                            mkdir -p /home/ubuntu/deploy-temp
                        '

                        echo "📤 Uploading files..."
                        scp -o StrictHostKeyChecking=no -r * ubuntu@${INACTIVE_IP}:/home/ubuntu/deploy-temp/

                        echo "🚀 Moving files to web root..."
                        ssh -o StrictHostKeyChecking=no ubuntu@${INACTIVE_IP} '
                            sudo cp -r /home/ubuntu/deploy-temp/* /var/www/html/
                            sudo systemctl restart nginx
                            sleep 5
                            echo "✅ Deployment complete"
                        '
                    """
                }
            }
        }

        stage('Health Check') {
            steps {
                script {
                    echo "🔍 Waiting for ALB health check..."
                    sleep 20

                    def health = sh(
                        script: "aws elbv2 describe-target-health --target-group-arn ${INACTIVE_TG} --query 'TargetHealthDescriptions[*].TargetHealth.State' --output text",
                        returnStdout: true
                    ).trim()

                    echo "Health status: ${health}"

                    if (!health.contains("healthy")) {
                        error "❌ Health check FAILED"
                    }

                    echo "✅ Health check PASSED"
                }
            }
        }

        stage('Switch Traffic') {
            when { expression { return params.AUTO_SWITCH } }
            steps {
                sh """
                    aws elbv2 modify-listener \
                    --listener-arn ${LISTENER_ARN} \
                    --default-actions Type=forward,TargetGroupArn=${INACTIVE_TG}
                """
                echo "🚀 Traffic switched to ${params.DEPLOY_TO}"
            }
        }
    }

    post {

        failure {
            echo "⚠️ Deployment failed! Rolling back..."

            sh """
                aws elbv2 modify-listener \
                --listener-arn ${LISTENER_ARN} \
                --default-actions Type=forward,TargetGroupArn=${ACTIVE_TG}
            """
        }

        success {
            echo "🎉 Deployment SUCCESSFUL!"
        }

        always {
            echo "📌 Pipeline finished"
        }
    }
}
