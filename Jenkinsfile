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
                    args '-u root'  // Run container as root
                }
            }
            steps {
                sh '''
                    apk add --no-cache bash
                    ls -la
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
                                reportName: 'Playwright HTML Report',
                                reportTitles: '',
                                useWrapperFileDirectly: true
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
                    args '-u root'
                }
            }
            steps {
                sh '''
                    echo "Installing Netlify CLI..."
                    npm install netlify-cli

                    echo "Netlify CLI version:"
                    node_modules/.bin/netlify --version || echo "❌ Netlify CLI not found"

                    echo "Checking build directory contents:"
                    ls -la build || { echo "❌ Build directory is missing"; exit 1; }

                    echo "Checking Netlify authentication status:"
                    node_modules/.bin/netlify status || echo "⚠️ Could not verify login (okay if using token)"

                    echo "Deploying to Netlify with site ID ${NETLIFY_SITE_ID}..."
                    node_modules/.bin/netlify deploy --auth=${NETLIFY_AUTH_TOKEN} --site=${NETLIFY_SITE_ID} --dir=build --prod --debug
                '''
            }
        }
    }
}
