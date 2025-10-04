pipeline {
    agent any
    
    triggers {
        pollSCM('H/2 * * * *') // Проверка изменений каждые 2 минуты
    }
    
    environment {
        DOCKER_IMAGE = 'nginx:alpine'
        CONTAINER_NAME = 'nginx-test-${BUILD_NUMBER}'
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
                    userRemoteConfigs: [[
                        url: env.REPO_URL,
                        credentialsId: '' // Оставьте пустым если публичный репозиторий
                    ]]
                ])
            }
        }
        
        stage('Calculate MD5') {
            steps {
                script {
                    // Вычисляем MD5 исходного файла
                    def originalMD5 = sh(
                        script: "md5sum index.html | awk '{print \$1}'",
                        returnStdout: true
                    ).trim()
                    env.ORIGINAL_MD5 = originalMD5
                    echo "Original MD5: ${env.ORIGINAL_MD5}"
                }
            }
        }
        
        stage('Run Nginx Container') {
            steps {
                script {
                    // Останавливаем и удаляем старый контейнер, если существует
                    sh "docker stop ${env.CONTAINER_NAME} || true"
                    sh "docker rm ${env.CONTAINER_NAME} || true"
                    
                    // Запускаем новый контейнер
                    sh """
                        docker run -d \
                          --name ${env.CONTAINER_NAME} \
                          -p ${env.HOST_PORT}:80 \
                          -v \$(pwd)/index.html:/usr/share/nginx/html/index.html \
                          ${env.DOCKER_IMAGE}
                    """
                    
                    // Ждем запуска контейнера
                    sleep 10
                }
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    // Проверяем HTTP статус
                    def response = sh(
                        script: "curl -s -o /dev/null -w '%{http_code}' http://localhost:${env.HOST_PORT} || echo '000'",
                        returnStdout: true
                    ).trim()
                    
                    echo "HTTP Response Code: ${response}"
                    
                    if (response != '200') {
                        currentBuild.result = 'FAILURE'
                        error("Health check failed! Expected 200, got ${response}")
                    }
                }
            }
        }
        
        stage('MD5 Verification') {
            steps {
                script {
                    // Скачиваем файл из контейнера и вычисляем MD5
                    def downloadedMD5 = sh(
                        script: """
                            curl -s http://localhost:${env.HOST_PORT} | md5sum | awk '{print \$1}'
                        """,
                        returnStdout: true
                    ).trim()
                    
                    echo "Downloaded MD5: ${downloadedMD5}"
                    echo "Original MD5: ${env.ORIGINAL_MD5}"
                    
                    if (env.ORIGINAL_MD5 != downloadedMD5) {
                        currentBuild.result = 'FAILURE'
                        error("MD5 verification failed! Expected ${env.ORIGINAL_MD5}, got ${downloadedMD5}")
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                // Всегда удаляем контейнер
                sh "docker stop ${env.CONTAINER_NAME} || true"
                sh "docker rm ${env.CONTAINER_NAME} || true"
                echo "Container ${env.CONTAINER_NAME} cleaned up"
            }
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            script {
                // Отправка уведомления в Telegram
                def message = "❌ CI Pipeline Failed\\n" +
                             "Repository: ${env.REPO_URL}\\n" +
                             "Build: ${env.BUILD_URL}\\n" +
                             "Job: ${env.JOB_NAME}\\n" +
                             "Build Number: ${env.BUILD_NUMBER}"
                
                def telegramURL = 'https://api.telegram.org/bot8460197612:AAGlVp01h1kYvdfLHpusRCXYhZB_fdUPhDc/sendMessage'
                sh """
                    curl -s -X POST ${telegramURL} \
                    -d "chat_id=900951534" \
                    -d "text=${message}" \
                    -d "parse_mode=Markdown" || echo "Telegram notification failed"
                """
            }
            echo 'Pipeline failed! Notification sent.'
        }
    }
}
