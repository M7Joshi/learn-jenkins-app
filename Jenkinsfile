pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = 'e3433b10-adb3-41e4-8559-9cc091a7d737'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = "1.0.${BUILD_ID}"
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
                    echo "📦 Building React app..."
                    echo "REACT_APP_VERSION=$REACT_APP_VERSION" > .env
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
                stage('Unit tests') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            echo "🧪 Running unit tests..."
                            npm test -- --ci --reporters=default --reporters=jest-junit
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
                        }
                    }
                    steps {
                        sh '''
                            echo "🧪 Running local E2E tests..."
                            npm install serve
                            node_modules/.bin/serve -s build &
                            sleep 10
                            EXPECTED_APP_VERSION=$REACT_APP_VERSION npx playwright test --reporter=html
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

        stage('Deploy staging') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.54.0-noble'
                    reuseNode true
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

                    export CI_ENVIRONMENT_URL=$(cat deploy-output.json | jq -r '.deploy_url')
                    echo "🌐 Staging URL: $CI_ENVIRONMENT_URL"

                    EXPECTED_APP_VERSION=$REACT_APP_VERSION CI_ENVIRONMENT_URL=$CI_ENVIRONMENT_URL npx playwright test --reporter=html
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

        stage('Deploy prod') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.54.0-noble'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = 'https://cheerful-souffle-d0935a.netlify.app'
            }

            steps {
                sh '''
                    echo "🚀 Deploying to production..."
                    apk add --no-cache bash curl jq
                    npm install netlify-cli

                    node_modules/.bin/netlify deploy \
                        --auth=$NETLIFY_AUTH_TOKEN \
                        --site=$NETLIFY_SITE_ID \
                        --dir=build \
                        --prod

                    echo "🔎 Running E2E on production: $CI_ENVIRONMENT_URL"
                    EXPECTED_APP_VERSION=$REACT_APP_VERSION CI_ENVIRONMENT_URL=$CI_ENVIRONMENT_URL npx playwright test --reporter=html
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
