pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '22cc4b5c-6923-49b2-87fc-e8c9a5f922d9'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        DOCKER_IMAGE = 'eevah/jenkinswdocker'
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_CREDENTIALS_ID = 'dockerhub_credentials'  
    }

    stages {
        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                echo "Build started..."
                ls -la
                node --version
                npm --version
                npm ci
                npm run build
                ls -la
                '''
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}:${env.BUILD_NUMBER}")
                }
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', "${DOCKER_CREDENTIALS_ID}") {
                        docker.image("${DOCKER_IMAGE}:${env.BUILD_NUMBER}").push()
                    }
                }
            }
        }

        stage('Tests') {
            parallel {
                stage('Unit Tests') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                        echo "Testler başlatılıyor.."
                        echo "Index dosyası kontrol ediliyor.."
                        if ls build/index.html; then
                            echo "index.html var"
                        else
                            echo "index.html yok"
                            exit 1
                        fi
                        npm test
                        '''
                    }
                    post {
                        always {
                            junit 'jest-results/junit.xml'
                        }
                    }
                }

                stage('E2E') {
                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                        npm install serve
                        node_modules/.bin/serve -s build &
                        sleep 10
                        npx playwright test --reporter=html
                        '''
                    }
                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Local Report', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }

        stage('Deploy to Docker') {
            steps {
                script {
                    def imageName = "${DOCKER_IMAGE}:${env.BUILD_NUMBER}"
                    
                    // Önceki container'ları durdur ve kaldır
                    sh '''
                    docker ps -q --filter 'ancestor=''' + imageName + ''' | xargs -r docker stop || true
                    docker ps -aq --filter 'ancestor=''' + imageName + ''' | xargs -r docker rm || true
                    '''
                    
                    // Kullanılabilir bir port bul ve container'ı bu port ile çalıştır
                    def basePort = 3012
                    def maxRetries = 10
                    def portToUse = basePort
                    for (int i = 0; i < maxRetries; i++) {
                        def isPortInUse = isUnix()
                            ? (sh(script: "lsof -i :${portToUse}", returnStatus: true) == 0)
                            : (sh(script: "netstat -an | findstr :${portToUse}", returnStatus: true) == 0)

                        if (!isPortInUse) {
                            echo "Kullanılabilir port bulundu: ${portToUse}"
                            break
                        } else {
                            echo "Port ${portToUse} kullanımda, başka bir port aranıyor..."
                            portToUse++
                        }
                    }

                    if (portToUse >= basePort + maxRetries) {
                        error "Tüm denenen portlar kullanımda. Lütfen portları kontrol edin."
                    }

                    // Container'ı dinamik port ile çalıştır
                    sh "docker run -d -p ${portToUse}:3000 ${imageName}"
                    echo "Uygulama başarıyla port ${portToUse} ile deploy edildi."
                }
            }
        }


                stage('Run Docker Container Locally') {
            steps {
                    script {
                        def usedPorts = []
                        def basePort = 3001
                        def maxRetriess = 17
                        
                        def portToUse = basePort
                        for (int i = 0; i < maxRetriess; i++) {
                            def isPortInUse = isUnix()
                                ? (sh(script: "lsof -i :${portToUse}", returnStatus: true) == 0)
                                : (sh(script: "netstat -an | findstr :${portToUse}", returnStatus: true) == 0)

                            if (!isPortInUse) {
                                echo "Kullanılabilir port bulundu: ${portToUse}"
                                break
                            } else {
                                usedPorts.add(portToUse)
                                portToUse++
                            }
                        }

                        if (usedPorts.size() == maxRetries) {
                            error "Tüm denenen portlar kullanımda. Lütfen portları kontrol edin."
                        }

                        sh '''
                        docker ps -q --filter 'ancestor=${DOCKER_IMAGE}:${env.BUILD_NUMBER}' | xargs -r docker stop || true
                        docker ps -aq --filter 'ancestor=${DOCKER_IMAGE}:${env.BUILD_NUMBER}' | xargs -r docker rm || true
                        '''

                        echo "Seçilen port: ${portToUse}"
                        sh """
                        docker run -d -p ${portToUse}:3000 ${DOCKER_IMAGE}:${env.BUILD_NUMBER}
                        """
                        echo "Yerel Docker container başarıyla ${portToUse} portu ile çalıştırıldı."
                    }
                }
            }

        stage('Check Application') {
            steps {
                script {
                    sh '''
                    echo "Container'ın başlatılması bekleniyor..."
                    sleep 10
                    curl http://localhost:3000 || echo "Bağlantı kurulamadı, tekrar deneyin"
                    '''
                }
            }
        }

        stage('Deploy to Netlify') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                npm install netlify-cli
                node_modules/.bin/netlify --version
                echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                node_modules/.bin/netlify status
                node_modules/.bin/netlify deploy --dir=build --prod
                '''
            }
        }
    }
}