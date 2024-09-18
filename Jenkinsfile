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
            steps {
                sh '''
                echo "Test süreci başlatılıyor.."

                if ls build/index.html; then
                    echo "index.html exists."
                else
                    echo "index.html does not exist."
                    exit 1
                fi
                '''
            }
        }
    }
}