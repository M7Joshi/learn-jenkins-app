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
                    args '-u root'  // Run as root for apk
                }
            }
            steps {
                sh '''
                    apk add --no-cache bash
                    node --version
                    npm --version
                    npm ci || { echo "❌ npm ci failed"; exit 1; }
                    npm run build || { echo "❌ Build failed"; exit 1; }

                    echo "✅ Verifying build output..."
                    ls -la
                    ls -la build || { echo "❌ Build folder not found!"; exit 1; }
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
                                reportName: 'Playwright HTML Report'
                            ])
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                    args '-u root'  // npm install + access perms
                }
            }
            steps {
                sh '''
                    npm install netlify-cli
                    node_modules/.bin/netlify --version

                    echo "📦 Checking if build folder exists before deploy..."
                    ls -la build || { echo "❌ Build directory not found! Cannot deploy."; exit 1; }

                    echo "🚀 Deploying to Netlify..."
                    node_modules/.bin/netlify deploy --auth=${NETLIFY_AUTH_TOKEN} --site=${NETLIFY_SITE_ID} --dir=build --prod
                '''
            }
        }
    }
}
