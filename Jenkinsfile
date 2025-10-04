pipeline {
    agent any
    
    triggers {
        pollSCM('H/2 * * * *') // Проверка изменений каждые 2 минуты
    }
    
    environment {
        DOCKER_IMAGE = 'nginx:alpine'
        CONTAINER_NAME = 'nginx-test'
        HOST_PORT = '9889'
        REPO_URL = 'https://github.com/yourusername/your-repo' // Замените на ваш репозиторий
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', 
                    url: env.REPO_URL,
                    credentialsId: 'github-credentials' // Создайте в Jenkins credentials для GitHub
            }
        }
        
        stage('Calculate MD5') {
            steps {
                script {
                    // Вычисляем MD5 исходного файла
                    def originalMD5 = sh(
                        script: "md5sum index.html | cut -d' ' -f1",
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
                        script: "curl -s -o /dev/null -w '%{http_code}' http://localhost:${env.HOST_PORT}",
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
                            curl -s http://localhost:${env.HOST_PORT} | md5sum | cut -d' ' -f1
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
                             "Причина: Проверка не пройдена"
                
                // Настройте webhook URL для Telegram бота
                def telegramURL = 'https://api.telegram.org/bot8460197612:AAGlVp01h1kYvdfLHpusRCXYhZB_fdUPhDc/sendMessage'
                sh """
                    curl -s -X POST ${telegramURL} \\
                    -d chat_id=900951534 \\
                    -d text="${message}" \\
                    -d parse_mode=Markdown
                """
            }
            echo 'Pipeline failed! Notification sent.'
        }
    }
}
