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
                    image 'node:18-alpine'
                    reuseNode true
                    args '-u root'
                }
            }
            steps {
                sh '''
                    apk add --no-cache bash
                    echo "📦 Building React app..."
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
                            echo "🧪 Running unit tests..."
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
                            image 'mcr.microsoft.com/playwright:v1.54.0-noble'
                            reuseNode true
                            args '-u root'
                        }
                    }
                    steps {
                        sh '''
                            echo "🧪 Running local E2E tests..."
                            npm install serve
                            node_modules/.bin/serve -s build &
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
                                reportTitles: '',
                                useWrapperFileDirectly: true
                            ])
                        }
                    }
                }
            }
        }

        stage('Deploy to Staging') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.54.0-noble'
                    reuseNode true
                    args '-u root'
                }
            }

            environment {
                CI_ENVIRONMENT_URL = ''
            }

            steps {
                sh '''
                    echo "🚀 Deploying to staging..."
                    apk add --no-cache bash curl jq
                    npm install netlify-cli

                    node_modules/.bin/netlify deploy \
                        --auth=$NETLIFY_AUTH_TOKEN \
                        --site=$NETLIFY_SITE_ID \
                        --dir=build \
                        --json > deploy-output.json

                    echo "✅ Deployed to staging:"
                    cat deploy-output.json | jq .

                    export CI_ENVIRONMENT_URL=$(cat deploy-output.json | jq -r '.deploy_url')
                    echo "🌐 CI_ENVIRONMENT_URL=$CI_ENVIRONMENT_URL"

                    CI_ENVIRONMENT_URL=$CI_ENVIRONMENT_URL npx playwright test --reporter=html
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
                        reportTitles: '',
                        useWrapperFileDirectly: true
                    ])
                }
            }
        }

        stage('Approval') {
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: 'Deploy to production?', ok: 'Yes, deploy!'
                }
            }
        }

        stage('Deploy to Production') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                    args '-u root'
                }
            }
            steps {
                sh '''
                    echo "🚀 Deploying to PRODUCTION..."
                    apk add --no-cache bash curl jq
                    npm install netlify-cli

                    node_modules/.bin/netlify deploy \
                        --auth=$NETLIFY_AUTH_TOKEN \
                        --site=$NETLIFY_SITE_ID \
                        --dir=build \
                        --prod \
                        --json > deploy-output.json

                    echo "✅ Production deployed:"
                    cat deploy-output.json | jq .
                '''
            }
        }

        stage('E2E (Production)') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.54.0-noble'
                    reuseNode true
                    args '-u root'
                }
            }

            environment {
                CI_ENVIRONMENT_URL = 'https://cheerful-souffle-d0935a.netlify.app'
            }

            steps {
                sh '''
                    echo "🔎 Running E2E tests on Production: $CI_ENVIRONMENT_URL"
                    CI_ENVIRONMENT_URL=$CI_ENVIRONMENT_URL npx playwright test --reporter=html
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
                        reportTitles: '',
                        useWrapperFileDirectly: true
                    ])
                }
            }
        }
    }
}
