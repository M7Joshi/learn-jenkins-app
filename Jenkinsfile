pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = 'e3433b10-adb3-41e4-8559-9cc091a7d737'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
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
                                reportName: 'Playwright Local',
                                reportTitles: '',
                                useWrapperFileDirectly: true
                            ])
                        }
                    }
                }
            }
        }

        stage('Deploy Staging') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.54.0-noble'
                    reuseNode true
                    args '-u root'
                }
            }
            environment {
                STAGING_URL = ''
            }
            steps {
                sh '''
                    echo "Installing tools..."
                    apk add --no-cache bash curl jq
                    npm install netlify-cli

                    echo "Deploying to Netlify (staging)..."
                    node_modules/.bin/netlify deploy \
                      --auth=$NETLIFY_AUTH_TOKEN \
                      --site=$NETLIFY_SITE_ID \
                      --dir=build \
                      --json > deploy-output.json

                    echo "✅ Deployment done. JSON written to deploy-output.json."
                    cat deploy-output.json | jq .

                    echo "🔗 Extracting staging URL..."
                    export STAGING_URL=$(cat deploy-output.json | jq -r '.deploy_url')
                    echo "STAGING_URL=$STAGING_URL" > staging_url.env
                '''
                script {
                    def url = sh(script: "cat staging_url.env | grep STAGING_URL | cut -d '=' -f2", returnStdout: true).trim()
                    env.CI_ENVIRONMENT_URL = url
                    echo "CI_ENVIRONMENT_URL set to: ${env.CI_ENVIRONMENT_URL}"
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'deploy-output.json', fingerprint: true
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

        stage('Deploy Prod') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                    args '-u root'
                }
            }
            steps {
                sh '''
                    apk add --no-cache bash curl jq
                    npm install netlify-cli

                    echo "Deploying to Netlify (production)..."
                    node_modules/.bin/netlify deploy \
                      --auth=$NETLIFY_AUTH_TOKEN \
                      --site=$NETLIFY_SITE_ID \
                      --dir=build \
                      --prod \
                      --json > deploy-output.json

                    echo "✅ Production deployed."
                    cat deploy-output.json | jq .
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'deploy-output.json', fingerprint: true
                }
            }
        }

        stage('Prod E2E') {
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
                    echo "Running Playwright against PRODUCTION..."
                    echo "Target URL: $CI_ENVIRONMENT_URL"
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
                        reportTitles: '',
                        useWrapperFileDirectly: true
                    ])
                }
            }
        }
    }
}
