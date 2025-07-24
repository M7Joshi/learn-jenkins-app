pipeline {
    agent any

    stages {
        
        stage('Checkout') {
            steps {
                checkout scm  // This will automatically checkout the repository configured in the Jenkins job
            }
        }

        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
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
        
        stage('Test') {
            agent {
                docker {
                    image 'node:18-alpine'
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
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    npm install serve
                    node_modules/.bin/serve -s build &
                    sleep 10
                    npx playwright test --reporter=junit --output=jest-results
                    # Verify the content of jest-results directory
                    ls -la jest-results  # Check if the JUnit test report is generated
                '''
            }
        }
    }

    post {
        always {
            junit '**/jest-results/junit.xml'  // Ensure the path to the test results is correct
        }
    }
}
