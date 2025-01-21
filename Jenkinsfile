pipeline {
    agent any
    environment {
        // Replace 'MySonarQube' with the name of your SonarQube server configuration in Jenkins
        SONARQUBE_SERVER = 'sonarqube'
        DOCKER_VERSION = '1.0.0' // Update this version as needed
        DOCKER_IMAGE = "tuonsreynit/vueapp:${DOCKER_VERSION}"
        DOCKER_CREDENTIALS_ID = 'dockerhub'
        TARGET_SERVER_CREDENTIALS_ID = 'server'
        TARGET_SERVER='root@159.223.64.36'
        TELEGRAM_BOT_TOKEN = credentials('telegram-bot-token') // ID for bot token credential
        TELEGRAM_CHAT_ID = credentials('telegram-chat-id')     // ID for chat ID credential
  
    }

    stages {
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv(SONARQUBE_SERVER) {
                    script {
                        def scannerHome = tool 'sonarscanner-tool'  // Replace with the name you gave to SonarScanner in Jenkins
                        sh "${scannerHome}/bin/sonar-scanner " +
                        "-Dsonar.projectKey=my-project " +
                        "-Dsonar.sources=src " +
                        "-Dsonar.host.url=${env.SONAR_HOST_URL} " +
                        "-Dsonar.login=${env.SONAR_AUTH_TOKEN}"
                    
                    }
                }
            }
        }

        stage("Wait for Quality Gate"){
            steps{
                script{
                    def qg = waitForQualityGate()
                    if (qg.status != 'OK'){
                        currentBuild.result='FAILURE'
                    }else{
                        currentBuild.result='SUCCESS'
                    }
                }
            }
        }

        stage('Build') {
            when{
                expression {
                    return currentBuild.result == 'SUCCESS'
                }
            }
            steps {
                script {
                    // Check if Docker is running and build the Docker image
                    sh """
                        docker ps
                        docker build -t $DOCKER_IMAGE .
                    """
                }
            }
        }
        
        stage('Push') {
            when{
                expression {
                    return currentBuild.result == 'SUCCESS'
                }
            }

            steps {
                script {
                    // Log in to Docker Hub and push the Docker image
                    docker.withRegistry('https://index.docker.io/v1/', DOCKER_CREDENTIALS_ID) {
                        sh "docker push $DOCKER_IMAGE"
                    }
                }
            }
        }

        stage("Deploy on workers"){
            steps{
                dir("ansible"){
                    sh """
                        export ANSIBLE_HOST_KEY_CHECKING=False
                        ansible -i hosts.yml workers -m ping
                        ansible-playbook -i hosts.yml deploy-poco.yml --extra-vars "docker_version=$DOCKER_VERSION"

                    """
                }
            }
        }
        // stage('Deploy') {
        //     steps {
        //         script {
        //             // SSH into the target server and deploy the Docker container
        //             sshagent([TARGET_SERVER_CREDENTIALS_ID]) {
        //                 sh """
        //                     ssh -o StrictHostKeyChecking=no $TARGET_SERVER '
        //                         docker stop vueapp-container || true && \
        //                         docker rm vueapp-container || true && \
        //                         docker pull $DOCKER_IMAGE && \
        //                         docker run -d --name vueapp-container -p 5000:80 $DOCKER_IMAGE && \
        //                         docker ps && \
        //                         sudo nginx -s reload
        //                         # Uncomment the following line if needed for adding a domain
        //                         # sudo bash /root/istad-devops/istad/utils/addDomainName.sh www 5000
        //                     '
        //                 """
        //             }
        //         }
        //     }
        // }

    }

    post {
        success {
            script {
                def message = "✅ *Build and Deployment Successful*\\nProject: ${env.JOB_NAME}\\nBuild Number: ${env.BUILD_NUMBER}"
                sendTelegramAlert(message)
            }
        }
        failure {
            script {
                def message = "❌ *Build Failed*\\nProject: ${env.JOB_NAME}\\nBuild Number: ${env.BUILD_NUMBER}\\nCheck Jenkins for details."
                sendTelegramAlert(message)
            }
        }
    }
}

def sendTelegramAlert(message) {
    sh """
        curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
            -d chat_id=${TELEGRAM_CHAT_ID} \
            -d parse_mode=Markdown \
            -d text="${message}"
    """
}