pipeline {
    agent any

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
                    sh 'npm ci'
                    sh 'npm run build'
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    // Run tests using Jest
                    echo 'Running tests...'
                    sh 'npm test -- --testResultsProcessor="jest-junit"'
                }
            }
        }

        stage('E2E Tests') {
            steps {
                script {
                    // Run the E2E tests in the Playwright Docker container
                    echo 'Running Playwright E2E tests...'
                    sh """
                        docker run -t -d -u 1000:1000 -w /var/jenkins_home/workspace/learn-jenkins-app \
                        -v /var/jenkins_home/workspace/learn-jenkins-app:/var/jenkins_home/workspace/learn-jenkins-app:rw,z \
                        -v /var/jenkins_home/workspace/learn-jenkins-app@tmp:/var/jenkins_home/workspace/learn-jenkins-app@tmp:rw,z \
                        mcr.microsoft.com/playwright:v1.39.0-noble /bin/bash -c "npm ci && npm run e2e"
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
