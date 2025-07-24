pipeline {
    agent any

    environment {
        PLAYWRIGHT_DOCKER_IMAGE = 'mcr.microsoft.com/playwright:v1.54.0-noble'
    }

    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        // Temporarily block the build stage by commenting it out
        // stage('Build') {
        //     agent {
        //         docker {
        //             image 'node:18-alpine'
        //         }
        //     }
        //     steps {
        //         sh '''
        //             ls -la
        //             node --version
        //             npm --version
        //             npm ci
        //             npm run build
        //             ls -la
        //         '''
        //     }
        // }

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
                    image "${PLAYWRIGHT_DOCKER_IMAGE}"
                    reuseNode true
                }
            }
            steps {
                sh '''
                    npm install serve
                    node_modules/.bin/serve -s build &
                    sleep 10
                    npx playwright test --reporter=junit --output=jest-results
                    ls -la jest-results
                '''
            }
        }
    }

    post {
        always {
            junit '**/jest-results/junit.xml'
        }
    }
}
