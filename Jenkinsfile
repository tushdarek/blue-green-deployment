pipeline {
    agent any

    parameters {
        choice(name: 'DEPLOY_TO', choices: ['blue', 'green'], description: 'Which environment is INACTIVE right now?')
        booleanParam(name: 'AUTO_SWITCH', defaultValue: true, description: 'Switch traffic after success?')
    }

    environment {
        LISTENER_ARN = 'arn:aws:elasticloadbalancing:ap-south-1:477973726849:listener/app/BlueGreen-ALB/fdff715c8fc7384b/98278ba4dd2f1f70'

        BLUE_TG_ARN   = 'arn:aws:elasticloadbalancing:ap-south-1:477973726849:targetgroup/TG-Blue/ad4a682e102029d0'
        GREEN_TG_ARN  = 'arn:aws:elasticloadbalancing:ap-south-1:477973726849:targetgroup/TG-Green/a5e3d80cb9ca8ead'

        BLUE_IP  = '13.201.124.219'
        GREEN_IP = '65.0.55.222'
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/aniketchougule108/Blue-Green-Deployment-Project-Jenkins.git'
                echo "✅ Code checked out"
            }
        }

        stage('Determine Inactive Environment') {
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
                    echo "🎯 Deploying to ${params.DEPLOY_TO}"
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

                        echo "🚀 Moving files..."
                        ssh -o StrictHostKeyChecking=no ubuntu@${INACTIVE_IP} '
                            sudo cp -r /home/ubuntu/deploy-temp/* /var/www/html/
                            sudo systemctl restart nginx
                            sleep 3
                            echo "✅ Deployment complete"
                        '
                    """
                }
            }
        }

        stage('Health Check') {
            steps {
                script {
                    echo "🔍 Checking app directly..."

                    def status = sh(
                        script: "curl -s -o /dev/null -w \"%{http_code}\" http://${INACTIVE_IP}",
                        returnStdout: true
                    ).trim()

                    echo "HTTP Status: ${status}"

                    if (status != "200") {
                        error "❌ App is NOT healthy"
                    }

                    echo "✅ App is healthy"
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
            echo "❌ Deployment failed - Rolling back"

            sh """
                aws elbv2 modify-listener \
                --listener-arn ${LISTENER_ARN} \
                --default-actions Type=forward,TargetGroupArn=${ACTIVE_TG}
            """
        }

        success {
            echo "🎉 Deployment SUCCESS"
        }

        always {
            echo "📌 Pipeline finished"
        }
    }
}
