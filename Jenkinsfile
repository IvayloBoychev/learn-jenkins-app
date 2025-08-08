pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '31979881-0b34-4a3e-ac11-1e4a489e11d3'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = "1.0.$BUILD_ID"
    }

    stages {

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

        stage('Tests') {
            parallel {
                stage('Unit tests') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
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

                stage('Local E2E') {
                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            npm install serve
                            node_modules/.bin/serve -s build &> /dev/null &
                            sleep 10
                            npx playwright test --reporter=html
                        '''
                    }
                    post {
                        always {
                            publishHTML([reportDir: 'playwright-report', reportFiles: 'index.html',
                                reportName: 'Local E2E', allowMissing: false, keepAll: false, useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }

        stage('Deploy to Staging') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    npm install netlify-cli@20.1.1
                    node_modules/.bin/netlify deploy --dir=build --site=$NETLIFY_SITE_ID --json > deploy-output.json
                '''
                script {
                    def json = readJSON file: 'deploy-output.json'
                    env.CI_ENVIRONMENT_URL = json.deploy_url
                }
            }
        }

        stage('Staging E2E') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    echo "Testing staging URL: $CI_ENVIRONMENT_URL"
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    publishHTML([reportDir: 'playwright-report', reportFiles: 'index.html',
                        reportName: 'Staging E2E', allowMissing: false, keepAll: false, useWrapperFileDirectly: true])
                }
            }
        }

        stage('Approval') {
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: 'Do you wish to deploy to production?', ok: 'Yes, deploy to prod'
                }
            }
        }

        stage('Deploy to Prod') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    npm install netlify-cli@20.1.1
                    node_modules/.bin/netlify deploy --dir=build --site=$NETLIFY_SITE_ID --prod
                '''
            }
        }

        stage('Prod E2E') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }
            environment {
                CI_ENVIRONMENT_URL = 'https://peaceful-daffodil-303af5.netlify.app'
            }
            steps {
                sh '''
                    echo "Testing prod URL: $CI_ENVIRONMENT_URL"
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    publishHTML([reportDir: 'playwright-report', reportFiles: 'index.html',
                        reportName: 'Prod E2E', allowMissing: false, keepAll: false, useWrapperFileDirectly: true])
                }
            }
        }
    }
}
