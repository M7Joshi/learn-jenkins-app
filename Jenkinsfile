pipeline {
    agent any

    stages {
        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                // Cache npm dependencies
                cache(key: 'npm-deps', path: 'node_modules/') {
                    sh '''
                        echo "Building application..."
                        node --version
                        npm --version
                        npm ci
                        npm run build
                        ls -la build/
                    '''
                }
            }
        }

        stage('Test') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                cache(key: 'npm-deps', path: 'node_modules/') {
                    sh '''
                        echo "Running unit tests..."
                        npm test
                    '''
                }
            }
        }

        stage('E2E') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.41.0-focal'
                    reuseNode true
                }
            }
            steps {
                cache(key: 'npm-deps', path: 'node_modules/') {
                    sh '''
                        echo "Starting server for E2E tests..."
                        npm install -g serve
                        node_modules/.bin/serve -s build &
                        echo "Waiting for server to start..."
                        timeout 30 {
                            sh '''
                                while ! curl -s http://localhost:3000 > /dev/null; do
                                    echo "Waiting for server..."
                                    sleep 5
                                done
                            '''
                        }
                        echo "Running E2E tests..."
                        npx playwright test
                    '''
                }
            }
        }
    }

    post {
        always {
            junit '**/jest-results/junit.xml'
            archiveArtifacts artifacts: 'build/**', allowEmptyArchive: true
            cleanWs()
        }
        
        success {
            echo 'Pipeline completed successfully!'
        }
        
        failure {
            echo 'Pipeline failed!'
            error 'Pipeline execution failed'
        }
    }
}
