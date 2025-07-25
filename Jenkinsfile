pipeline {
    agent any

    stages {
        /*
        stage('Build') {
            agent {
                docker {
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
        */

        stage('Test') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    test -f build/index.html
                    npm test
                '''
            }
        }

        stage('E2E') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.54.0-noble'  // Fixed Docker image syntax
                    reuseNode true
                }
            }
            steps {
                sh '''
                    npm install --save-dev serve  // Ensure serve is installed locally
                    serve -s build &  // Run serve in the background
                    until curl --output /dev/null --silent --head --fail http://localhost:5000; do
                        echo "Waiting for server to start..."
                        sleep 1
                    done
                    npx playwright test  // Run Playwright tests once the server is ready
                '''
            }
        }
    }

    post {
        always {
            junit '**/jest-results/tunit.xml'  // Ensure this path is correct for your Jest test results
        }
    }
}
