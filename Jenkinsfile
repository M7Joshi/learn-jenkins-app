pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = 'e3433b10-adb3-41e4-8559-9cc091a7d737'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = "1.0.$BUILD_ID"
        PLAYWRIGHT_IMAGE = 'mcr.microsoft.com/playwright:v1.54.1-jammy'
    }

    stages {
        stage('Build') {
            agent {
                docker {
                    image 'node:18-bullseye'
                    reuseNode true
                    args '-u root'
                }
            }
            steps {
                sh '''
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la build
                '''
            }
        }

        stage('Tests') {
            parallel {
                stage('Unit Tests') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                            args '-u root'
                        }
                    }
                    steps {
                        sh '''
                            npm install
                            npm test
                        '''
                    }
                    post {
                        always {
                            junit 'jest-results/junit.xml'
                        }
                    }
                }

                stage('E2E (Local)') {
                    agent {
                        docker {
                            image "${PLAYWRIGHT_IMAGE}"
                            reuseNode true
                            args '-u root'
                        }
                    }
                    steps {
                        sh '''
                            npm install
                            npm install serve
                            npx playwright install --with-deps
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
                                reportName: 'E2E Report - Local'
                            ])
                        }
                    }
                }
            }
        }

        stage('Deploy & Test Staging') {
            agent {
                docker {
                    image "${PLAYWRIGHT_IMAGE}"
                    reuseNode true
                    args '-u root'
                }
            }

            environment {
                CI_ENVIRONMENT_URL = ''
            }

            steps {
                sh '''
                    npm install netlify-cli node-jq
                    npm install
                    npx playwright install --with-deps

                    echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"
                    npx netlify deploy --auth $NETLIFY_AUTH_TOKEN --site $NETLIFY_SITE_ID --dir=build --json > deploy-output.json

                    export CI_ENVIRONMENT_URL=$(npx node-jq -r '.deploy_url' deploy-output.json)
                    echo "Staging URL: $CI_ENVIRONMENT_URL"

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
                        reportName: 'E2E Report - Staging'
                    ])
                }
            }
        }

        stage('Approval') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    input message: 'Deploy to Production?', ok: 'Yes, Deploy'
                }
            }
        }

        stage('Deploy & Test Prod') {
            agent {
                docker {
                    image "${PLAYWRIGHT_IMAGE}"
                    reuseNode true
                    args '-u root'
                }
            }

            environment {
                CI_ENVIRONMENT_URL = 'https://cheerful-souffle-d0935a.netlify.app'
            }

            steps {
                sh '''
                    npm install netlify-cli
                    npm install
                    npx playwright install --with-deps

                    echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                    npx netlify deploy --auth $NETLIFY_AUTH_TOKEN --site $NETLIFY_SITE_ID --dir=build --prod

                    echo "Running Playwright tests on: $CI_ENVIRONMENT_URL"
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
                        reportName: 'E2E Report - Production'
                    ])
                }
            }
        }
    }
}
