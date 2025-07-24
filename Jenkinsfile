pipeline {
    agent any

    environment {
        // Define environment variables
        PLAYWRIGHT_IMAGE = 'mcr.microsoft.com/playwright:v1.39.0-noble'  // Playwright Docker image version
        GIT_REPO_URL = 'https://github.com/M7Joshi/learn-jenkins-app.git' // Git repository URL
        WORKSPACE_DIR = '/var/jenkins_home/workspace/learn-jenkins-app'  // Jenkins workspace directory
        NPM_COMMAND = 'npm ci'  // Command to install npm dependencies
        BUILD_COMMAND = 'npm run build'  // Command to build the React app
        TEST_COMMAND = 'npm test -- --testResultsProcessor="jest-junit"'  // Command to run tests
        E2E_COMMAND = 'npm run e2e'  // Command to run Playwright E2E tests
    }

    stages {
        stage('Checkout SCM') {
            steps {
                script {
                    // Checkout the latest code from the Git repository
                    checkout scm
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    // Build the React app using npm
                    echo 'Building the React application...'
                    sh "${NPM_COMMAND}"
                    sh "${BUILD_COMMAND}"
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    // Run tests using Jest
                    echo 'Running tests...'
                    sh "${TEST_COMMAND}"
                }
            }
        }

        stage('E2E Tests') {
            steps {
                script {
                    // Pull the correct Playwright Docker image using the variable
                    echo 'Pulling Playwright Docker image...'
                    sh "docker pull ${PLAYWRIGHT_IMAGE}"

                    // Run the E2E tests in the Playwright Docker container
                    echo 'Running Playwright E2E tests...'
                    sh """
                        docker run -t -d -u 1000:1000 -w ${WORKSPACE_DIR} \
                        -v ${WORKSPACE_DIR}:${WORKSPACE_DIR}:rw,z \
                        -v ${WORKSPACE_DIR}@tmp:${WORKSPACE_DIR}@tmp:rw,z \
                        ${PLAYWRIGHT_IMAGE} /bin/bash -c "${NPM_COMMAND} && ${E2E_COMMAND}"
                    """
                }
            }
        }

        stage('Post Actions') {
            steps {
                script {
                    // Ensure that test results are recorded correctly
                    echo 'Recording test results...'
                    junit '**/jest-results/**/*.xml'
                }
            }
        }
    }

    post {
        always {
            // Perform cleanup or other necessary actions after the pipeline run
            echo 'Pipeline run completed.'
        }
    }
}
