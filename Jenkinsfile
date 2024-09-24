pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '22cc4b5c-6923-49b2-87fc-e8c9a5f922d9'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
    }

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
        stage('Tests'){
            parallel {
                stage('Unit Tests'){

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
                        always{
                            junit 'jest-results/junit.xml'
                        }
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
                            npx playwright test --reporter=html
                        '''
                    }
                    post {
                        always{
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright HTML Report', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }

                
            }
        }

        stage('Deploy') {
            agent{
                docker{
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