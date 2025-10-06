pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = 'nginx:alpine'
        CONTAINER_NAME = "nginx-test-${BUILD_NUMBER}"
        HOST_PORT = '9889'
        REPO_URL = 'https://github.com/Dariong-LN/Continuous-Integration.git'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    extensions: [],
                    userRemoteConfigs: [[url: env.REPO_URL]]
                ])
            }
        }
        
        stage('Test Failure for Telegram') {
            steps {
                script {
                    // Искусственно создаем ошибку для теста
                    error("Тестовая ошибка для проверки Telegram уведомлений")
                }
            }
        }
        
        stage('Calculate MD5') {
            steps {
                script {
                    def originalMD5 = sh(
                        script: "md5sum index.html | awk '{print \$1}'",
                        returnStdout: true
                    ).trim()
                    env.ORIGINAL_MD5 = originalMD5
                    echo "Original MD5: ${env.ORIGINAL_MD5}"
                }
            }
        }
        
        // остальные stages...
    }
    
    post {
        always {
            script {
                sh "docker stop ${env.CONTAINER_NAME} || true"
                sh "docker rm ${env.CONTAINER_NAME} || true"
                echo "Container ${env.CONTAINER_NAME} cleaned up"
            }
        }
        failure {
            script {
                def message = "❌ CI Pipeline Failed\\n" +
                             "Repository: ${env.REPO_URL}\\n" +
                             "Build: ${env.BUILD_URL}\\n" +
                             "Job: ${env.JOB_NAME}\\n" +
                             "Build Number: ${env.BUILD_NUMBER}\\n" +
                             "Тестовое уведомление"
                
                def telegramURL = 'https://api.telegram.org/bot8460197612:AAGlVp01h1kYvdfLHpusRCXYhZB_fdUPhDc/sendMessage'
                
                echo "Отправка уведомления в Telegram..."
                echo "URL: ${telegramURL}"
                echo "Message: ${message}"
                
                def response = sh(
                    script: """
                        curl -s -X POST '${telegramURL}' \\
                        -d 'chat_id=900951534' \\
                        -d 'text=${message}' \\
                        -d 'parse_mode=Markdown'
                    """,
                    returnStdout: true
                ).trim()
                
                echo "Response from Telegram: ${response}"
                
                // Альтернативная попытка с другим форматом
                sh """
                    curl -s -X POST '${telegramURL}' \\
                    -d 'chat_id=900951534' \\
                    -d 'text=Тестовое сообщение от Jenkins' \\
                    -d 'parse_mode=Markdown' || echo "CURL failed"
                """
            }
        }
    }
}
