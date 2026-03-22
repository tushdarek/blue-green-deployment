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
                    echo "Deploying to inactive: ${params.DEPLOY_TO}"
                }
            }
        }

        stage('Deploy New Version to Inactive Env') {
            steps {
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${INACTIVE_IP} '
                            sudo rm -rf /var/www/html/*
                            sudo mkdir -p /var/www/html
                        '
                        scp -r * ubuntu@${INACTIVE_IP}:/var/www/html/
                        ssh ubuntu@${INACTIVE_IP} '
                            sudo systemctl restart nginx
                            echo "Deployed Version \$(cat index.html | grep Version)" 
                        '
                    """
                }
            }
        }

        stage('Health Check') {
            steps {
                script {
                    echo "Waiting for health check..."
                    sleep 10
                    def health = sh(script: "aws elbv2 describe-target-health --target-group-arn ${INACTIVE_TG} --query 'TargetHealthDescriptions[0].TargetHealth.State' --output text", returnStdout: true).trim()
                    if (health != "healthy") {
                        error "Health check FAILED on ${params.DEPLOY_TO}"
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
                echo "🚀 TRAFFIC SWITCHED to ${params.DEPLOY_TO}!"
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
        always {
            echo "Pipeline finished. Check ALB DNS to verify."
        }
    }
}
