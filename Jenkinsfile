pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = 'e3433b10-adb3-41e4-8559-9cc091a7d737'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = "1.0.$BUILD_ID"
    }

    stages {

        stage('Build') {
            agent {
                docker {
                    image 'node:18-bullseye'
                    reuseNode true
                    args '--user root'
                }
            }
            steps {
                sh '''
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }

        stage('Tests') {
            parallel {
                stage('Unit tests') {
                    agent {
                        docker {
                            image 'node:18-bullseye'
                            reuseNode true
                            args '--user root'
                        }
                    }
                    steps {
                        sh '''
                            npm test || true
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
                            args '--user root'
                        }
                    }
                    steps {
                        sh '''
                            npm install serve
                            npx serve -s build &

                            sleep 10
                            npx playwright test --reporter=html
                        '''
                    }
                    post {
                        always {
                            publishHTML([
                                allowMissing: false,
                                alwaysLinkToLastBuild: false,
                                keepAll: false,
                                reportDir: 'playwright-report',
                                reportFiles: 'index.html',
                                reportName: 'Local E2E',
                                useWrapperFileDirectly: true
                            ])
                        }
                    }
                }
            }
        }

        stage('Deploy & Test Staging') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                    args '--user root'
                }
            }

            steps {
                sh '''
                    npm install netlify-cli node-jq

                    echo "Deploying to STAGING..."
                    npx netlify deploy --dir=build --json > deploy-output.json

                    export CI_ENVIRONMENT_URL=$(npx node-jq -r '.deploy_url' deploy-output.json)
                    echo "Staging URL: $CI_ENVIRONMENT_URL"

                    npx playwright test --reporter=html
                '''
            }

            post {
                always {
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: false,
                        keepAll: false,
                        reportDir: 'playwright-report',
                        reportFiles: 'index.html',
                        reportName: 'Staging E2E',
                        useWrapperFileDirectly: true
                    ])
                }
            }
        }

        stage('Approval') {
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: 'Do you wish to deploy to production?', ok: 'Yes, deploy to Prod'
                }
            }
        }

        stage('Deploy & Test Prod') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                    args '--user root'
                }
            }

            steps {
                sh '''
                    npm install netlify-cli

                    echo "Deploying to PRODUCTION..."
                    npx netlify deploy --dir=build --prod

                    npx playwright test --reporter=html
                '''
            }

            post {
                always {
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: false,
                        keepAll: false,
                        reportDir: 'playwright-report',
                        reportFiles: 'index.html',
                        reportName: 'Prod E2E',
                        useWrapperFileDirectly: true
                    ])
                }
            }
        }
    }
}
