pipeline {
    agent any

    stages {
        stage('Build') {
            agent{
                docker{
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                ls -la
                node --version
                npm --version
                npm ci
                npm run build
                ls -la
                '''
            }
        }

        stage('Test'){

            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }


            steps {
                sh '''
                echo "Test süreci başlatılıyor.."

                echo "Index dosyası kontrol ediliyor.."

                if ls build/index.html; then
                    echo "index.html var"
                else
                    echo "index.html yok"
                    exit 1
                fi
                echo "NPM VERSIYON KONTROL---------------"
                npm --version

                echo "Testler sürdürülüyor.."

                npm test
                '''

            }
        }
        stage('E2E'){
            agent{
                docker{
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }

            steps{
                sh '''
                    npm install serve
                    node_modules/.bin/serve -s build &
                    sleep 10
                    npx playwright test
                '''
            }
        }
    }

    post {
        always{
            junit 'jest-results/junit.xml'
        }
    }
}