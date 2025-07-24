pipeline {
    agent any

    environment {
        // Ensure Playwright version 1.39.0 is used in the Docker image
        PLAYWRIGHT_DOCKER_IMAGE = 'mcr.microsoft.com/playwright:v1.39.0-noble'
    }

    stages {
        // Checkout SCM
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        // Build the project
        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                }
            }
            steps {
                sh '''
                    # Display file listing for debugging
                    ls -la
                    node --version
                    npm --version

                    # Install dependencies and build the project
                    npm ci
                    npm run build

                    # Display file listing after build for debugging
                    ls -la
                '''
            }
        }

        // Test the project with Jest
        stage('Test') {
            agent {
                docker {
                    image 'node:18-alpine'
                }
            }
            steps {
                sh '''
                    # Check if build/index.html exists
                    test -f build/index.html

                    # Run Jest tests with Jest-JUnit results processor
                    npm test
                '''
            }
        }

        // Run E2E tests with Playwright
        stage('E2E') {
            agent {
                docker {
                    image "${PLAYWRIGHT_DOCKER_IMAGE}" // Use Playwright image version 1.39.0-noble
                    reuseNode true
                }
            }
            steps {
                sh '''
                    # Install serve to serve the build
                    npm install serve

                    # Serve the build on localhost:3000 in background
                    node_modules/.bin/serve -s build &

                    # Wait for the server to start
                    sleep 10

                    # Run Playwright tests and output JUnit results to jest-results
                    npx playwright test --reporter=junit --output=jest-results

                    # Debugging step: List contents of jest-results to confirm junit.xml is created
                    ls -la jest-results
                '''
            }
        }
    }

    post {
        always {
            // Ensure the correct path to the JUnit report
            junit '**/jest-results/junit.xml'  // Point to JUnit report file
        }
    }
}
