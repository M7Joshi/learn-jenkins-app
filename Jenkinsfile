pipeline {
    agent any

    stages {
        
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
                    npx playwright test --reporter=junit --output=jest-results
                    # Ensure the Playwright test results are in the correct directory
                    ls -la jest-results  # To verify test results are generated
                '''
            }
        }
    }

    post {
        always {
            junit '**/jest-results/**/*.xml'  // Correct the path to the test results
        }
    }
}
