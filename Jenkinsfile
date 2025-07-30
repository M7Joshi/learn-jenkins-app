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
                    args '-u root'
                }
            }
            steps {
                sh '''
                    node --version
                    npm --version
                    npm ci
                    npm run build
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
                            args '-u root'
                        }
                    }
                    steps {
                        sh 'npm test'
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
                            image 'mcr.microsoft.com/playwright:v1.54.0-noble'
                            reuseNode true
                            args '-u root'
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
                                reportName: 'E2E Report',
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
                    image 'mcr.microsoft.com/playwright:v1.54.0-noble'
                    reuseNode true
                    args '-u root'
                }
            }
            steps {
                sh '''
                    npm install netlify-cli node-jq
                    echo "Deploying to staging..."
                    npx netlify deploy --site=$NETLIFY_SITE_ID --auth=$NETLIFY_AUTH_TOKEN --dir=build --json > deploy-output.json
                    STAGING_URL=$(npx node-jq -r '.deploy_url' deploy-output.json)
                    echo "Staging deployed at: $STAGING_URL"

                    # Run tests on the staging site
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
                        reportName: 'Staging E2E Report',
                        useWrapperFileDirectly: true
                    ])
                }
            }
        }

        stage('Approval') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    input message: 'Deploy to production?', ok: 'Yes, deploy'
                }
            }
        }

        stage('Deploy & Test Prod') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.54.0-noble'
                    reuseNode true
                    args '-u root'
                }
            }
            steps {
                sh '''
                    npm install netlify-cli
                    echo "Deploying to production..."
                    npx netlify deploy --site=$NETLIFY_SITE_ID --auth=$NETLIFY_AUTH_TOKEN --prod --dir=build

                    # Run tests on prod
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
                        reportName: 'Production E2E Report',
                        useWrapperFileDirectly: true
                    ])
                }
            }
        }
    }
}
